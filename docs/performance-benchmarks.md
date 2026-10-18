# VRChat Avatar Performance Benchmarks & Measurement — State of the Art, Late 2025 / Early 2026

*Numbers from published community benchmarks are marked **[MEASURED]** with hardware context; derived figures marked **[ESTIMATED]**. VRChat's performance rank system is static analysis only — "Very Poor" and "actually expensive" are correlated but not identical.*

---

## 1. How to measure avatar performance

**Unity Profiler** — the workhorse for CPU frame time. VRChat's Android optimization docs recommend it; PC numbers won't match Quest but relative attribution ("this component eats 3 ms") transfers. Caveats:
- VRChat's runtime differs from raw Unity — PhysBones behave differently in-game; jellejurre's benchmark harness (github.com/jellejurre/UnityBenchmark) was run *inside VRChat* for PhysBone data.
- With GPU skinning on (default, dramatically improved in Unity 2021.2+ where skinning runs as compute shaders), most skinning cost moves off the CPU thread — the CPU profiler **under-reports** it. GPU cost needs a frame capture.
- **Measured quirk:** the presence of **any** state behaviour on any layer of any animator controller increases runtime of all controllers on that avatar by **~50%** (jellejurre). Since default Action/Gesture layers have behaviours, essentially every avatar pays this.

**RenderDoc / Nsight Graphics** — GPU-side truth: vertex cost, skinning compute dispatches, blendshape buffers, overdraw. Overdraw isn't exposed anywhere; infer via RenderDoc capture or Meta's "render fewer pixels" method (drop render scale to ~0 — if frame time collapses, you're fill/fragment bound).

**VRAM measurement** — no single authoritative tool. The rank system's Texture Memory is an *estimate* (40/75/110/150 MB PC tiers). Community tools (Thryrallo's VRAM Checker, lilAvatarUtils) compute from resolution/format, but d4rkc0d3r showed estimates can overshoot: Thry's calculator said 49.27 MiB for a mesh whose measured GPU buffers totaled 26.16 MiB (tools assume uncompressed storage; blendshape deltas are deduplicated on GPU). **[MEASURED discrepancy, one data point]**

## 2. Real-world cost data of avatar features

Best public dataset: **jellejurre's VRChat Performance Benchmarks** (wiki.vrchat.com/wiki/Community:VRChat_Performance_Benchmarks) — curve-fitting methodology: spawn 1/8/64/256 of a component, fit frame-time-vs-count. Hardware: Ryzen 7 5800X + RTX 4090. Frame budget: 90 fps ≈ 11 ms, 60 fps ≈ 16 ms. All **[MEASURED]** in VRChat:

| Feature | Measured cost (per 1000 units) |
|---|---|
| PhysBone affected transforms | **0.66 ms** / 1000 transforms |
| Animator FX layers (2-state toggle, WD on, idle) | ~0.01 ms/layer, **superlinear at high counts**; +20–30% if actively toggled |
| Direct Blend Trees (as layer replacement) | ~¼ of equivalent toggle-layer cost |
| Unity Constraints | ~3 ms / 1000 (big jumps past ~1250 active) |
| VRC Constraints (editor-converted) | ~0.25 ms / 1000 at depth 1; ~0.5 ms at depth 100 |
| Auto-converted VRC Constraints | ~0.75 ms / 1000 |
| Contact senders/receivers (idle) | ~0.5 ms / 1000 |
| Receivers actively toggled | ~0.75 ms / 1000 |
| Avatar parameters (local only) | ~1.5 ms / 1000 |
| Cloth vertices | ~0.2 ms / 1000 (linear to ~200k lobby-wide, then superlinear); +7%/collider |
| Draw calls (skinned, RTX 4090) | ~2 ms / 1000 |

**Key structural findings [MEASURED]:**
- **PhysBones scale on transform count, not component count.** Hierarchy shape adds up to a 33% spread (fewer, larger components cheaper). Colliders have only slight effect. Matches VRChat's claim that PhysBones are 12–20× faster than DynamicBone.
- **Animator layers are the hidden tax.** One 100-layer controller ≈ ten 30-layer controllers; 40 avatars × 50 layers would cost 68 ms if linear, actually ~6.4 ms (superlinear in per-controller layer count). WD-off increases toggle-layer cost ~50%. AnyState with "Can Transition to Self" costs ~20% extra.
- **Constraints are a known footgun.** Real avatars documented where constraint hierarchies consumed **11.1 ms of an 11.5 ms CPU frame** (612 constraints) — ask.vrchat.com developer-update thread, May 2024.
- **Cloth vs PhysBones ranking mismatch:** one Aug 2025 report measured 128 copies of an avatar: cloth <2 ms, PhysBones 14–16 ms — yet the avatar rated Very Poor *solely* by cloth vertex count. VRChat acknowledged cloth limits date to Unity 2017 and are under re-analysis.
- **Shader tiers:** no rigorous cross-shader benchmark table exists. One measurement: VRChat-vs-Unity draw call throughput regression of 2.5–6× depending on shader (lilToon vs Standard), skinned meshes 20–30% slower than raw Unity. Consensus tiering: opaque toon/Standard Lite < multi-pass transparency < heavy mask/alpha setups with grabpasses < tessellation fur.

