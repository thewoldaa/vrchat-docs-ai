# The Modular Avatar / VRCFury / AAO Module Ecosystem for VRChat Avatars (2025–2026)

*Based on direct fetches of official docs (modular-avatar.nadena.dev, vrcfury.com, vpm.anatawa12.com). Features not verifiable in current docs flagged.*

---

## 1. Modular Avatar: Complete Component Reference

Modular Avatar (MA, bd_/anatawa12, VCC package `nadena.dev.modular-avatar`) is a **non-destructive prefab composition framework**: drop prefabs onto your avatar, add MA components, everything resolves at build time via NDMF — your scene is never permanently modified. MA 1.12–1.14 era (2025).

**MA Avatar Root** — marks the avatar root; required on the same GameObject as the VRC Avatar Descriptor for MA to process. If nothing is processed, check this plus "Apply on Play".

**Bone Proxy** — reparents an object from your prefab into an existing avatar bone at build time. Attachment modes: *As child at root* (zeroes local position — recommended for non-avatar-specific prefabs), *As child keep world pose* (precise, avatar-specific). "Match Parent Scale" compensates for scaled parents. MA rewrites animator paths referencing the old location. **Gotcha:** not for clothing — use Merge Armature.

**Merge Animator** — adds an entire animator controller to a chosen layer (FX, Action, etc.). Options: *Path mode* — Relative (paths relative to the component; prefab works anywhere; humanoid bone animations always apply avatar-wide) vs Absolute. *Layer priority* (higher overrides; existing animator = 0). *Merge mode*: Append or Replace. *Match Avatar Write Defaults* auto-adjusts your states. **Gotchas:** two Replace-mode Merge Animators on the same layer is an error; `VRCAnimatorLayerControl` resolves only within the same animator; some blend trees can't convert to WD-off.

**Merge Armature** — the clothing workhorse. Merges a clothing prefab's armature onto the avatar's, matching bones by name with prefix/suffix stripping. Rewrites SMR bone references, PhysBone/contact targets, animator references; unmatched bones become children of the matched parent; leftover nodes cleaned unless they hold components. Position lock: Not locked / Base⇒Target (unidirectional, default after "Setup Outfit") / Base⇔Target (bidirectional — prefabs that animate the base avatar, e.g. hair physics). Nested merges supported since 1.7.0. **Gotcha:** assumes target bones exist and don't move — avatar-specific, not for generic multi-avatar prefabs (that's VRCFury Armature Link or Bone Proxy).

**Menu Item / Menu Group / Menu Installer / Menu Install Target** — the menu authoring stack. *Menu Item* defines one control (button/toggle/submenu/two- & four-axis puppet); the GameObject's name is the label. Submenus from a VRC Expressions Menu asset or **Children** (Menu Items on child objects), with **automatic pagination** over the 8-control limit. *Automatic parameter creation*: naming an undeclared parameter creates it (saved/synced/Is Default checkboxes). *Menu Group* bundles items for installation elsewhere. *Menu Installer* installs a menu asset or its children — users click "Select Menu" and pick a destination. *Menu Install Target* installs into another prefab's installed menu. **Gotcha:** unbound Menu Items (no parent/Group/Installer) silently do nothing; multiple "Is Default" items = undefined behavior.

**MA Parameters** — declares animator parameters with type (Bool/Int/Float/Animator Only/Prefix), default, synced/saved flags, and **automatic renaming** to avoid clashes between prefabs (or deliberate linking: rename two gimmicks to the same name). Prefix mode handles PhysBone-generated parameter families. Renamings cascade down the hierarchy. "Override Animator Defaults" rewrites controller defaults. **Gotcha:** Saved resolution takes the outermost component; siblings → saved if *any* says so.

**Mesh Settings** — sets Anchor Override (light probes) and Bounds (Root Bone) for all meshes under an object, with Inherit/Set/Don't Set/Set-or-Inherit modes. "Setup Outfit" auto-adds it. **Gotcha:** bounds only affect Skinned Mesh Renderers.

**Material Setter / Material Swap (Reactive)** — *Material Setter* targets specific renderers; *Material Swap* matches by material ("when this object is active, swap X → Y under Target Root"), with **Quick Swap** browsing modes for cycling color variants in-editor. Both reactive (§2). (Older "Material Modifier" concept = Material Setter + Object State now.)

