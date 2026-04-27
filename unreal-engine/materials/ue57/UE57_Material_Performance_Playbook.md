# UE5.7 Material Performance Playbook

## Goal

这份文档不是“材质性能概念介绍”，而是一份实战排查手册。

目标：

1. 帮你建立 UE5.7 材质性能的正确成本模型。
2. 帮你知道先看什么，再看什么，不要陷入无效微优化。
3. 给出常见症状 -> 原因 -> 优化动作的对照表。
4. 把传统材质、Material Layers、Substrate、WPO、透明、Scene Texture、Distance Field 的性能问题放到同一张图里理解。

---

## 1. The Only Cost Model That Really Matters

大多数项目里，材质性能问题主要不是被 `Add`、`Multiply`、`Clamp` 这些基础运算拖垮的。

真正常见的成本来源是：

1. 纹理采样数和带宽
2. 透明 overdraw
3. SceneTexture / SceneDepth / DBuffer 读取
4. Distance Field 查询
5. Noise / VectorNoise / 程序化复杂图
6. World Position Offset
7. Pixel Depth Offset
8. Layered Materials / Material Layers / Substrate 的层级增长
9. Static Switch 导致的 permutation 爆炸

如果这 9 件事没抓住，几乎所有“材质优化”都会变成错方向。

---

## 2. A Cost Ladder

### 2.1 Usually Cheap

- 常量
- 标量和向量参数
- `Add / Subtract / Multiply / Divide`
- `Min / Max / Clamp / Saturate`
- `DotProduct`
- `Lerp`
- `ComponentMask / AppendVector`
- 普通 UV 运算

### 2.2 Usually Medium

- 少量纹理采样
- World / Local / Tangent 空间变换
- Fresnel
- BumpOffset
- 少量 SceneDepth
- 普通法线重建
- 小规模 Material Attributes 混合

### 2.3 Usually Expensive

- 大量纹理采样
- Translucent 材质
- SceneTexture 重度依赖
- Noise / VectorNoise
- DistanceToNearestSurface / DistanceFieldGradient
- WPO on dense mesh
- PixelDepthOffset
- 深层分层材质
- 复杂 Substrate 材质树

---

## 3. The Workflow Order

优化顺序建议固定成下面这样：

1. 先判断是不是透明问题
2. 再看纹理采样数
3. 再看 SceneTexture / Distance Field
4. 再看 WPO / PDO
5. 再看 Layers / Substrate
6. 最后才看节点级算术微优化

这样做的原因很简单：

- 前面几项的收益最大
- 后面的微优化通常是最后 10% 的事情

---

## 4. The First-Line Tools

### 4.1 Material Editor Stats

永远先看：

- instruction count
- texture sample count
- different platform preview stats

这是最快的第一层筛查。

### 4.2 Shader Complexity View Mode

本地源码里，编辑器 viewport 命令明确包含：

- `Shader Complexity View Mode`
- `Shader Complexity & Quads visualization`

这两个模式适合看：

- 纯 shader 成本热点
- 透明 / overdraw 热点

### 4.3 Material Analyzer

官方提供 `Material Analyzer`，适合看：

- 母材质和实例链
- Static Switch / Static Component Mask 使用情况
- 参数覆盖范围
- 哪些实例链已经变复杂到难以维护

### 4.4 ProfileGPU

本地 `BaseInput.ini` 里明确注册了：

- `ProfileGPU`
- `ProfileGPUHitches`

这意味着你可以从材质编辑器统计再往下一层走：

- 到 GPU pass 级别看真正花在哪个 pass

### 4.5 Scalability / MaterialQualityLevel

本地 `BaseScalability.ini` 可以看到：

- `r.MaterialQualityLevel`
- `r.TranslucencyLightingVolume.*`

所以项目里做材质高低配时，不能只靠“感觉降复杂度”，而应该显式设计：

- quality switch
- feature level switch
- scalability mapping

---

## 5. Symptom -> Cause -> Action

### 5.1 症状：普通不透明材质也很重

常见原因：

- 采样数太多
- 不必要的多层混合
- 重复的世界空间计算
- 法线 / 粗糙度 / mask 没有打包

优先动作：

1. 合并纹理采样
2. 把 ORM 或 masks 打包
3. 复用 UV
4. 减少重复 normal blending
5. 把对象局部效果改到 local space

