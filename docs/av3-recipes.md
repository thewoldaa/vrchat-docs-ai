# Avatar 3.0 Hands-On Recipes — A Practical Guide for VRChat Avatar Creators (2025–2026 Edition)

*Verified against creators.vrchat.com documentation and the VRChat wiki, including SDK 3.8.1+ behavior (per-platform overrides, Toon Standard, iOS build & test).*

---

## 0. Foundations (read this first)

Every humanoid Avatar 3.0 avatar has **five Playable Layers** that stack in this order — **Base → Additive → Gesture → Action → FX** — with later layers overriding earlier ones when they animate the same property:

| Layer | Purpose |
|---|---|
| **Base** | Locomotion, transforms only |
| **Additive** | Things added on top of Base (breathing) — transforms only |
| **Gesture** | Hand-triggered or menu-triggered pose stuff — transforms only |
| **Action** | Full-body overrides (emotes, sitting alternatives) — transforms only |
| **FX** | **Everything else**: GameObject toggles, blendshapes, material/shader swaps, particles. The only layer whose non-transform animation is copied to mirrors and remote players. |

**Golden rule:** toggles, blendshapes, material swaps → **FX layer**. Poses, tail wagging, outfits that move bones → Gesture. Sitting/dancing overrides → Action.

Custom parameters live in a **VRC Expression Parameters** asset; menu controls in a **VRC Expressions Menu** asset; both assigned in the Expressions section of the **VRC Avatar Descriptor**. Synced parameters share a **256-bit budget**: `bool` = 1 bit, `int` = 8 bits (0–255), `float` = 8 bits (−1.0 to 1.0, synced in 1/127 steps). Up to 8,192 total custom parameters. Menu pages hold up to **8 controls** each.

Names of parameters in the Expression Parameters asset must match the Animator parameter names in your controllers **exactly** (case-sensitive).

## 1. Building a Toggle From Absolute Scratch

Goal: a menu toggle that shows/hides a pair of glasses.

**Scene prep**

