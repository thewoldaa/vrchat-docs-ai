# VRChat Optimization Deep-Dive: Avatars & Worlds (2025–2026 Edition)

*A hands-on guide to texture memory, meshes, materials, PhysBones, Quest conversion, world budgets, and profiling — verified against official VRChat creator docs and community tooling.*

**Primary sources:**
- https://creators.vrchat.com/avatars/avatar-optimizing-tips
- https://creators.vrchat.com/avatars/avatar-performance-ranking-system
- https://creators.vrchat.com/platforms/android/quest-content-optimization
- https://creators.vrchat.com/platforms/android/quest-content-limitations
- https://wiki.vrchat.com/wiki/Guides:Avatar_Performance_Ranking
- https://wiki.vrchat.com/wiki/Guides:Occlusion_Culling_and_You
- Tool docs: AAO (github.com/anatawa12/AvatarOptimizer), TexTransTool, VRCQuestTools (github.com/kurotu/VRCQuestTools), d4rkAvatarOptimizer, lilAvatarUtils, VRWorld Toolkit

---

## 0. Know Your Budget First

Everything below flows from the Performance Rank tables. Current thresholds (PC / Mobile):

| Metric | Excellent (PC/Mobile) | Good | Medium | Poor |
|---|---|---|---|---|
| Triangles | 32,000 / 7,500 | 70,000 / 10,000 | 70,000 / 15,000 | 70,000 / 20,000 |
| **Texture Memory** | **40 MB / 10 MB** | 75 MB / 18 MB | 110 MB / 25 MB | 150 MB / 40 MB |
| Skinned Meshes | 1 / 1 | 2 / 1 | 8 / 2 | 16 / 2 |
| Material Slots | 4 / 1 | 8 / 1 | 16 / 2 | 32 / 4 |
| PhysBone Components | 4 / 0 | 8 / 4 | 16 / 6 | 32 / 8 |
| PB Affected Transforms | 16 / 0 | 64 / 16 | 128 / 32 | 256 / 64 |
| PB Collision Check Count | 32 / 0 | 128 / 16 | 256 / 32 | 512 / 64 |
| Bones | 75 / 75 | 150 / 90 | 256 / 150 | 400 / 150 |
| Lights | 0 | 0 | 0 | 1 |
| Particle Systems | 0 / 0 | 4 / 0 | 8 / 0 | 16 / 2 |
| Total Particles Active | 0 | 300 | 1000 | 2500 / 200 |
| Audio Sources | 1 | 4 | 8 | 8 |

Two critical facts most creators miss:

1. **Mesh Read/Write enabled anywhere** → automatic "Very Poor," plus the triangle count is reported as disabled. This is the #1 accidental Very Poor cause.
2. **Disabled GameObjects and components still count.** That "off" outfit in your FX layer is fully counted by the rank system and (without tools) still occupies texture memory.

Quest/mobile hard caps: avatars **10 MB compressed** (aim 5–8 MB), worlds **100 MB compressed**. Mobile avatars exceeding component limits get those components stripped server-side — "Show Avatar" cannot restore them.

## 1. Texture Memory Mastery

### 1.1 Computing it exactly

Texture memory is not download size. It is the **decompressed-on-GPU size** of the texture at its max imported resolution, plus mipmaps:

```
bytes = (width × height × bits_per_pixel) / 8 × 1.333   [×1.333 = +33% for mipmaps]
```

| Format | bpp |
|---|---|
| RGBA32 (uncompressed) | 32 |
| BC7 (PC default for albedo) | 8 |
| DXT1/BC1 (PC, no alpha) | 4 |
| DXT5/BC3 (PC, alpha) | 8 |
| BC5 (PC normal maps) | 8 |
| BC4 (single channel) | 4 |
| ASTC 4×4 (Android high quality) | 8 |
| ASTC 6×6 (Android) | ~3.56 |
| ASTC 8×8 / 10×10 (Android) | 2 / 1.28 |

**Worked examples (with mipmaps):**

- 2048×2048 RGBA32: 2048² × 4 B = 16.78 MB → **22.4 MB**
- 2048×2048 BC7: **2.8 MB** — an 8× win over RGBA32
- 1024×1024 BC7: **1.4 MB**
- 2048×2048 BC5 normal: **5.6 MB** (halve to 1024 → 1.4 MB)
- 1024×1024 ASTC 6×6: **0.61 MB**; 2048² ASTC 6×6: **2.4 MB**
- 4096×4096 BC7: **11.2 MB** — a single 4K diffuse nearly burns an entire Excellent PC budget

