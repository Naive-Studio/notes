# UE Material Node Reference

## Scope

This note is a study-oriented reference for Unreal Engine material nodes.

- Audience: gameplay engineers, technical artists, and VFX artists who want to understand material graphs from both math and runtime perspectives.
- Engine basis: the local engine source in this workspace plus Epic's material documentation.
- Goal: build a usable mental model first, then provide a searchable node inventory.

Two important caveats:

1. "All material nodes" is not the same as "all nodes you should use every day". The engine source contains helper nodes, output nodes, switches for specific rendering paths, and a few internal/editor-facing classes.
2. Category placement changes across UE versions. Some nodes that used to be grouped under `Coordinates` in older docs now appear under `Vector`, `Depth`, or `Utility`.

---

## How To Learn Material Nodes

Do not memorize nodes as isolated tools. Learn them in this order:

1. Data types
2. Coordinate spaces
3. Scalar and vector math
4. Texture sampling
5. Masking and blending
6. Lighting/view-dependent effects
7. Vertex deformation
8. Expensive procedural features
9. Platform and shading path switches

If this order is clear, most nodes become variations of the same small set of ideas.

---

## The Core Mental Model

A material graph is mostly doing one of these things:

- Build a scalar field: a single value such as roughness, opacity, height, or a mask.
- Build a vector field: a direction or color such as normal, tangent, world direction, or RGB.
- Transform coordinates: UV, local space, world space, view space, screen space.
- Blend fields together: `Lerp`, masks, layer weights, material attributes.
- Sample a stored signal: texture, scene texture, virtual texture, font atlas.
- Approximate geometry: fake depth, parallax, Fresnel edge, vertex offset.

The fastest way to read a material graph is to ask:

- What space is this value in
- Is it scalar or vector
- Is it sampled or procedurally built
- Does it run mostly per vertex or per pixel
- Does it introduce texture reads, scene reads, or special rendering path cost

---

## Performance First Principles

In real projects, the biggest costs usually come from:

- Texture sample count and bandwidth
- Translucent overdraw
- Scene texture reads
- Distance field queries
- Procedural noise
- Expensive world-position logic on large surfaces
- World Position Offset on dense meshes
- Shader permutation explosion from too many static switches

The cheapest things are usually:

- Constants
- Parameters
- Basic arithmetic such as `Add`, `Multiply`, `Min`, `Max`, `Clamp`, `Saturate`
- Simple masks and channel swizzles

Rule of thumb:

- ALU is often cheaper than extra texture fetches.
- A clean graph with repeated samples is often slower than a messier graph that samples less.
- A correct-looking opaque material is usually far cheaper than a fancy translucent one.

---

## Official Learning Buckets

The official docs organize nodes roughly like this:

- Atmosphere
- Color
- Constant
- Coordinates
- Custom
- Depth
- Font
- Material Functions
- Landscape
- Material Attributes
- Math
- Parameters
- Particles
- Texture
- Utility
- Vector
- Vector Operations

For learning, this note uses a more practical grouping:

- Constants and parameters
- Coordinates and spaces
- Math and vector operations
- Texture and scene sampling
- Masks, blending, and attributes
- Lighting and view effects
- Vertex and deformation
- Landscape and terrain
- Utility and platform switches
- Specialized outputs and advanced rendering nodes

---

## 1. Constants And Parameters

These nodes do not create meaning by themselves. They control values.

### Common nodes

| Node | Meaning | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `Constant` | Single scalar | Roughness, metallic, thresholds | Very low | Prefer constants for fixed values |
| `Constant2Vector` | 2D vector | UV scale, offsets | Very low | Good for UV math |
| `Constant3Vector` | 3D vector / RGB | BaseColor, direction, tints | Very low | Use for debug colors and quick prototypes |
| `Constant4Vector` | 4D vector / RGBA | Packed controls | Very low | Good for grouped authoring values |
| `ScalarParameter` | Tunable scalar in instances | Roughness, emissive power | Very low | Prefer instances instead of duplicating materials |
| `VectorParameter` | Tunable color or vector | Tint, custom directions | Very low | Great for look-dev iteration |
| `CollectionParameter` | Global shared parameter | Time-of-day, weather, global wetness | Very low at runtime | Use for project-wide controls, not per-asset overrides |
| `DynamicParameter` | Particle-driven runtime parameter | Niagara / Cascade inputs | Low | Keep channel meaning documented |
| `StaticSwitchParameter` | Compile-time branch | Feature toggles, high/low quality paths | Runtime almost free, build cost high | Only use for meaningful branches |
| `StaticComponentMaskParameter` | Compile-time channel mask | Reuse packed textures | Runtime almost free, build cost high | Useful when channel use changes by material family |

### Geometric meaning

These nodes have no direct geometric meaning. They are control points.

### Study tip

If a graph is hard to read, replace every `ScalarParameter` and `VectorParameter` with a temporary constant in your head. Then read the real logic.

---

## 2. Coordinates And Spaces

This is where most beginners get lost. A node is often simple, but the space is not.

### Common nodes

