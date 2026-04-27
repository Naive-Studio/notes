# UE5.7 Material Compile Pipeline

## Goal

这份文档专门解释一个问题：

`材质图是怎么变成 shader 的？`

它不是材质节点手册，也不是材质系统总览，而是把 UE5.7 的材质编译链拆开来看。

阅读目标：

1. 读懂 `UMaterial`、`UMaterialInstance`、`FMaterial`、`FMaterialShaderMap` 各自负责什么。
2. 理解 Main Material Node 和 `CompilePropertyEx()` 的关系。
3. 看清旧 translator、新 IR/HLSL generator、DDC、ShaderMap 是怎么串起来的。
4. 知道什么改动会触发重编译，什么改动不会。

本文基于：

- 本地引擎源码 `UE 5.7.4`
- Epic 官方材质文档

---

## 1. The Short Version

把整个流程压缩成一句话：

材质资产先把图表达式按材质属性编译成中间结果，再由 translator 生成 HLSL 片段，注入到引擎 shader 模板里，最后按平台和静态参数编译成 `FMaterialShaderMap` 并缓存。

如果再展开一点，就是：

```mermaid
flowchart LR
    A["UMaterial / UMaterialInstance"] --> B["Main Material Node + Expressions"]
    B --> C["CompilePropertyEx(AttributeID)"]
    C --> D["UMaterialExpression::Compile(FMaterialCompiler)"]
    D --> E["Material Translator"]
    E --> F["/Engine/Generated/Material.ush"]
    F --> G["FMaterialShaderMap"]
    G --> H["FMaterialResource / FMaterialRenderProxy"]
    H --> I["Renderer"]
```

---

## 2. Core Runtime And Asset Types

### 2.1 UMaterial

`UMaterial` 是材质的源定义。

它持有：

- Material Domain
- Blend Mode
- Shading Model
- 表达式图
- 各个材质属性输入
- Substrate Front Material
- 若干 editor-only 结构和编译辅助数据

材质图本身的“真相”主要存在这里。

### 2.2 UMaterialInstance

`UMaterialInstance` 不重新定义整张图，它主要做：

- 继承父材质
- 覆盖参数
- 提供静态参数集
- 派生编译 key

本地源码里，`UMaterialInstance::CompilePropertyEx()` 直接委托给父材质：

```cpp
return Parent ? Parent->CompilePropertyEx(Compiler, AttributeID) : INDEX_NONE;
```

所以实例的价值不是“重新画一张图”，而是“在父图框架上派生变体”。

### 2.3 UMaterialInstanceDynamic

`UMaterialInstanceDynamic` 是运行时改参入口。

它暴露了大量：

- `SetScalarParameterValue`
- `SetVectorParameterValue`
- `SetTextureParameterValue`
- `ByInfo`
- `ByIndex`

这些接口说明 MID 的设计目标是：

- 高频、低延迟、运行时参数覆盖
- 不改 shader 结构

### 2.4 FMaterial / FMaterialResource / FMaterialRenderProxy

这是资产层和渲染层之间的桥。

你可以粗略理解成：

- `UMaterial*` 是 UObject 资产
- `FMaterial` 是编译与 shader 语义中心
- `FMaterialResource` 是某平台 / 某特性级别 / 某质量级别下的具体资源版本
- `FMaterialRenderProxy` 是渲染线程消费的代理

---

## 3. The Real Compile Entry: CompilePropertyEx

很多人以为“材质编译”是整张图一次性直出。
实际上，UE 的入口更像“按材质属性逐项编译”。

在本地 `UMaterial::CompilePropertyEx()` 中，可以看到它首先把 `AttributeID` 映射成 `EMaterialProperty`，然后根据属性选择对应输入。

例如：

- `MP_BaseColor`
- `MP_Roughness`
- `MP_Metallic`
- `MP_Normal`
- `MP_EmissiveColor`
- `MP_WorldPositionOffset`
- `MP_PixelDepthOffset`
- `MP_ShadingModel`
- `MP_FrontMaterial`

如果启用了 `bUseMaterialAttributes`，大多数属性会改走：

- `EditorOnly->MaterialAttributes.CompileWithDefault(...)`

这说明：