## 3. Crowd scenarios: 40 Very Poor vs Medium

**[ESTIMATED from measured per-unit costs]** — no controlled 40-avatar A/B study published:
- **CPU:** 40 Medium avatars (~100 PB transforms each) ≈ 2.6 ms; 40 Very Poor (300+) ≈ 8+ ms. Animator layers dominate: 40 avatars × 30 layers ≈ 6 ms; 40 × 150+ layers ≈ 20–30 ms — 90 fps impossible on CPU alone. Consistent with community observation that hiding 10–20% of the heaviest avatars measurably improves framerate.
- **GPU:** 40 Medium at ~8–16 material slots ≈ 320–640 skinned draw calls (plus shadow/VR passes); Very Poor with 20–30 slots, transparency, extra passes can triple that. Each pixel light multiplies passes across every material.
- **VRAM:** Very Poor PC avatars routinely 100–200 MB download; 40 of them = gigabytes of texture VRAM → pooled-out VRAM hitches.

## 4. Skinned mesh optimization

**GPU skinning [MEASURED, d4rkc0d3r, RTX 4090, Unity 2019.4.31f1]** for a 115k-vertex mesh:
- Merged mesh, no active blendshapes: 0.04 ms total.
- Merged mesh, 1 active blendshape: **0.75 ms** (0.37 ms of that is a buffer copy — pipeline stall).
- Merged mesh, 10 active: 0.86 ms.
- *Same mesh split* into blendshape mesh + plain mesh: 0.09 ms (1 shape) / 0.17 ms (10) — **8–5× cheaper** because the 105k non-blendshape vertices skip the copy path.
- On Unity 2021.3.21f1 the merged path collapses to 0.03–0.105 ms (compute-only) — post-2022 VRChat made splitting largely unnecessary, but it explains why blendshape-animated faces were historically a hidden GPU tax. 20 copies of the merged avatar with 1 blendshape active caused massive fps drop; the split version didn't.

**Bone count:** **0.32 ms per 1000 moving bones** [MEASURED, jellejurre] — add PhysBone's 0.66 ms/1000 on top if PB-driven. Vertex count did not materially affect bone/material tests.

**Blendshape buffers [MEASURED]:** the GPU buffer always contains *all* blendshapes of the mesh (even unused) with delta normals and tangents, but stores deltas only for vertices with non-zero deltas (40 bytes/vertex × 3 components for pos/normal/tangent). Stripping unused shapes saves real VRAM (8.05 MiB for 24 shapes vs 20.15 MiB for 61 on the test mesh); separating face/hair meshes limits which meshes pay the buffer.

**AAO Trace and Optimize runtime effects:** no independent frame-time benchmark published; claimed savings (fewer blendshapes → smaller GPU buffers, merged bones → less skinning+PB cost, merged meshes → fewer draw calls) follow directly from §2 per-unit costs — **[ESTIMATED savings, mechanism-verified]**. One hard number [MEASURED]: an AAO-internal optimization reduced *editor* AutoMergeBlendShape processing from ~370 ms to ~22 ms (build time, not runtime).

## 5. Draw call reality

- Typical PC avatar: **~5–15 material slots** → 5–15 skinned draw calls, doubled by the shadow pass, tripled for your own avatar (VRChat renders three clones of it: local, mirror, shadow) **[ESTIMATED from architecture]**.
- **1000 draw calls ≈ 2 ms on an RTX 4090** [MEASURED]; scaling identical whether across meshes or submeshes.
- Realtime lights multiply: 20 materials × 3 passes × 3 lights = 180 calls for a previously trivial avatar [MEASURED arithmetic on a real scenario].
- Skinned meshes in crowds with different bones/textures generally do **not** batch — VRChat crowds are draw-call heavy; the realistic crowd win is fewer materials per avatar, not instancing. **[ESTIMATED]**

## 6. Mirror cost multiplication

A mirror is a second camera from the reflected viewpoint — in VR, effectively a third render of the scene. VRChat Wiki: a whole-world mirror "will double the performance cost of rendering the world"; **occlusion culling does not work inside mirror reflections**. A 100 fps → ~50 fps drop with a mirror matches one measured anecdote. Every reflected avatar's skinned meshes/transparency/particles render again; your own avatar's mirror+shadow clones mean it's simulated/rendered three times.