| Node | Space meaning | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `TextureCoordinate` | UV space | Base texture mapping | Low | Share UV logic between samples |
| `Panner` | UV translation over time | Water, scrolling effects | Low | Move to customized UVs when possible |
| `Rotator` | UV rotation | Radar sweep, rotating decals | Low | Combine with scale and offset in one path |
| `WorldPosition` | Pixel position in world space | World-aligned textures, triplanar | Low to medium | Avoid repeated transforms |
| `LocalPosition` | Position in local/object space | Local dissolve, object-relative effects | Low | Prefer over world space when effect is object-local |
| `ActorPositionWS` | Actor origin in world space | Distance-to-center effects | Low | Often pair with `WorldPosition` subtraction |
| `ObjectPositionWS` | Object position in world space | Per-object gradients | Low | Useful for shared materials |
| `CameraPositionWS` | Camera world position | Distance fade, view-dependent offsets | Low | Cache repeated direction calculations |
| `CameraVectorWS` | View direction | Fresnel, fake reflections | Low | Normalize once if reused |
| `ReflectionVectorWS` | Reflected view vector | Cubemap reflection | Low to medium | Good cheap fake reflection input |
| `ScreenPosition` | Screen UV / clip-related position | Screen-space effects | Medium | Be careful with perspective assumptions |
| `ViewSize` | View resolution | Pixel-size-aware effects | Low | Useful for screen-space thresholds |
| `SceneTexelSize` | Scene texture texel size | Neighbor lookups, blur steps | Low | Keep offsets minimal |
| `LightmapUVs` | Lightmap UV channel | Static-lighting utilities | Low | Niche, do not confuse with base UV |
| `PixelNormalWS` | Per-pixel world normal | Fresnel, detail comparisons | Low to medium | If a per-vertex approximation works, use that |
| `VertexNormalWS` | Per-vertex world normal | WPO, directional offsets | Low | Best used in vertex path |
| `PreSkinnedPosition` / `PreSkinnedNormal` | Pre-skinning local data | Stable character-space effects | Medium | Great for skeletal mesh-local masks |

### Geometric meaning

- `TextureCoordinate`: position on a 2D parameter surface.
- `WorldPosition`: position in the 3D world field.
- `LocalPosition`: position in the mesh's own frame.
- `CameraVectorWS`: direction from surface toward camera.
- `ReflectionVectorWS`: mirror reflection of the view direction around the surface normal.

### Classic example

World-space snow:

1. Get `PixelNormalWS`
2. Dot with world up vector `(0,0,1)`
3. Clamp or smoothstep
4. Use as a mask in `Lerp`

This works because the dot product measures how much the surface normal points upward.

### Optimization notes

- Repeated world-space math across many texture layers adds up.
- If the effect is tied to the object, local space is usually cheaper and easier to reason about.
- If UV motion is shared by several texture samples, build the UV once and fan it out.

---

## 3. Math And Vector Operations

This group gives the graph its actual logic.

### The must-know operators

| Node | Geometric interpretation | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `Add` / `Subtract` | Translation / offset | Biasing ranges, center-relative math | Very low | Free enough for heavy use |
| `Multiply` / `Divide` | Scale / inverse scale | Contrast, amplitude, remapping | Very low | Prefer multiply by reciprocal if value is constant |
| `Min` / `Max` | Region clipping | Mask combination | Very low | Great cheap alternative to branches |
| `Clamp` / `Saturate` | Range restriction | Keep mask in `[0,1]` | Very low | `Saturate` is the usual fast mask clamp |
| `OneMinus` | Value inversion | Invert masks | Very low | Better than adding custom subtract logic |
| `Abs` | Distance from zero | Symmetric patterns | Very low | Common in mirrored shapes |
| `Frac` / `Floor` / `Ceil` / `Round` | Quantization and periodic decomposition | Tiling, step patterns | Low | Useful but easy to alias visually |
| `Fmod` / `Modulo` | Periodic remainder | Repeating pulses | Low | Good for segmented procedural effects |
| `Power` | Nonlinear shaping | Contrast, Fresnel shaping | Low to medium | Avoid stacking multiple powers |
| `SquareRoot` | Distance shaping | Falloffs, remaps | Medium | Keep it only where needed |
| `Sine` / `Cosine` | Periodic oscillation | Waves, pulses, flow | Medium | Try baking if pattern is fixed |
| `Step` / `SmoothStep` | Hard or soft threshold | Stylized masks, borders | Low | `SmoothStep` is usually visually safer |
| `DotProduct` | Projection / angle cosine | Facing masks, snow, Fresnel basis | Low | One of the most important nodes |
| `CrossProduct` | Orthogonal vector | Basis reconstruction | Low | Use when you need a perpendicular direction |
| `Normalize` | Preserve direction only | Direction math | Low to medium | Cache if reused |
| `Length` / `Distance` | Metric magnitude | Radial masks, falloff | Low to medium | `Distance` is conceptually clear |
| `AppendVector` / `ComponentMask` | Rebuild or split channels | Pack/unpack logic | Very low | Essential for texture packing |
| `Transform` / `TransformPosition` | Change coordinate basis | Tangent/world/view conversions | Medium | Minimize repeated conversions |
| `Lerp` (`LinearInterpolate`) | Convex blend | Texture layering, masks | Low | Prefer one clean alpha path |

### Three concepts you should internalize

#### Dot product

The dot product answers:

- Are these directions aligned
- Opposed
- Or orthogonal

Use it for:

- Snow accumulation
- Fresnel-like masks
- Forward-facing decals
- Anisotropy helpers

#### Normalize

Normalization throws away magnitude and keeps only direction.

If a graph compares directions, normalization usually makes the math stable.

#### Lerp

`Lerp(A, B, Alpha)` is not just "mix two colors".
It means "move continuously from field A to field B under the control of Alpha".

That mental model makes layered materials much easier to read.

---

## 4. Texture And Scene Sampling

A lot of material work is simply reading stored signals efficiently.

### Common nodes

| Node | Meaning | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `TextureSample` | Read a texture using UVs | BaseColor, normal, ORM maps | Medium | The main cost driver in many materials |
| `TextureObject` | Pass texture object without sampling | Material functions, shared sample path | Very low by itself | Sample later, not early |
| `TextureProperty` | Query texture metadata | Resolution-aware logic | Low | Rarely needed in simple materials |
| `SceneTexture` | Read scene buffers | Post/decal/special effects | Medium to high | Use only when required |
| `SceneColor` | Read scene color | Distortion, refraction-like tricks | Medium to high | Overuse hurts bandwidth |
| `SceneDepth` | Read depth buffer | Soft intersections, water edges | Medium | Good, but still a scene read |
| `SceneDepthWithoutWater` | Depth ignoring water surface | Water-specific logic | Medium | Specialized |
| `RuntimeVirtualTextureSample` | Read RVT data | Landscape blending, cached layers | Medium | Great for amortizing expensive layers |
| `SparseVolumeTextureSample` | Read sparse volume texture | Clouds, volumetric data | Medium to high | Highly specialized |
| `DBufferTexture` | Read decal-related buffer | Deferred decal workflows | Medium to high | Use only inside the proper pipeline |

