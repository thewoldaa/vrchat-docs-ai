# The 3D Model Creation Pipeline for VR / Social Platforms (VRChat Focus)

**Research report — compiled October 2026 from 2025–2026 sources**

---

## 1. 3D Fundamentals

### 1.1 Meshes, vertices, edges, faces

A mesh is a collection of polygons describing a surface. The atomic elements:

- **Vertex** — a point in 3D space `(x, y, z)`, usually with extra attributes: normals, UVs, tangents, vertex colors, bone weights. GPUs render vertices, not "polygons."
- **Edge** — a line connecting two vertices.
- **Face** — a surface bounded by edges. Quads (4 sides) are the artist's standard for modeling; **triangles** are what GPUs actually rasterize. Every quad = 2 triangles. All engines report triangle counts.
- **N-gons** (5+ sides) are fine for modeling but must be triangulated before export; they can cause shading/UV errors.

**Triangulation:** Blender keeps quads natively and triangulates on export (or via the *Triangulate* modifier). Rule of thumb: keep quads through modeling/sculpting for clean edge-loops and deformation; triangulate only as the final step. Edge-flow around joints (shoulders, hips, knees) should follow muscle lines so deformation bends cleanly.

**Polycount conventions (2025–2026):**

| Asset type | Triangle budget |
|---|---|
| VRChat avatar (PC, "Excellent") | ≤ 32,000 tris |
| VRChat avatar (PC, any rank) | ≤ 70,000 tris |
| VRChat avatar (Quest/Android/iOS, Excellent) | ≤ 7,500 tris |
| VRChat avatar (Mobile, max usable) | 20,000 tris |
| AAA game hero character | 80,000–150,000 tris |
| Background NPC | 5,000–20,000 tris |
| Prop | 500–10,000 tris |

### 1.2 UV mapping and atlases

UV unwrapping flattens the 3D surface onto a 2D plane so a texture can be painted onto it. Key concepts:

- **Seams** — cuts where the unwrap "peels" the mesh. Place seams in hidden areas (underarms, inside clothing, back of head).
- **Texel density** — pixels per meter of surface. Consistent density across an avatar prevents one part looking blurry while another is sharp. For a VRChat avatar at 2048 px covering a ~1.7 m character: 2048 / ~2.5 m of unwrapped area ≈ 800 px/m — good target; faces typically get 2–4× the body's density (a dedicated face UV island).
- **UV islands** — separate connected patches. Keep 4–8 px padding between islands so mipmapping and bilinear filtering don't bleed between them.
- **Atlas** — multiple materials' textures packed into one texture so multiple meshes can share one material. This is the single most important optimization for VRChat (see §5).

### 1.3 Materials and PBR

**PBR (Physically Based Rendering)** models light interaction with two workflows:

- **Metallic/Roughness** (Standard, Unity URP/Lit): `Albedo (base color) + Metallic (0–1 mask) + Smoothness (1−Roughness) + Normal + AO + Height + Emission`
- **Specular/Gloss** (legacy): direct specular color + glossiness.

Parameter-by-parameter:

| Map | Contents | Notes |
|---|---|---|
| **Albedo/BaseColor** | Raw color, no lighting/shadows baked in | sRGB; no ambient occlusion painted in (that's AO's job) |
| **Normal** | Surface angle perturbation (RGB encoded XYZ) | **Linear data — never mark sRGB.** Tangent-space is standard; object-space rare |
| **Metallic** | Grayscale mask: white = metal, black = dielectric | Linear; often packed with other masks |
| **Smoothness/Roughness** | Grayscale microsurface | Stored in albedo's alpha channel (specular workflow) or metallic's alpha (metallic workflow) in Unity |
| **AO (Ambient Occlusion)** | Grayscale contact shadowing | Linear; multiplies indirect light only |
| **Height** | Grayscale displacement info for parallax/tessellation | Optional; expensive in VR |
| **Emission** | Self-illuminated color + intensity mask | Makes things glow; counts against bloom budgets |

**Channel packing:** pack grayscale masks into one RGBA texture — e.g., Unity's HDRP/URP *Mask Map* = (R: Metallic, G: AO, B: Detail mask, A: Smoothness). This cuts 3–4 textures into 1 with zero quality loss and is standard practice for hitting texture-memory budgets.

