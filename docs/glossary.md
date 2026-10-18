# VRChat Creator Glossary

*Jargon decoder for AI assistants and newcomers. One line each.*

## Core

| Term | Meaning |
|---|---|
| **AV3 / Avatar 3.0** | VRChat's avatar system: 5 playable animator layers (Base/Additive/Gesture/Action/FX) + expression menus/parameters |
| **SDK3** | Current VRChat SDK generation (SDK2 uploads permanently shut off Nov 2025) |
| **VCC** | VRChat Creator Companion — official Unity project/package manager |
| **VPM** | VRChat Package Manager — package system used by VCC (`Packages/` folder, `vpm-manifest.json`) |
| **VRCFury / MA / AAO** | VRCFury (GUI toggle/gesture tool) · Modular Avatar (non-destructive prefab assembly) · AAO: Avatar Optimizer (anatawa12's optimization suite) |
| **NDMF** | Non-Destructive Modular Framework — the build-hook foundation all three run on; phases: Resolving→Generating→Transforming→Optimizing |
| **WD (Write Defaults)** | Animator state setting; ON = writes defaults when unkeyed, OFF = keeps last value. Community standard: OFF everywhere + explicit off-states |
| **AAP** | Animated Animator Parameter — a parameter driven by an animation curve; overrides menu writes |
| **FBT** | Full-Body Tracking (3/6/11-point tracker setups) |
| **IK 2.0** | Current VRChat inverse-kinematics solver (2022.2.1+): saved calibrations, 11 tracked points, elbow anti-clip |

## Avatar

| Term | Meaning |
|---|---|
| **PhysBone** | VRChat's physics component (replaced DynamicBone — legacy, now blocked at upload) for hair/tail/skirt motion |
| **Contacts** | VRC Contact Sender/Receiver — proximity-triggered parameter system (poke, hug) |
| **Visemes** | 15 mouth blendshapes (`vrc.v_sil`…`vrc.v_u`) driven by lip sync |
| **HeadChop** | System hiding head mesh in first person; "Visible Head Accessory" (MA) re-reveals hair/hats via proxy bones |
| **Impostor** | Auto-generated billboard body-double shown when an avatar is blocked or platform-missing (replacing fallbacks since 2025) |
| **Fallback** | Deprecated simple avatar shown when the real one can't load; custom fallbacks must be Good+ on Android |
| **Blueprint ID** | Immutable content ID (`avtr_…`/`wrld_…`) — re-uploading to it updates in place |
| **Performance rank** | Excellent→Good→Medium→Poor→Very Poor; static analysis (tris, texture memory, PhysBones…); server-computed since Aug 2025 |
| **Texture memory** | GPU footprint of all textures incl. mipmaps — the dominant avatar budget (PC Excellent ≤40 MB) |
| **VRCFT** | VRCFaceTracking — OSC bridge feeding Unity-Expressions face/eye tracking from webcams/trackers |

## World

| Term | Meaning |
|---|---|
| **Udon / UdonSharp (U#)** | VRChat's sandboxed scripting VM / its C# compiler. No async, no try/catch, no generics |
| **Soba** | Announced Udon successor (C#-parity); in development as of 2026 |
| **Owner** | The one player whose writes to synced vars are authoritative (`Networking.SetOwner`) |
| **Master** | First player in the instance; re-elected on leave (`isMaster`) |
| **Manual sync** | Reliable sync mode; requires `RequestSerialization()` after changes |
| **Network event** | One-shot remote method call — NOT replayed for late joiners |
| **Late joiner problem** | Events aren't replayed to joiners; sync state via synced variables + `OnDeserialization` |
| **Station** | Seat/sit component (`VRC_Station`) |
| **Pickup** | Grabbable object component (`VRCPickup`) |
| **ProTV / VideoTXL / USharpVideo** | Community video-player systems (ProTV = current standard) |
| **AudioLink** | Audio-reactive system broadcasting `_AudioTexture` (4 bands + delay lines) to all shaders in the instance |
| **LTCGI** | Realtime area-light/IBL system for screens and emitters (attribution required) |
| **VRC Light Volumes (VRCLV)** | Per-voxel SH light-volume system for avatars/worlds (REDSIM; supported by lilToon 1.10+, Poiyomi) |
| **ClientSim** | Official in-editor world simulator (play mode, no networking) |
| **ClientSim / CyanEmu** | (CyanEmu = ClientSim's ancestor) |
| **Community Labs** | Public-testing ground for new worlds; good ones auto-graduate to Public |
| **Instance types** | Public, Friends(+), Invite(+), Group(+), Group Public — plus age-gated (~ageGate) |
| **Trust rank** | Visitor → New User (can upload) → User (can publish worlds/reports) → Known User → Trusted User |
| **Nuisance** | Trust flag making your voice/avatars blocked by default |
| **Persistence / PlayerData / PlayerObjects** | Per-player saved state (100 KB each, live since Nov 2024); save before `OnPlayerLeft` — too late |
| **VRCJson / DataList / DataDictionary** | Udon's JSON + container types (since 3.7.4) |

## 3D pipeline

| Term | Meaning |
|---|---|
| **FBX** | Autodesk interchange format; VRChat's standard path (Blender→Unity). Watch Apply Scalings/units |
| **glTF/GLB** | Khronos open format ("JPEG of 3D"); VRM sits on top |
| **VRM** | glTF-based humanoid-avatar format (0.x z− forward, 1.0 z+ forward); VRoid ecosystem |
| **PMX** | MMD model format; import via MMD Tools |
| **Blendshape / Shape key** | Per-vertex morph target (Unity term / Blender term) |
| **Texel density** | Pixels per meter of surface — keep consistent; faces get 2–4× body |
| **Channel packing** | Multiple grayscale masks in one RGBA texture (R/G/B/A slots) — mandatory for mobile texture budgets |
| **BC7 / DXT1/5 / BC5 / BC4** | PC GPU texture compression formats (RGBA / RGB / normals / mask) |
| **ASTC** | Mobile GPU texture compression (Quest/iOS); BC formats don't work on Quest |
| **Mipmaps** | Precomputed halved texture chain (+33% memory; required) |
| **Crunch** | Download-size-only compression; saves ZERO texture memory |
| **Draw call** | One material on one mesh per pass — minimized via atlasing/merging |
| **Atlasing** | Packing multiple materials' textures into one atlas so one material covers all |
| **Retopology** | Rebuilding sculpted mesh as clean low-poly quads |
| **T-pose / A-pose** | Bind poses; VRChat upload requires T; pose and rig must agree |
| **Digitigrade / plantigrade** | Toe-walking vs flat-footed leg rigs (furry avatars toggle between) |
| **Cats / Tuxedo** | Blender add-ons (Neoneko-maintained Cats Unofficial; Tuxedo for atlassing/decimation) |
| **GoGoLoco (GGL)** | Community-standard locomotion prefab (desktop poses/crawl/fly); ships as avatar layer |
| **SPS / DPS / TPS** | Plug-and-socket interaction systems (VRCFury's SPS is the current one) |
| **FT** | Face Tracking (tag on avatar products) |
| **Booth / Gumroad / Jinxxy** | Asset marketplaces (JP / EN general / EN community) |
| **Tilia / Thunes** | VRChat's payment partner for Creator Economy payouts |
| **VRC+ (VRChat Plus)** | $9.99/mo subscription (trust boost, gift drops, Age Verification eligibility) |