### Texture sampling optimization

- Pack roughness, AO, metallic, masks into channels.
- Reuse UVs.
- Avoid sampling the same texture more than once unless LOD/bias differs intentionally.
- Prefer a baked texture over procedural reconstruction if the pattern is stable.
- A visually rich material with 4 smart samples often beats a 12-sample graph built from reusable fragments.

---

## 5. Masks, Layering, And Material Attributes

This is the bridge from math to actual material authoring.

### Common nodes

| Node | Meaning | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `Lerp` | Blend two signals | Dirt, wetness, detail maps | Low | Keep alpha readable |
| `SphereMask` | Radial soft mask | Impact decals, local dissolve | Low | Great cheap localized effect |
| `AntialiasedTextureMask` | Texture-driven mask with AA awareness | Crisp UI-like cutouts | Medium | Great for hard-edge masks |
| `MakeMaterialAttributes` | Build a full material payload | Layered materials | Low by itself | Organize complex graphs |
| `SetMaterialAttributes` | Set selected attributes | Cleaner layered graphs | Low by itself | Usually better than giant monoliths |
| `BreakMaterialAttributes` / `GetMaterialAttributes` | Extract specific channels | Layer mixing | Low by itself | Avoid over-fragmenting graphs |
| `BlendMaterialAttributes` | Blend two material payloads | Clearcoat, layered surfaces | Medium | Powerful, but both sides can stay expensive |
| `ShadingModel` | Select shading model in expression-based workflows | Advanced material setups | Context-dependent | Use only when architecture needs it |

### Geometric meaning

These nodes usually do not define geometry directly.
They define where one appearance wins over another.

That "where" often comes from:

- world-space masks
- normal-based masks
- height maps
- distance fields
- painted vertex colors
- landscape weights

---

## 6. Lighting And View-Dependent Effects

These are the nodes that make a surface react to viewing angle or lighting context.

### Common nodes

| Node | Meaning | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `Fresnel` | Grazing-angle falloff from normal vs view | Rim light, glass edge | Low | Cheap and effective |
| `ReflectionVectorWS` | Reflection direction | Cubemap sampling | Low to medium | Great for fake reflections |
| `BlackBody` | Kelvin to color mapping | Fire, heated metal | Low | Author with temperature, not RGB guesswork |
| `Desaturation` | Color intensity reduction | State effects, style passes | Low | Cheap utility |
| `GIReplace` / `LightmassReplace` / `ShadowReplace` | Alternate values for specific lighting paths | Baked-lighting tricks | Specialized | Keep usage documented |
| `EyeAdaptation` / `EyeAdaptationInverse` | Exposure-related response | HDR-aware emissive tuning | Medium | Useful for post-sensitive effects |
| `AtmosphericFogColor`, `AtmosphericLightColor`, `SkyAtmosphere*` | Atmosphere lighting inputs | Sky, cloud, scattering effects | Specialized | Usually for sky and VFX shaders |

### Geometric meaning

The main geometry idea here is angle dependence:

- surface normal
- view vector
- light vector
- reflected vector

If an effect changes when the camera moves around the object, this bucket is involved.

---

## 7. Vertex, Deformation, And Pseudo-Geometry

These nodes change the apparent or actual shape.

### Common nodes

| Node | Meaning | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `VertexNormalWS` | Vertex-space world normal | Push along normal | Low in vertex stage | Great for simple expansion |
| `WorldPositionOffset` input | Actual vertex displacement | Wind, wobble, breathing | Depends on vertex count | Expensive on dense meshes |
| `BumpOffset` | Height-based UV parallax | Bricks, shallow grooves | Medium to high | Good fake depth, cheaper than real geometry |
| `DeriveNormalZ` | Reconstruct Z from XY normal | Normal math cleanup | Low | Useful with packed procedural normal logic |
| `RotateAboutAxis` | Rotate vectors around axis | Swirl and local deformation helpers | Medium | Keep math local and bounded |
| `MotionVectorWorldOffsetOutput` | Motion vector support for WPO | Correct temporal rendering | Specialized | Needed for high-quality animated WPO |
| `SphericalParticleOpacity` | Spherical fade model | Particles | Low | Specialized but cheap |

### Important warning

World Position Offset cost scales with vertex density.

A tiny WPO effect on a dense mesh can cost more than a big-looking pixel shader trick on a simpler asset.

---

## 8. Landscape And Terrain

Landscape materials are powerful but are often where shader cost gets out of control.

### Common nodes

| Node | Meaning | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `LandscapeLayerBlend` | Blend terrain layers | Grass, dirt, rock, snow | Medium to high | The core terrain blend node |
| `LandscapeLayerWeight` | Single-layer weight access | Custom logic per layer | Medium | Useful for local branch control |
| `LandscapeLayerSwitch` | Skip work if a layer is absent | Terrain optimization | Runtime savings, compile cost | Very valuable in large materials |
| `LandscapeLayerSample` | Read layer value | Mask-driven effects | Medium | Often cleaner than overloading blend logic |
| `LandscapeLayerCoords` | Landscape UV generation | Tile control, macro/micro mapping | Low | Centralize coordinate logic |
| `LandscapeVisibilityMask` | Visibility / holes | Caves, mesh cutouts | Low | Simple but important |
| `LandscapePhysicalMaterialOutput` | Per-layer physical material output | Footsteps, traces | Specialized | Useful for gameplay hooks |