1. Main Material Node 不是单一输出，而是一组属性槽的总线。
2. `Use Material Attributes` 会改变编译入口组织方式。
3. 图结构最终仍要归结为具体材质属性的编译请求。

---

## 4. Expression Graph Compilation

### 4.1 UMaterialExpression::Compile

每个材质节点都是一个 `UMaterialExpression` 子类。

这些节点最关键的方法是：

- `Compile(FMaterialCompiler* Compiler, int32 OutputIndex)`

也就是说：

- 节点自己知道如何把自己的语义提交给编译器
- 材质编译更像对象图递归求值，而不是先序列化成文本再统一解释

### 4.2 FMaterialCompiler 是抽象代码生成接口

`FMaterialCompiler` 是这个系统的核心抽象之一。

从头文件能看到，它提供了大量虚接口，比如：

- 常量和参数：`Constant`、`NumericParameter`
- 数学：`Add`、`Mul`、`Dot`、`Normalize`
- 坐标：`WorldPosition`、`LocalPosition`、`TextureCoordinate`
- 采样：`TextureSample`、`SceneTextureLookup`
- 控制流：`If`、`Switch`
- 特殊路径：`GIReplace`、`NaniteReplace`、`LightmassReplace`
- 导数：`DDX`、`DDY`
- 材质缓存、VT、Sparse Volume Texture 等高级功能

所以可以把 `FMaterialCompiler` 理解成：

- 材质图 DSL 的后端接口

节点不直接写 HLSL 字符串，而是调用这个抽象接口生成更高层的编译结果。

### 4.3 为什么这样设计

这样做有几个重要好处：

1. 节点与具体后端解耦
2. 同一份图可以走不同 translator
3. 编译器可以顺带收集：
   - texture usage
   - scene texture usage
   - uniform expression
   - errors
   - compile insights

---

## 5. Legacy Translator vs New IR/HLSL Generator

这是 UE5.7 材质编译链里最值得注意的一部分。

### 5.1 Legacy Path

传统路径核心是：

- `FHLSLMaterialTranslator`

在本地 `FMaterial::Translate_Legacy()` 里，它会：

1. 构造 `FHLSLMaterialTranslator`
2. 调用 `Translate(false)`
3. 必要时因 DDC 失效重试 `Translate(true)`
4. 生成材质 HLSL 代码
5. 写入虚拟 include：
   - `/Engine/Generated/Material.ush`

也就是说 legacy 路径最终输出的是：

- 一段材质专属 HLSL 片段

### 5.2 New Path

UE5.7.4 本地源码里已经能看到完整的 Material IR 相关模块：

- `MaterialIRModule`
- `MaterialIRModuleBuilder`
- `MaterialIRToHLSLTranslator`
- `MaterialIRValueAnalyzer`

在 `FMaterial::Translate_New()` 中，流程变成：

1. 构建 `FMaterialIRModule`
2. 用 `FMaterialIRModuleBuilder` 把材质转成 IR
3. 运行 `FMaterialIRToHLSLTranslation`
4. 用模板解析器插值成最终材质 shader 代码
5. 同样塞进 `/Engine/Generated/Material.ush`

这说明新的系统不是简单“换个函数名”，而是：

- 在图表达式和最终 HLSL 之间引入了显式 IR 层

### 5.3 UE5.7.4 的现实状态

本地 `FMaterial::Translate()` 里有一个关键判断：

- 若 `bUsingNewHLSLGenerator` 为真
- 且 `Substrate` 未启用
- 才走 `Translate_New()`

源码注释还明确写着：

- `the new translator does not support Substrate yet`

所以在 `UE 5.7.4` 中：

- 新 IR/HLSL generator 已经存在
- 但 Substrate 仍依赖 legacy translator

这个结论非常重要，因为它直接说明：

- UE5.7 的材质编译系统处在迁移阶段
- 不是所有材质路径都已经统一到新编译后端

---

## 6. Why /Engine/Generated/Material.ush Matters

无论 legacy 还是 new path，最终都会把材质生成代码塞进：

- `/Engine/Generated/Material.ush`

这说明 UE 的材质并不是独立 shader 文件，而是：

- 被注入到引擎 shader 模板体系中的一段材质实现

这背后的好处是：

1. 引擎可以复用统一的 pass shader 框架
2. 材质代码只关注“材质逻辑”
3. 平台、pass、渲染路径的差异由更外层 shader 模板承担