### 1.4 Texture formats

| Format | Channels | Use |
|---|---|---|
| **PNG** | RGBA 8/16-bit | Lossless, universal default; no layers |
| **PSD** | Layered | Photoshop source; Unity imports PSD directly (flattened) |
| **TGA** | RGBA 8-bit | Legacy game-industry standard; smaller than PNG for 8-bit |
| **EXR** | Half/float HDR | Lighting data, normal maps at high precision, Substance output |
| **JPEG/WebP** | Lossy | Never for albedo in a production pipeline (artifacts + no alpha) |

**GPU texture memory & compression** — the file format is irrelevant once imported; the GPU format is what costs memory:

| GPU format | Bits/pixel | 1024² | 2048² | 2048² + mipmaps |
|---|---|---|---|---|
| RGBA32 (uncompressed) | 32 | 4 MB | 16 MB | 21.3 MB |
| DXT1 / BC1 (RGB) | 4 | 0.5 MB | 2 MB | 2.7 MB |
| DXT5 / BC3 (RGBA) | 8 | 1 MB | 4 MB | 5.3 MB |
| BC7 (high-quality RGBA) | 8 | 1 MB | 4 MB | 5.3 MB |
| BC5 (normal maps, 2-ch) | 8 | 1 MB | 4 MB | 5.3 MB |
| BC4 (single mask) | 4 | 0.5 MB | 2 MB | 2.7 MB |
| BC6H (HDR) | 8 | 1 MB | 4 MB | 5.3 MB |
| ASTC 4×4 (mobile best) | 8 | 1 MB | 4 MB | 5.3 MB |
| ASTC 6×6 | 3.56 | 0.45 MB | 1.8 MB | 2.4 MB |
| ASTC 8×8 | 2.0 | 0.25 MB | 1 MB | 1.3 MB |

**The memory formula:**

```
memory = Σ over each mip level: ceil(width/blk) × ceil(height/blk) × bitsPerBlock / 8
```

Mipmaps add ~33% (the sum 1 + ¼ + 1/16 + … = 4/3). Block-compressed textures require dimensions divisible by 4 — a 1023×1023 texture silently falls back to uncompressed RGBA32 (8× memory). If Read/Write is enabled, memory doubles (CPU + GPU copies).

Platform defaults: **Desktop (DX11+)** → BC7 for RGBA, DXT1 for RGB, BC5 for normals. **Android/iOS (Quest, phones)** → ASTC (4×4 for faces/detail, 6×6–8×8 elsewhere); Quest does *not* support BC formats, so PC-compressed textures decompress to RGBA32 on mobile if not overridden.

### 1.5 LODs

Level of Detail: a chain of progressively simpler meshes (LOD0 full detail → LOD1 ~50% tris → LOD2 ~25% → LOD3/cull). The engine swaps them by on-screen size. **VRChat avatars don't use LODs** (you're always near the viewer), but worlds do. Useful rule for LOD steps: each level ≈ 50% triangles, textures can drop a power of two. LOD generation tools: Blender Decimate modifier, Simplygon, Unity's own (none built-in), Blender's "Decimate → Collapse."

---

## 2. Blender (as of Blender 4.x, 2025–2026)

### 2.1 Modeling

- **Workflow:** blockout → box-model or sculpt → retopology → UV → bake → texture.
- **Key tools:** Extrude (`E`), Inset (`I`), Bevel (`Ctrl+B`), Loop Cuts (`Ctrl+R`), Knife (`K`), proportional editing (`O`).
- **Modifiers (non-destructive stack):** *Mirror* (symmetry — always use it), *Subdivision Surface* (Catmull-Clark smoothing; keep for render meshes, NOT for game/VR export — export the base cage), *Solidify* (clothing thickness), *Decimate* (polycount reduction, ratio 0.5 halves tris roughly), *Triangulate*, *Weighted Normal*, *Data Transfer* (copy weights/normals between meshes — essential for outfit workflows).
- **Sculpting:** Dyntopo or multires; typical sculpt 2–20 M tris. Sculpts are never exported directly — retopologize to 10–40k tris for an avatar and bake the detail into normal maps.