**Object Toggle / Shape Changer / Scale Adjuster** — see §2. *Scale Adjuster*: non-uniform bone scaling without shearing rotated children — for fitting outfits from other avatars. Only cooperates with the dedicated scale tool, not the combined gizmo.

**Platform Filter** — include/exclude GameObjects per platform (VRChat, Resonite…). Mixing Include+Exclude on one object is an error. Often redundant (Merge Animator is already platform-aware).

**World Fixed Object / World Scale Object** — anchors a prop to the world with one shared generated constraint (constant cost whether 1 or 50). Position via Parent Constraints.

**Visible Head Accessory** — makes head-attached objects visible in first person via VRCHeadChop + mesh surgery (triangles spanning visible/hidden bones get proxy bones + re-weighted vertices). **Gotchas:** can't be a child *of* a PhysBone chain; applying to all head children means your bangs block your view.

**Others:** Blendshape Sync (mirror body blendshapes onto outfits), PhysBone Blocker, Convert Constraints (Unity→VRC), Floor Adjuster, Move Independently, Move To Outfit Root, MMD Layer Control, Remove Vertex Color, Rename VRChat Collision Tags, Replace Object, **Sync Parameter Sequence** (stable cross-platform parameter order), Global Collider, Merge Motion (Blend Tree). Full list: https://modular-avatar.nadena.dev/docs/reference/

## 2. Reactive Components: The Animator-Killer

MA's reactive system (https://modular-avatar.nadena.dev/docs/reference/reaction/) is the flagship reason to stop hand-building animator layers. A reactive component is **active** when its GameObject and all parents are active, *or* when it sits on/under a selected Menu Item. Effects compose; **lowest component in the hierarchy wins** on conflicts. Compiled into animator logic at build time.

**Canonical example — shirt hides bra chains:** Shape Changer on the shirt targeting the body (Delete/shrink chest blendshapes) + Object Toggles on the shirt's Menu Item turning off the bra-chain object and on the shirt. Enable shirt → chains vanish, chest flattened; disable → everything restores. No clips, no layers, no parameters.