### Key rule

Always design terrain materials around pruning.

If you build every expensive branch for every pixel and only blend at the end, the material will scale badly.

---

## 9. Utility, Switches, And Specialized Outputs

These nodes solve edge cases, platform differences, and rendering-path-specific needs.

### Common utility nodes

| Node | Meaning | Typical use | Cost | Optimization |
|---|---|---|---|---|
| `FeatureLevelSwitch` | Switch by feature level | Mobile vs desktop | Runtime cheap, permutation cost | Good for platform tailoring |
| `QualitySwitch` | Switch by quality level | Scalability tiers | Runtime cheap, permutation cost | Keep branch count small |
| `RayTracingQualitySwitch` | RT quality path split | High-end rendering | Runtime cheap, permutation cost | Specialized |
| `PathTracingQualitySwitch` | Path tracing path split | Offline/high-fidelity path | Runtime cheap, permutation cost | Specialized |
| `ShaderStageSwitch` | Vertex vs pixel stage switch | Split logic by stage | Context-dependent | Good for moving math upstream |
| `PreviousFrameSwitch` | Access previous-frame logic | Temporal effects | Specialized | Use sparingly |
| `DistanceToNearestSurface` | Query global distance field | Contact highlights, intersection masks | Medium to high | Very useful, but not cheap |
| `DistanceFieldGradient` | Gradient of distance field | Flow and direction from distance field | Medium to high | Specialized and powerful |
| `DistanceFieldApproxAO` | Approximate AO from distance fields | Soft occlusion helpers | Medium to high | Specialized |
| `DDX` / `DDY` | Screen-space derivatives | Mip logic, analytic anti-aliasing | Medium | Advanced usage only |
| `Custom` | Inject HLSL | Anything not expressible cleanly | Depends entirely on code | Last resort, not first instinct |

### When to use `Custom`

Use `Custom` only when:

- a node graph would be unreadable
- you need a missing math primitive
- you need a tight loop or algorithm the graph does not express well

Do not use it just because it looks "more programmer-like".

---

## A Small Cost Ladder

### Usually cheap

- constants
- parameters
- scalar math
- masks
- dot products
- `Lerp`
- `Fresnel`

### Usually moderate

- a few texture samples
- world-space transforms
- scene depth reads
- parallax-like UV tricks
- repeated normal-space conversions

### Usually expensive

- many texture samples
- procedural `Noise`
- `VectorNoise`
- distance field queries
- scene texture heavy paths
- deep layered landscapes
- WPO on dense meshes
- complex translucent materials

---

## Practical Optimization Checklist

- Reduce texture samples before you micro-optimize arithmetic.
- Pack masks into channels.
- Share UV logic.
- Move suitable math to the vertex stage or customized UVs.
- Bake stable procedural effects.
- Avoid unnecessary world-space work in object-local effects.
- Prefer opaque or masked over translucent when possible.
- Use static switches only for meaningful architectural splits.
- Use material instances for look iteration, not duplicated master materials.
- Check shader complexity, material stats, and instruction counts early.

---

## Recommended Study Order For Daily Practice

Week 1:

- `Constant`
- `ScalarParameter`
- `VectorParameter`
- `TextureCoordinate`
- `TextureSample`
- `Multiply`
- `Lerp`
- `Clamp`
- `Saturate`

Week 2:

- `WorldPosition`
- `LocalPosition`
- `DotProduct`
- `Normalize`
- `Fresnel`
- `SphereMask`
- `Panner`
- `Rotator`

Week 3:

- `BumpOffset`
- `SceneDepth`
- `DepthFade`
- `PixelNormalWS`
- `VertexNormalWS`
- `RuntimeVirtualTextureSample`
- `LandscapeLayerBlend`

Week 4:

- `DistanceToNearestSurface`
- `DDX`
- `DDY`
- `Custom`
- quality and feature switches
- expression-based shading model workflows

---

## 10. Material Space And Math Primer

This section is the "why" behind many common material graphs.

Most difficult material bugs are not caused by a missing node.
They are caused by one of these:

- wrong space
- wrong value type
- wrong normalization
- wrong assumption about what a vector means

If this section is clear, the graph becomes much easier to design and debug.

### The spaces you should know first

| Space | What it means | Common nodes | Best for | Common mistake |
|---|---|---|---|---|
| UV space | 2D parameterization on a surface | `TextureCoordinate`, `Panner`, `Rotator` | Texture mapping, scrolling, tiling | Treating UV as if it were world-stable |
| Tangent space | Surface-local orientation basis | normal maps, tangent-space normal input | Standard normal map workflows | Mixing tangent normal with world vectors directly |
| Local / object space | Coordinates relative to the mesh origin | `LocalPosition`, `ObjectPositionWS` as reference anchor | Object-local gradients, local dissolve | Using world space for object-local effects |
| Pre-skinned local space | Skeletal mesh local space before skinning | `PreSkinnedPosition`, `PreSkinnedNormal` | Stable character masks | Using current skinned world position for a body-space effect |
| World space | Global level coordinates | `WorldPosition`, `ActorPositionWS`, `PixelNormalWS` | World-aligned textures, snow, terrain blending | Forgetting that the effect becomes world-stable, not object-stable |
| View / camera space | Relative to the camera | `CameraVectorWS`, `CameraPositionWS` | Fresnel, view-facing masks | Using it when you really wanted object orientation |
| Screen space | Relative to the viewport | `ScreenPosition`, `SceneTexelSize` | Post-process, scene lookups, screen UV tricks | Expecting it to stay fixed on an object in the world |

### A practical rule

Always ask:

- Where does this value live
- Does it move with the object
- Does it move with the camera
- Does it stay fixed in the world

