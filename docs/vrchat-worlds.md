# VRChat World Creation — Comprehensive Research Report (2025–2026)

## 1. Toolchain: Unity, VCC, VPM, and SDK3 Setup

### 1.1 Unity Version

- **Current supported Unity version: 2022.3.22f1** (officialized with SDK 3.6.0 + VCC 2.3.0 in May 2024). Do **not** upgrade to newer Unity LTS versions — content built on other versions will not load once uploaded to VRChat.
- **October 2025 Unity security vulnerability note:** the standalone-player Unity vulnerability disclosed in Oct 2025 does **not** affect VRChat content creation. VRChat explicitly tells creators to ignore Unity Hub's upgrade warning and stay on 2022.3.22f1.
- When installing via Unity Hub, select **Android Build Support** (+ SDK/NDK/OpenJDK) and **iOS Build Support** for Quest/Android/iOS targets.
- **Built-in Render Pipeline only.** URP and HDRP are not supported.

### 1.2 VRChat Creator Companion (VCC) & VPM

- The **VCC** (https://vcc.docs.vrchat.com/) is the standard project manager. It installs Unity Hub + the correct Unity editor, and creates new **Worlds**, **Avatars**, or **UdonSharp** template projects.
- Install location default: `%LocalAppData%\Programs\VRChat Creator Companion`. Windows 10/11 64-bit required.
- **VPM (VRChat Package Manager)** packages live in the project's `Packages/` folder (not `Assets/`), declare dependencies, and install cleanly. Legacy `.unitypackage` SDK distribution was discontinued at the end of 2022.
- Community package repositories (e.g., Poiyomi, z3y, MerlinVR) can be added as VCC curated lists.

### 1.3 SDK3 (Worlds SDK) & Project Structure

- SDK3 = the current SDK generation (SDK2 uploads were **permanently switched off November 3, 2025** — the blocking was announced/enforced across Oct–Nov 2025; old SDK2 content still loads but can no longer be updated).
- A typical worlds project contains:
  - `Packages/com.vrchat.worlds/` — the Worlds SDK (inspectors, prefabs, example scene at `Samples/UdonExampleScene/`).
  - `Packages/com.vrchat.base/` — the Base SDK (shared components).
  - `Packages/com.vrchat.avatars/` (if added) — avatar SDK.
  - `Packages/com.vrchat.udonsharp/` — **UdonSharp is bundled by default with the Worlds SDK** (since SDK 3.5.0).
  - `Assets/` — your scenes, materials, meshes, textures, UdonSharp scripts.
- Authentication: `VRChat SDK > Show Control Panel > Authentication`. You need a **VRChat account (not Steam/Meta-linked only) of at least "New User" trust rank** to upload. As of **SDK 3.9.0 (Oct 2025)**, visitors can now Build & Test locally; upload still requires rank.
- **Oct 2025**: VRChat began **blocking new uploads from older SDKs** (upgrade to 3.9.x required for new uploads).

## 2. Udon / UdonSharp

### 2.1 Udon Graph vs UdonSharp

- **Udon** is VRChat's custom scripting runtime for worlds. Two authoring modes:
  - **Udon Graph** — visual node-based editor (each node = an "extern" call into native C# land).
  - **UdonSharp (U#)** — C#-like source compiled to Udon programs; the de-facto standard for anything nontrivial. Created by MerlinVR, bundled officially.
- Programs attach to GameObjects as **UdonBehaviour** components. Execution model: single-threaded event-driven; heavy loops block the whole instance's Udon budget (Udon behaviours across all players share CPU time, so per-frame `Update` work is expensive in busy instances).

### 2.2 Events (the core API surface)

- Lifecycle: `Start`, `Update`, `LateUpdate`, `FixedUpdate`, `OnEnable`/`OnDisable`, `OnDestroy`.
- Interaction: `Interact()` (triggered by raycast on a collider with a UdonBehaviour — the interaction text is customizable), `OnPickup`, `OnDrop`, `OnPickupUseDown/Up` (trigger squeeze), `OnTriggerEnter/Exit` (collider set as isTrigger), `OnPlayerTriggerEnter(VRCPlayerApi)` / `OnPlayerTriggerExit` (player-trigger colliders configured on the UdonBehaviour).
- Networking: `OnOwnershipRequest`, `OnOwnershipTransferred`, `OnPlayerJoined(VRCPlayerApi)`, `OnPlayerLeft`, `OnDeserialization`, `OnPreSerialization`.
- Input/respawn: `OnPlayerRespawn`, `InputJump(bool)` etc. (Input* hooks only on local player, require "Input Hooks" enabled in project settings).
- Video: `OnVideoStart`, `OnVideoEnd`, `OnVideoLoop`, `OnVideoPause`, `OnVideoPlay`, `OnVideoReady`, `OnVideoError(VideoError)`.

### 2.3 Sync Patterns

- Variables synced with `[UdonSynced]` (U#) or the Sync mode dropdown (Graph): `None`, `Continuous`, `Manual`. Use `RequestSerialization()` after changing a synced var in Manual mode.
- Ownership: `Networking.SetOwner(VRCPlayerApi.player, gameObject)`; check `Networking.IsOwner(gameObject)`.
- `SendCustomNetworkEvent(NetworkEventTarget.All | Others | Owner | Self, "MethodName")` — since **SDK 3.8.1 (May 2025)**, network events support **up to 8 parameters**, declared in U# with the `[NetworkCallable]` attribute (default rate limit 5/sec per event, max 100/sec; `NetworkEventTarget.Self` bypasses rate limits; `NetworkCalling.CallingPlayer` identifies the sender). `NetworkEventTarget.Others` excludes the sender.
- Timers: `SendCustomEventDelayedSeconds("Method", float)`, `SendCustomEventDelayedFrames(...)`, `SendCustomEvent(...)`.
- `VRCPlayerApi` exposes `TeleportTo(position, rotation)`, `Respawn()`, `SetVelocity`, `GetVelocity`, `GetPosition`, `PlayHapticEventInHand`, `UseAttachedStation`, `Immobilize(bool)`, etc. `localPlayer = Networking.LocalPlayer`.

### 2.4 Common World Patterns (UdonSharp recipes)

- **Toggle button**: collider + `Interact()` → flip a bool, toggle GameObject active / renderer enabled; sync with a synced bool + `RequestSerialization()`.
- **Doors**: `Interact()` or `OnPlayerTriggerEnter` rotates via `Quaternion.RotateTowards` in `Update`, or an Animator. For networked doors, take ownership, sync an "open" bool.
- **Sliders**: UI `Slider` on a world-space Canvas with `VRC_UiShape`; hook `OnValueChanged` to a public U# method via UnityEvents.
- **Respawn / kill floor**: trigger collider + `OnPlayerTriggerEnter` → `player.Respawn()`; teleport zones → `player.TeleportTo(spawnPoint.position, spawnPoint.rotation)`.
- **Teleport pads**: `Interact()` or trigger + `localPlayer.TeleportTo(...)`.
- **Audio zones**: trigger colliders that enable/disable `AudioSource` or crossfade volume in `Update`; combine with `VRC_SpatialAudioSource`.
- **Pickups**: add `VRC_Pickup`, script `OnPickup`/`OnDrop`/`OnPickupUseDown`; `OnPickupTriggerDown/Up` for trigger-based guns. Synced pickups use synced Vector3 positions + ownership transfer.
- **Particle triggers**: `ParticleSystem.Play()`/`Stop()` from Udon.

## 3. VRChat-Specific Scene Components

Allowlist: https://creators.vrchat.com/worlds/components/ — components not on the list are stripped at build.

| Component | Purpose / key notes |
|---|---|
| **VRC_AvatarPedestal** | Spawn avatars by ID. Fields: `avatarUrls` (VRCUrl[]), `useManifest`. |
| **VRC_Station** | Seats/sitting. Used for chairs, pods, cinema seats; `UseAttachedStation()` from Udon. |
| **VRC_SpatialAudioSource** | Wraps AudioSource for VRChat's spatializer. Key settings: **Gain (dB, up to +12)**, **Enable Spatialization**, **Use AudioSource Volume Curve**, **Custom Curve**. Common pitfall: silence after upload — fix with Enable Spatialize + Spatial Blend = 1 + linear/custom rolloff. |
| **VRC_UiShape** | Makes world-space UI canvases clickable. Required on any world-space canvas you want to click. |
| **VRC_Pickup** | Grab-and-hold objects. Fields: pickupHand, allowDisposal, maxPickupDistance, Throw Velocity Boost, Auto Hold (**AutoHold checkbox** in SDK 3.9.0). |
| **VRC_ObjectSync** | Network-synced transform for physical objects (pickup + sync). |
| **VRC_PortalMarker / VRC_PortalSpawner** | World portals; `VRC_PortalSpawner` creates portals from Udon (dynamic spawning). |
| **VRC_Mirror** (VRCMirrorReflection prefab) | Realtime reflection camera. **Extremely expensive** — **one mirror max per room, never enabled by default**, behind a user toggle. |
| **VRC_SceneResetSpawn** | Custom spawn reset point. |

**Note:** SDK2's `VRC_Trigger` does **not** exist in SDK3 — all event triggers are done in Udon.

### Post-processing & cameras
- Post-processing (PPv2) works on **PC worlds** but must be VR-safe (avoid chromatic aberration, SSR, SSAO). **Post-processing is fully disabled on Android/Quest.**
- **SDK 3.8.1** exposed `VRCCameraSettings` & `VRCQualitySettings` to Udon; **3.9.0** added `VRCCameraSettings.CullingMask`, `GetCurrentCamera()`, and **Camera Dolly** components (`VRCCameraDollyAnimation`, `VRCCameraDollyPath`, `VRCCameraDollyPoint`).

## 4. Physics

- **PhysBones** are avatar-oriented; in **worlds** you use standard Unity physics: Rigidbody, Colliders, Joints — all permitted on Android too, but use moderately.
- Udon physics access: `Physics.Raycast`, `Physics.OverlapSphere`, Rigidbody externs; SDK 3.6.0 exposed `Physics.bounceThreshold` to Udon.
- VRCPickup + Rigidbody + VRC_ObjectSync = throwable/synced physics props; note **Items in Udon Worlds**: players can spawn their own items into your world.
- Unity Constraints are permitted in worlds but cost more than expected on mobile.

## 5. Performance & Limits

### 5.1 World-specific hard limits (PC & Android)

| Metric | Limit |
|---|---|
| **Compressed world size, Android/Quest** | **100 MB hard cap** after build-time compression |
| **Practical size guidance (Public worlds)** | **Keep under 200 MB** (VRChat may delist oversized Public worlds) |
| **Android triangle budget** | ~**250,000 triangles** for the whole world (leave room for avatars) |
| **Textures (mobile)** | ≤ 1024×1024 recommended; atlas aggressively; **ASTC compression**; Crunch helps download size only, NOT runtime memory |
| **Materials** | Minimize; enable GPU Instancing; don't over-merge (kills occlusion culling) |
| **Video players** | > 2 in a room = performance red flag |
| **Mirrors** | > 1 per room = red flag; never enable by default |
| **FPS target** | ≥ 45 FPS with a single VR user at spawn |

### 5.2 Reference budgets

Worlds do **not** get a per-world performance rank like avatars, but texture memory is the dominant VRAM cost: keep ~150 MB textures on PC / ~40 MB on mobile for healthy territory (avatar rank thresholds: PC Excellent ≤40 MB → Very Poor >150 MB; Mobile Excellent ≤10 MB → Very Poor >40 MB).

### 5.3 Android/Quest limitations (worlds)

Source: https://creators.vrchat.com/platforms/android/quest-content-limitations/

- **Post-processing: completely disabled.**
- **Cameras/mirrors**: permitted in worlds but heavy — use sparingly.
- **Video players**: work with limitations.
- **Cloth**: disabled on avatars; careful in worlds.
- **Audio Sources**: allowed in worlds; disabled on avatars.
- **Rigidbodies/colliders/joints**: allowed, use moderately.
- **Shaders on Android worlds**: worlds can use custom shaders on Android (unlike avatars) — performance is your responsibility. SDK mobile world shaders: `Lightmapped`, `Skybox`, `Supersampled UI`; **`VRChat/Mobile/Toon Standard` arrived in SDK 3.8.1** (Outline variant PC-only).
- **iOS**: SDK iOS build/upload support began late 2024; SDK 3.8.1 added iOS Build & Test and per-platform overrides; full phone release October 2025.

## 6. Upload Pipeline

1. **Build & Test** (`VRChat SDK > Show Control Panel > Build & Test`) runs the world locally — no trust rank needed as of 3.9.0. Use **ClientSim** (VPM package) to test in-editor: simulates players, stations, pickups, mirrors.
2. **Build & Publish** (Windows target). Switch active build target to Android and repeat for Quest/Android; iOS via multi-platform publishing (3.8.1+).
3. First upload creates a private world; **"World Visibility"** then lets you **Publish to Community Labs**.
   - Requirements: trust rank **"User"+**, one new world per 7 days, updates allowed any time.
   - Worlds that perform well are **automatically made Public** — no manual approval.
4. **Content Warnings** (Content Gating): set in SDK **Builder tab → Content Warnings**, or the website world page. Labels: **Sexually suggestive, Adult language and themes, Graphic violence, Excessive gore, Extreme horror**. Users filter these in Settings → Comfort & Safety. Age-based forced gating was disabled Dec 2023; VRChat announced in Dec 2025 it will be **reintroduced alongside Age Verification in H1 2026**.
5. **Safety settings** on the user side: Community Labs worlds hidden by default; clients can block untrusted URLs, set max download sizes, mirror resolution.
6. **Thumbnails**: SDK captures a thumbnail; ensure it accurately represents the world.

### Scene setup & lighting best practices
- **Bake lighting** (Baked GI / lightmaps) — "exceedingly important and can give you huge performance gains." Use light probes for dynamic objects. Avoid real-time lights except a few key ones.
- **Bake occlusion culling**; keep occlusion areas aligned to final geometry; don't over-merge meshes.
- Use **tonemapping** to avoid blowout; test lighting with a Standard/PBR shader, not Toon.
- Shaders must support **single-pass stereo** (no grab-pass-dependent or screen-space-only effects that break VR).
- Supersampled UI shader for world-space text crispness.
- Avoid `.blend` direct imports; export FBX from Blender.

## 7. Current 2025–2026 State & Community Best Practices

- **SDK versions:** 3.6.0 (May 2024, Unity 2022.3.22f1) → 3.7.x (2024) → **3.8.1 (May 2025: Toon Standard shader, network events with parameters, iOS Build & Test, camera/quality Udon APIs)** → **3.9.0 (Oct 2025: Camera Dolly, pickup AutoHold, Network ID Utility, VRCCameraSettings.CullingMask, visitor Build & Test)**. Latest: https://creators.vrchat.com/releases/
- **Udon 2 → Soba:** Udon 2 (a new compiled Udon runtime) was **discontinued in Dec 2024** in favor of **Soba**, a new C#-based object-oriented programming system that will coexist with (not replace) Udon. As of Aug 2026, Soba is still pre-closed-beta. Source: https://ask.vrchat.com/t/on-udon-2-soba-and-why-we-changed-directions/28484
- **I18n/localization:** no official SDK-level i18n system for worlds; community pattern is per-language GameObject groups or Udon dictionary lookups. (Official guidance gap — say so plainly.)
- **Community tools ecosystem (VPM):** UdonSharp (bundled), ClientSim, ProTV 3 (https://protv.dev/, open beta Mar 2025 — RenderTexture pipeline, LTCGI/AreaLit integration; supersedes VideoTXL), Udon Toolkit, VRSL (stage lighting), LTCGI (image-based lighting), VRWorldToolkit (editor-side world validation), CyanTrigger, z3y shaders.
- **Practical checklist (consensus across official + community guides):**
  1. VCC + Unity 2022.3.22f1, Worlds template, latest SDK 3.9.x.
  2. Bake lights, bake occlusion, ≤~250k tris on Android, 100 MB compressed Android cap.
  3. One toggleable mirror; ≤2 video players; stagger video URL loads 5s+.
  4. Supersampled UI shader; VR-safe post-processing only on PC.
  5. Test with ClientSim in-editor, then Build & Test on PC and Quest; 45+ FPS at spawn.
  6. Label content warnings; publish to Community Labs; keep under 200 MB.

## Source URLs

- Getting started / SDK setup: https://creators.vrchat.com/sdk/ , https://vcc.docs.vrchat.com/
- Current Unity version: https://creators.vrchat.com/sdk/upgrade/current-unity-version/
- Worlds overview & components allowlist: https://creators.vrchat.com/worlds/
- Udon video players: https://creators.vrchat.com/worlds/udon/video-players/
- Android optimization: https://creators.vrchat.com/platforms/android/quest-content-optimization/ , https://creators.vrchat.com/platforms/android/quest-content-limitations/
- Community Labs / publishing: https://docs.vrchat.com/docs/vrchat-community-labs/
- Content gating: https://wiki.vrchat.com/wiki/Content_Gating , https://hello.vrchat.com/creator-guidelines
- SDK releases: https://creators.vrchat.com/releases/release-3-8-1/ , https://creators.vrchat.com/releases/release-3-9-0/
- Udon 2 / Soba: https://ask.vrchat.com/t/on-udon-2-soba-and-why-we-changed-directions/28484
- SDK2 shutdown: https://ask.vrchat.com/t/developer-update-14-august-2025/46303
- ProTV: https://protv.dev/
- Avatar performance ranking (context): https://creators.vrchat.com/avatars/avatar-performance-ranking-system/