### 2.2 Rigging

- **Armature** = bone hierarchy. Each bone has a head/tail; rotation happens around the head. Use **X-Axis Mirror** for symmetric rigging.
- **Humanoid rig:** spine chain (Hips → Spine → Chest → Neck → Head), 2–3 bones per limb segment, fingers (3 joints × 3 bones + thumb), eye bones, optional breast/hair/skirt/sleeve bones for physics.
- **Weight painting:** each vertex gets weights (0.0–1.0) per bone; weights sum to 1.0. Each vertex is typically influenced by up to 4 bones in Unity. Common pitfalls: bleeding weights across seams (fix with Weight smoothing / "Clean" at 0.01 threshold), painting clothing that needs *Data Transfer* from the body instead.
- **Bone constraints:** IK (`IK constraint`, 2-bone IK for arms/legs), *Copy Rotation* (eyes), *Damped Track*, *Child Of*. For VRChat, constraints are baked at export — the runtime uses Unity/VRChat constraints instead.

### 2.3 Shape keys (blendshapes)

Per-vertex morph targets stored as offsets from a basis mesh. Used for:

- **Facial expressions:** smiling, brow raise, blink (eye closed L/R)
- **Visemes** (lip sync): `vrc.v_aa, vrc.v_ss, vrc.v_dd, vrc.v_ff, vrc.v_th, vrc.v_ch, vrc.v_pp, vrc.v_kk, vrc.v_nn, vrc.v_rr, vrc.v_sil, vrc.v_ih, vrc.v_e, vrc.v_oh, vrc.v_ou` — VRChat uses a 15-viseme set derived from the A–I–O–U–E scheme (see §8.5)
- Corrective shapes for deformation

Sculpt each shape on a duplicated mesh, import as shape key. Name them with the VRChat convention (`vrc.blink`, `vrc.v_aa`, etc.) so the SDK auto-detects. Keep shape-key count reasonable — each adds a vertex-offset buffer (memory scales with mesh vertex count × shape count; a 30k-vertex mesh with 40 shapes stores ~40 × 30k × 3 floats ≈ 14 MB of offsets, though Unity compresses on import).

### 2.4 Export

**FBX export (the VRChat path):** `File → Export → FBX (.fbx)`
- Path Mode: `Copy` + **Embed Textures** checkbox (or leave textures out and import to Unity separately — commonly preferred)
- Limit to: Selected Objects
- Transform: Scale **1.0** (not 0.01!), Forward = **-Z**, Up = **Y** — or use the "experimental" Apply Transform option; the safest workflow is Scale 1.0 with default axes and let Unity's FBX import fix units
- Apply all modifiers (armature modifier exports as skin); triangulate or leave to Unity

**glTF 2.0 export:** `File → Export → glTF 2.0` — supports PBR (metallic/roughness), morph targets, skinning, KHR_materials extensions. Best for web/AR/VRM pipelines; **not** used directly by VRChat (which is Unity/FBX-based).

**Scale discipline:** Blender default unit = 1.0 m. A 1.7 m human is 1.7 Blender units. If your character is 170 units tall, everything downstream (physics, camera, VR tracking) breaks. Verify with `N-panel → Dimensions` showing meters.

### 2.5 VRM add-on

The **VRM Add-on for Blender** imports and exports `.vrm` (both 0.x and 1.0) directly:

1. Import the VRM (armature + MToon materials come in as Blender nodes)
2. Edit mesh/materials
3. Export → set VRM version 0.x or 1.0, map humanoid bones in the panel, set expressions (BlendShape → Expression mapping), spring bone colliders, first-person mesh annotation
4. Export validates bone naming against the humanoid map

### 2.6 MMD / PMX import

The **MMD Tools** add-on imports `.pmx` (Polygon Movie Maker eXtended) models — the MMD community format:

- Imports mesh, materials (with toon/sphere textures), armature (standard MMD rig with IK bones, ~130+ bones), morphs (vertex/group/bone/material/UV), rigid bodies & joints (physics)
- Typical flow for VRChat: import PMX → delete Japanese-only morphs you don't need → fix weights → export FBX. Watch for: MMD's inverted/left-right naming, texture path encoding (Shift-JIS), and enormous polycounts (MMD models often 100k–400k tris).
- Export back to PMX for MMD is supported but lossy.