That one habit solves a huge number of material problems.

### The three most important coordinate decisions

#### 1. Should the effect move with the object

If yes, prefer:

- `LocalPosition`
- `PreSkinnedPosition`
- UV space

Examples:

- local dissolve from the center of a mesh
- a body-space gradient on a character
- a weapon heat mask that should rotate with the gun

#### 2. Should the effect stay fixed in the level

If yes, prefer:

- `WorldPosition`
- `PixelNormalWS`
- world up or world directional vectors

Examples:

- snow accumulation
- moss on the north-facing side
- world-aligned dirt

#### 3. Should the effect react to the camera

If yes, prefer:

- `CameraVectorWS`
- `ReflectionVectorWS`
- `ScreenPosition`

Examples:

- rim light
- force-field edge glow
- post-process distortion

### Normal, tangent, and why people get confused

In materials, a "normal" is not just "a direction".
Its meaning depends on which basis it belongs to.

#### Tangent-space normal

This is the usual output from a normal map texture.

Use it when:

- the material input expects tangent-space normal
- you are in a standard surface workflow

#### World-space normal

Use it when:

- you want to compare a surface with world up
- you want a camera-facing or reflection-facing effect
- you are doing triplanar or world-projected logic

Typical nodes:

- `PixelNormalWS`
- `VertexNormalWS`

### The shortest useful math toolkit

The following small set of operations explains most material graphs:

| Operation | Meaning | Typical use |
|---|---|---|
| `Add/Subtract` | Offset / relative position | shift masks, compute delta |
| `Multiply/Divide` | Scale / remap | contrast, UV tiling |
| `DotProduct` | Angle alignment / projection | facing masks, snow, Fresnel basis |
| `CrossProduct` | Build a perpendicular direction | custom basis construction |
| `Normalize` | Keep direction only | stable directional math |
| `Length/Distance` | Radial metric | sphere masks, distance fades |
| `Clamp/Saturate` | Keep values in range | masks and safe blending |
| `Lerp` | Controlled blend | layer mixing |
| `Abs` | Symmetry around zero | mirrored masks, triplanar weights |
| `Frac/Floor/Fmod` | repetition and segmentation | tile patterns, pulses |

### The four formulas worth memorizing

#### 1. Normalize a range to 0..1

```text
t = saturate((x - minValue) / (maxValue - minValue))
```

Use it for:

- height masks
- distance fades
- slope bands

#### 2. Radial mask

```text
d = length(P - Center)
mask = 1 - saturate(d / Radius)
```

Use it for:

- hit pulses
- local glow
- circular dissolve

#### 3. Facing mask

```text
facing = saturate(dot(normalize(A), normalize(B)))
```

Use it for:

- snow from world up
- camera-facing masks
- directional wear

#### 4. Signed plane mask

```text
planeValue = dot(P - Origin, PlaneNormal)
```

Use it for:

- clipping along a plane
- top-to-bottom dissolve
- directional reveal

### Dot product in plain language

Dot product is the most valuable node in UE materials.

You can read it like this:

- close to `1`: two directions point the same way
- close to `0`: they are perpendicular
- close to `-1`: they point opposite ways

Common pairings:

- `PixelNormalWS · WorldUp`
- `CameraVectorWS · PixelNormalWS`
- `normalize(WorldPosition - Center) · Direction`

### Lerp in plain language

`Lerp(A, B, Alpha)` means:

- use A when Alpha is 0
- use B when Alpha is 1
- blend between them for values in the middle

This is not just for color.
It is the general-purpose node for:

- mixing roughness
- blending normals
- switching emissive intensity
- layering dirt, snow, wetness, burns

### Triplanar math in one paragraph

Triplanar mapping works because:

1. you project the texture from X, Y, and Z directions
2. you compute weights from the absolute normal components
3. you normalize those weights
4. you blend the three projections

In simplified form:

```text
w = abs(normal)
w = w / (w.x + w.y + w.z)
result = XProj * w.x + YProj * w.y + ZProj * w.z
```

The visual cost comes from multiple texture samples, not from the weight math itself.

### Space conversion rules you should follow

#### Rule 1

Do not compare a tangent-space normal directly with a world-space vector.

If you do, the result is meaningless.

#### Rule 2

If an effect should stay fixed on a skeletal mesh body part, prefer:

- `PreSkinnedPosition`
- `PreSkinnedNormal`

instead of current world position.

#### Rule 3

If an effect is purely per-object, start in local space before reaching for world space.

That often makes the graph easier and cheaper.

#### Rule 4

If an effect is screen-based, accept that it will not be object-stable.

This is expected behavior, not a bug.

### Common beginner mistakes

| Mistake | What it looks like | Actual issue | Fix |
|---|---|---|---|
| Snow mask rotates strangely with the mesh | snow appears object-relative | used local normal instead of world normal | use `PixelNormalWS` and world up |
| Dissolve shifts when actor moves | local effect becomes world-locked | used `WorldPosition` | use `LocalPosition` |
| Normal blend looks wrong | lighting breaks or flips | mixed tangent and world data | convert or stay in one basis |
| Screen-space distortion swims on object | effect follows camera | used `ScreenPosition` intentionally or accidentally | switch to object/world logic if you need stability |
| Rim light is too strong or unstable | edge glow covers whole object | missing normalization or wrong sign | normalize vectors and clamp result |
| Triplanar is too expensive | nice result but heavy | 3x or more texture fetches | reserve for assets that truly need it |

### A quick debugging checklist for math and space

When a graph feels wrong, ask these in order:

1. What space is every major vector in
2. Which values are scalar and which are vector
3. Did I normalize directional vectors before comparing them
4. Should this effect move with the object, world, or camera
5. Am I using `WorldPosition` where `LocalPosition` would be enough
6. Am I using `PixelNormalWS` where tangent-space normal data was intended

