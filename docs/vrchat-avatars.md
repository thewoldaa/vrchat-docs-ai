# VRChat Avatar Creation: Comprehensive Research Report (as of late 2025 – early 2026)

**Sources of record:** creators.vrchat.com, docs.vrchat.com, wiki.vrchat.com, vcc.docs.vrchat.com, ask.vrchat.com, vpm.anatawa12.com, vrcfury.com.

---

## 1. Toolchain & Pipeline Overview

### 1.1 Environment: VCC + Unity 2022.3.22f1

- **Current Unity version: 2022.3.22f1** (pinned LTS; version numbers below were current at research time and drift — verify in VCC). Do **not** upgrade Unity — content built on other versions won't load.
- **VCC** installs Unity, manages packages via **VPM**, creates Avatar/World templates. Windows 10/11 64-bit full support.
- Workflow: VCC → Create New Project → **Avatars** template → add packages (Modular Avatar, AAO, VRCFury, Poiyomi, lilToon) → Open Project.
- In Unity: `VRChat SDK → Show Control Panel` for auth, build, and performance-rank display. Install Unity with **Android Build Support** (+ iOS).
- Upload requires **New User** trust rank (Steam/Oculus-only accounts can't upload). Build & Test no longer requires a trust rank (SDK 3.9.0+).

### 1.2 Model prep (Blender / Substance)

1. **Modeling** — Blender/Maya. Target ~20–60k tris PC body; **1 skinned mesh** strongly preferred. No non-manifold geometry.
2. **UVs** — one UV set, laid out for atlasing; beware overlapping UVs on mobile bake paths.
3. **Texturing** — Substance/Blender; on mobile texture memory dominates (10–40 MB): packed masks, ≤2K body, ≤1K elsewhere.
4. **Rigging** — **Humanoid** rig required for full AV3. Standard humanoid 55-bone set (Hips…fingers); extra bones (hair/tail/clothes) below.
5. **Blendshapes** — minimum: **15 visemes** (`vrc.v_sil, vrc.v_PP, vrc.v_FF, vrc.v_TH, vrc.v_DD, vrc.v_kk, vrc.v_CH, vrc.v_SS, vrc.v_nn, vrc.v_RR, vrc.v_aa, vrc.v_E, vrc.v_ih, vrc.v_oh, vrc.v_ou`), **blink** (L/R), optional expression/lowerlid shapes.
6. **Export** — FBX with blendshapes, no subdivisions, scale handled by Unity import.

### 1.3 Unity import & SDK3 setup

1. Import FBX → Rig → **Humanoid**, check T-pose mapping.
2. Extract textures/materials, build materials.
3. Add **VRC_AvatarDescriptor**:
   - **Playable Layers** (Base/Additive/Gesture/Action/FX)
   - **Expressions** (Expression Parameters + Expressions Menu assets)
   - **Lip Sync**: *Viseme Blend Shape* (recommended) / *Viseme Parameter Only* / legacy jaw flap
   - **Eye Look** (bones or blendshapes)
4. SDK Control Panel → **Build & Publish for Windows** (or Build & Test local).

## 2. Avatar 3.0 (AV3) System

### 2.1 Playable Layers

Up to five root controllers merged at runtime. **Generic (non-humanoid) avatars: only Base, Action, FX.** Generic setup: `Generic` rig, avatar reference in root Animator, leave root controller blank (stripped at runtime), put everything in playable layers.

| Layer | Purpose | Animates |
|---|---|---|
| **Base** | Locomotion (idle/walk/run/crouch/fall/seated) | Transforms only |
| **Additive** | Additive motion (breathing, sway) | Transforms only |
| **Gesture** | Per-body-part overrides via hand gestures or menu; tail/ear idle via masks | Transforms only |
| **Action** | Full-body override, emotes/dances | Transforms only |
| **FX** | Everything EXCEPT transforms — blendshapes, material swaps, particle toggles | Non-transform properties |

Order: Base → Additive → Gesture → Action → FX; later overrides earlier. Sample controllers in `Packages/com.vrchat.avatars/Samples/AV3 Demo Assets/`. Never assign the same controller to two slots. Avoid unreachable "dead end" states.

### 2.2 Parameters & Sync

- **Built-in** (free, read-only): `Viseme` (int 0–14), `Voice` (float), `GestureLeft/Right` (int **0–7**: Neutral, Fist, HandOpen, FingerPoint, Victory, RockNRoll, HandGun, ThumbsUp), `GestureLeft/RightWeight`, `VelocityX/Y/Z` + `VelocityMagnitude`, `Upright`, `AngularY`, `Grounded`, `InStation`, `Seated`, `AFK`, `TrackingType`, `ExpressionMode`, `MuteSelf`, `InMute`, `Earmuffs`, `IsLocal`, `IsOnFriendsList`, `SuppressRemoteVoice`, scale params `ScaleFactor`, `ScaleFactorInverse`, `EyeHeightAsMeters`, `EyeHeightAsPercent`.
- **Custom** in an **Expression Parameters** asset (`VRCExpressionParameters`): name, type, default, **Saved** + **Synced** flags. Writable by: menus, Parameter Drivers, PhysBones, Contact Receivers, OSC.

### 2.3 The 256-bit budget

| Type | Range | Cost |
|---|---|---|
| `int` | 0–255 | **8 bits** |
| `float` | −1.0 to 1.0 | **8 bits** (quantized 1/127) |
| `bool` | true/false | **1 bit** |

- Total synced budget: **256 bits** (32 floats/ints or 256 bools). Hard cap 8192 total custom params (synced+local). Saved-but-unsynced costs 0.
- **Cross-platform sync is positional, not by name**: PC + Quest variants must keep the parameter list identical in order.

### 2.4 Expression Menus

`VRCExpressionsMenu` tree; root shown via radial menu. **Max 8 controls per page.** Types: Button, Two-Axis Puppet, Four-Axis Puppet (incl. **Radial Puppet** — one float for sliders/hue), Toggle, Sub-menu. Menus themselves cost no sync budget.

## 3. VRChat-Specific Components

- **VRC_AvatarDescriptor** — playable layers, expressions, lip-sync, eye-look, colliders, view/audio position, T-Pose/IK/Sitting poses.
- **Visemes** — 15 `vrc.v_*` blendshapes (Shape mode recommended) or `Viseme` int parameter.
- **Eye Look** — bones or blendshapes; per-eye min/max yaw/pitch, blink + lower-lid shapes, emotion shapes.
- **VRC PhysBone** — replaces legacy DynamicBone (no longer allowed in uploads; VRCFury/AAO convert). Per-chain: pull, spring, stiffness, gravity, limits, stretch, colliders. Costs "Components", "Affected Transforms", "Colliders", "Collision Check Count" separately. SDK 3.10.x: multi-edit roots, collider-jitter fix.
- **PhysBone Collider** — sphere/capsule/plane on body bones (e.g., thighs for skirts).
- **Contacts** — **Sender** (emits tag+radius) and **Receiver** (writes float 0–1 into FX parameter). Enables poke/grab/proximity. Receivers can be Local Only (excluded from rank count). 2026: `matchingTags` on ContactExitInfo.
- **VRChat Constraints** — Parent/Rotation/Position/Scale/Aim/Look At/Roll/World Scale; better integrated than Unity constraints; Count + Depth ranked.

## 4. Expressions, Animations & Animator Details

- **Toggles**: two-state layer per toggle, or single-state blendtree keyed on the parameter. Always author explicit off-states.
- **Gestures**: Gesture layer driven by `GestureLeft/Right` (0–8) with `GestureWeight` analog blends, masked per-hand.
- **Radial puppet**: menu float (−1..1) → blendtree of morphs or hue-shift shader property.
- **Blink sync**: handled by descriptor — blinks sync to remote viewers without animations.
- **Write Defaults**: best practice 2025+ = **WD OFF consistently on every state in every layer** with complete off-state animations (VRCFury computes defaults automatically). Mixed WD causes state bleed. **Direct Blend Trees** (one state per layer, continuous parameter mapping) avoid many WD issues.
- **Humanoid vs Generic**: humanoid = 5 layers + IK + FBT; generic = 3 layers, no humanoid IK, but expressions/contacts/physbones work. Generic suits quadrupeds/vehicles.
- **Extra Animator components cost rank** — keep everything in playable layers.

## 5. Non-Destructive Workflows (the 2025–2026 standard)

Build-time hooks (`IVRCSDKPreprocessAvatarCallback`, ordered by `callbackOrder`) — scene avatar never modified.

- **NDMF** — shared foundation ordering tools (Resolving → Generating → Transforming → Optimizing).
- **Modular Avatar** (nadena.dev) — component-based assembly: `MA Avatar Root`, `Bone Proxy`, `Merge Animator`, `Merge Armature`, `Menu Item/Installer`, `Parameters` (consolidation + auto-renaming). Callback order ≈ −25 (early).
- **AAO: Avatar Optimizer** (anatawa12) — `Trace And Optimize` (one-click: unused blendshapes/PhysBones removal, never-animated-bone merge, mesh merge), `Merge Skinned Mesh`, `Remove Mesh By BlendShape/Box/Sphere`, `Material Modifier`, etc. Post-optimization the SDK rank display is unreliable — use **"Actual Performance Window"** gist in play mode.
- **d4rkAvatarOptimizer** — alternative: mesh combination, texture atlasing, animation merging. "Create Optimized Copy" disabled with VRCFury present (use "Apply on Upload").
- **VRCFury** — GUI: **Full Controller**, **Toggle** (no clips needed), **Gestures**, Haptic Socket, DPS/TPS, "Fix Write Defaults", clip path rewriting. Runs after MA/AAO.
- Also: **TexTransTool** (non-destructive atlasing), **lilNDMFMeshSimplifier** (decimation), **Av3Emulator** (play-mode testing — enable "Run Preprocess Avatar Hook"), **GestureManager**.

**Typical modern stack:** Modular Avatar (assembly) → TexTransTool (atlas) → AAO Trace And Optimize → VRCFury where needed → Av3Emulator to verify.

## 6. Performance Ranks — Exact Limits

All GameObjects/Components count, **including disabled ones**. Disabling Mesh Read/Write anywhere = **Very Poor** (upload blocked). Bold-stat violations above your Minimum Displayed Rank fully block the avatar (fallback robot); non-bold violations partially strip components. Mid-2025: ranks computed **server-side** (may differ slightly from SDK preview).

### PC limits (max per rank)

| Stat | Excellent | Good | Medium | Poor |
|---|---|---|---|---|
| Triangles | 32,000 | 70,000 | 70,000 | 70,000 |
| Bounds Size | 2.5 m³ | 4 m³ | 5×6×5 m | 5×6×5 m |
| **Texture Memory** | 40 MB | 75 MB | 110 MB | 150 MB |
| Skinned Meshes | 1 | 2 | 8 | 16 |
| Basic Meshes | 4 | 8 | 16 | 24 |
| Material Slots | 4 | 8 | 16 | 32 |
| PhysBone Components | 4 | 8 | 16 | 32 |
| PB Affected Transforms | 16 | 64 | 128 | 256 |
| PB Colliders | 4 | 8 | 16 | 32 |
| PB Collision Checks | 32 | 128 | 256 | 512 |
| Contacts | 8 | 16 | 24 | 32 |
| Constraints Count/Depth | 100/20 | 250/50 | 300/80 | 350/100 |
| Animators | 1 | 4 | 16 | 32 |
| Bones | 75 | 150 | 256 | 400 |
| Lights | 0 | 0 | 0 | 1 |
| Particle Systems (active) | 0 | 4 (300) | 8 (1000) | 16 (2500) |
| Trail/Line Renderers | 1/1 | 2/2 | 4/4 | 8/8 |
| Cloths (verts) | 0 | 1 (50) | 1 (100) | 1 (200) |
| Physics Colliders/Rigidbodies | 0 | 1 | 8 | 8 |
| Audio Sources | 1 | 4 | 8 | 8 |

### Mobile (Android/Quest & iOS) limits

| Stat | Excellent | Good | Medium | Poor |
|---|---|---|---|---|
| Triangles | 7,500 | 10,000 | 15,000 | 20,000 |
| Texture Memory | 10 MB | 18 MB | 25 MB | 40 MB |
| Skinned Meshes | 1 | 1 | 2 | 2 |
| Basic Meshes | 1 | 1 | 2 | 2 |
| Material Slots | 1 | 1 | 2 | 4 |
| Animators | 1 | 1 | 1 | 2 |
| Bones | 75 | 90 | 150 | 150 |
| PB Components/Transforms | 0/0 | 4/16 | 6/32 | 8/64 |
| PB Colliders/Checks | 0/0 | 4/16 | 8/32 | 16/64 |
| Contacts | 2 | 4 | 8 | 16 |
| Constraints Count/Depth | 30/5 | 60/15 | 120/35 | 150/50 |
| Particle Systems | 0 | 0 | 0 | 2 (200 active, 400 polys) |
| Trail/Line Renderers | 0 | 0 | 0 | 1 |

**Download size**: hard caps are **PC: 200 MB download / 500 MB uncompressed; Android/iOS: 10 MB download / 40 MB uncompressed** (enforced in SDK 3.5.2 Android / 3.7.0 PC). Mobile practical guideline: 5–8 MB total — Toon Standard's texture set frequently busts this → mask packing + atlasing essential.

**Fallback avatars**: must be Good+ by Android limits, uploaded first on Android with "Is a Fallback" flag, then re-uploaded on Windows/iOS to the same blueprint ID. Impostors (auto-generated) show before fallbacks when the avatar is merely missing for your platform.

## 7. Shaders, Mesh Combining, Atlasing

- **PC shaders**: **Poiyomi Toon** (Western de-facto, feature-heavy: audio-link, TPS, UV-tile-discard) and **lilToon** (JP dominant, cheaper at similar visuals). Both PC-only.
- **Mobile shaders**: VRChat native set only — `VRChat/Mobile/Standard Lite`, `Toon Lit`, and **Toon Standard** (SDK 3.8.1, 2025): ramp shadows, normal maps, emission, matcap, rim, metallic, hue shift. Mask-channel packing essential; **VRCQuestTools 2.10+** automates conversion from lilToon. Unlit/Transparent variants for mobile transparency fixes.
- **Quest conversion workflow**: VRCQuestTools ("Convert Avatar for Android") + TexTransTool atlasing + lilNDMFMeshSimplifier + material merge + bake transparency.
- **Mesh combining/atlasing**: fewer material slots = fewer draw calls. Tools: AAO Merge Skinned Mesh, d4rk, TexTransTool AtlasTexture. **Caution**: blanket-combining multi-outfit avatars can *worsen* performance (all textures buffered, everything skinned) — combine per-state.
- **VRCFallback tags**: on PC upload, SDK assigns fallback mobile shaders per material so PC-only shaders degrade gracefully on Quest.

## 8. 2025–2026 State

- Unity 2022.3.22f1 pinned; Oct 2025 Unity vulnerability does not apply to VRChat content.
- **SDK 3.10.x**: menu editor UX, Parameter Driver random-no-repeat, Contact matchingTags, Toon Standard family, PhysBone multi-edit, server-side perf calc.
- Platforms: Windows, Android (Quest 2+/PICO/Vive XR/phones), **iOS 17+** (out of beta; same mobile table).
- Official roadmap: native SDK-side auto-optimization/atlasing (starting Standard Lite), Soba.

**Bottom-line recommended workflow (2026):** Blender (≤70k tris PC / ≤10k mobile, 1 skinned mesh, 15 visemes + blink, humanoid rig) → textures with packed masks → VCC avatar project (2022.3.22f1) → Modular Avatar + VRCFury or hand-built FX (WD OFF, ≤256-bit params) → lilToon/Poiyomi PC + Toon Standard mobile → AAO + VRCQuestTools + TexTransTool for Android → verify Av3Emulator + Actual Performance Window → upload PC + Android/iOS to same avatar ID.

### Key sources

- https://creators.vrchat.com/avatars/avatar-performance-ranking-system
- https://creators.vrchat.com/avatars/animator-parameters/
- https://creators.vrchat.com/avatars/playable-layers
- https://creators.vrchat.com/sdk/upgrade/current-unity-version
- https://docs.vrchat.com/docs/avatar-fallback-system
- https://docs.vrchat.com/docs/vrchat-safety-and-trust-system
- https://creators.vrchat.com/avatars/avatar-optimizing-tips
- https://creators.vrchat.com/platforms/
- https://vrcfury.com/ · https://vpm.anatawa12.com/avatar-optimizer/en/docs/tutorial/basic-usage/