---

## 3. File Formats

| Format | Structure | Units/scale | Pros | Cons | Best for |
|---|---|---|---|---|---|
| **FBX** (Autodesk, proprietary) | Binary or ASCII scene graph + geometry + skin + morphs + animation + embedded textures | **FBX internal unit = cm**; Blender-Unity roundtrip introduces 100× scale factors (Blender m → FBX cm → Unity re-scales). Unity's importer auto-applies `File Scale 0.01` | Full-featured: skinning, morphs, multiple UV sets, animation; the de-facto DCC→Unity path; VRChat standard | Proprietary black box; version incompatibilities; axis/scale confusion (Y-up vs Z-up); ASCII FBX poorly supported | **VRChat avatars**, Unity, Unreal |
| **glTF 2.0 / GLB** (Khronos) | JSON + binary buffer (GLB = single binary container) | Meters, Y-up, right-handed, +Z forward (VRM 1.0 changed to z+ forward) | Open standard; "JPEG of 3D"; PBR built-in; extensions (KHR_materials_unlit, VRM sits on top); web/AR native | Animation features limited (no IK constraints natively) | Web, AR, VRM, Godot, three.js, Cluster |
| **VRM 0.x** | glTF 2.0 + `VRM` extension: humanoid bone map, BlendShapeProxy (expressions), first-person, springBone, MToon shader | Meters; **z− forward** | Humanoid-avatar standard since 2018; toon shading via MToon; works across VRM apps | Deprecated (2022) but still widely supported; expression system limitations | Legacy VRM apps, VRoid Hub |
| **VRM 1.0** | glTF + `VRMC_vrm`, `VRMC_springBone`, `VRMC_node_constraint`, `VRMC_materials_mtoon` (MToon 1.0) | Meters; **z+ forward** (breaking change); "expressions" replaces "blendshapes" naming | Fixes 0.x issues: capsule spring-bone colliders, per-joint spring tuning, matcap masking in MToon, node constraints, HDR emissive multiplier | Some apps still 0.x-only (VRoid exports both) | Cluster, VRoid Studio 2023+, modern VRM ecosystem |
| **PMX** (MMD) | Binary, Japanese community format; mesh + materials + morphs + rigid-body physics | Varies, ~ 8 units = 1 m historically (modeler-dependent) | Rich morph/physics ecosystem; massive free-model library | Japanese-only tooling; no PBR; scale chaos; never use as an interchange format | MMD, dance animation |
| **OBJ** | Plain text; geometry + UVs + MTL material refs | Unitless | Dead simple, universal, lossless geometry | **No skinning, no morphs, no animation, no PBR**; one UV set; huge text files | Static mesh interchange, sculpting handoff |

**Pipeline verdict:** Blender → FBX → Unity for VRChat; Blender → glTF/GLB → VRM for VRM platforms; OBJ only for static props.

---

## 4. Texturing

### 4.1 Substance Painter workflow