### Practice drills

Use these as small exercises:

1. Build a top-down gradient twice: once with `WorldPosition.Z`, once with `LocalPosition.Z`, then move the actor.
2. Build a snow mask with `PixelNormalWS · WorldUp`.
3. Build a rim mask with `1 - saturate(dot(N, V))`.
4. Build a radial pulse using `Distance(WorldPosition, Center)`.
5. Build a local hit sphere using `LocalPosition`.
6. Build a triplanar prototype and compare its sample count with a regular UV material.

---

## 11. Material Cookbook

This section turns the previous theory into repeatable practice.

Each recipe uses a small core set of nodes and gives you:

- a target effect
- the key idea
- the core nodes
- the main cost warning

### Recipe 01: Scrolling Water

Target:

- a cheap moving water surface

Core nodes:

- `TextureCoordinate`
- `Panner`
- `TextureSample`
- `Lerp`
- `Normal`

Build:

1. Start from one UV channel.
2. Feed it into a `Panner`.
3. Sample a normal map and a foam/noise map with that panned UV.
4. Optionally use two panners at different speeds and directions, then blend.
5. Use the result to drive normal and subtle emissive/roughness variation.

Why it works:

- UV motion gives the illusion of flow without changing geometry.

Performance note:

- Usually cheap to medium.
- The danger is stacking too many panned samples.

### Recipe 02: World-Space Snow

Target:

- snow collects on upward-facing surfaces

Core nodes:

- `PixelNormalWS`
- `Constant3Vector(0,0,1)`
- `DotProduct`
- `Saturate`
- `Power`
- `Lerp`

Build:

1. Use `PixelNormalWS`.
2. Dot it with world up.
3. Clamp to `0..1`.
4. Shape the mask with `Power` or `SmoothStep`.
5. Use that mask to blend snow color, roughness, and normal detail.

Why it works:

- the dot product measures how much the surface faces upward

Performance note:

- Cheap.
- One of the best examples of useful world-space logic.

### Recipe 03: Fresnel Force Field

Target:

- bright edge glow around an energy shield or hologram

Core nodes:

- `Fresnel`
- `Power`
- `Multiply`
- `Emissive`
- `Opacity`

Build:

1. Start from `Fresnel`.
2. Shape the edge width with exponent.
3. Multiply by color and intensity.
4. Feed into emissive.
5. Optionally use a softer version for opacity.

Why it works:

- grazing angles receive stronger response than front-facing angles

Performance note:

- Usually cheap.
- The real cost comes if you make the material translucent and full-screen.

### Recipe 04: Height Dissolve With Edge Glow

Target:

- object dissolves upward with a glowing edge

Core nodes:

- `LocalPosition`
- `ComponentMask`
- `ScalarParameter`
- `Subtract`
- `SmoothStep`
- `OneMinus`
- `Emissive`
- `OpacityMask`

Build:

1. Use `LocalPosition.Z` as a height value.
2. Compare it against a dissolve threshold parameter.
3. Generate a thin band near the threshold for edge glow.
4. Use the main mask for opacity mask or opacity.
5. Use the thin band for emissive.

Why it works:

- you are effectively moving a clipping plane through object-local space

Performance note:

- Cheap to medium.
- Excellent masked-material effect.

### Recipe 05: Local Hit Pulse

Target:

- circular pulse where the actor was hit

Core nodes:

- `LocalPosition`
- `Distance`
- `Radius parameter`
- `OneMinus`
- `SmoothStep`
- `Lerp`

Build:

1. Define a hit center in local space.
2. Compute `Distance(LocalPosition, HitCenter)`.
3. Turn that into a radial mask.
4. Optionally animate radius over time.
5. Use the mask to boost emissive, tint, or roughness.

Why it works:

- distance in local space gives an object-stable radial region

Performance note:

- Cheap.
- Very good for gameplay-driven effects on characters and props.

### Recipe 06: Wetness Overlay

Target:

- material becomes darker, smoother, and slightly more saturated when wet

Core nodes:

- `ScalarParameter`
- `Lerp`
- base color
- roughness
- normal attenuation logic

Build:

1. Create a wetness mask or scalar amount.
2. Darken base color slightly.
3. Lower roughness noticeably.
4. Optionally soften high-frequency normal detail.
5. Blend all of them with the same wetness alpha.

Why it works:

- wet surfaces are usually darker and less rough

Performance note:

- Cheap.
- Great candidate for material instances or MPC control.

### Recipe 07: Triplanar Rock

Target:

- rock material without visible UV seams

Core nodes:

- `WorldPosition`
- `PixelNormalWS`
- `Abs`
- `TextureSample`
- `Lerp` or weighted blend math

Build:

1. Project texture in X, Y, and Z directions.
2. Compute blending weights from `abs(PixelNormalWS)`.
3. Normalize weights.
4. Blend the three projections.
5. Apply same logic to color and normal with care.

Why it works:

- surfaces facing each axis receive the matching projection

Performance note:

- Medium to expensive.
- Powerful, but texture sample count grows quickly.

### Recipe 08: Cheap Hologram

Target:

- stylized hologram with scrolling lines and edge glow

Core nodes:

- `ScreenPosition` or UV
- `Panner`
- `Frac`
- `Sine`
- `Fresnel`
- `Emissive`

Build:

1. Generate repeating scanlines in UV or screen space.
2. Animate them with `Panner`.
3. Add a Fresnel edge.
4. Blend a cool emissive tint.
5. Keep opacity simple.

Why it works:

- the effect combines motion, repetition, and edge emphasis

Performance note:

- Cheap to medium if opaque or masked.
- Can become expensive if built as a large translucent sheet.

### Recipe 09: Soft Intersection Foam

Target:

- foam or bright edge where water meets geometry

Core nodes:

- `SceneDepth`
- `PixelDepth`
- `DepthFade`
- `Lerp`
- foam texture