So: **one 4K uncompressed texture ≈ the entire 40 MB Excellent PC budget**. Switching a 2048 RGBA32 to BC7 saves ~19.6 MB by itself.

### 1.2 Reading the numbers in the SDK

SDK Control Panel → Avatar Descriptor shows **"In Memory" texture size** — this is the Texture Memory metric that feeds the rank. Cross-check individual textures via Unity's texture inspector (note: inspector "Size" reflects the *currently selected build platform tab* — switch to Android/iOS tab to see ASTC numbers) or tools like lilAvatarUtils' texture report.

### 1.3 Per-platform overrides (the Android/iOS tab)

Every texture importer has a Default tab plus per-platform tabs. For PC+Quest dual-upload:

1. Default/PC: BC7 color, BC5 normals, BC4 masks.
2. **Android tab → Override → Max Size 1024, Format ASTC 6×6** (4×4 for faces/detail, 8×8 for flat color). Same for iOS tab.
3. Rebuild — the Android build's In Memory figure now reflects ASTC.

If you leave Override unchecked, Unity picks defaults (often ETC2/RGBA32 at original size) — how Quest avatars end up at 30 MB with "compressed" textures.

### 1.4 Crunch: download size ≠ memory

**Crunch only reduces download/package size. At load time the texture decompresses to its normal BC/DXT format in VRAM — crunch saves zero texture memory.** Official docs: "Crunch compression does not help with in-memory size!" Use when over the compressed cap but memory budget is fine; note crunch can break across Unity versions.

### 1.5 Texture streaming

Mip Streaming behaves inconsistently on avatars and VRChat's rank estimate does not credit it. Do not treat streaming as an optimization strategy — reduce Max Size and use proper compressed formats instead.

## 2. Mesh Optimization Workflow

### 2.1 Finding the heaviest meshes

- SDK Avatar Descriptor stats + performance report (per-renderer triangle counts).
- **AAO Avatar Report window** or lilAvatarUtils "Mesh Info" — table of all meshes, tris, blendshape counts, read/write flags.
- Target: one SkinnedMeshRenderer; PC Excellent 32k tris, mobile Excellent 7,500 (Poor 20,000).

### 2.2 Decimation strategy

- Decimate **accessories and hidden geometry first**; leave the face untouched.
- Blender Decimate (Collapse) non-destructive; 30–60% reductions usually survive visually on hair/props.
- **Remove unseen geometry before decimating seen geometry**: mouth interior, body under clothing, soles — free quality.
- Below 10k tris (mobile), retopology (not just decimation) is often required.

### 2.3 Blendshape stripping

Blendshapes cost memory + CPU even unused. Safe to remove:
- **Body mesh**: keep only what FX/menus use; all visemes/expressions live on the face mesh — strip everything else from body.
- **Keep**: 15 visemes, eye-look shapes (if blendshape mode), shapes used by animations/menus, tool-flagged referenced shapes.
- Tools: **AAO Trace and Optimize** (community standard), d4rkAvatarOptimizer, **lilAvatarUtils Fix BlendShapes**.

⚠️ Never delete blendshapes by hand without checking your animator — a deleted shape referenced by FX silently breaks toggles.

### 2.4 Read/Write, visibility, interior geometry

- **Disable Mesh Read/Write on every mesh** (import settings + particle meshes). Enabled = instant Very Poor.
- **Interior head geometry**: delete eyes-back/teeth/tongue if never needed; if kept, jaw-open-driven scale/collapse animation hides them (but they still count tris).
- Toggled-off clothing: FX should animate the **Renderer enabled flag**, not scale-to-zero (scale still costs skinning).

## 3. Material / Atlas Workflow

### 3.1 Why

Each material slot = submesh = draw call. PC Excellent ≤4 slots; mobile ≤1. Particle systems count one slot (two with trails).

### 3.2 Atlasing + merging step-by-step

**AAO route (simplest, current standard):**
1. Version-control first.
2. Add **Merge Skinned Mesh** (or **Trace and Optimize**) on the avatar root.
3. AAO material-merge / "Merge Texture" (ATX atlas) handles non-destructive atlasing for compatible materials.
4. Verify in Play mode + Av3Emulator — merged renderers share one enabled flag, so per-renderer toggles need rework.

**TexTransTool route (more control):** `TTT AtlasTexture` component → drag materials in → island padding → TTT repacks UV islands at build time. Add `TTT MaterialModifier` to unify shader properties.