1. **Export from Blender:** low-poly FBX (with UVs) and high-poly FBX (or separate high-poly pieces).
2. **Import low-poly** into Painter; assign the high-poly in the *Mesh Map baking* settings.
3. **Bake:** Normal (tangent, MikkTSpace — matches Blender's default), AO, Curvature, Position, Thickness, ID (from material IDs). Baking resolution 2048–4096; cage/max frontal ray distance tuned to avoid projection errors.
4. **Texturing:** smart materials + procedural generators (grunge, edge wear driven by curvature/AO) + hand paint layers. Non-destructive layer stack.
5. **Channel setup:** BaseColor (sRGB), Normal (DirectX vs OpenGL — Unity/Blender = **OpenGL/Y+**, DirectX = flipped green), Metallic, Roughness, Emissive, AO, Height.
6. **Export:** texture set → choose preset (Unity HD Render Pipeline / URP / custom channel-packed). Painter can pack AO/Rough/Metal into one RGB PNG automatically.

**Normal map baking gotchas:** match tangent space (MikkTSpace everywhere), match flip convention (Unity/Blender = OpenGL), always bake at a power-of-two, and check baking with a preview shader before painting.

### 4.2 Texture sizes & atlasing for avatars

- **Face:** dedicated 2048 (or 1024 for mobile) — the face gets the most close-up scrutiny.
- **Body/hair/outfit:** 1024–2048 each, or atlased to a shared 2048/4096.
- **4096** is the practical ceiling and costs 32 MB RGBA / 21 MB BC7 + mips for a single texture — almost never justified on an avatar; on mobile it's disqualifying.
- **Atlas strategy:** UV-unwrap all outfit pieces + body into one shared 0–1 UV space (Blender: select all → follow active quads, or use an atlas packer like UVPackmaster / Texel Density tools), then one material covers everything.

### 4.3 VRAM math — worked examples

Formula: `Σ mips: ceil(w/blk)·ceil(h/blk) · bitsPerBlock/8`, mips ≈ ×4/3.

- **2048² BC7:** 2048² = 4,194,304 px → 262,144 blocks (4×4) × 1 byte = **4 MB**; ×4/3 = **5.33 MB**
- **2048² DXT1:** **2 MB** (+mips = 2.67 MB)
- **2048² RGBA32:** **16 MB** (+mips = 21.33 MB) — the "accident" case
- **1024² ASTC 6×6:** ≈ **0.08 MB** (+mips ≈ 0.10 MB)

**Avatar budget example (PC "Good" = 75 MB):** face 2048 BC7 (5.3) + body 2048 BC7 (5.3) + hair 2048 (5.3) + outfit 4096 BC7 (21.3) + normal 2048 BC5 (5.3) + masks 1024 (1.3) + 4 expression textures 1024 (4×1.3) ≈ **51.8 MB** ✓. Swap the 4096 to RGBA32 uncompressed and you're at 114 MB → rank drops to Medium.

**VRChat counts "Texture Memory"** — the sum of all textures' GPU footprint including mipmaps and per-platform compression, as displayed in the SDK's *Avatar Performance* window (PC Excellent ≤ 40 MB, Good ≤ 75, Medium ≤ 110, Poor ≤ 150; Mobile: 10 / 18 / 25 / 40 MB).

---

## 5. Optimization Theory

### 5.1 Draw calls

One draw call = one material on one (sub)mesh rendered in one pass. Cost: CPU driver overhead per call (state changes, sorting). Typical desktop budget thousands of calls; Quest/mobile much lower. An avatar's draw calls ≈ **material slots × shader passes** (e.g., an outline pass doubles it).

### 5.2 Batching, combining, material merging

- **Static batching:** combines static geometry at build time (worlds, not avatars).
- **Dynamic batching:** per-frame vertex transform combining — largely obsolete.
- **SRP Batcher (URP):** batches by *shader variant* rather than geometry — the big URP win; avatars sharing shaders batch well.
- **GPU instancing:** many copies of the same mesh+material (props, grass).
- **Mesh combining:** physically merge meshes (Blender `Ctrl+J`, or Unity's **Avatar Optimizer / Trace and Optimize**, d4rkAvatarOptimizer). VRChat guideline: **1 SkinnedMeshRenderer + ≤3 static mesh renderers** for Excellent.
- **Material slot merging:** identical material on multiple submeshes still costs separate draw calls — either merge the meshes or atlas the textures so one material covers all. VRChat Excellent requires **≤4 material slots**.

### 5.3 Triangle budget

The vertex stage cost is vertices × (skinning + shading). Quest/mobile GPUs (Adreno, up to ~120k tris/frame comfortable across a whole scene) mean avatar triangles matter enormously in crowded instances. Decimation order of operations: decimate → check silhouette → fix UV seams → re-bake normals from the original high-poly if needed.

### 5.4 Bone budget

Each animated bone = per-frame matrix update + per-vertex 4-bone skinning read. VRChat PC: **Excellent ≤75 bones** — that's the standard humanoid (Hips→head ~17 + 2×(arm 8 + leg 8) + 30 finger bones ≈ 70), meaning Excellent leaves almost no room for extra bones; Good (≤150) is the realistic target with hair/skirt physics. Mobile: 75/90/150/150.

### 5.5 Overdraw

Transparent pixels still cost fill rate (each overlapping transparent layer shades every pixel). Fixes: fewer/smaller transparent particles, cutout (alpha-test) instead of alpha-blend for hair edges where possible, smaller particle meshes.

### 5.6 Mipmaps & anisotropy

- **Mipmaps:** precomputed halved chains; required for 3D content (prevents shimmering, improves cache coherence). Cost +33% memory — worth it everywhere except UI.
- **Anisotropic filtering:** keeps textures sharp at grazing angles (floor textures, long hair). Costs bandwidth; aniso 2–4 on avatars is fine, avoid 16 on mobile.

### 5.7 VRChat-specific rank targets (summary table)

| Metric | PC Excellent | PC Good | Mobile Excellent | Mobile Good |
|---|---|---|---|---|
| Triangles | 32,000 | 70,000 | 7,500 | 10,000 |
| Texture Memory | 40 MB | 75 MB | 10 MB | 18 MB |
| Skinned Meshes | 1 | 2 | 1 | 1 |
| Material Slots | 4 | 8 | 1 | 1 |
| Bones | 75 | 150 | 75 | 90 |
| PhysBones Components | 4 | 8 | 0 | 4 |
| Animators | 1 | 4 | 1 | 1 |
| Lights | 0 | 0 | 0 | 0 |

Very Poor = anything beyond Poor; hidden by default (mobile can never fully show Very Poor). Disabling Mesh Read/Write anywhere = instant Very Poor.

---

## 6. Shaders (Unity)

### 6.1 ShaderLab & Shader Graph

- **ShaderLab:** Unity's declarative shader format — `Shader` → `Properties` (exposed uniforms) → `SubShader` → `Pass` with HLSL `CGPROGRAM`/`HLSLPROGRAM` blocks. Built-in pipeline uses `CGPROGRAM`; URP uses `HLSLPROGRAM` with includes from the URP ShaderLibrary.
- **Shader Graph:** node-based, produces URP/HDRP shaders. Nodes: *Sample Texture 2D*, *Lerp*, *Step/Smoothstep*, *Fresnel Effect*, *Time*, *Position/Normal Vector* — enough to build stylized toon, dissolve, outline, and hologram effects without code.
- **Vertex/fragment vs surface:** URP = explicit vertex+fragment; the old Built-in "Surface Shaders" don't exist in URP.

### 6.2 URP vs Built-in

| | Built-in (BiRP) | URP |
|---|---|---|
| Architecture | Fixed legacy pipeline | Scriptable Renderer, RenderGraph (URP 17+/Unity 6) |
| VRChat | **VRChat PC & Quest currently run Built-in-style pipelines** (VRChat SDK targets the Unity version VRChat ships with — Unity 2022.3, Built-in for avatars) | Not usable for VRChat avatars today; standard for new standalone games |
| Performance features | None | SRP Batcher, RenderGraph, Forward+, Vulkan/Metal |
| Migration | — | Use Render Pipeline Converter; "pink materials" = Built-in shader in URP project |

Note: VRChat's own SDK pins the pipeline, so VRChat creators optimize within the Built-in feature set.

### 6.3 Toon shading

Cel-shading formula: `lit = step(smoothstep(N·L, threshold), ...)` — quantize `N·L` with a ramp texture (2–4 bands) sampled by `saturate(dot(N, L) * 0.5 + 0.5)`. **lilToon** (free, the 2025–2026 VRChat community standard), **Poiyomi Toon** (feature-rich, animated effects), and **MToon** (VRM standard) all implement: quantized diffuse ramp, rim light, matcap, outline, emission animation, audio-link reactive params.

### 6.4 Transparency issues

- **Alpha cutout (alpha-test):** binary on/off via `clip(alpha − cutoff)`. No sorting problems; hard edges. Use for hair cards.
- **Alpha blend:** `SrcAlpha, OneMinusSrcAlpha`; requires back-to-front sorting which fails between meshes; z-fighting with outlines; costs overdraw. Depth-write is typically off → self-sorting artifacts inside one mesh.
- **Best practice:** cutout for most things; additive blend for glow; avoid multilayer transparency on mobile.

### 6.5 Outline shaders

Three techniques: (1) **inverted-hull** — duplicate mesh, flip normals, expand along normals, render backfaces in outline color (cheapest, per-material second pass; lilToon/MToon default); (2) **post-process depth/normals edge detection** (worlds only); (3) **vertex-shader screen-space extrusion** for constant screen thickness.

### 6.6 Matcap

Material Capture: a sphere-lit lookup texture sampled by view-space normal `uv = N_view.xy × 0.5 + 0.5` — zero lights, zero PBR, instant "shiny anime" shading. Extremely cheap; standard for hair shine and metals on avatars. MToon 1.0 added matcap masking.

---

## 7. Animation Basics

- **Skeletal animation:** animation channels drive bone transforms (usually rotation only for humanoid limbs); skinned vertices follow bones via weights. Keyframe curves interpolate (linear, Bezier, quaternion slerp).
- **Humanoid vs Generic (Unity):** *Humanoid* maps the rig onto Unity's Mecanim humanoid avatar (Hips, Spine, Head, LeftUpperArm…) enabling **retargeting** — one animation file works on any humanoid. *Generic* plays raw bone transforms (faster, needed for tails/wings/props). VRChat avatars are Humanoid; animation layers (Base, Gesture, Action, FX, Sitting, T-Pose, IK Pose) each have masks (e.g., FX layer masks to right hand + face).
- **Root motion:** hips-driven locomotion transferred by the engine (`Apply Root Motion`). VRChat **disables root motion** — locomotion is driven by tracking/play space, and the avatar plays in-place animations.
- **Animation clips & curves:** a clip is sampled property curves (transform TRS, material floats `_Color`, blendshape weights `Face.Mesh.vrc.blink`, slider-driven Animator parameters). Blend trees blend clips by 1D/2D parameters (e.g., viseme or velocity). Curves can drive anything animatable, which is how VRChat toggle menus work (Animator int/float parameters ↔ object toggles via animated GameObject active states).
- **Frame rates:** key at 30–60 fps; bake IK before export; strip scale curves (rotation-only curves are smaller and safer).

---

## 8. Character Design Conventions for VR

### 8.1 T-pose / A-pose

The **bind pose** — the pose in which the mesh is skinned to the armature, and the pose Unity's Humanoid mapping requires at the "T-Pose" calibration. T-pose (arms straight out) is required for VRChat uploads; A-pose (arms ~45° down) is more natural for skinning the shoulders but many pipelines (VRoid, VRM standard) mandate T. VRM requires exact T-pose with palms facing down and fingers spread. VRChat's SDK animates the avatar to T-pose briefly for calibration. Practical: model and rig in A-pose if you prefer, but ensure the final bind pose is T.

### 8.2 Proportions & height scaling

- 1 Unity unit = 1 m. Average real human: 1.6–1.8 m. VRChat's default avatar height is ~1.6–1.7 m; users can scale ±100% in-game.
- **Eyes = camera height in VR:** eye position (derived from the head bone + eye offsets) determines the player's view height; a 1.2 m avatar genuinely sees the world from 1.2 m.
- Stylized proportions (anime 6–7 heads tall, chibi 3 heads) all work — but the **arms must reach** naturally to hand-tracking/tracker positions, so extreme proportions may need IK offset configuration (VRChat's "Shoulder width / Arm span" calibration adjusts).
- Keep the mesh's real-world metric size correct in Blender (1.7 m tall), or fix in Unity (`Model → Scale Factor`, or the VRChat SDK's scale validation warnings).

### 8.3 Tracking points (Avatar 3.0)

- **Full-body tracking (FBT):** head, chest/hip trackers (3-point) or full body (Vive Ultimate/Tracker 3.0, Tundra, SlimeVR): Head, Neck (optional), Chest, Hip, Knees, Feet — mapped to Humanoid bones Head, Chest, Hips, Left/RightLowerLeg, Left/RightFoot; IK drives the rest.
- **Hands:** controllers → LeftHand/RightHand bones via wrist rotation mapping + finger tracking (Index, Quest 3 controller capacitive, or hand tracking) driving finger bones via blendshape/rotation curves.
- **Eyes:** LookAt via VRM lookAt (bone or expression type) / VRChat eye-look IK — needs eye bones or blendshape-based gaze.
- **Lip sync (visemes):** VRChat drives 15 blendshapes `vrc.v_aa` (A), `vrc.v_ss`, `vrc.v_dd`, `vrc.v_ff`, `vrc.v_th`, `vrc.v_ch`, `vrc.v_pp`, `vrc.v_kk`, `vrc.v_nn`, `vrc.v_rr`, `vrc.v_ih` (E/I), `vrc.v_e` (E), `vrc.v_oh` (O), `vrc.v_ou` (U), `vrc.v_sil` (silence) — derived from the classic **A-I-O-U-E** phoneme set. Set them in the SDK's Lip Sync menu (Shape Key mode recommended over jaw-bone mode).
- **Blinking:** `vrc.blink` (+ `vrc.blink_l`/`vrc.blink_r`) driven by auto-blink or eye tracking.
- **Expressions:** `vrc.happy, vrc.angry, vrc.sad, vrc.relaxed, vrc.surprised, vrc.excited` mapped in the SDK (matching VRM's standard expression presets `happy/angry/sad/relaxed/surprised/aa/ih/ou/ee/oh/blink/blinkLeft/blinkRight/lookUp/lookDown/lookLeft/lookRight/neutral`).

---

## 9. End-to-end VRChat Workflow (step-by-step)

1. **Blender:** block out → model/sculpt → retopo → 30–60k tris (mobile: ≤10k)
2. **UV:** unwrap, set texel density, atlas planned (≤4 material slots target)
3. **Rig:** humanoid armature (T-pose), weights, corrective shape keys, `vrc.*` morphs named
4. **Substance Painter:** bake high→low, texture, export channel-packed 1024–2048 BC-bound textures
5. **Export FBX:** scale 1.0, -Z forward, Y up, apply modifiers
6. **Unity (VRChat Creator Companion + SDK3):** import FBX → set Model scale → Humanoid rig mapping (check T-pose preview) → assemble avatar prefab: VRCAvatarDescriptor, one SkinnedMesh, lilToon/Poiyomi materials, PhysBones (hair/skirt/collar), VRCExpressionMenus/Parameters (toggles, sliders), animation layers
7. **Optimize:** check SDK Performance panel → target Good (PC) / Medium (mobile): merge meshes/materials, compress textures to max size per platform override (ASTC on Android tab!), strip unused blendshapes (AAO/Fix-Blendshape tools), disable read/write, reduce PhysBones colliders
8. **Test:** local test in VRChat, check visemes, blink, eye-look, first-person camera (hide head/hair interior via mesh annotation or scale-0 animation), then upload (trust rank permitting)

---

## Sources

- VRChat Performance Ranks — https://creators.vrchat.com/avatars/avatar-performance-ranking-system/
- VRChat Avatar Optimization Tips — https://docs.vrchat.com/docs/avatar-optimizing-tips
- VRChat Configuration (uncompressed size limits) — https://docs.vrchat.com/docs/vrchat-configuration-window
- Unity GPU texture formats reference — https://docs.unity3d.com/Manual/texture-choose-format-by-platform.html
- Texture memory calculation — https://tech.sumzap.co.jp/entry/texture-memory-analyzation
- ASTC compression — https://developer.nvidia.com/astc-texture-compression-for-game-assets
- VRM 1.0 spec — https://github.com/vrm-c/vrm-specification/blob/master/specification/VRMC_vrm-1.0/README.md ; https://vrm.dev/
- VRoid Studio VRM export — https://vroid.pixiv.help/hc/en-us/
- Blender manual — https://docs.blender.org/manual/en/latest/
- MMD Tools add-on — https://github.com/UuuNyaa/blender_mmd_tools
- VRM Add-on for Blender — https://github.com/saturday06/VRM-Addon-for-Blender
- lilToon — https://github.com/lilxyzw/lilToon ; Poiyomi — https://github.com/poiyomi/PoiyomiToonShader
- Khronos glTF 2.0 spec — https://github.com/KhronosGroup/glTF/tree/master/specification/2.0
- Avatar Optimizer (anatawa12) — https://github.com/anatawa12/AvatarOptimizer

**Caveats:** Performance-rank numbers verified against live docs; pipeline specifics (VRChat's Unity version, any future URP adoption) change with SDK releases — verify against SDK release notes when it matters.
