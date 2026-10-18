# VRChat Avatar Troubleshooting Index

*Symptom → cause → fix. The fastest-path index for "my avatar is broken."*

---

## Symptoms at a glance

| Symptom | Most likely causes (in order) | Fix — see section |
|---|---|---|
| **Avatar shows as gray robot / impostor** | Performance blocked · platform missing · safety settings | §1 |
| **Avatar pink/magenta in editor** | Shader missing or wrong pipeline | §2 |
| **Avatar pink to some users in VRChat** | They blocked custom shaders → fallback | §2 |
| **Mouth doesn't move** | Lip sync mode/mesh wrong, visemes unmapped, FX fighting | §3 |
| **Eyes don't blink / half-closed / frozen** | Blink shape unmapped, FX conflicts, weight <100 | §3 |
| **Eyes look down / cross-eyed / dead stare** | Eye Look angles wrong, FX fighting eye-look, eye bones bad | §3 |
| **Toggles stuck on / "state bleed" / weird residue** | Mixed Write Defaults, missing off-states, weight-0 layers | §4 |
| **Menu item does nothing** | Parameter name mismatch, not in FX, missing menu binding | §4 |
| **Others don't see my toggle** | Not synced, budget overflow, positional mismatch, animation outside FX | §4 |
| **Upload fails: "too many parameters"** | 256-bit overflow (or 256-total client limit) | §4 |
| **Instant Very Poor** | Mesh Read/Write enabled, uncompressed 4K textures | §5 |
| **"Medium" locally, Very Poor in-game** | Server-side recalculation, material-swap textures counted | §5 |
| **Hair flaps while talking** | Jaw auto-mapped to a hair bone | §6 |
| **Avatar deforms wrong in Unity but fine in Blender** | Bone rolls, scale not applied, humanoid mapping | §6 |
| **FBT: hip flips / feet skate** | Hip placement, proportions, foot animation in FX | §7 |
| **Mesh explodes / 100× wrong scale** | FBX Apply Scalings, unapplied transforms | §6 |
| **Crashes people / flagged as crasher** | Accidental heavy feature — audit before appeal | §8 |

---

## 1. Gray robot / impostor instead of my avatar

The **viewer's** safety settings hid the avatar — usually not your bug:

- Performance blocked: your avatar is Poor/Very Poor and the viewer's "Minimum Displayed Performance Rank" blocks it (mobile default block = Medium). Check your rank in the SDK; target Medium or better for public avatars.
- Platform missing: no Android/iOS build → viewers on those platforms see the impostor. Upload cross-platform to the same blueprint ID.
- Impostor vs fallback: since 2025, impostors (billboard body doubles) replace fallbacks in most cases; wearers can toggle "Use Impostor as Fallback". A "Show Avatar" click by the viewer forces the real avatar (perf-blocked only — a missing platform asset can't be forced).
- File-size blocked or load error: nameplate badge icon tells you which.

See `platform-features.md` §1–2, `optimization-deep-dive.md`.

## 2. Pink / magenta materials

- **In editor:** the shader failed to compile or isn't present — wrong render pipeline (URP material in a Built-in project — VRChat is Built-in), missing VPM package (reinstall lilToon/Poiyomi via VCC), corrupted import (reimport). Check the Console, not the Inspector.
- **In VRChat only:** those users have "Allow Untrusted Shaders"/safety shaders blocked — your materials fall back per `VRCFallback` tags. Set explicit fallback tags in your shader/material; design Quest-style fallbacks deliberately. Toon+Transparent falls back to Transparent Unlit (flat glow) — cutout hair geometry survives better.
- See `shaders.md` §8.

## 3. Face systems broken

**Mouth doesn't move:**
1. Descriptor → Lip Sync mode left as "Default" or set to a mode that doesn't match your asset (use **Viseme Blend Shape** for blendshape faces).
2. Wrong mesh selected in the lip-sync field — it must be the ONE mesh carrying the `vrc.v_*` shapes.
3. Viseme slots not filled — run **Auto Detect!**; if your shapes aren't named with the `vrc.` prefix, map all 15 manually.
4. An FX animation clips the viseme blendshapes — conflicting animation wins unpredictably; remove those keys.
5. Mode is "Jaw Flap" with no jaw bone mapped.

**Blink broken:** blink shape not assigned in the descriptor's Blink section; shape max weight <100; an FX clip animating the blink shape (conflict); blink done via an FX toggle instead of the descriptor (won't sync to others).

**Eyes look down/cross-eyed:** Eye Look angles authored while the head was pitched — with the avatar in T-pose looking forward, set Horizontal ≈ ±60, Vertical ≈ −10 to +10. Crosseyed = eye bones' forward axes wrong (Blender export convention). Dead stare = FX animating eye bones/shapes while eye-look is on, or eye-look mode mismatched (blendshape vs bone).

**Hair flaps while I talk:** Unity's auto-mapper assigned a hair bone to **Jaw** — in Configure Avatar, set Jaw to None (unless you have one).