Build:

1. Use `DepthFade` or compare scene and pixel depth.
2. Build a narrow intersection mask.
3. Multiply by a foam texture.
4. Use the result for emissive, opacity, or color shift.

Why it works:

- depth difference approximates how close the translucent surface is to other geometry

Performance note:

- Medium.
- Valuable, but scene-depth reads are not free.

### Recipe 10: Grass Wind WPO

Target:

- foliage bends in the wind

Core nodes:

- `VertexNormalWS`
- `Simple gradient`
- wind phase from `Time`
- `Sine`
- `WorldPositionOffset`

Build:

1. Create a height mask so the base stays stable.
2. Generate a wave over time.
3. Multiply by a directional wind vector.
4. Scale by the vertex mask.
5. Feed into `WorldPositionOffset`.

Why it works:

- the base remains anchored while upper vertices move

Performance note:

- Vertex cost, not just pixel cost.
- Watch mesh density carefully.

### Recipe 11: Lava Cracks

Target:

- dark cooled rock with hot emissive cracks

Core nodes:

- crack mask texture
- `Lerp`
- `Emissive`
- `Roughness`
- `Panner` or subtle noise

Build:

1. Use a crack mask to separate hot and cold regions.
2. Darken the base surface heavily.
3. Put strong emissive only in the cracks.
4. Optionally animate emissive slightly over time.
5. Lower roughness near the hot regions if needed.

Why it works:

- high contrast between emissive lines and cooled rock sells the effect quickly

Performance note:

- Cheap to medium.
- Most of the look should come from the mask, not heavy procedural noise.

### Recipe 12: Runtime Damage Flash

Target:

- character or prop flashes when damaged

Core nodes:

- `ScalarParameter`
- `VectorParameter`
- `Lerp`
- `Emissive`
- optional `Fresnel`

Build:

1. Expose a damage amount scalar.
2. Blend base color toward a damage tint.
3. Optionally boost emissive during the peak.
4. Drive the parameter from MID in gameplay code or Blueprint.

Why it works:

- parameter-driven blend is the cheapest reliable gameplay feedback path

Performance note:

- Cheap.
- One of the best examples of when MID is more important than graph complexity.

### Recipe 13: World-Space Moss

Target:

- moss appears on shaded or upward-facing world surfaces

Core nodes:

- `PixelNormalWS`
- world direction
- optional world position noise or mask
- `Lerp`

Build:

1. Start from a world-facing mask, similar to snow.
2. Bias it toward surfaces that should collect moss.
3. Modulate with noise or artist mask.
4. Blend moss color, roughness, and normal detail.

Why it works:

- it leverages a believable environment direction cue

Performance note:

- Cheap to medium.
- Very effective if you keep texture count under control.

### Recipe 14: Contact Highlight With Distance Field

Target:

- object glows where it approaches nearby geometry

Core nodes:

- `DistanceToNearestSurface`
- `SmoothStep`
- `OneMinus`
- `Emissive`

Build:

1. Query distance to nearest surface.
2. Convert low distances into a bright mask.
3. Shape the response with `SmoothStep`.
4. Use for emissive or rim tint.

Why it works:

- the material reacts to actual nearby world geometry

Performance note:

- Medium to expensive.
- Save for high-value interactive materials.

### Recipe 15: Post-Process Edge Tint

Target:

- screen-space stylized edge emphasis

Core nodes:

- `SceneTexture`
- `SceneDepth`
- screen UV logic
- post-process domain

Build:

1. Work in `Post Process` domain.
2. Use scene data to detect edges or depth differences.
3. Tint those edges.
4. Keep the pass focused and minimal.

Why it works:

- post-process materials have direct access to screen-space scene information

Performance note:

- Full-screen cost.
- Be very deliberate about how much math and how many scene reads you add.

### How to practice this cookbook

A good way to study these recipes is:

1. Rebuild the effect with only the listed core nodes.
2. Verify that the basic version already reads correctly.
3. Add only one upgrade at a time.
4. Check instruction count and sample count after each upgrade.

That habit will teach you both visual construction and cost intuition at the same time.

---

## Node Inventory Snapshot From Local Engine Source

The local engine source currently exposes 277 `MaterialExpression*.h` headers under the runtime material and landscape folders.

This raw inventory includes:

- regular palette nodes
- landscape nodes
- advanced output nodes
- path/platform switches
- a few helper or specialized internal classes

Use this section as a lookup index, not as a "learn in this order" list.

### Inventory