### 5.2 症状：一开透明性能就掉很多

常见原因：

- 透明 overdraw
- 半透明排序路径
- SceneColor / refraction
- 多层粒子叠加
- 透明材质还带复杂噪声和深度逻辑

优先动作：

1. 能改 `Masked` 就别用 `Translucent`
2. 压缩屏幕覆盖面积
3. 减少粒子堆叠层数
4. 简化 refraction 和 scene reads
5. 对远距离版本做材质简化

### 5.3 症状：树叶 / 草地很贵

常见原因：

- Masked 材质 overdraw
- 双面 + 法线 + 风摆动
- 贴图采样过多
- 远距离还保留完整近景逻辑

优先动作：

1. 优化 alpha coverage
2. 合并贴图采样
3. WPO 分近中远级别
4. 为 foliage 设计专用低成本母材质
5. 检查两面材质是否真的必要

### 5.4 症状：角色材质很重

常见原因：

- 多层肤质 / 布料 / clear coat
- 大量 mask 和 detail normal
- Substrate 层数过深
- 每个部件都挂不同复杂材质

优先动作：

1. 拆解哪些层真正可见
2. 合并角色同类材质族
3. 降低近似重复的 detail layer
4. 将高价值部件与普通部件分层预算

### 5.5 症状：场景特效材质拖慢帧率

常见原因：

- Translucent + noise + scene depth + panner 同时存在
- 粒子数量大
- 屏幕占比高
- 还开启局部阴影或复杂光照特性

优先动作：

1. 先压屏幕覆盖面积
2. 再压粒子数量
3. 再简化材质图
4. 必要时改成 flipbook / baked flow texture

---

## 6. Texture Sample Playbook

### 6.1 第一原则

先减少采样，再想其他。

因为大量材质的瓶颈，本质上就是：

- 采样次数
- 带宽
- sampler 压力

### 6.2 最有效动作

- 把 AO / Roughness / Metallic / masks 打包
- 多个 layer 共享同一组 UV
- 避免同一纹理在同一材质里重复采样
- 不稳定程序图案尽量 bake
- detail normal 和 detail mask 要有明确收益再上

### 6.3 常见反模式

- “为了图整洁”把同一纹理 sample 多次
- 每个 layer 都有自己独立 UV 动画
- 同时保留 macro、micro、detail、overlay 四层采样却没有明确收益

---

## 7. Transparency Playbook

### 7.1 先记一句话

透明材质的问题很多时候不是 shader 指令，而是：

- 画了太多次

### 7.2 优化优先级

1. 屏幕覆盖面积
2. 粒子叠层数
3. 材质采样和逻辑复杂度
4. 光照与阴影特性

### 7.3 典型动作

- 远距离切换到更简单材质
- 把边缘羽化改成 masked 或 dithered 方案
- 减少 scene color / scene depth 使用
- 让透明粒子素材本身更聚焦，少大片低 alpha

### 7.4 什么时候一定要警惕

当一个材质同时具备下面几项时，几乎肯定要重点审：

- `Translucent`
- `Noise`
- `SceneTexture`
- `DepthFade`
- `Refraction`
- 大面积屏幕覆盖

---

## 8. Scene Texture And Depth Playbook

### 8.1 为什么这类节点敏感

从本地源码的 `HLSLMaterialTranslator` 可以看到：

- `SceneTextureLookup`
- `SceneDepth`
- `SceneColor`

这些都不是普通贴图采样，它们是对场景缓冲的读取。

### 8.2 什么时候值得用

- 水边交界
- 软粒子
- 后处理逻辑
- 特定折射 / 扭曲
- 需要屏幕空间关系判断的少量高价值效果

### 8.3 什么时候不值得用

- 只是为了做一个“差不多像”的边缘效果
- 可以用 object/local/world space 直接解决的问题
- 可以预烘焙成 mask 的问题

### 8.4 排查动作

- 看 scene texture 读了几次
- 看是否同时用了 filtered / clamped 版本
- 看是否在 translucent 路径里反复使用

---

## 9. Distance Field Playbook

### 9.1 为什么它很强

`DistanceToNearestSurface`、`DistanceFieldGradient`、`DistanceFieldApproxAO` 很强，因为它们能让材质感知：

