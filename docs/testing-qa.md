# VRChat Testing & QA Practices: A 2025–2026 Field Report

*The SDK's validation catches only a fraction of what breaks in-game; the rest surfaces only with deliberate testing. Consolidated current practice across eight areas.*

---

## 1. In-Editor Testing Tools: Av3Emulator vs Gesture Manager vs ClientSim

Neither VRChat nor the SDK ships an official avatar play-mode emulator — a long-standing community request. VRChat staff acknowledge that Av3Emulator and Gesture Manager behave differently from each other and from the client (particle simulation, shader backends, fallback-shader preview can't be fully replicated in-editor). Both are Curated VPM packages.

### Av3Emulator (Lyuma)

- **Simulates**: the most faithful end-to-end AV3 runtime — all playable layers, parameter syncing, radial menus, gestures, viseme/eye-look parameters, HeadChops, OSC input. Crucially, spawns **non-local clones** — the only in-editor way to approximate how your avatar appears *to other players* (remote behavior differs: viewpoint scaling, locally-viewed layer suppression, parameter owner semantics).
- **Setup**: VCC (Curated) or .unitypackage (github.com/lyuma/Av3Emulator/releases). Clear the Console, `Tools → Avatars 3.0 Emulator → Enable` → an "Avatars 3.0 Emulator Control" object spawns. Set *Default Animator To Debug* = FX, enter Play Mode, click the avatar for a control panel mimicking the in-game expression menu. Parameter values drivable manually; Animator window updates live.
- **Limitations**: no runtime PhysBone config modification; locomotion "velocity" only drives proxy animations; no real networking (only parameter-sync semantics via clones). SDK 3.8.0+ PhysBone tweak-in-playmode regressions affect all editor tooling.

### Gesture Manager (BlackStartx)

- **Simulates**: animation-focused testing with a UI mimicking the in-game radial menu — all 8 gestures both hands, radial puppets, locomotion previews (walk/run/crouch/prone/fall), state previews (AFK/Seated/IK/T-pose), tracking-point count and VRMode, parameter overrides (MuteSelf, IsLocal, InStation, gesture weights). **Scene Camera** matches Game view to Scene view for PhysBone testing; **Clickable Contacts** triggers Contact Senders with the mouse (better than Av3Emulator for this). Debug tab exposes animator params, layer weights, active states. Edit-Mode clones the avatar with controllers attached for live animation editing.
- **Setup**: VCC → `Tools → Gesture Manager Emulator` → GestureManager prefab in scene → Play Mode → select GestureManager.
- **Limitations**: less faithful to parameter-sync and non-local behavior; some bugs occur *only* under Gesture Manager. **Practical hybrid: run both** — Av3Emulator's simulation with GestureManager's nicer radial menu (Av3Emulator explicitly supports this).

### ClientSim (official, worlds)

Official worlds counterpart (fork of CyanEmu), docs at creators.vrchat.com/worlds/clientsim/. Replicates client behavior in editor: desktop player controller, pickup grabbing (I/J/K/L/U/O + mouse wheel), interacts, UI via VRC_UiShape, stations, Udon variable inspection in Play Mode, EditorOnly deletion on play. Ships with the Worlds SDK — open your scene and press Play. Esc opens settings; SDK tab exposes PlayerData/PlayerObject debug windows.

**Known limitations**: simulates *only* the local player (remote players are stand-ins without full avatar systems); deserialization events fire only for the local player; `SerializationResult.byteCount` means "properties serialized" in ClientSim but "bytes sent" in VRChat. Requires both Unity input systems + Oculus audio spatializer + SDK Build Control Panel layer setup. Known bugs: `Physics.RaycastNonAlloc` misses moved colliders without rigidbodies; occasional exceptions on exiting Play Mode; broken highlight shader on Metal/Mac. The operative rule: **"Always test your world in VRChat before making it public!"**

## 2. Test Worlds and Multi-Client Testing

**Private avatar-test world.** Minimal private world = standard avatar QA sandbox: spawn, good lighting, mirrors, open space for locomotion/FBT. Upload with **visibility Private**. Include: a mirror (both quality modes), varied lighting, ceiling-height open area, second spawn point to test avatar reset. Keep it unlisted.

**Local multi-client testing (Build & Test).** Canonical workflow (docs.vrchat.com/docs/using-build-test): SDK control panel Settings tab → point "VRChat Client" at VRChat.exe; Builder tab → *Setup Layers for VRChat* + *Set Collision Matrix*; set **Number of Clients = 2** → **Build & Test**. Two local client instances launch — the first is instance master/object owner, the second observes. The only way to test ownership transfer, OnOwnershipRequest, synced variables, and "how does my avatar look to someone else" without a second account.

**Build & Reload** accelerates iteration: clients = 0 → Build & Test becomes Build & Reload — rebuilds and teleports all open clients into the new local instance, skipping login. Manual clients join via `VRChat.exe --watch-worlds --profile=0 --no-vr --enable-debug-gui --enable-sdk-log-levels --enable-udon-debug-logging -screen-width 1920 -screen-height 1080`. Caveats: new clients may not join reloaded instances (re-run Build & Reload at 0 to regroup); stations historically misbehave with multiple test clients; EAC can block the SDK's multi-client launcher — the **VRC Quick Launcher** (VCC Tools tab) launches local EAC-free instances with multiple profiles and debug flags.

**Multiboxing for remote-perspective testing** (wiki.vrchat.com/wiki/Multiboxing): two client instances signed into two accounts — log out, restart to login on all instances, sign into secondary accounts first, main last. Same-machine instances share settings and perf cost; two avatars of one user can share a *local offline* world but not the same online instance. Accepted for testing; input-sync automation is not.

**Avatars:** SDK **Build & Test** builds the avatar to `%LocalAppData%\LocalLow\VRChat\VRChat\Avatars\` as a local test avatar ("SDK Test Avatars" — available even below upload trust rank). Local test avatars don't run full server processing — a final uploaded-private-avatar pass is still required. Cross-platform: build/test the Android/iOS variant separately; **cross-platform parameter sync matches by position and type, not name** — reordering silently breaks sync.

## 3. Test Checklists

### Pre-upload avatar checklist

**Face & eyes**
- [ ] **Visemes**: lip-sync mode set (Viseme Blend Shape recommended), Auto Detect run, all 15 shapes mapped; speak into mic in Play Mode and check each viseme; verify `sil` exists (wiki.vrchat.com/wiki/Visemes).
- [ ] **Blink**: blendshape/bone mapped; test blink-rate sliders; eye-look eyelids don't conflict with animated eyelids.
- [ ] **Eye look**: configure straight/up/down/left/right rotations; test preview sliders + gaze-at-remote-player; disable eye-look in states where custom eye animation plays.

**Body & tracking**
- [ ] **Viewpoint** between the eyes; first-person: head hidden or HeadChop configured (hair survives via `Always Apply`/`VR Only`); mirror clone shows the full head.
- [ ] **FBT poses**: calibrate with 3-point and 6-point proxies; hip/chest/feet spacers; "Use Auto Footstep"/"Force Locomotion Animations" match design; test prone (Z), crouch (C), T-pose calibration.
- [ ] **T-pose/rest pose**: T-poses correctly; hands don't deform on gesture transitions.

**Logic**
- [ ] **All toggles** on AND off; no residue (off-states restore every animated property); rapid toggling tested.
- [ ] **Gestures**: all 8 per hand (Shift+F1–F8 desktop); gesture-weight blendtrees at 0/50/100%.
- [ ] **Radial puppets**: sweep full range; puppet defaults set; radial + menu + Contact writes to the same parameter agree.
- [ ] **Saved states**: Save/Load, leave and rejoin, parameters restore; Reset Avatar long-press returns to defaults.
- [ ] **First person + mirrors**: every toggle in first person; both mirror quality modes; shadows (shadow clone renders head full-scale — hair/face can look wrong in shadow only).
- [ ] **Remote view**: Build & Test 2-client or Av3Emulator non-local clone — animated toggles, particles, layer suppression behave for the observer.

**Performance & platform**
- [ ] Rank reviewed (≤ Medium for public); size limits — **PC: 200 MB download / 500 MB uncompressed; Android: 10 MB / 40 MB** (SDK 3.7.0 enforces PC limits at build; not enforced on Build & Test).
- [ ] Quest/Android variant built; both share blueprint ID.

### World checklist

- [ ] **Spawn**: spawn points ordered/shuffled as intended, none inside geometry, respawn height above playable space (SDK 3.7.0+ gizmo).
- [ ] **Interactions**: every interactable used from both master and non-master; pickups grab/throw/drop; stations enter/exit; UI via raycast and touch.
- [ ] **Late joiner**: join mid-session with all systems non-default — the #1 world bug class. Verify OnDeserialization from storage, OnPlayerJoined ordering, correct synced state.
- [ ] **Multiple players**: 2-client Build & Test for ownership/master transfer (`OnMasterTransferred`, SDK 3.7.0+) and networked interactions; Players debug view (Right Shift+~+4) shows ping/delay per player.
- [ ] **Quest/mobile**: Android variant built; real-device testing required — shaders, performance, input, component-removal all differ; use Android Build & Test with a real device.

## 4. Common Upload Error Catalog

| Error | Cause | Fix |
|---|---|---|
| **Size exceeded** ("download/uncompressed size exceeds the limits") | Bundle over platform caps (200/500 MB PC, 10/40 MB Android). Enforcement: 3.5.2 (Android), 3.7.0 (PC); older SDKs fail *server-side* instead of warning | Reduce max texture size, merge textures/materials, power-of-two textures, trim audio (mono/lower quality), remove unused blendshapes/keyframes. "High Quality" texture compression *increases* both sizes |
| **"VRCExpressionParameters has too many parameters defined"** | Synced parameters exceed **256 bits**. Also a client-side hard limit of **256 total** parameters regardless of sync status — going over breaks *all* parameters and defaults in-game even when the SDK accepted the upload | Open the Expression Parameters asset (SDK 3.7.0 editor shows a byte-meter), delete blank/unused synced entries; convert to local-only where sync isn't needed; Modular Avatar's "Expression Parameters Usage" window shows the running total |
| **"Avatar validation failed"** (generic) | SDK-speak for "check the Console" — real cause is elsewhere. Classic trap: SDK reports an unrelated error when the actual problem is a broken script or an MA Parameters entry set to "Animator only" | Read the Console tab, not the dialog. Strip recently added components; check MA parameter types |
| **Missing scripts / compile errors** | Prefab references missing assemblies, or console errors block build | Clear console, reinstall missing VPM dependencies, remove orphaned "Missing Script" components |
| **Hierarchy / duplicate parameters** | Same parameter with different types across controllers; FX references parameters absent from Expression Parameters | Align names/types across FX, Expression Parameters, menus; SDK 3.7.0's rebuilt panel flags most mismatches |
| **Dynamic Bones present** | Upload blocked since SDK 3.7.0 — auto-conversion mandatory | Convert via the SDK prompt |
| **Server-side failures after "successful" build** | Usually stale SDK (size enforcement and security checks run server-side) | Update SDK to ≥3.5.2 (Android) / 3.7.0 (PC) |

## 5. Debugging Deployed Content

**Output logs.** One log per launch: `%AppData%\..\LocalLow\VRChat\VRChat\output_log_yyyy-MM-dd_HH-mm-ss.txt` (retained 24 hours). Contains Udon debug output, avatar-loading errors, world/networking messages; SDK log-level output with `--enable-sdk-log-levels` and `--enable-udon-debug-logging`. For Udon logic: scatter `Debug.Log` with unique markers before each critical operation (docs.vrchat.com/docs/debugging-udon-projects).

**World debug overlay.** **Right Shift + ~ (backtick) + number** — must be Right Shift (on non-QWERTY, the *position* left of 1). Requires `--enable-debug-gui` or Quick Menu Settings toggle. Pages: **2** Version & Info; **3** Log Viewer (live output log, type filters; background collection is performance-costly); **4** Players (master/local/VR flags, ping, send interval, delay, "Suffering" indicator = Udon sending too much data); **6** Net Objects (per-object owner, sleep state, bytes/serialization — requires World Debugging enabled or ownership); **7–9** world-space overlays (Network Object Info atop synced objects, Player Info at players' feet). Performance tab with graphs opens from the menu (no keybind).

**Avatar-side in-game debugging.** Circle menu `Options → Avatar → Debug` — live parameter values + active state of every playable layer; the definitive tool for "is the parameter updating but the transition failing, or neither." Shift+F1–F8 force gestures (Left Shift = left hand, Right Shift = right). `Options → Avatar → Avatar Overlay` visualizes PhysBone/Contact volumes live; Shift+F11 = Safe Mode isolation. Full desktop key table: docs.vrchat.com/docs/keyboard-and-mouse.

## 6. PhysBones/Contacts: Editor vs Runtime

Official position ([debugging-avatar-components](https://creators.vrchat.com/avatars/avatar-components/debugging-avatar-components/)): **PhysBones and Contacts run in the Unity editor as they do in VRChat** — Play Mode without any emulator exercises the SDK's native implementations; grab/pose PhysBones through a MainCamera-tagged Game view. Constraints run in edit mode (toggle: SDK Settings → "Execute VRChat Constraints in Edit Mode").

Real differences:
- **The client layer, not the components.** Missing in-editor: real controller input, permissions interplay (a receiver with Allow Self=false behaves differently when *you* test), avatar-interaction settings, the four-collider global limit per avatar, grabbing bounding-box caps, mirroring/shadow-clone rendering. A contact system can be perfect in Play Mode and dead in-game because *the other player* has interactions disabled.
- **No animation-driven motion** in Play Mode — move the avatar root from Scene view to induce swing (Av3Emulator issue #136).
- **Runtime tweak regressions**: SDK 3.8.0 (Apr 2025) broke real-time PhysBone tuning in Play Mode (feedback thread; check SDK version before blaming your rig).
- **Emulator coverage**: Gesture Manager's Clickable Contacts (mouse-driven senders); Av3Emulator lets you manually set a receiver's parameter to test transitions (VRChat staff's recommendation). Beware: user parameters starting with `/` (e.g., `/HandL`) collide with VRChat's internal namespace and are ignored — a frequent "works in editor, not in game" cause.

Community-standard workflow — four escalating levels (vrcreators.net): static validation → Unity Play Mode → local Build & Test with Avatar Overlay → uploaded private avatar with a real second user. Each level catches a failure class the previous cannot.

## 7. Automated Test Frameworks

- **ClientSim's built-in framework** (clientsim.docs.vrchat.com/systems/automated-testing/): fixtures `ClientSimTestBase` (prefab-level) and `ClientSimWorldTestBase` (full-world). Enable by adding `"testables": ["com.unity.inputsystem", "com.vrchat.clientsim"]` to the package manifest; tests populate Unity's Test Runner. `ClientSimTestHelpers` verifies actions against ClientSim events; `ClientSimTestInput` injects desktop input. Constraint: ClientSim initializes via `InitializeOnLoad` → tests run with **Domain Reloading disabled** (handled by the fixture base classes); only one world test at a time (play mode starts once).
- **Unity Test Framework + Av3Emulator for avatars**: no official path; the pattern is a UTF suite validating avatar structure in EditMode (descriptor present, viseme blendshapes found, parameter names consistent between FX and Expression Parameters, bit budget ≤ 256, menus reference defined params) via reflection over VRC SDK assets, then PlayMode tests with Av3Emulator asserting animator outcomes. Feasible but fragile — SDK upgrades break assumptions.
- **CI**: no official headless upload API; SDK builds run inside the Editor. Practical CI: (1) compile/validate + UTF EditMode headlessly, (2) optionally `-batchmode -executeMethod` calling SDK build methods to produce local bundles and assert size limits, (3) artifacts for manual Build & Test. Uploading in CI isn't viable for most creators (interactive login). **Realistic consensus: automate validation and structure checks; keep the final build-and-test human-in-the-loop.**

## 8. Beta Branch Testing

**Client openbeta:** Steam (Library → VRChat → Properties → Betas → Open Beta) or Quest (Library → VRChat → `...` → Details → Release Channels). Meta PCVR has no beta channel. As of 2026 the openbeta branch carries Unity 6 builds (the `unity-6` branch folded into openbeta in 2026.3.3); side branches like `steam-audio-beta` exist for specific features.

**SDK betas:** VCC → Settings → Packages → **Show Pre-Release Packages** → pick `-beta` versions per package.

**Risks and discipline:**
- **Network incompatibility**: non-network-compatible betas can't see live users; content (notably Items/spawned objects) is explicitly incompatible between beta and live branches (2026.3.2 notes).
- **Client betas don't change what you upload**: "Do NOT upgrade your SDK projects to Unity 6" until announced — client Unity version and SDK-supported Unity are decoupled; mixing is the most common way testers brick their own project.
- **SDK betas can corrupt content**: 3.6.2-beta toggle menu entries were silently broken and needed recreation; beta-saved animations may embed assumptions that change. Always work in version control; never treat a beta-only project as your only copy.
- **Testing content *on* openbeta**: upload from your normal SDK project, join a Friends instance on the beta client. Upload test builds as private avatars/worlds; re-upload finals from a stable SDK.
- **Repro etiquette**: state your branch + build number (Quick Menu Settings → Debug row); retest on live before reporting — the delta between branches *is* the bug.

## Sources

- https://vrc.school/docs/Avatars/Troubleshooting/ · https://github.com/lyuma/Av3Emulator · https://github.com/BlackStartx/VRC-Gesture-Manager
- https://creators.vrchat.com/worlds/clientsim/ · https://clientsim.docs.vrchat.com/systems/automated-testing/
- https://docs.vrchat.com/docs/using-build-test · https://wiki.vrchat.com/wiki/VRC_Quick_Launcher · /wiki/Multiboxing · /wiki/Open_Beta
- https://creators.vrchat.com/avatars/avatar-size-limits/ · /avatars/avatar-components/debugging-avatar-components/
- https://notes.sleightly.dev/VRCExpressionParameters-has-too-many-parameters-defined-b277d6a2efe942eab2acdaabf0f7d95c
- https://help.vrchat.com/hc/en-us/articles/9521522810899 (output logs) · https://docs.vrchat.com/docs/debugging-udon-projects · /docs/keyboard-and-mouse
- https://creators.vrchat.com/worlds/udon/world-debug-views/ · https://wiki.vrchat.com/wiki/Debug_Menu
- https://wiki.vrchat.com/wiki/Visemes · https://vrcreators.net/docs/vrchat/vrchat-avatar-dynamics-tutorial-part-3-how-to-test-4-level-testing
- https://docs.vrchat.com/docs/vrchat-202632-open-beta · https://docs.vrchat.com/docs/vrchat-202633-open-beta