Components: **Object Toggle** (controls other objects' active states; deliberate one-frame delay so removing outer clothing doesn't flash naked frames; A→B→C chains cascade), **Shape Changer** (Set or **Delete** blendshape values; Delete actually removes polygons — real poly savings; Threshold widens affected vertices; don't use on blendshapes other animations drive), **Material Setter/Swap**, **Mesh Cutter** (delete/hide vertices by axis/bone/mask/shape/UV tile). **Invert condition** flips the trigger. **Reaction Debugger** (right-click → Show Reaction Debugger) simulates toggles and shows which reaction fires.

**Caveat:** reaction timing (one-frame delays) is an implementation detail subject to change — don't choreograph frame-precise effects; use Merge Animator for those.

## 3. VRCFury: Complete Module List

VRCFury (vrcfury.com): components + "Actions" generating entire FX layers, menus, and parameters at build time, on a copied avatar ("your files are never touched"). Works with WD ON or OFF; auto-computes default states so you only author "on" animations.

**Toggle** — the core. Menu path with slashes creates submenus; Add Action + Object Toggle for zero-clip props, or animation clips. Options: *Saved Between Worlds*, *Use Slider Wheel* (0%=default, 100%=enabled), *Default On*, *Exclusive Tags* (mutually exclusive sets — tag `tops`/`bottoms`, full-body outfits tagged `tops,bottoms` turn both off), *Security* flag, *Hold Button*, *Separate Local State*, *Enable Transition State*, *Use a Global Parameter* (OSC access), custom icons.

**Full Controller** — merges an entire pre-made animator + menu + parameters. Parameters get a `VF##` prefix to isolate them unless listed under Global Parameters (`*` = all global, `!name` = exceptions). Path Rewrite Rules repurpose avatar-specific controllers as portable props. **Gotcha:** author merged controllers with WD **on** — converting off→on loses stickiness.

**Armature Link** — VRCFury's Merge Armature equivalent, more tolerant: recursive matching, automatic suffix/prefix prediction, transform alignment, rewrites animations pointing at pre-merge paths. Unmatched bones reported in Debug Info.

**Gestures** — per-hand, per-gesture actions (blendshapes, clips, Disable Blinking). COMBO mode needs both hands; Gesture Lock menu toggle; Exclusive Tags; Gesture Weight for gradual fist animation.

**Global Collider** — globally synced PhysBone collider + contact sender on any bone. **Big gotcha:** VRChat's collider budget means it *steals finger colliders* — Ring > Little > Middle, max 6.

**Other components (highlights):** *Blink Controller* (drives blinking from a closed-eye frame; stops during eye gestures), *Anchor Override Fix*, *Bounding Box Fix*, *Cross-Eye Fix*, *Blendshape Link* (clothing follows body sliders), *Blendshape Optimizer* (bakes non-animated blendshapes — free VRAM), *Fix Write Defaults* (aligns every state to the avatar's dominant convention, or Force On/Off), *Direct Tree Optimizer* (collapses compatible layers into one direct blend tree — frametime), *MMD Compatibility*, *Advanced Visemes* (bone-based; jerky caveat), *Apply/Delete During Upload*, *Parameter Compressor (Beta)* (16-bit packing), *Security Pin Number*, *Show In First Person* (incompatible with Armature Link), *Toes Puppet*, *When-Talking State*, *Droppable (World Constraint)*.

**Actions** (inside Toggles/Gestures): Object Toggle, BlendShape, Animation Clip (paths auto-rewritten relative to the component — what makes prefabs portable), Poiyomi Flipbook Frame, Poiyomi UV Tile Discard, Material Property, Material Swap, Scale, Enable SPS / Change SPS Tag, Set an FX Float, Disable Blinking/Visemes, Reset Physbone, Flipbook Builder, Smooth Loop Builder.

**On SPS/DPS/TPS:** current official docs center on **SPS (Super Plug Shader)** — VRCFury's plug-and-socket suite with tag-based logic — as successor to old TPS/DPS integration; legacy DPS/TPS auto-setup still exists but isn't the documented headline. Legacy modules (Sanity Props, motorbike sitter, "Whatever OR", "Plus children") are community-known but **not verifiable in current docs** — check the Discord.

**Automatic Fixes** (vrcfury.com/fixes) — silently patches dozens of SDK/Unity bugs: streaming-mipmap and menu-icon upload failures, particle security-check triggers, duplicate FX layers, Android parameter-order sync, Animator Override flattening, contact/dynamics play-mode breakage, and more.

## 4. AAO: Avatar Optimizer Reference

AAO (anatawa12, `com.anatawa12.avatar-optimizer`, v1.9.x) — the *performance* leg of the trio. Components classed as Avatar Global / Source Edit Skinned Mesh / Modifying Edit Skinned Mesh.

**Trace and Optimize (T&O)** — one component on the avatar root, scans animations, auto-applies safe optimizations (individually toggleable):
- **Optimize BlendShape** — removes/freezes/merges blendshapes animations never touch.
- **Remove unused Objects** — deletes unreachable GameObjects/components/textures; auto-toggles PhysBones serving only toggled-off objects.
- **Preserve EndBone**, **Optimize PhysBone Settings** (merge identical colliders, merge non-grabbable PhysBones, replace end bones with Endpoint Position, clear needless Is Animated).
- **Optimize Animator** — converts AnyState/complete-graph layers to Entry-Exit diamonds + blend trees (smaller animators, cheaper frametime).
- **Merge Skinned Meshes** (automatic), **Allow Shuffling Material Slots** (draw-call reduction), **Optimize Texture** (UV packing + downsizing for shaders with ShaderInformation support).
- **MMD World Compatibility** checkbox keeps MMD-used blendshapes intact.
- Advanced: zero-sized-polygon removal (can break some shaders — careful).

Design contract: *never change the avatar's appearance* — if it does, it's a reportable bug.

**Merge Skinned Mesh (MergeSMR)** — manual merge of SMRs and basic MeshRenderers into one. Options: Remove Empty Renderer GameObject, Skip Enablement Mismatched Renderers, Copy Enablement Animation (copies exactly one animated activeness property; errors on multiple), BlendShape Mode (Rename/Merge same-name/v1.7.x compat). **Gotchas:** merged meshes can't be toggled individually (unless Copy Enablement Animation); set Anchor Override via target SMR or MA Mesh Settings. In most cases T&O does this automatically.

**Remove Mesh By BlendShape** — deletes vertices/polygons displaced by selected blendshapes (chest/butt sliders hidden under clothing) with a Tolerance slider — permanent geometry removal, real poly/VRAM savings.

**Remove Mesh By Box / Mask / UV Tile** — geometry removal by box volume, texture mask, or UV-tile discard regions (pairs with Poiyomi UV tile toggles).

**Merge Material** — texture atlasing for materials sharing one shader. Requires ShaderInformation support (built-in for Standard, VRC ToonLit/ToonStandard, lilToon; shader authors can add more). **Limitations:** no support for animation-swapped materials; textures must have fixed transforms (parallax/UV-scroll excluded). Combine with Merge Skinned Mesh on the same GameObject to merge across renderers.

## 5. How the Three Tools Interact

All three run on **NDMF** (Non-Destructive Modular Framework) with named pipeline phases (Resolving → Generating → Transforming/Optimizing) and per-plugin execution order. MA and AAO declare explicit ordering so MA's *generation* completes before AAO's *optimization* (T&O runs late, sees the final avatar, safely deletes unused objects created by MA logic). VRCFury is NDMF-based in current versions and likewise runs generation before optimizers.

**Practical split:**
- **Modular Avatar** — structure & interactivity: outfit installation, menus, parameter namespacing, reactive toggles. Best when the asset creator ships MA prefabs.
- **VRCFury** — end-user convenience & heavy animation logic: toggles/gestures you add to *any* avatar, Full Controller, gestures, SPS, automatic fixes. Best when *you* author the behavior, or a prefab ships "VRCFury-ready".
- **AAO** — pure optimization, last, no matter which authoring tool you used.

**Conflicts:** VRCFury + MA on the same avatar is common and generally fine, but: VRCFury Armature Link vs MA Merge Armature on the *same* outfit (pick one); both auto-managing Write Defaults (decide a convention, don't let both fight); MA Parameters renaming vs VRCFury global parameters (VF## prefixing keeps it isolated — Full Controller's Global Parameters list exists when you need OSC access); AAO's Remove Unused Objects can delete objects an MA reactive component counts on if animations aren't detected. Mesh editing (Remove Mesh By BlendShape) should be configured *after* MA Setup Outfit so you cut the merged result, not duplicated pre-merge geometry.

## 6. Prefab Ecosystem Norms

MA's [Distributing Prefabs guide](https://modular-avatar.nadena.dev/docs/distributing-prefabs/): point users to official MA distribution (never bundle a possibly-outdated copy), use nested prefabs with a "base" prefab so non-MA users still get the core outfit, use **internal (auto-renamed) parameters** so gimmicks can't clash. The outfit-creator guide standardizes "Setup Outfit"-compatible clothing: correct armature naming, Mesh Settings in Set-or-Inherit, Shape Changer on the outfit targeting the body, Blendshape Sync. VRCFury's "for Artists" pitch: prefabs embed their own Toggles/Full Controllers with relative paths, so "drag in, upload" works with zero menu editing.

Marketplace norms 2025–2026: a good outfit/hair prefab ships MA Merge Armature pre-configured (often with a "remove MA components" fallback variant), optionally a VRCFury variant, states tested avatar/Unity versions, and increasingly AAO-friendly notes (which blendshapes are safe to delete). "Gogo Loco" variants remain the classic Full Controller use case.

## 7. Common Failure Modes

- **Menu not appearing:** unbound MA Menu Item; Menu Installer pointed at a full menu; VRCFury toggle path typo; menu-limit errors.
- **Animations path-broken after merge:** clips authored against absolute avatar paths inside a Relative Merge Animator break; VRCFury Full Controller's Debug Info lists unmatched paths (fix with Path Rewrite Rules). AAO bone merges remap paths automatically; third-party tools bypassing NDMF do not.
- **NDMF error console:** MA's error report window (Tools → Modular Avatar → Show Error Report) with clickable object links. "Nothing processed at all" = missing MA Avatar Root or Apply-on-Play disabled.
- **Version mismatches:** the biggest ecosystem hazard — a prefab bundling an old MA/NDMF can *downgrade* your project and break every other prefab (hence MA's "don't redistribute MA" rule). Update all four (NDMF, MA, AAO, VRCFury) together via VCC/ALCOM.
- **Write Defaults confusion:** mixed WD states break MA's Match-Avatar-WD heuristic and produce "toggle stuck on" bugs; VRCFury Fix Write Defaults is the blunt-force remedy.

**Sources:** https://modular-avatar.nadena.dev/docs/reference/ (+ reaction/, distributing-prefabs/, problems/) · https://vrcfury.com/components (+ /toggle, /full-controller, /armature-link, /gestures, /other, /global-collider) · https://vrcfury.com/actions · https://vrcfury.com/fixes · https://vpm.anatawa12.com/avatar-optimizer/en/docs/reference/