- 周边几何距离
- 表面接近关系
- 近似方向

### 9.2 为什么它也贵

因为它不是普通局部图逻辑，而是在读全局距离场相关数据。

### 9.3 适合场景

- 接触边缘高亮
- 交界溶解
- 地面接触泡沫
- 某些世界交互感知效果

### 9.4 优化原则

- 只给关键材质用
- 有替代方案时优先用局部 mask / baked texture
- 不要给大规模普通资产批量挂

---

## 10. WPO And Pixel Depth Offset Playbook

### 10.1 WPO 的关键点

WPO 成本不只看节点复杂度，更看：

- 顶点数量

一个轻微的 WPO 放在高密 Nanite 或高密 skinned mesh 上，可能比想象中贵得多。

### 10.2 本地源码里的直接信号

`HLSLMaterialTranslator` 会显式跟踪：

- `bUsesWorldPositionOffset`
- `bUsesPixelDepthOffset`

并把它们写入 compilation output 和 shader defines。

这说明：

- WPO / PDO 不是普通表达式细节
- 它们会影响整份材质编译语义

### 10.3 WPO 优化动作

- 给 WPO 设置明确的 `MaxWorldPositionOffsetDisplacement`
- 近中远景采用不同变形强度
- 只对真正需要运动的部分启用
- 把大范围程序噪声位移改为烘焙动画或骨骼驱动

### 10.4 PDO 使用建议

Pixel Depth Offset 的收益常见于：

- 软化交界
- 某些表面融接

但它很容易引入：

- 额外复杂度
- 视觉调试困难

所以建议：

- 只在确有视觉收益的局部材质使用

---

## 11. Noise And Procedural Material Playbook

### 11.1 为什么它容易超预算

`Noise` 和 `VectorNoise` 常见问题不是“不能用”，而是：

- 很容易叠多层
- 很容易和世界空间逻辑绑定
- 很容易继续带动更多 math

### 11.2 最佳策略

- 用它们做 look-dev 和原型验证
- 一旦图样稳定，尽量 bake

### 11.3 可以 bake 的东西

- 溶解 mask
- 云纹
- 流动噪声
- 磨损 pattern
- 局部斑驳分布

### 11.4 不该 bake 的东西

- 真正需要对场景实时响应的世界逻辑
- 与 gameplay 实时驱动强相关的程序结果

---

## 12. Material Layers And Layered Materials Playbook

### 12.1 它们常见的性能陷阱

问题不在“用了 Layer 就一定贵”，而在：

- 每层都完整执行一遍复杂材质逻辑
- 最后才做混合

### 12.2 正确思路

- 先裁掉不必要的层
- 让 layer 本身尽量轻
- 把昂贵逻辑放在真正需要的层里

### 12.3 典型优化动作

- 对共享采样做上提
- 对 layer 专属逻辑做下沉
- 用更轻的材质属性组合代替整套属性混合

---

## 13. Substrate Performance Playbook

### 13.1 Substrate 的成本不是“节点多”

更准确地说，它的成本增长主要来自：

- 材质树深度
- BSDF 层数
- 混合关系
- 透射 / 透明路径
- 每层内部仍然可能有大量采样

### 13.2 最常见错误

- 看见 Substrate 能表达很多，就把所有现象全堆进去

### 13.3 推荐规则

- 从 1 层开始
- 有明确收益再加第 2 层
- 超过 2 到 3 个有实义的层时必须做预算审查
- 复杂效果尽量留给 hero asset

### 13.4 实际判断标准

如果一个 Substrate 材质回答不了下面两个问题，它大概率已经太复杂：

1. 每一层的视觉职责是什么
2. 去掉这一层后，玩家能否明显看出来

---

## 14. Domain-Specific Rules

### 14.1 Surface

主战场，优先优化：

- 采样数
- 分层
- WPO

### 14.2 Deferred Decal

重点关注：

- 覆盖面积
- DBuffer 成本
- 不必要的复杂 shading

### 14.3 Post Process

重点关注：

- SceneTexture 读取次数
- 分辨率敏感逻辑
- 全屏成本

### 14.4 UI

重点关注：

- 简洁稳定
- 少采样
- 不把表面材质逻辑直接照搬进 UI