### 3.3 When atlasing hurts

- Outfits that **swap textures per-state**: those textures must stay separate.
- Different tiling/offset needs, mixed normal-map needs, wildly divergent resolution requirements (face 2K vs shoes 256).
- Rule: **atlas per outfit-state group**, not the whole avatar. Over-aggressive atlasing defeats batching and occlusion.

## 4. PhysBone Optimization

### 4.1 What counts

Three metrics: **component count**, **affected transforms**, **collision check count** (transforms × colliders affecting them). Mobile Excellent allows **zero** PhysBones.

### 4.2 Reducing cost

1. **Consolidate components**: one PhysBone can drive an entire multi-chain hierarchy — convert "one component per hair strand" into one root component.
2. **Reduce affected transforms**: affect every 2nd/3rd bone in a chain; rest follow rigidly. Or merge chain bones in Blender.
3. **Reduce collision checks**: fewer colliders, per-chain collider assignment (not "affect all"), ignore lists, simple volumes over many small ones.
4. `Is Animated` off unless FX animates PB params; disable PB simulation on hidden clothing.

### 4.3 Very Poor configs

- 32+ components (one per bone — typical of auto-converted DynamicBones).
- 100+ transform hair chains with every collider "affect everything."
- Leftover colliders from deleted clothing still checking all chains.

## 5. Particle / Light / Audio Audits

**Particles** (mobile Poor caps at 2 systems / 200 active):
- Kill **particle collision** (physics per particle) — fake with stretching.
- Avoid **trails** below Poor (doubles slots + geometry).
- High-poly **mesh particles** — use quads.
- Tighten **maxParticles** to actual use (Total Active counts maxParticles, not visible).

**Lights**: PC beyond 1 light = below Excellent; **mobile strips lights entirely**. "Avoid real-time lights at all possibilities — bake lighting into textures or use emissive materials."

**Audio**: Excellent = 1 on PC; **0 on mobile (removed)**. Vorbis/mono, lower sample rate for loops.

## 6. Quest Conversion Full Checklist

**Shaders — VRCQuestTools workflow:**
1. Install VRCQuestTools via VCC.
2. `VRChat Quest Tools → Convert Material for Quest` — converts PC materials to nearest `VRChat/Mobile/*` shader.
3. Mobile restricted to **only SDK shaders**: Toon Standard, Standard Lite, Bumped Diffuse, Bumped Mapped Specular, Diffuse, Matcap Lit, Toon Lit, Particles/Additive, Particles/Multiply. Toon Standard Outline variant falls back to non-outline on mobile.
4. Bake matcaps/emission into textures or drop them; no normal map → use non-Bumped variant.
5. **Enable GPU Instancing on every material.**

**Texture overrides**: Android tab → ASTC 6×6, Max Size 1024 (2048 only for face). Mobile budgets: 10 MB Excellent / 40 MB Poor.

**Pruning**: particle systems → "PC-only" toggle or delete; remove Unity Constraints (use VRChat Constraints), Final IK, Audio Sources, Cameras, Lights, Cloth, rigidbodies from Quest build.

**Fallback testing**:
1. Build & Test Android; test on real Quest (grabpass/transparency behave differently).
2. Check shader conversion visually — mobile shaders tint diffuse by vertex colors; stray black vertex colors darken everything (fix in Blender).
3. Verify EVERY menu toggle — post-optimization breakage is the classic failure.
4. "Good" on PC is often "Poor" on mobile.

## 7. World Optimization

### 7.1 Lightmap baking (Unity 2022.3)

- **Progressive GPU** lightmapper; **Non-directional** on Quest targets.
- Lightmap Resolution low: 5–10 texels/unit indoor, lower outdoors — "even with low lightmap resolution, lighting can look very good."
- **No realtime GI** — Baked GI only; mark lights Baked/Mixed.
- Mark world geometry **Lightmap Static**; check for accidental non-static meshes (they go unlit-black).
- **Light probes** along walkable space for dynamic objects.

### 7.2 Occlusion culling

- Large solid occluders: **Occluder Static**; things hidden: **Occludee Static**. "Only big, solid things should be occluders" — not trees/fences/transparent things.
- **Occlusion Areas** over playable zones (not one giant view volume); Occlusion Portals on doors.
- Keep meshes **object-separated** so culling can hide whole buildings.
- Bake after final geometry; verify heaviest sightlines in Occlusion Preview.

### 7.3 LOD

LOD Group on dense props/trees (LOD1 ~40% tris, cull at distance); LOD + GPU instancing together for foliage. LODs multiply asset count — prioritize most-seen props.