所以材质编译更准确的说法是：

- 生成可嵌入 renderer 模板的材质代码

---

## 7. ShaderMap And Compilation Environment

### 7.1 FMaterialShaderMap 是什么

`FMaterialShaderMap` 是某份材质在某种编译上下文下的 shader 集合。

它受很多因素影响：

- 平台
- Feature Level
- Quality Level
- 静态参数
- 使用的渲染特性
- Substrate 配置
- Material Layers / 函数依赖状态

### 7.2 BeginCompileShaderMap 做了什么

本地 `FMaterial::BeginCompileShaderMap()` 里可以看到核心流程：

1. 新建 `FMaterialShaderMap`
2. 调用 `Translate(...)`
3. 生成 `FMaterialCompilationOutput`
4. 准备 `FSharedShaderCompilerEnvironment`
5. 应用 BRDF headers 和 derived defines
6. 把 uniform expression 信息塞进编译环境
7. 触发 shader job 编译与缓存

### 7.3 为什么材质改一个静态开关就会重新编译

因为静态开关不只是参数，它会影响：

- shader map key
- material environment
- 代码裁剪后的最终 HLSL

所以静态参数的本质是：

- 改编译结果，不只是改运行时值

---

## 8. DDC And Translation Caching

`FHLSLMaterialTranslator::Translate()` 里可以看到显式的 DDC 查询逻辑。

流程大致是：

1. 计算 translation DDC key
2. 异步查询 DDC
3. 同步开始材质翻译
4. 如果命中缓存，则反序列化翻译结果
5. 如果缓存无效，走完整翻译

这说明材质编译缓存不只是最终 shader cache，还有：

- 材质翻译阶段本身的缓存

所以当你觉得材质“为什么还是慢”，问题可能出在：

- translation DDC 没命中
- shader map key 频繁变化
- 静态参数太多
- 平台与质量级别覆盖太多

---

## 9. Static Parameters, Instances, And Permutations

### 9.1 为什么实例通常不重编译

普通材质实例改的是：

- scalar
- vector
- texture

这些通常不改变 shader 结构，只改变 uniform / resource binding。

所以：

- 普通实例参数改动大多不触发重新编译

### 9.2 为什么静态参数危险

静态参数会改变：

- 分支是否被裁掉
- 某些路径是否启用
- shader map key

所以它的代价常常不是运行时，而是：

- 编译量
- permutation 数量
- DDC 占用
- PSO / shader 管理复杂度

### 9.3 实例继承链的真正价值

实例系统最大的价值不是“更快执行”。
更准确地说，它提供的是：

- 结构复用
- 编译结果复用
- 更低的维护成本

---

## 10. Material Functions And Material Layers In The Compile Path

### 10.1 Material Functions

Material Function 本质上是：

- 可被调用的图子程序

它在编译时会被联入调用者图中，而不是像普通程序函数那样在运行时调用。

所以它带来的收益主要是：

- 图组织复用
- 维护性提升

而不是运行时“调用开销更低”。

### 10.2 Material Layers

Material Layers 在编译链上比普通函数更进一步，因为它们会进入：

- Layer IDs
- Blend IDs
- Layer states
- DDC / shader key 计算

本地 `MaterialLayersFunctions.h` 里的 `FMaterialLayersFunctionsID` 就是直接证据。

这意味着：

- Material Layers 不只是编辑器 UI 资产系统
- 它们明确参与编译 identity

---

## 11. Substrate In The Compile Pipeline

Substrate 对编译链的影响远不止“多了一些节点”。

### 11.1 它改变了输出语义

在 `UMaterialExpression` 基类中，Substrate 扩展了这些能力：

- `IsResultSubstrateMaterial`
- `GatherSubstrateMaterialInfo`
- `SubstrateGenerateMaterialTopologyTree`

这说明图已经不只是“普通属性表达式图”，而可以表达：

- Substrate 材质树

### 11.2 它让图结构变成 BSDF 树

官方 API 和本地源码都说明：

- `FrontMaterial` 是根
- BSDF 节点是叶子
- mixing / layering / add / weight / select 是中间 operator

所以 Substrate 编译不是单纯给几个 pin 赋值，而是：

- 先理解一棵材质拓扑树
- 再据此生成最终材质表示