### 14.5 Volume

重点关注：

- 屏幕覆盖
- 采样
- 体积路径的单独预算

---

## 15. Material Quality And Feature Switch Strategy

### 15.1 为什么要做分档

项目里真正成熟的材质体系，应该天然支持：

- Low
- Medium
- High / Epic

而不是只有一个“豪华默认版本”。

### 15.2 用什么做分档

- `QualitySwitch`
- `FeatureLevelSwitch`
- 平台专用母材质
- 远近景材质版本

### 15.3 用什么不要做分档

不要用一堆零碎 `StaticSwitchParameter` 拼出“所有情况都想覆盖”的万能母材质。

那样会带来：

- permutation 爆炸
- 编译时间上升
- 资产维护灾难

---

## 16. Quick Triage Checklist

如果你要现场排一个材质，我建议按下面这 10 条顺序走。

1. 这是 `Opaque / Masked / Translucent` 哪一类？
2. 屏幕覆盖面积大吗？
3. texture sample 数量是多少？
4. 是否使用 `SceneTexture / SceneDepth`？
5. 是否使用 `Distance Field`？
6. 是否用了 `Noise / VectorNoise`？
7. 是否使用 `WPO / PDO`？
8. 是否有多层 Material Attributes / Layers / Substrate？
9. 是否暴露了过多 static switches？
10. 这个效果有没有更便宜的近似方案？

---

## 17. Anti-Patterns

这些做法最容易把项目拖慢。

- 一个万能母材质覆盖所有资产类型
- 为了“图整洁”重复 sample 同一纹理
- 透明材质承担太多本该放在贴图里的工作
- WPO 没有顶点密度预算
- Noise 原型效果直接原样上生产
- Layered 材质每层都完整跑一遍复杂逻辑
- Substrate 不设层数和资产范围边界
- 把 static switch 当运行时开关用

---

## 18. Landscape, Water, And Foliage Special Topics

These three categories deserve their own section because they are the most common material-heavy content types in real projects.

They also fail in very different ways:

- Landscape usually fails by layer accumulation
- Water usually fails by translucency and scene reads
- Foliage usually fails by masked overdraw plus wind and shadow cost

### 18.1 Landscape

Landscape material cost usually grows from:

- too many painted layers
- every layer carrying a full expensive graph
- repeated macro/micro sampling per layer
- world-aligned or triplanar logic applied too broadly
- slope, height, wetness, RVT, detail and biome logic all stacked together

#### What to optimize first

1. Count how many layers can actually affect a pixel.
2. Move expensive logic behind meaningful layer gating.
3. Share UV and shared sampling work where possible.
4. Decide which detail belongs in RVT or baked masks instead of per-layer live evaluation.

#### Good habits

- Keep a cheap baseline for terrain.
- Use `LandscapeLayerSwitch` or equivalent gating aggressively where it makes sense.
- Separate macro color logic from micro detail logic.
- Treat triplanar and world-aligned sampling as exceptions, not defaults.

#### Bad habits

- Every layer has its own full normal, roughness, height, noise, macro and wetness path.
- One terrain master tries to cover every biome with static switches.
- High-value terrain close-ups and ordinary background ground use the same complexity budget.

### 18.2 Water

Water materials fail for a different reason:

- they often combine translucency, refraction, scene color, scene depth, shoreline masks and animated normals in one place

#### The first decision that matters

Ask what kind of water you actually need:

| Water class | Typical recommendation |
|---|---|
| Cheap gameplay water | scrolling normals, simple shading, minimal scene reads |
| Standard surface water | dedicated water workflow, controlled depth interaction |
| Hero cinematic water | more advanced path, stronger budget, explicit profiling |

#### What to optimize first

1. Screen coverage
2. Scene color / refraction usage
3. Number of moving normal layers
4. Shoreline and foam logic
5. Whether this should really be translucent at all times

#### Good habits

- Separate shoreline logic from open-water logic when possible.
- Keep far-distance water materially simpler.
- Check whether a single convincing normal stack beats many weak ones.
- Compare expensive hero water against a simpler reference before committing.

#### Bad habits

- Large translucent water plane with many panners, foam layers, scene reads and distortion all active everywhere.
- One water material serving puddles, rivers, oceans and cinematic shots with only switches.