## 4. Animator / toggles / parameters

**Stuck toggles / state bleed:** mixed Write Defaults within a layer is cause #1 — pick WD OFF everywhere, author explicit off-state clips, set every layer weight to 1 (WD-OFF + weight-0 layers don't reset). Checklist: `av3-recipes.md` §5.

**Menu item does nothing:**
- Menu parameter name ≠ animator parameter name (exact, case-sensitive — `Props/Glasses` ≠ `Glasses`).
- Parameter exists in Expression Parameters but not in the FX controller (or vice versa).
- The animated property lives outside FX (material/blendshape animation in Gesture doesn't apply the same way and doesn't mirror).
- AAP conflict: a clip animates the same parameter the menu writes — remove the keyframe.
- Dead-end state: the layer has an unreachable state with no exits.

**Others can't see my toggle:**
- Parameter not marked **Synced** in Expression Parameters.
- 256-bit synced budget exhausted — overflow parameters **silently** don't sync.
- PC/Quest variants: sync is **positional** — parameter lists must match in order and type across platform variants.
- bool parameters and the menu control disagree on the default value.

**Upload fails — "VRCExpressionParameters has too many parameters":** synced parameters exceed 256 bits. Note the separate client-side limit of 256 *total* parameters (all types, synced or not) — exceeding it breaks ALL parameters in-game even if the SDK accepted the upload. Delete unused entries, demote to unsynced (Saved costs nothing), pack bools. The SDK 3.7.0+ editor shows a byte-meter.

## 5. Performance rank problems

- **Instant Very Poor:** any mesh (including particle meshes) with Read/Write enabled. Also: >150 MB texture memory, >70k tris (PC), >32 animators, >400 bones.
- **Local Medium, in-game Very Poor:** ranks are computed **server-side** since Aug 2025 and count textures referenced by material-swap animations; known server-side counting bugs exist. Use anatawa12's "Actual Performance Window" gist in play mode for the honest local number; if the server number is provably wrong, report with the avatar ID.
- **Slimming fast:** Read/Write off → texture formats (BC7/BC5 PC, ASTC Android, Max Size 1024–2048) → AAO Trace and Optimize → PhysBone consolidation. Full decision tree: `optimization-deep-dive.md` §9 Tree A.

## 6. Import / rig / mesh problems

- **100× scale:** FBX exported without "Apply Scalings: FBX Units Scale", or unapplied transforms. Fix in Blender (`Ctrl+A → All Transforms`), re-export; don't hand-edit Unity's Scale Factor (PhysBone constants inherit the error). `blender-workflow.md` §10.1.
- **Deforms right in Blender, wrong in Unity:** bone rolls (check `N → Item → Roll`), "Only Deform Bones" dropped weighted bones, humanoid retarget vs weights (test rig as Generic — right in Generic = mapping problem, not weights). §10.2.
- **Shape keys vanished:** a modifier above the shape-key stack baked them away during FBX export; or "Shape" unchecked; or visemes live on a different mesh than the descriptor's lip-sync target. §10.3.
- **Bones unmapped in Configure:** `.001` suffixes (duplicates), mixed `LeftXxx`/`xxx.R` conventions, unmapped Chest (mandatory in VRChat), Jaw auto-grabbing hair. §10.4–10.5.

## 7. FBT problems

- **Hip flips:** hip bone below the thigh line, zero-length bones in the spine chain, or "full-body hacks" (flipped hips, extra leg bones).
- **Feet skate / arm pull / stilts:** proportions off — limbs disproportionate to a human trigger arm-pull (short arms) and stilts (long legs); fix with Immersive Scaler or remodel. Don't animate feet/hips in FX or Gesture (fights IK).
- **Calibration pain:** no IK Pose assigned, or height-vs-armspan measurement mismatched to the avatar's proportions. Ship a sensible default View Position; "FBT compat" is a selling point.
- Gate pose layers by the built-in `TrackingType` parameter so poses only apply in 3/6-point, never 11-point FBT. See `avatar-design-patterns.md` §5.

## 8. Crashers / disruption flags

If your avatar/world gets flagged for crashing others, audit for: full-screen shaders or full-viewport transparent quads, full-world audio sources, oversized models/bounds, particle storms with collision, shader keywords incompatible with the client. **Crashers are a TOS violation even when accidental** — fix and re-upload; appeals go through moderation. Deliberate crashers are banned.

## 9. Quick diagnosis tools

- `Options → Avatar → Debug` (in-game): live parameter values + active state per playable layer — the definitive "is the parameter changing but the transition failing, or neither" tool.
- Avatar Overlay (in-game): visualizes PhysBone/Contact volumes.
- Right Shift + `~` + 2/3/4: world debug views (FPS, logs, players).
- Output log: `%USERPROFILE%\AppData\LocalLow\VRChat\VRChat\output_log_*.txt`.
- Shift+F1–F8: force gestures on desktop; Shift+F11: Safe Mode isolation.
- Full testing workflow: `testing-qa.md`.