```text
Abs
AbsorptionMediumMaterialOutput
ActorPositionWS
Add
Aggregate
AntialiasedTextureMask
AppendVector
Arccosine
ArccosineFast
Arcsine
ArcsineFast
Arctangent
Arctangent2
Arctangent2Fast
ArctangentFast
AtmosphericFogColor
AtmosphericLightColor
AtmosphericLightVector
BentNormalCustomOutput
BindlessSwitch
BlackBody
Blend
BlendMaterialAttributes
Bounds
BreakMaterialAttributes
BumpOffset
CameraPositionWS
CameraVectorWS
Ceil
ChannelMaskParameter
ChannelMaskParameterColor
Clamp
ClearCoatNormalCustomOutput
CloudLayer
CollectionParameter
CollectionTransform
ColorRamp
Comment
ComponentMask
Composite
Constant
Constant2Vector
Constant3Vector
Constant4Vector
ConstantBiasScale
Convert
Cosine
CrossProduct
CurveAtlasRowParameter
Custom
CustomOutput
DataDrivenShaderPlatformInfoSwitch
DBufferTexture
DDX
DDY
DecalColor
DecalDerivative
DecalLifetimeOpacity
DecalMipmapLevel
DeltaTime
DepthFade
DepthOfFieldFunction
DeriveNormalZ
Desaturation
Distance
DistanceCullFade
DistanceFieldApproxAO
DistanceFieldGradient
DistanceFieldsRenderingSwitch
DistanceToNearestSurface
Divide
DotProduct
DoubleVectorParameter
DynamicParameter
Exponential
Exponential2
ExternalCodeBase
EyeAdaptation
EyeAdaptationInverse
FeatureLevelSwitch
FirstPersonOutput
FloatToUInt
Floor
Fmod
FontSample
FontSampleParameter
FontSignedDistance
Frac
Fresnel
FunctionInput
FunctionOutput
GenericConstant
GetMaterialAttributes
GIReplace
HairAttributes
HairColor
HsvToRgb
If
IfThenElse
InverseLinearInterpolate
IsFirstPerson
IsOrthographic
LandscapeGrassOutput
LandscapeLayerBlend
LandscapeLayerCoords
LandscapeLayerSample
LandscapeLayerSwitch
LandscapeLayerWeight
LandscapePhysicalMaterialOutput
LandscapeVisibilityMask
LayerStack
Length
LightmapUVs
LightmassReplace
LightVector
LinearInterpolate
LocalPosition
Logarithm
Logarithm10
Logarithm2
MakeMaterialAttributes
MapARPassthroughCameraUV
MaterialAttributeLayers
MaterialCache
MaterialFunctionCall
MaterialLayerOutput
MaterialProxyReplace
Max
MeshPaintTextureCoordinateIndex
MeshPaintTextureObject
MeshPaintTextureReplace
Min
Modulo
MotionVectorWorldOffsetOutput
Multiply
NamedReroute
NaniteReplace
NeuralPostProcessNode
Noise
Normalize
ObjectBounds
ObjectLocalBounds
ObjectOrientation
ObjectPositionWS
ObjectRadius
OneMinus
Operator
Panner
Parameter
ParticleColor
ParticleDirection
ParticleMacroUV
ParticleMotionBlurFade
ParticlePositionWS
ParticleRadius
ParticleRandom
ParticleRelativeTime
ParticleSize
ParticleSpeed
ParticleSpriteRotation
ParticleSubUV
ParticleSubUVProperties
PathTracingBufferTexture
PathTracingQualitySwitch
PathTracingRayTypeSwitch
PerInstanceCustomData
PerInstanceFadeAmount
PerInstanceRandom
PinBase
PixelDepth
PixelNormalWS
PostVolumeUserFlagTest
Power
PrecomputedAOMask
PreSkinnedLocalBounds
PreSkinnedNormal
PreSkinnedPosition
PreviousFrameSwitch
QualitySwitch
RayTracingQualitySwitch
RecordTextureStreamingInfo
ReflectionCapturePassSwitch
ReflectionVectorWS
RequiredSamplersSwitch
Reroute
RerouteBase
RgbToHsv
RotateAboutAxis
Rotator
Round
RuntimeVirtualTextureCustomData
RuntimeVirtualTextureOutput
RuntimeVirtualTextureReplace
RuntimeVirtualTextureSample
RuntimeVirtualTextureSampleParameter
SamplePhysicsField
Saturate
ScalarParameter
SceneColor
SceneDepth
SceneDepthWithoutWater
SceneTexelSize
SceneTexture
ScreenPosition
SetMaterialAttributes
ShaderStageSwitch
ShadingModel
ShadingPathSwitch
ShadowReplace
Sign
Sine
SingleLayerWaterMaterialOutput
SkyAtmosphereLightDirection
SkyAtmosphereLightIlluminance
SkyAtmosphereViewLuminance
SkyLightEnvMapSample
SmoothStep
Sobol
SparseVolumeTextureBase
SparseVolumeTextureObject
SparseVolumeTextureSample
SpeedTree
SphereMask
SphericalParticleOpacity
SquareRoot
SRGBColorToWorkingColorSpace
StaticBool
StaticBoolParameter
StaticComponentMaskParameter
StaticSwitch
StaticSwitchParameter
Step
Substrate
SubsurfaceMediumMaterialOutput
Subtract
Switch
Tangent
TangentOutput
TemporalResponsivenessOutput
TemporalSobol
TextureBase
TextureCollection
TextureCollectionParameter
TextureCoordinate
TextureObject
TextureObjectFromCollection
TextureObjectParameter
TextureProperty
TextureSample
TextureSampleParameter
TextureSampleParameter2D
TextureSampleParameter2DArray
TextureSampleParameterCube
TextureSampleParameterCubeArray
TextureSampleParameterSubUV
TextureSampleParameterVolume
ThinTranslucentMaterialOutput
Time
Transform
TransformPosition
Truncate
TruncateLWC
TwoSidedSign
UserSceneTexture
VectorNoise
VectorParameter
VertexColor
VertexInterpolator
VertexNormalWS
VertexTangentWS
ViewProperty
ViewSize
VirtualTextureFeatureSwitch
VolumetricAdvancedMaterialInput
VolumetricAdvancedMaterialOutput
WorldPosition
```

---

## Suggested Next Steps

If you continue studying this topic, the best follow-up notes would be:

1. Rebuild the recipes in Section 11 from memory
2. Turn Section 10 into a personal cheat sheet of formulas and space rules
3. Cross-read this file with `UE57_Material_Compile_Pipeline.md`
4. Cross-read this file with `UE57_Material_Performance_Playbook.md`

---

## References

- Epic documentation: Material Expressions Reference
- Epic documentation: Constant Material Expressions
- Epic documentation: Material Parameter Expressions
- Epic documentation: Vector Material Expressions
- Epic documentation: Utility Material Expressions
- Epic documentation: Depth Material Expressions
- Epic documentation: Landscape Material Expressions
- Local engine source:
  - `Engine/Source/Runtime/Engine/Public/Materials`
  - `Engine/Source/Runtime/Landscape/Classes/Materials`