1. Select avatar root → Inspector → **VRC Avatar Descriptor** → *Expressions* → both slots filled (Expression Parameters asset + Expressions Menu asset). If empty: right-click Project window → *Create → VRChat → Avatars → Expression Parameters* (and *Expressions Menu*), assign both.
2. Put the toggleable object somewhere sensible, e.g. `AvatarRoot/Accessories/Glasses`. Make sure it starts **active** (you'll animate it off).

**Create the parameter**

3. Select Expression Parameters asset → *Add Parameter* → name `Glasses` → Type `Bool` → Default checked → check **Saved** and **Synced** (1 bit; if out of budget, leave Synced off — others won't see it).
4. Open your FX controller (Descriptor → Playable Layers → FX). Never edit SDK sample controllers in place — duplicate first.

**Create the layer and states**

5. Animator window → Layers tab → **+** → name `GlassesToggle`. Gear icon → set **Weight = 1**.
6. Right-click grid → *Create State → Empty* → `ON`. Create another → `OFF`.
7. Right-click `ON` → *Set as Layer Default State* (orange).
8. Create the clips (next step), then assign: `ON` state Motion = `Glasses_ON`, `OFF` state Motion = `Glasses_OFF`.

**Create the animation clips**

9. Project window → right-click → *Create → Animation* → `Glasses_ON` and `Glasses_OFF`.
10. Select avatar in scene → Window → Animation (Ctrl+6) → load `Glasses_OFF` → press record → in Hierarchy **deactivate the Glasses GameObject** → stop recording. This writes an `m_IsActive` keyframe at time 0. Reverse for `Glasses_ON`.

⚠️ Common mistake: recording while the wrong object is selected so the clip keys the wrong path. Verify the clip's property list shows `Glasses : Active` under GameObject.

**Wire the transitions**

11. Right-click `ON` → *Make Transition* → drag onto `OFF`. Click the arrow → uncheck **Fixed Duration**, **Transition Duration = 0**, **Exit Time = 0**.
12. Conditions: `ON → OFF` = `Glasses` / `Is false`. `OFF → ON` = `Glasses` / `Is true`.

**Add the menu control**

13. Select Expressions Menu asset → *Add Control* → name "Glasses" → Type **Toggle** → Parameter `Glasses`.
14. Build & Test (SDK Builder tab). In the Test world, flip the toggle. If nothing happens: check Animator in Play mode, and confirm menu parameter name == animator parameter name exactly.

That's the whole pipeline: **menu control → expression parameter → animator parameter → two states with conditioned transitions → clip that changes `m_IsActive`**. Every other recipe is a variation of it.

## 2. Gesture-Based Toggles

Goal: **left fist = outfit swap**; analog gesture-weight control for horns that grow as you grip.

1. In FX (or Gesture for transform/pose based), add built-in parameters `GestureLeft`/`GestureRight` (Type Int) to the controller. These are **built-in**: do *not* add them to Expression Parameters (0=neutral, 1=fist, 2=open hand, 3=point, 4=victory, 5=rock'n'roll, 6=hand gun, 7=thumbs up).
2. Layer `OutfitGesture`: idle state + one state per outfit.
3. Transitions from idle: `GestureLeft Equals 1` → Casual; `Equals 2` → Formal; etc. Back-transitions on `Equals 0`.
4. Enable **Interrupt Source: Current State Then Next** so swapping fists mid-hold doesn't get stuck.

**The double-fire problem:** holding fist re-triggers on grip flicker. Classic fix — **gesture weight + hysteresis**:
- Add built-in `GestureLeftWeight` (Float 0–1 from analog grip on Index controllers).
- Idle → outfit only when `GestureLeft = 1` **and** `GestureLeftWeight > 0.9`; return when `GestureLeft = 0` **and** `GestureLeftWeight < 0.1`.

**Analog gesture control:**
5. Horns that grow as you squeeze: single state with a **1D blend tree** on `GestureLeftWeight`, motions `Horns_Small` (scale ×0.5) at threshold 0, `Horns_Big` (×1.2) at threshold 1.
6. Transform animations live fine in Gesture; scale/blendshape animations belong in FX.
7. Desktop users get gestures via keybinds (Shift+F1–F8) but **no weight** — always provide a menu fallback (e.g., radial puppet driving the same float).

## 3. Radial Puppet: Hue-Shift / Morph Slider

Goal: a radial menu dial that hue-shifts hair continuously.

1. Expression Parameters → Add → `HairHue`, Type **Float**, Default 0, **Saved + Synced**.
2. Expressions Menu → Add Control → Type **Radial Puppet** → name "Hair Color" → Parameter `HairHue`.
3. In FX, new layer with one state set as **Blend Tree**: Type **1D**, Parameter `HairHue`, motions at thresholds −1, −0.5, 0, 0.5, 1 (5 clips keying the hair material's color/hue property).
4. Each clip animates the same **material property** on the same renderer. For lilToon/Poiyomi animate their documented hue properties (e.g., `_MainTexHSVG.z` for lilToon hue rotation) rather than baking colors.

**Material property animation notes**
- Only FX-layer material animation syncs/mirrors. Gesture-layer material animation looks right to *you* but wrong to everyone else.
- Animating a material in a clip modifies the **instantiated material**, not the asset — fine and normal.
- If the slider "snaps" when the menu closes, that's usually an AAP or Write Defaults interaction (§5, §12).

**Morph slider variant:** same skeleton, clips animate a **SkinnedMeshRenderer blendshape weight** (e.g., `Cheeks` 0 → 100). The standard "body slider" recipe.

## 4. Direct Blend Trees

**What it is:** a single state per layer whose blend tree *is* the whole system. The parameter directly selects/interpolates clips. Fewer states, no transition-condition bugs, excellent with **Write Defaults OFF** because every clip explicitly keys everything.

**Concrete setup — 3-way hairstyle selector:**
1. Expression Parameters: `HairStyle`, Int, default 0, saved+synced. Menu: three Toggles (or Buttons set to values 0/1/2 targeting the same Int).
2. FX layer `Hair`, **one** state as blend tree: Type **1D**, Parameter `HairStyle`, thresholds 0/1/2 with clips `Hair_A/B/C`. Each clip keys *all three* hairstyles' active flags (Hair_A clip: A on, B off, C off; etc.).
3. Done. No transitions, no conditions.

**Motion-time variant:** a state whose Motion is a 1D blend tree, with **Motion Time** enabled on the state driven by a float parameter — scrubs an animation's timeline proportionally (0 = start, 1 = end). Classic uses: expression intensity, animation scrubbed by Voice. Motion Time expects a **Float** parameter.

**Float analog example:** `BreathingDepth` float slider → 1D tree: `Breathe_Off` at −1, `Breathe_Deep` at 1 — continuous cross-fade. Recommended structure for anything radial-puppet-driven.

**Gotchas:** thresholds must cover the parameter's full range (pad floats to −1/+1); gaps between thresholds produce blended output; clips in the tree should have identical property sets to avoid "one property stuck" under WD OFF.

## 5. Write Defaults Deep-Dive

**What WD does:** checked → the state writes animated properties to their *default values* (captured at clip-creation time) whenever active. Unchecked → properties not keyed are **left at whatever the last animation left them**.

**VRChat reality:** community standard (and most tools/bases) is now **WD OFF** — far more predictable across Unity versions (2022.3 multiplies WD quirks). Mixed WD in the same layer is the #1 cause of state bleed.

**What breaks with WD ON:**
- Toggle clips that only key "on" but not "off" → stuck properties.
- Animations from other layers bleed in.
- Changing Unity versions silently changes "defaults" → avatar breaks after re-import.

**What breaks with WD OFF:**
- Every animated property must be keyed by *some* clip on an active state with layer weight > 0, else it freezes.
- The default state must contain an **off-state clip** resetting everything.

**Converting to WD OFF safely:**
1. Back up controllers.
2. WD off for **every state in every layer** (batch tools exist).
3. Every toggle layer's idle/base state gets an off-state clip keying *every property the layer touches* to neutral.
4. Add reset clips to transitional states.
5. **Layer weight = 1 everywhere** (WD OFF + weight-0 layers don't reset states — the classic "toggles stopped working after conversion").
6. Test exhaustively: toggle on/off, close menu, rejoin, check Saved params restore.

**Off-state clip checklist:**
- [ ] Every property animated anywhere in the layer keyed in default state's clip
- [ ] `m_IsActive` keyed for every toggled object
- [ ] All touched blendshape weights keyed (even at 0)
- [ ] Every animated material property keyed at default
- [ ] No stray keys on unrelated properties
- [ ] Layer weight = 1; default state is actually default (orange arrow)

## 6. Parameter Drivers

`VRC Avatar Parameter Driver` = State Behaviour that sets/changes parameters when a state is entered. "Local Only" affects only you.

**Recipe A — self-clearing button:** menu Button sets `Boop` (Int) = 1 → state keyed on `Boop = 1` plays squish → that state has Driver *Set* `Boop` = 0 on enter. Fires once, self-resets. (Drivers can't override a parameter currently being animated as an AAP on the same controller.)

**Recipe B — random idle:** Driver action **Random**: set `IdleVariant` (Int) random 0–2 on a loop-entry state; transitions to `Idle_A/B/C` on Equals.

**Recipe C — int cycling (3-way hair color):** menu Toggle sets bool `ColorNext`; a driver state does **Add** `HairColorIndex` +1, immediately returns to idle, another driver clears `ColorNext` (Local Only bookkeeping bool — save the bit). Colors via **direct blend tree** on `HairColorIndex`. One toggle steps 0→1→2→0.

Drivers are local — drive a **synced** parameter if remote players must see the result.

## 7. Blink / Viseme / Eye-Look Verification Checklist

**Blink**
- [ ] Descriptor → Blink section → blendshape(s) assigned, threshold ~0.35.
- [ ] Blendshapes exist on the SkinnedMeshRenderer, spelled identically.
- [ ] **No FX animation animates the blink blendshape** — conflicting animation wins unpredictably.
- [ ] "Blink but half-closed": shape max weight <100 or animated elsewhere.
- [ ] Blink implemented via an FX toggle animation will NOT sync — use the descriptor's Blink.

**Visemes**
- [ ] Lip Sync mode **Viseme Blend Shape**, correct face mesh selected, **Auto Detect!** fills all 15 slots. Common failure: mode left as "Default" or detection ran on the wrong mesh — mouth never moves.
- [ ] Face mesh not animated by FX on those viseme shapes.
- [ ] Test: mic on, make "ooo"/"sss" — mouth moves with no input.

**Eye Look**
- [ ] Mode Blend Shape or Bone; eyes/bones assigned.
- [ ] **Eyes looking down / crosseyed:** with avatar in T-pose looking forward, set Horizontal ~±60, Vertical ~−10 to ~+10. Eyes-down almost always means vertical angles were authored while the head was pitched.
- [ ] Leave both Tracking and Movement enabled.
- [ ] No FX clip animates eye bones/blendshapes while eye-look is on (they fight).

## 8. PhysBone Recipes

**Hair with 2 chains:**
1. One PhysBone per chain (cleaner, independent settings) on each pigtail root.
2. **Multi-Child Type = Ignore** for single rigid strand; **Average Child** when the bone has children.
3. Starting point: Pull 0.4–0.6, Momentum 0.2–0.4, Stiffness 0.1–0.3, Gravity 0.2. Tune one property at a time.
4. Add the **Head** standard collider (auto-provided by descriptor) to the Colliders list — without it hair clips through your head when looking down.

**Skirt with colliders:**
1. PhysBone on hip bone, roots = skirt "petal" roots. **Average Child**. Low stiffness (0–0.1), gravity 0–0.2.
2. Add both **thigh** standard colliders — this is why skirts stop clipping through legs.
3. Add a Limit Angle if petals spread too far; multi-layer skirts: outer stiff, inner loose.

**Breast/butt physics (tasteful config):**
1. Small **Max Angular Limit** (~15–25°), Pull 0.3–0.5, Stiffness 0.4–0.6, gravity ≤0.1.
2. Chest standard collider to prevent torso clipping.
3. Limit Type = Angle — the difference between "subtle bounce" and "nightmare fuel."
4. Add an FX toggle to disable (component animation → FX layer).

**Limit types explained:**
- **None:** free rotation, can fold through itself.
- **Angle:** cone — max angle from rest direction in any direction. Usually enough.
- **Hinge:** rotation around one axis within a range — flags, tails that shouldn't twist.
- **Polar:** yaw range around a polar direction — hair strands that swing but don't fold backward.
- **Limits 2:** second-generation solver, better stability, fixes boundary jitter — generally recommended now.
- **Multi-Child Type:** *Ignore* = each branch independent; *Average Child* = averaged direction drives one chain (skirts: almost always Average Child).

## 9. Contacts Recipes

**Hand poke → expression:**
1. **VRC Contact Sender** on index fingertip bone, Radius ~0.03, Collision Tags: `Poke`.
2. **VRC Contact Receiver** on cheek bone, Radius ~0.05, Tag `Poke`, Receiver Type **Proximity**, Parameter `CheekPoke` (add float param; leave **unsynced** if only you need it).
3. FX layer: idle → `Poked` (xD-eyes blendshape clip) on `CheekPoke > 0.5`, back on `< 0.1`. Contacts output 0–1 by proximity — use thresholds.
4. Receivers fire only for **other avatars'** senders by default; set "Self = On" to let your own hands trigger (handy for testing).

**Proximity hug:**
1. Chest **Contact Receiver**, sphere radius ~0.35, Tag `Hug`, Proximity, Parameter `HugProx`.
2. Works between avatars that both ship the tag (community conventions: `Hug`, `Hand`, `Head`, `Pointer`, `Face`).
3. FX: hug state (arms out / heart particles) on `HugProx > 0.8`, release on `< 0.2` — hysteresis prevents flicker.
4. Contacts run locally on the receiving avatar; only sync what must be seen.

## 10. FBT Setup & FBT-Friendly Fixes

**Setup (avatar side):**
1. Descriptor: assign **T-Pose** and **IK Pose** (slightly bent knees, arms out prevents calibration drift). SDK "Auto assign" works for standard humanoid rigs.
2. Set **Eye Height** correctly — FBT calibration uses it.
3. Base locomotion layer: SDK defaults support FBT out of the box.

**FBT-friendly fixes:**
- **Hips offset:** don't animate the hip bone in Gesture/Action (fights IK); adjust avatar scale/eye height instead of hip-animation hacks.
- **Feet drift/skating:** ensure T-Pose and IK Pose correct; avoid foot-bone animation in FX/Gesture (overrides IK → slide).
- **FBT toggle systems:** FX toggles that disable upper-body-override poses, expose PhysBone stiffness sliders (menu float + blend tree animating PB params), gate pose layers by **`TrackingType`** (built-in Int) so poses only apply in 3-point/6-point, never FBT.

## 11. Cross-Platform: Parameter Ordering & Per-Platform Overrides

**Parameter ordering rule (critical):** PC + Quest/iOS versions sync parameters **by position in the Expression Parameters list and by type, not by name**. Two avatars sync correctly only if the parameter at index N has the same type on both.

1. Use the **same Expression Parameters asset** on both versions whenever possible — unused-but-listed parameters still maintain alignment.
2. If you must diverge: shared parameters identical order+type at the **top**; platform-specific appended **only at the end**.
3. Same for animator parameter names within each build; sync is positional across builds.

**Per-platform override workflow (SDK 3.8.1+):**
1. Set up all platform variants in the **same scene**.
2. Builder tab → select base avatar → **⋮** next to "Selected Avatar" → *Add per-platform override* → pick platform + variant GameObject.
3. **Multi-Platform Build & Publish** — uploads each variant under **one avatar ID**. A `VRC Per Platform Overrides` component appears on the root.
4. Works with prefabs/prefab variants. Each platform build can diverge FX controllers and menus — as long as parameter ordering rule (2) holds for anything synced between mixed instances.

## 12. Common AV3 Bugs & Fixes

**State bleed** — property stuck animating from a state you left. Cause: mixed Write Defaults, or WD-ON writing defaults from a stale clip. Fix: uniform WD OFF + off-state clips, layer weights ≥ 1.

**Transitions not firing.** Checklist: parameter exists in *this* controller (Expression Parameters ≠ animator parameters — both must exist); condition name/type matches exactly; no "dead end" states; "Can Transition To Self" isn't eating re-entry; any **AAP** (parameter animated by a clip) overrides menu-set values — remove the keyframe.

**Animations overriding each other.** Layer order: Base → Additive → Gesture → Action → FX; FX wins non-transform. Higher layer within the controller also wins. Fix: **Avatar Masks** to scope layers, weights 1 = full override. A mask with nothing enabled animates nothing — the classic "my layer does nothing."

**"Avatar resets when menu closes."** Sub-menu control with a stray parameter binding, or a Button-type control resetting on release, or WD ON writing defaults when a menu-gated state exits. Fix: separate UI bookkeeping params from displayed params; Parameter Driver "Set on enter" to persist.

**Parameter not syncing.**
- Not marked **Synced** in Expression Parameters (or 256-bit budget exhausted — overflow silently doesn't sync).
- PC/Quest **positional mismatch** (§11).
- Animation lives outside FX (material/blendshape animation in Gesture doesn't mirror).
- Saved restores between sessions but **not** across avatar switches unless the avatar ID matches.

**Quick debugging toolkit:** avatar console logs, `IsLocal` parameter to split local vs remote logic, and **OSC** (enable in VRChat settings) to read live parameter values — fastest way to see whether a parameter actually changes.

## Sources

- https://creators.vrchat.com/avatars/playable-layers
- https://creators.vrchat.com/avatars/animator-parameters/
- https://creators.vrchat.com/avatars/per-platform-avatar-overrides/
- https://creators.vrchat.com/releases/release-3-8-1/
- https://wiki.vrchat.com/wiki/Expressions · https://wiki.vrchat.com/wiki/Animations · https://wiki.vrchat.com/wiki/Avatar_Descriptor · https://wiki.vrchat.com/wiki/Toggles