### 11.3 5.7.4 下的工程结论

Substrate 已经进入正式主线，但本地 5.7.4 里它仍依赖 legacy translator。

所以如果你调试 Substrate 编译问题，第一反应应该是：

- 去看 legacy material translator 和 Substrate 表达式实现

而不是默认它已经走新 IR path。

---

## 12. What Triggers Recompile

下面这张表适合日常判断。

| 改动 | 一般是否重编译 | 说明 |
|---|---|---|
| 改母材质图结构 | 会 | 直接改编译输出 |
| 改母材质 Domain / Blend / Shading Model | 会 | 改输入语义和 shader path |
| 改 Static Switch / Static Bool / Static Component Mask | 通常会 | 改 permutation |
| 改 Material Layer 结构或状态 | 会 | 参与编译 ID |
| 改普通实例 scalar/vector/texture 参数 | 一般不会 | 只改运行时覆盖 |
| 改 MID 参数 | 不会 | 运行时更新 |
| 改 MPC 值 | 不会 | 统一更新引用材质 |
| 改函数体 | 会影响所有引用材质 | 因为调用图被重建 |

---

## 13. Debugging Checklist

当你碰到“材质编译慢”或“改了参数为什么重新编译”的问题时，我建议按这个顺序排查：

1. 看是不是改了静态参数，而不是普通参数。
2. 看是不是改了 Domain / Blend / Shading Model。
3. 看是不是 Material Function 或 Material Layer 资产发生了变化。
4. 看 shader map key 是否因为平台或质量级别变化而重新派生。
5. 看是否命中 DDC。
6. 看是不是 Substrate 路径，把问题带到了 legacy translator。

---

## 14. Mental Model For Engineers

如果你需要一个程序员视角的最终心智模型，我建议记成下面这样：

### Layer 1: Authoring

- `UMaterial`
- `UMaterialExpression`
- Material graph

### Layer 2: Semantic Routing

- Domain
- Blend Mode
- Shading Model
- Material Attributes / Front Material

### Layer 3: Graph Compilation

- `CompilePropertyEx`
- `UMaterialExpression::Compile`
- `FMaterialCompiler`

### Layer 4: Translation

- legacy `FHLSLMaterialTranslator`
- new Material IR + HLSL generator

### Layer 5: Shader Resource Build

- material environment
- shader map id
- DDC
- `FMaterialShaderMap`

### Layer 6: Runtime Consumption

- `FMaterialResource`
- `FMaterialRenderProxy`
- renderer pass templates

只要这 6 层关系清楚，UE 材质编译链就不会再显得神秘。

---

## 15. Code Map

最值得读的源码位置：

- `E:/Slash/Engine/Source/Runtime/Engine/Public/Materials/Material.h`
- `E:/Slash/Engine/Source/Runtime/Engine/Private/Materials/Material.cpp`
- `E:/Slash/Engine/Source/Runtime/Engine/Public/Materials/MaterialInstance.h`
- `E:/Slash/Engine/Source/Runtime/Engine/Private/Materials/MaterialInstance.cpp`
- `E:/Slash/Engine/Source/Runtime/Engine/Public/MaterialCompiler.h`
- `E:/Slash/Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.h`
- `E:/Slash/Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp`
- `E:/Slash/Engine/Source/Runtime/Engine/Public/Materials/MaterialIRModule.h`
- `E:/Slash/Engine/Source/Runtime/Engine/Public/Materials/MaterialIRModuleBuilder.h`
- `E:/Slash/Engine/Source/Runtime/Engine/Public/Materials/MaterialIRToHLSLTranslator.h`
- `E:/Slash/Engine/Source/Runtime/Engine/Public/MaterialShared.h`
- `E:/Slash/Engine/Source/Runtime/Engine/Private/Materials/MaterialShared.cpp`

---

## 16. References

- UE5.7 Release Notes  
  https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-7-release-notes?application_version=5.7

- Material Inputs  
  https://dev.epicgames.com/documentation/it-it/unreal-engine/material-inputs-in-unreal-engine

- Material Properties  
  https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-material-properties?application_version=5.6

- Create Materials and Material Instances  
  https://dev.epicgames.com/documentation/en-us/unreal-engine/artist-03-create-materials-and-material-instances