**Disable Pixel Lights** disables pixel-shaded point/spot lighting in the reflection (falls back to vertex lighting) — eliminates N_lights × N_materials draw calls *inside the mirror only*. Does **not** reduce the mirror's geometry baseline, shadow passes, or transparency overdraw. Layer culling (Reflect Layers) and lower mirror resolution usually save more. **[ESTIMATED, mechanism-verified]**

## 7. Quest profiling

- **Tools:** Unity Profiler (relative attribution only); **RenderDoc Meta Fork** for GPU captures; Meta's `ovrgpuprofiler` for tile-level tracing (~10% GPU overhead while profiling).
- **Frame budget:** Meta requires 72 fps (13.9 ms); Quest 3 supports 90/120 Hz (11.1/8.3 ms). Triage: disable the render camera → unaffected = CPU-bound; drop render scale → improves = fill-bound (overdraw).
- **Overdraw** is the dominant mobile GPU killer — "Alpha fill rate is a significant performance sink for mobile GPUs"; post-processing disabled entirely on Android. One contributor's estimate: Quest ≈ 4.8× slower polygon throughput, 1.78× pixel throughput vs their PC baseline — 70k-tri multi-pass avatar ≈ 40%+ frame-time penalty; heavily overdrawn scenes at 23–30 fps dual-eye. **[MEASURED by one author, informal methodology]**
- **Thermal throttling:** no public benchmark of VRChat's Quest throttling curves [ESTIMATED: sustained Very Poor crowds push the chip into throttled clocks within minutes] — which is why mobile rank blocking is hard-locked (Very Poor hidden by default, mobile users cannot disable the block; exceeding component caps physically strips components).

## 8. Tools catalog

| Tool | Author | Status (late 2025) | What it does |
|---|---|---|---|
| **Actual Performance Window** | anatawa12 (Gist Pack) | Actively maintained | Runs the SDK's own `AvatarPerformance.CalculatePerformanceStats` at Build & Publish or Play Mode — the way to see *post-non-destructive-tools* rank (SDK panel shows pre-bake rank, wrong when AAO/MA installed) |
| **lilAvatarUtils** | lilxyzw | 2.1.2 (Aug 2025), MIT | Texture tool: all avatar textures with computed VRAM, batch replace, resolution/import overrides; Mesh/Animator/Shader utilities |
| **Thryrallo VRC-Avatar-Performance-Tools** | Thryrallo | VPM, maintained | VRAM Checker (incl. animated materials) + Avatar Evaluator (VRAM, grabpasses, blendshapes, AnyState counts, WD audit) |
| **AAO: Avatar Optimizer** | anatawa12 | 1.9.x line | Non-destructive runtime cost reduction (see §4) |
| **jellejurre/UnityBenchmark** | jellejurre | Public repo | Harness reproducing §2 measurements |
| **RenderDoc (Meta fork) / Nsight Graphics** | — | Stable | Ground-truth GPU captures |
| **ovrgpuprofiler** | Meta | Stable | On-device Quest GPU stage tracing |

## Bottom line

The measurement stack is mature — Unity Profiler + RenderDoc/Nsight + Actual Performance Window covers CPU, GPU, and rank — but VRChat's rank measures *component presence*, not runtime cost. The actual costs live elsewhere: animator layers (~0.01 ms/layer, superlinear), constraint hierarchies (up to 11 ms disasters), PhysBone transform counts (0.66 ms/1000), and blendshape buffers (an order-of-magnitude GPU tax pre-Unity-2021 that mesh-splitting fixed). Crowds are won or lost on animator/PhysBone CPU time and material-slot draw calls; mirrors re-pay everything once more per eye; on Quest, overdraw and thermal headroom — not triangle counts — end the party.

## Sources

- https://wiki.vrchat.com/wiki/Community:VRChat_Performance_Benchmarks (jellejurre) · https://github.com/jellejurre/UnityBenchmark
- https://gist.github.com/d4rkc0d3r/f77c1e96d4aeefd0d1eaf13fb096a2de (GPU skinning/blendshape study)
- https://creators.vrchat.com/platforms/android/quest-content-optimization · https://creators.vrchat.com/avatars/avatar-performance-ranking-system/
- ask.vrchat.com developer-update threads (constraints 11 ms report; cloth vs PhysBone report)
- github.com/anatawa12/AvatarOptimizer · github.com/lilxyzw/lilAvatarUtils · github.com/Thryrallo/VRC-Avatar-Performance-Tools
- developers.meta.com (RenderDoc Meta fork, ovrgpuprofiler) · wiki.vrchat.com (mirror cost)
