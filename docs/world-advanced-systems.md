# Advanced VRChat World Systems & Techniques — 2025–2026

*Survey of current techniques, APIs, and community tooling for advanced world creation. All API details cross-checked against live documentation; unverifiable items flagged.*

---

## 1. Mirrors, In Depth

`VRCMirrorReflection` ([docs](https://creators.vrchat.com/worlds/components/vrc_mirrorreflection/)) renders the scene from a mirrored camera into the `_MainTex` of the first material on the same GameObject's MeshRenderer — the SDK ships `VRCMirror.prefab` as reference.

**Key parameters:**

| Parameter | Behavior |
|---|---|
| **Disable Pixel Lights** | Point/spot lights fall back to vertex lighting inside the reflection. Biggest mirror perf win. |
| **Turn Off Mirror Occlusion** | Disables occlusion culling in the reflection; enable if objects flicker. |
| **Reflect Layers** | Layer mask. Note: the **Water layer is never rendered in mirrors** (hard-coded). |
| **Mirror Resolution** | `Auto` renders at display/HMD resolution **capped at 2048×2048 per eye**. No U# API to raise it — you can only lower it (512/1024/2048). |
| **Maximum Antialiasing** | MSAA cap; client graphics settings can override downward. |
| **Custom Shader** | Replaces the default mirror shader — the hook for tinted/frosted/animated mirrors. |
| **Custom Clear Color** | Alpha channel respected — enables cutout-style mirror tricks. |

**Recursive layer masks:** mirrors don't reflect other mirrors (no recursion). Workaround: **Mirror-Clone-Toggle** (VRLabs) — a second mirror activated for the reflected view. **Local-Mirror-Detection** (VRLabs) tells you locally whether the player looks at a mirror (swap quality tiers). Mirrors exist locally per-viewer; cloning/toggling is a local operation — the basis of **per-user mirror toggles**. Best practice: **off by default**, enable on approach, quality presets, reduce resolution first.

**Fake mirrors:** Camera → RenderTexture on a quad — cheaper, arbitrary scale, post-processable, non-planar. Caveats: cameras are expensive (manage culling mask, resolution, update rate yourself — enable on demand via Udon); no parallax correction (reads as "screens" unless viewer is constrained).

**Fake windows:** (1) unlit/emissive plane sampling a panorama texture + realtime directional light + LTCGI screen to sell light spill; (2) reflection probe baked at the window. For "windows into other places," a second scene area behind the wall with parallax-corrected shader beats a real camera render.

## 2. Post-Processing Stacks

Unity **PPv2**: `PostProcessVolume` (global or local) + `PostProcessLayer` on the reference camera (SDK VRCWorld setup handles layer binding). The [post-processing-settings persistence example](https://creators.vrchat.com/worlds/examples/persistence/post-processing-settings) shows exposing volume parameters to Udon so users save preferences — copy this pattern for accessibility.

**Safe for VR:** Bloom, mild vignette, color grading (LUT/tonemapping), grain, subtle chromatic aberration. **Do not use:** motion blur (instant nausea), depth of field (breaks stereo convergence — desktop-only worlds), lens distortion, heavy SSR (stereo reprojection artifacts). Heavy SSAO is the usual Quest framerate killer — mark volumes to be skipped on Android. Keep bloom modest — HDR bloom in an HMD reads far stronger than on a monitor.

## 3. Lighting Artistry

**Light probes & LPPV:** Light Probe Groups for dynamic objects + Light Probe Proxy Volumes on large non-static meshes. Bake with mixed directional mode for normal-mapped lightmaps.

**Reflection probes:** one baked "master" probe + small local probes near reflective zones. Realtime probes off the table; swap baked probe textures at runtime via Udon for day/night. Box-projected probes on interiors prevent light leaking from wrong directions.

**Emission + bloom:** the workhorse illusion — bake ambient light low, make perceived lighting come from emissive surfaces picked up by bloom, real lights only where shadows matter. The core of "neon city" worlds.

**LTCGI** ([ltcgi.dev](https://ltcgi.dev)): realtime area light for TVs/windows/neon strips.
1. Install via VCC (vpm.pimaker.at).
2. Receivers need compatible shader: Poiyomi 8 (LTCGI module), ORL, z3y, Mochie, Silent's Filamented, Hekky, lilToon 1.8+; URP Lit variant exists.
3. Add **LTCGI_Controller** (one per scene); optional single "Video Texture" RenderTexture slot for realtime video lighting (one video input only). Fast Sampling forced on Quest.
4. **LTCGI_Screen** components on each area light mesh: Color Mode (Static/Texture/Single UV/AudioLink), Affected Renderers (mask = perf — don't leave "everything").
5. Baked shadows: allocate a lightmap channel per screen, bake (Unity lightmapper or Bakery).

LTCGI supports **VRC Light Volumes** since 1.7.0. Attribution required.

**Day/night cycles:** (a) **Lightmap swaps** — bake 2–4 times-of-day sets, swap `lightmapIndex` + ambient via Udon; instant, free, discrete steps (crossfade via overlapping geometry sets is brute-force). (b) **Realtime rig** — rotating directional + ambient lerp + runtime probe swaps; flexible, loses baked quality. (c) **Hybrid** — baked neutral lightmaps + LTCGI screens + emissive-animated fixtures driven by an Udon clock. Sync the clock as an owner-serialized float so all users see the same sky.

## 4. Advanced Udon Patterns

**State machines:** idiomatic Udon FSM = `int _state` checked against a `string[] _stateNames`, or UdonBehaviour state objects each with `Enter()`/`Exit()` custom events dispatched via `SendCustomEvent`. Centralized transition tables (`int[,]`) keep it data-driven. Avoid string indirection in hot loops — cache UdonBehaviour references, prefer direct variable writes + one dispatch event.

**Event bus / singleton:** a "hub" UdonBehaviour that others register into at `Start()` (write into array slot + call `Register()`). Fixed-size object arrays with a compaction counter (no dynamic collections). Singletons: no static lookup exists — each consumer holds a serialized reference, or first `Start()` claims the object and others find it via `GetComponentInChildren`.

**GC pressure:** Udon heap-allocates on string concatenation, array instantiation, `object`-typed boxing of value types, and `GetComponents`-style calls. Rules: allocate in `Start()`, never in per-interaction code; reuse scratch arrays as fields; avoid `object`-typed cross-behaviour calls in loops; never rebuild UI strings when text hasn't changed.

**Timers / coroutine emulation:** `SendCustomEventDelayedSeconds` is the primitive; scheduler systems use a fixed array of `{time, eventIndex}` pumped from a low-frequency tick (0.25–1 s), avoiding one delayed event per item. **VRCTween** (official), and VRLabs' **BlendTreeBuilder** for driving state via Animator instead of Udon ticks (Animator blendtrees consume normalizedTime at zero Udon cost).

## 5. Player Systems

**Custom locomotion:** `Immobilize(true)` strips walking; then `TeleportTo(position, rotation)` per-tick (hard snap — interpolate first, TeleportTo on the final frame). VRLabs' **Follower** (head/chest-follow objects) and climbing systems are built on Immobilize + hand-driven TeleportTo with rig lerping (never lerp the camera itself).

**Sit systems beyond stations:** stations are rigid (no physics, forced pose). Alternatives: Immobilize + TeleportTo + manual animation override via playable layers; constraint-based sitting (Station only for the interaction, then release to a custom rig). Vehicles: station in parented rig with `canUseEnterStation` + Udon-driven rigidbody motion.

**Custom gravity:** no direct API. (a) Immobilize + per-frame velocity-matched TeleportTo along a gravity field (expensive); (b) walking players don't honor `SetVelocity` — most "walking on walls" worlds use teleport-interpolation with heavy damping; (c) restrict the effect to pickups, which do honor rigidbody physics.

**Player collision:** three detection families ([docs](https://creators.vrchat.com/worlds/udon/players/player-collisions/)): `OnPlayerTriggerEnter/Stay/Exit` (capsule vs trigger), `OnPlayerCollisionEnter/Stay/Exit` + `OnControllerColliderHitPlayer` (moving physics objects vs players — *not* called when a player walks into a stationary object), `OnPlayerParticleCollision`. To push players: [player-forces](https://creators.vrchat.com/worlds/udon/players/player-forces/) — velocity impulses from Udon.

**Voice — verified current API** ([player-audio docs](https://creators.vrchat.com/worlds/udon/players/player-audio/)) — **not** `SetVoiceDistance`; per player:
- `SetVoiceGain(float)` — dB boost, 0–24, default 15.
- `SetVoiceDistanceNear(float)` — 0–1,000,000 m; keep 0 for realism.
- `SetVoiceDistanceFar(float)` — 0–1,000,000 m, default 25. **0 = effectively mute.**
- `SetVoiceVolumetricRadius(float)` — 0–1000, keep 0 unless simulating a large source; must be < Far.
- `SetVoiceLowpass(bool)` — disable the distance low-pass (DJ voice channels).

Avatar audio: `SetAvatarAudioGain` (0–10 dB), `SetAvatarAudioFarRadius` (default 40), `SetAvatarAudioNearRadius`, `SetAvatarAudioVolumetricRadius`, `SetAvatarAudioForceSpatial`, `SetAvatarAudioCustomCurve`.

**Range zones:** a trigger zone calling `SetVoiceDistanceFar` on enter/exit per affected player. **Mute zones** (Far = 0) and **safe zones** (combined with [Udon Moderation Tool guidelines](https://creators.vrchat.com/worlds/udon/udon-moderation-tool-guidelines/) + UI hiding) are community-standard. Voice settings apply as *you* hear the other player — the zone owner pattern is "each player's local script adjusts everyone's voice when in range."

## 6. Stations, Radar, Queues, Whitelists, Pedestals

**Radar arrays:** `OnPlayerJoined/Left` maintaining your own `VRCPlayerApi[]` — fixed-capacity array (e.g., 82) with a count and swap-remove. Radar displays iterate on a slow tick, sampling `GetTrackingData`/`GetVelocity`. VRLabs' Contact-Tracker and Camera-System follow this fixed-pool pattern.

**Join queues:** ownership-based tickets — each joiner requests ownership of the first free slot object (`Networking.SetOwner`), slot index = position; `OnDeserialization` updates everyone's displayed queue; a trigger admits only the slot-0 owner.

**Nameplates:** the official nameplates-customization docs URL currently 404s — status in flux, **verify before building on it**. The full `VRCNameplates` Udon API remains unshipped. De facto: world-space TextMeshPro labels updated from your radar array.

**Avatar pedestals with access control** ([docs](https://creators.vrchat.com/worlds/components/vrc_avatarpedestal/)): `SetAvatarUse(blueprintId)` swaps the local player's avatar; Udon can gate the interaction (whitelist check on `displayName` — or better, user IDs — before calling). Public avatars work for everyone; **private avatars only for the uploader**; marketplace avatars show a details page to non-owners.

## 7. Persistence (2025–2026 status)

Live since Nov 2024, production-grade. Current shape ([persistence docs](https://creators.vrchat.com/worlds/udon/persistence/)):

- **PlayerData**: per-player key-value store, 100 KB per world per player (compressed). Ints, floats, doubles, vectors, colors, strings, byte arrays. Any script can read it.
- **PlayerObjects**: `VRCPlayerObject` templates auto-instantiated per player; `VRCEnablePersistence` on the UdonBehaviour persists its synced variables. Docs now recommend PlayerObjects *over* PlayerData for prefabs — changing any PlayerData re-sends the player's *entire* PlayerData set. PlayerObjects also double as unstealable per-player object pools (owner can't transfer ownership).
- **`OnPlayerRestored(VRCPlayerApi)` is the gate** — don't read/write persisted state before it. A 2025.1.1 fix guarantees it executes before other serializations.
- Storage introspection: `GetPlayerObjectStorageLimit/Usage`, `RequestStorageUsageUpdate` (call rarely); events `OnPersistenceUsageUpdated`, `OnPlayerObjectStorageExceeded/Warning`.
- Users can reset their data per-world or globally from the website.
- **2025.2.1 data-loss incident:** a serialization change broke `VRCUrl`, `VRCUrl[]`, `string[]` persistence decoding, overwriting data with defaults (post-mortem: May 8 2025 dev update). Lesson: version your persisted schema defensively; don't persist strings you can recompute; `GetPlayerRestoreStatus` API request still open.
- **2025.4.1 network-compatibility protection:** joining instances of an older, layout-incompatible world version is blocked. Consequence: changing synced variable types/order/count locks existing instances against new joiners until re-upload — careful during events. 2025.4.1 also shipped **World Dynamics** (world-side PhysBones/Contacts/Constraints, SDK 3.9.1+).
- Data must be saved **before** the local player leaves — you cannot persist in your own `OnPlayerLeft`.

## 8. Particle Wizardry

- `OnParticleCollision` for gameplay hits; `OnPlayerParticleCollision` for zones (Collision module + Send Collision Messages).
- **Vertex streams:** Custom1/Custom2 into shaders enables per-particle data-driven effects (VRLabs **Particle-Shader** and **Particle-Driver** are reference implementations) — per-particle glow, AudioLink-reactive particles, billboard impostors.
- Particle collisions as cheap physics: swarms of projectiles with zero rigidbodies.
- **Budget illusions:** one looping emitter with sub-emitters instead of many systems; `Culling Mode = Local` only for always-visible ambient; texture-sheet animations instead of meshes; Udon-driven `Emission` rate scaling for idle/active states without enable/disable (re-warm spikes).

## 9. Screen-Space Illusions

- **Portals:** real inter-world portals = `VRCPortalMarker` (World ID `wrld_...`, Home/Hub modes, custom name). Intra-world fake portals: stencil-buffer shaders (render far room through stencil mask with offset camera) or render textures. VRChat's own portal shader turns off in mirrors — stencil portals that *should* show in mirrors need layer planning.
- **Skybox swapping:** `RenderSettings.skybox` is Udon-writable (per-PC — owner-sync if shared canon); pairs with mirror Custom Skybox so reflections match.
- **Camera tricks:** **VRCCameraSettings** ([docs](https://creators.vrchat.com/worlds/udon/vrc-graphics/vrc-camera-settings/)): `ScreenCamera` (current view camera — stereo in VR) and `PhotoCamera` (handheld camera). Limited control + quality info — enough for FOV-aware shaders, camera-position-locked skyboxes, "am I being filmed?" detections (VRLabs IsRendering-Detection / Local-Mirror-Detection). No API to *arbitrarily* render from the user's camera.
- **Dolly cams:** `VRC_CameraDolly` drives the handheld camera along spline paths with control points — fly-through intros, preset "cinema mode" seating.

### Sources

- https://creators.vrchat.com/worlds/components/vrc_mirrorreflection/ · /vrc_avatarpedestal/ · /vrc_portalmarker/ · /vrc_cameradolly/
- https://creators.vrchat.com/worlds/udon/players/player-audio/ · /player-collisions/ · /player-forces/
- https://creators.vrchat.com/worlds/udon/vrc-graphics/vrc-camera-settings/ · /persistence/ · /udon-moderation-tool-guidelines/
- https://docs.vrchat.com/docs/vrchat-202511 · /vrchat-202541
- https://ask.vrchat.com/t/developer-update-25-november-2024/28125 · /developer-update-8-may-2025/43409
- https://ltcgi.dev · https://github.com/PiMaker/ltcgi · https://github.com/VRLabs

*Caveats: nameplates docs URL 404s (status unverified); no Udon mirror-resolution setter beyond the component dropdown; `SetVoiceDistance` does not exist — correct API is `SetVoiceDistanceFar`/`Near`. PPv2 specifics from established community practice (no dedicated official page).*