### 7.4 Mirrors, video, transparency

- **Mirrors**: on Quest max one small mirror with limited recursive layer mask (players only) + Udon toggle; PC mirrors also toggleable.
- **Video**: one AVPro instance, restricted resolution; avoid multiple concurrent screens on Quest.
- **Transparency**: "avoid needing transparency completely — alpha fill rate is a significant performance sink for mobile GPUs." Cutout > blend; opaque > everything; beware fullscreen transparent quads (worst-case overdraw).

### 7.5 Draw calls

GPU Instancing on all materials; static batching via Static flags; atlas per logical object group while preserving culling boundaries; fewer real-time lights = fewer shadow passes.

### 7.6 Quest 100 MB cap — levers in order

1. **Textures**: Max Size downscale, ASTC block size up (4×4→6×6/8×8), delete unused.
2. **Lightmaps**: lower resolution, non-directional, delete stale LightingDataAsset + re-bake.
3. **Audio**: Vorbis, mono, lower bitrate; Compressed In Memory for long clips.
4. **Meshes**: Mesh Compression Medium; decimate; delete unreferenced assets.
5. Verify with a real Build & Publish — read Editor.log statistics; the **post-compression** figure counts against 100 MB (avatars: 10 MB).

## 8. Reading the Profiler

- **Unity Profiler** attached to a Build & Test instance: per-draw-call costs, Camera.Render overdraw.
- **Overdraw debugging**: Scene view draw-mode dropdown → "Overdraw" — bright = stacked transparent layers. Shrink quads, use cutout/additive.
- **VRAM**: Memory Profiler package snapshots by actual GPU footprint.
- **Test loop**: ClientSim (worlds) + Av3Emulator (avatars) — run every toggle after each optimization pass.

## 9. Decision Trees

### Tree A: "My avatar is Poor — check first?"

```
Is any mesh Read/Write enabled?
├─ YES → disable on all meshes/particle meshes. Done (#1 cause).
└─ NO ↓
Texture Memory over budget? (PC Poor = 150 MB)
├─ YES → Max Size 1024–2048, BC7/BC5/BC4 on PC tab
│         Android tab Override → ASTC 6×6
│         strip unused outfit textures (unused toggles still count)
│         atlas AFTER formats are right, not before
└─ NO ↓
Triangles > 70,000? (mobile > 20,000)
├─ YES → delete interior/hidden geometry; decimate accessories; merge meshes
└─ NO ↓
PhysBones: Affected Transforms > 256 (PC) or Collision Checks > 512?
├─ YES → consolidate to one component per chain-group; prune colliders;
│         restrict per-chain targets
└─ NO ↓
Material Slots > 32 (PC) / 4 (mobile)?
├─ YES → atlas + merge (AAO/TexTransTool); check particle slot double-count
└─ NO ↓
Particles: trails/collision on, maxParticles huge, >16 systems?
├─ YES → tighten maxParticles, kill collision/trails
└─ NO ↓
Bones > 400, Animators > 32, Contacts > 32?
└─ YES → strip unused constraint chains/contacts; merge animators
```

### Tree B: "My world is over 100 MB on Quest — what do I cut?"

```
Rank asset types by size (Editor.log build statistics / VRWorld Toolkit Build Report).

Textures dominate?
├─ YES → any >1024? Max Size down; ASTC 4×4 → 6×6/8×8; delete unused;
│         lightmap branch next
└─ STILL OVER ↓
Lightmaps huge?
├─ YES → lower resolution; Non-directional; delete stale LightingDataAsset, re-bake
└─ STILL OVER ↓
Audio big?
├─ YES → Vorbis, mono, lower bitrate; Compressed In Memory
└─ STILL OVER ↓
Meshes big?
├─ YES → Mesh Compression Medium; decimate; delete unreferenced assets
└─ STILL OVER ↓
Last resorts: re-author hero textures at 512; fewer reflection probes; cut a mirror;
split decorative content behind Udon toggle (still counts toward limits — prefer cutting)
```

## Final Priorities

If you only do five things: (1) kill Read/Write, (2) fix texture formats per-platform + Max Sizes, (3) one skinned mesh + stripped blendshapes via AAO, (4) consolidate PhysBones + prune collider targets, (5) bake world lighting + occlusion, minimal transparency on Quest.

*All rank tables and limits cited from creators.vrchat.com as retrieved 2025–2026; limits change — re-check the Performance Ranking System page before uploading.*