### 18.3 Foliage

Foliage cost is usually a mix of:

- masked overdraw
- alpha coverage
- two-sided lighting
- wind via WPO
- shadowing cost

#### What to optimize first

1. Alpha silhouette efficiency
2. Overdraw from card stacking
3. Wind complexity
4. Material samples
5. Shadow interaction

#### Good habits

- Author tighter alpha shapes.
- Keep wind deformation simple and stable.
- Build dedicated foliage masters instead of reusing generic masked masters blindly.
- Use different complexity levels for hero foliage and background fill.

#### Bad habits

- Tiny alpha detail everywhere on big cards.
- High-frequency WPO on dense foliage meshes.
- Treating foliage like regular masked props without considering how many times it overlaps on screen.

### 18.4 A shared triage order for these three categories

When any of these systems is slow, check in this order:

1. How many pixels are being shaded
2. How many times the same screen region is being shaded
3. How many samples the material graph takes
4. Whether expensive logic is localized or applied globally
5. Whether there is a cheaper near/far or hero/background split available

---

## 19. My Default Optimization Priorities

如果我接手一个项目，要开始治理材质性能，我会按这个顺序：

1. 透明特效和大面积透明
2. 场景中的 foliage 材质
3. 角色高频可见材质
4. Landscape 与地表多层材质
5. Hero asset 的复杂 Substrate 材质
6. 再回过头做母材质结构治理

原因是：

- 这是最容易拿到实帧收益的顺序

---

## 20. Material Debugging Handbook

This section is the practical debugging companion to the rest of the playbook.

The point is not to memorize every possible bug.
The point is to establish a repeatable order of suspicion.

### 19.1 If the material looks black

Common causes:

- wrong texture or missing texture binding
- a mask is zeroing out the result
- normal or shading path is broken
- material domain or blend mode is wrong
- Substrate graph is incomplete or invalid
- feature not supported by the active path or platform

Fast checks:

1. Replace BaseColor or Emissive with a constant color.
2. Bypass all masks and lerps temporarily.
3. Replace normal with a flat default.
4. Check the material domain and shading model.
5. If Substrate, reduce to one minimal BSDF.

What this tells you:

- if constant color works, the graph logic is the problem
- if it still fails, the issue is likely structural or path-related

### 19.2 If normals look wrong

Common causes:

- tangent-space normal mixed with world-space math
- normal map intensity pushed too far
- incorrect normal blending
- wrong tangent basis assumptions

Fast checks:

1. Feed a flat normal.
2. Disable all detail normals.
3. Compare tangent-space path versus world-space path separately.
4. Test with `PixelNormalWS` outputs only for debugging.

Typical fix:

- keep normal math in one basis as long as possible

### 19.3 If the effect moves when the actor moves

Common causes:

- world-space logic used for an object-local effect

Fast checks:

1. Replace `WorldPosition` with `LocalPosition`.
2. Move and rotate the actor in the level.
3. Check whether the effect should really be world-locked.

Typical fix:

- choose the correct space first, then rebuild the mask

### 19.4 If the effect swims with the camera

Common causes:

- `ScreenPosition`
- scene-texture-based logic
- view-dependent vectors used where object-stable logic was intended

Fast checks:

1. Disable screen-space inputs.
2. Replace them with local or world-space equivalents.
3. Move the camera around the asset.

Typical fix:

- reserve screen-space logic for effects that are meant to be screen-space

### 19.5 If WPO causes flicker or instability

Common causes:

- displacement too large
- mesh too dense or too sparse in the wrong places
- previous-frame offset path not matching current logic
- motion vectors and temporal logic not cooperating

Fast checks:

1. Clamp displacement amplitude hard.
2. Disable all but one WPO contributor.
3. Compare near and far camera distances.
4. Check whether the mesh density supports the deformation pattern.

Typical fix:

- simplify the deformation field and establish a maximum displacement budget

### 19.6 If PixelDepthOffset causes odd artifacts

Common causes:

- PDO used too aggressively
- effect fighting against geometry expectations
- interaction with masked / translucent setup

Fast checks:

1. Disable PDO entirely.
2. Restore it with very small amplitude.
3. Test the same material without translucent features.

Typical fix:

- use PDO only for narrow, justified cases

### 19.7 If SceneTexture logic breaks outside Post Process

Common causes:

- scene reads being used in the wrong domain or wrong blend path
- assumptions about filtering, UVs, or domain support

Fast checks:

1. Confirm domain first.
2. Check whether the logic belongs in `Surface`, `Deferred Decal`, or `Post Process`.
3. Replace scene reads with constants to isolate the rest of the graph.

Typical fix:

- move the logic to the right material domain or redesign the effect around object/world-space data

### 19.8 If the compile time suddenly explodes

Common causes:

- too many static switches
- material layers or functions changed in shared assets
- platform / quality permutations multiplied
- Substrate graph complexity increased

Fast checks:

1. Count static switches and static component masks.
2. Check whether a shared function or layer asset changed.
3. Compare compile scope before and after the change.
4. Reduce one recent feature branch at a time.

Typical fix:

- remove low-value static branches and split mothers by true rendering family instead of one universal graph

### 19.9 If runtime is slow but material stats seem acceptable

Common causes:

- overdraw
- screen coverage
- too many instances of the effect on screen
- pass-level cost rather than per-material instruction cost

Fast checks:

1. Use `Shader Complexity`.
2. Use `Shader Complexity & Quads`.
3. Run `ProfileGPU`.
4. Compare the same asset in isolation versus in the full scene.

Typical fix:

- optimize visibility, overdraw, and scene usage before micro-optimizing node math

### 19.10 If only one platform looks broken

Common causes:

- feature level differences
- unsupported path assumptions
- quality switches
- precision or path-specific fallback behavior

Fast checks:

1. Check `FeatureLevelSwitch` and `QualitySwitch`.
2. Preview the material on the target platform level.
3. Remove advanced features until the discrepancy disappears.

Typical fix:

- treat platform materials as deliberate branches, not accidental fallbacks

### 19.11 If a Substrate material fails or behaves strangely

Common causes:

- too many layers added before validating a minimal stack
- incomplete or conflicting operator topology
- pushing a hero-style material into the wrong asset class
- expecting the new HLSL generator path to own the issue when 5.7.4 still routes Substrate through legacy translator

Fast checks:

1. Collapse to one BSDF.
2. Re-add one operator at a time.
3. Verify the visual job of every layer.
4. Compare against a simpler legacy baseline.

Typical fix:

- rebuild from the simplest physically coherent structure instead of patching the large graph

### 19.12 The five-step emergency reduction method

When a material is broken and you need signal fast, do this:

1. Replace all major outputs with constants.
2. Re-enable only the base texture path.
3. Re-enable normal logic.
4. Re-enable masks and blends.
5. Re-enable expensive features one by one:
   - scene reads
   - noise
   - WPO
   - PDO
   - layers
   - Substrate operators

This is often faster than staring at the whole graph.

### 19.13 A debugging mindset that saves time

The best debugging question is usually not:

- what is the whole graph doing

It is:

- what is the first stage where the result becomes wrong

That shift makes large material graphs much more tractable.

---

## 21. Suggested Companion Reading

建议和下面三份一起读：

1. [UE57_Material_System_Analysis.md](/E:/Slash/CH/Docs/UE57_Material_System_Analysis.md:1)
2. [UE57_Material_Compile_Pipeline.md](/E:/Slash/CH/Docs/UE57_Material_Compile_Pipeline.md:1)
3. [UE57_Substrate_Practical_Guide.md](/E:/Slash/CH/Docs/UE57_Substrate_Practical_Guide.md:1)

---

## 22. References

- Material Analyzer  
  https://dev.epicgames.com/documentation/ja-jp/unreal-engine/unreal-engine-material-analyzer-tool

- UE5.7 Release Notes  
  https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-7-release-notes?application_version=5.7

- Local engine source:
  - `E:/Slash/Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp`
  - `E:/Slash/Engine/Source/Runtime/Engine/Public/Materials/Material.h`
  - `E:/Slash/Engine/Source/Editor/UnrealEd/Private/EditorViewportCommands.cpp`
  - `E:/Slash/Engine/Config/BaseInput.ini`
  - `E:/Slash/Engine/Config/BaseScalability.ini`
  - `E:/Slash/Engine/Config/ConsoleVariables.ini`
