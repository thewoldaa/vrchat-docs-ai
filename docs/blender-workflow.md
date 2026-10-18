# The Complete Blender → VRChat Avatar Workflow Deep-Dive (Blender 4.x, 2025–2026 Edition)

*A hands-on guide covering modeling, rigging, shape keys, FBX round-trips, VRM, MMD, texturing, add-ons, scripting, and troubleshooting.*

---

## 0. Toolchain Status Check (Read This First)

The VRChat Blender ecosystem has gone through a mass extinction event:

**Cats Blender Plugin — the original is dead.** `absolute-quantum/cats-blender-plugin` has had no release since June 2021. What people actually use is **Cats Blender Plugin Unofficial** by **Team Neoneko** (https://catsblenderplugin.xyz). Versioning matches Blender versions (4.2.x for 4.2, 4.5.x for 4.5 LTS). Note: Cats is on **life support** — Blender 4.5 was the last version with active fixes; all functionality is migrating into a successor ("Avatar Toolkit", planned May 2026).

**Tuxedo Blender Plugin** (baking/atlassing/decimation) — also revived by Neoneko; their fork targets Blender 5.0+ (v1.2.0, Nov 2025), while feilen's original v0.4.2-alpha works on 4.2/4.3/4.5. On 4.x use feilen's original; on 5.0 the Neoneko fork.

**MMD Tools** is alive: `MMD-Blender/blender_mmd_tools` v4.x supports Blender 4.2–5.2, installs from Blender's *Get Extensions* panel.

**VRM Add-on** ("VRM format" by saturday06) is the actively-maintained VRM importer/exporter — v4.10.x on Blender Extensions (https://extensions.blender.org/add-ons/vrm/), Blender 2.93–5.2, VRM 0.x + 1.0, MToon node generation, Python automation API.

**Practical version recommendation:** Blender **4.2 LTS** or **4.5 LTS** is the sweet spot — every add-on has a working build, and Unity-side tutorials target it.

## 1. Modeling: Reference Setup, Pose Discipline, and Topology

### 1.1 Scene setup and scale

VRChat measures avatars by **eye height**, not total height:

| Landmark | Target |
|---|---|
| Eye height | **1.6 m** from ground |
| Total height | ~1.7 m |
| Arm span (T-pose) | ≈ total height |

In Blender:
1. `Scene Properties → Units` — Metric, Unit Scale **1.0** (1 Blender unit = 1 m, matching Unity).
2. Drop a 1 m cube at origin as a sanity reference; delete before export.
3. Build the character standing on the ground plane (Z=0), facing **−Y**.

### 1.2 T-pose discipline

- Export in a clean **T-pose**: arms straight out, palms down/slightly in, legs straight, all bones at rest rotation.
- Reason: Unity's humanoid Mecanim rig **retargets from the T-pose bind**.
- Pragmatic exception: professionals often model/rig in slight **A-pose (~45°)** because it deforms better at rest — but the *armature rest pose must agree with the mesh*, and all bone rotation values must be 0 in rest state. `Ctrl+A → All Transforms` before rigging.
- Symmetry: model one half, Mirror modifier, apply.

### 1.3 Edge flow for deformation

- **Shoulders/armpits:** rings wrapping the deltoid perpendicular to the arm + a dedicated armpit loop + a **twist zone** (2–3 loops down the upper arm) to prevent "candy-wrapper" collapse. Minimum 6–8 rings on the upper arm.
- **Elbows/knees:** strictly ring-parallel loops, 3–4 across the bend, center loop exactly on the joint pivot. Diagonal loops pinch the joint into a point.
- **Hips/groin:** hardest area. Loops wrapping each leg vertically + a horizontal belt across the pelvis at leg-split level. Avoid a single vertex star in the crotch (classic crotch-tear bug).
- **Neck:** 3+ rings — the head turns constantly in VR.
- **Hands:** 2–3 rings per knuckle (full finger tracking / gestures).
- Quads everywhere on deforming areas; tris/poles pushed to non-deforming zones.

### 1.4 Common topology mistakes

1. Loops that **spiral** around limbs instead of ringing them → twisting shears on rotation.
2. Not enough loops at joints — 2 loops across a knee is not enough.
3. Poles (5+ star vertices) exactly on elbow/knee/shoulder pivot → pinching, shading artifacts.
4. Uniform density everywhere — face 20k tris but elbow has 3 loops. Budget density where it deforms.
5. Separate mesh objects for parts that deform together → seams break when the spine bends. Merge, or weight-lock seam vertices.
6. Non-manifold geometry / interior faces from booleans → explode in normal maps, break weight painting.

## 2. Rigging: Building the Unity Mecanim Armature

### 2.1 The bone naming contract

Unity's Mecanim maps bones by **name pattern recognition**:

```
Hips, Spine, Chest, UpperChest (optional), Neck, Head
LeftShoulder/RightShoulder (clavicle)
LeftArm/RightArm (upper arm), LeftForeArm/RightForeArm, LeftHand/RightHand
LeftUpLeg/RightUpLeg (thigh), LeftLeg/RightLeg (shin), LeftFoot/RightFoot, LeftToeBase/RightToeBase
Thumb01..03, Index01..03, Middle01..03, Ring01..03, Pinky01..03 (per side)
```

Conventions: `Left`/`Right` prefix **or** `.L`/`.R` suffix — **do not mix styles**. Beware `.001` suffixes (`LeftArm.001` will not map). Mapping is name-driven on import — with correct names, Unity's *Configure Avatar* auto-assigns nearly everything.

### 2.2 Building the armature

Option A: Rigify with a humanoid meta-rig → export with **Only Deform Bones** checked. Option B: Cats Unofficial "Fix Model" (best for MMD/XPS imports). Option C — manual (recommended the first time):

1. Add → Armature. Enable **X-Axis Mirror**.
2. Build: Hips → Spine → Chest → Neck → Head. Hips → thigh → shin → foot → toe (per side). Chest → clavicle → upper arm → forearm → hand → five finger chains of 3 bones.
3. **Bone rolls matter.** Edit Mode → `N → Item → Roll`. Arm bones rolled so local Y points down the bone. Wrong rolls produce twisted deformations even with perfect weights.
4. All root bones parented to a single armature object named cleanly (no `.001`).
5. Rest pose = T-pose, all rotation values zero (pose mode → select all → `Alt+R`, `Alt+G` to verify).
6. **Fingers:** full 15-bone-per-hand set, named per §2.1. VRChat finger tracking expects them; actually weight them — gestures bend fingers visibly.
7. Optional: **twist bones** (one per upper/lower arm), weighted to the outer half of the limb. Tuxedo can generate them automatically ("Generate Twistbones").

### 2.3 Constraints (Blender-side only — they will NOT export)

IK, Copy Rotation, drivers stay in Blender; the FBX export bakes only rest transforms + explicitly baked animation. Use constraints freely for testing; bake (`Pose → Animation → Bake`) if a pose must survive. VRChat does its own IK on top.

### 2.4 Weight painting workflow

1. **Auto weights first**: select mesh, shift-select armature, `Ctrl+P → With Automatic Weights`. Multi-part meshes: auto-weight each part individually to avoid cross-part bleed.
2. **Clean up immediately**: Weight Paint mode → `Weights → Clean` (threshold 0.01) — removes most "mesh twitches when unrelated bones move" bugs.
3. **The big four problem areas:**
   - **Shoulder:** deltoid/arm region → *arm* bone only; torso → *chest* only; hard-ish gradient at the crease. Arm weights bleeding onto chest = "spikes pulled toward the arm."
   - **Hips:** thigh weights stop below the waistband; pelvis/spine owns the belt area.
   - **Twist zones:** upper-arm influence split arm/twist 50/50 mid-limb, full twist at the elbow.
   - **Hands:** finger weights must not touch other fingers (webbed palms on fist).
4. `Weights → Normalize All` per body region at the end.
5. **Test deformation before export** — Blender and Unity use different skinning math; do one Unity in-scene pose check on first export.
6. Blender 4.x: bone groups → **bone collections**.

### 2.5 Common weight bugs and fixes

| Symptom | Cause | Fix |
|---|---|---|
| Vertex won't paint | Zero weights / no vertex group / armature modifier missing | Check modifier target; assign to a group then paint |
| Mesh tears at crotch/shoulder | Hard weight boundary, no shared influence | Soften gradient over 2–3 loops |
| Unrelated part twitches | Stray 0.01 weights from auto weights | `Weights → Clean` (0.005–0.01) |
| Arm candy-wraps on twist | No twist bone | Add twist bone or redistribute |
| Chest follows the arm | Arm weights on chest vertices | Subtract arm group from chest region |
| Fine in Blender, wrong in Unity | Non-zero roll/scale; non-applied transforms | `Ctrl+A → All Transforms` before export |

## 3. Shape Keys: Visemes, Blinks, Correctives

### 3.1 The viseme set

VRChat uses the **Oculus LipSync 15-viseme set**, driven by blendshape names on the mesh the Avatar Descriptor points at:

| Index | Name | Shape target |
|---|---|---|
| 0 | `vrc.v_sil` | Neutral, lips relaxed |
| 1 | `vrc.v_pp` | p/b/m — lips fully closed, slight pout |
| 2 | `vrc.v_ff` | f/v — lower lip against upper teeth |
| 3 | `vrc.v_th` | th — tongue between teeth |
| 4 | `vrc.v_dd` | t/d — slightly open, teeth near-touching |
| 5 | `vrc.v_kk` | k/g — back of mouth open |
| 6 | `vrc.v_ch` | ch/sh/j — lips rounded, jaw slightly down |
| 7 | `vrc.v_ss` | s/z — teeth nearly closed, lips wide |
| 8 | `vrc.v_nn` | n/l — lips apart, mid-open |
| 9 | `vrc.v_rr` | r — slightly rounded, mid-open |
| 10 | `vrc.v_aa` | ah — jaw open, oval |
| 11 | `vrc.v_e` | eh — mouth wide, mid-open |
| 12 | `vrc.v_i` | ee/ih — lips stretched wide |
| 13 | `vrc.v_o` | oh — lips rounded, jaw mid-open |
| 14 | `vrc.v_u` | ou — lips rounded and forward, jaw closed |

Use exactly `vrc.v_i`, `vrc.v_o`, `vrc.v_u` (VRChat's names, not Oculus's "ih/oh/ou").

### 3.2 Building them from scratch

You only need to sculpt **three** shapes; derive the rest (this is exactly what Cats' viseme generator does):

1. Single object, add the `Basis` key first.
2. Sculpt **A** (`vrc.v_aa`): jaw down, oval aperture. **Don't touch anything outside the lips/jaw region** — unmoved vertices stay at rest, which keeps derived keys clean.
3. Sculpt **O** (rounded circle, jaw mid-open) and **CH** (pursed forward, small aperture).
4. Add the remaining keys, driven by mixing:

   - `vrc.v_aa` = A ×1.0 · `vrc.v_ou` = O ×1.0 · `vrc.v_ch` = CH ×1.0
   - `vrc.v_dd` = A 0.3 + CH 0.7 · `vrc.v_e` = CH 0.7 + O 0.3
   - `vrc.v_ff` = A 0.2 + CH 0.4 · `vrc.v_ih` = A 0.5 + CH 0.2
   - `vrc.v_kk` = A 0.7 + CH 0.4 · `vrc.v_nn` = A 0.2 + CH 0.7
   - `vrc.v_oh` = A 0.2 + O 0.8 · `vrc.v_rr` = CH 0.5 + O 0.3
   - `vrc.v_ss` = CH 0.8 · `vrc.v_th` = A 0.4 + O 0.15
   - `vrc.v_pp`, `vrc.v_sil` ≈ 0 (closed/rest)

5. Implement mixing with shape-key drivers, or generate via Cats and hand-fix.
6. Rename to exact `vrc.v_*` names. Order doesn't matter (VRChat reads names); keep Basis first.

### 3.3 Blink and correctives

- **Blink (`vrc.blink_L`/`vrc.blink_R` or combined `Blink`):** upper lid **rotates down around the eyeball's center**, not just translated — translate-only blinks clip through the eyeball. VRChat plays a 0→1→0 curve.
- **Corrective shape keys:** sculpt the defect (bent elbow pinch, crotch tear at squat, shoulder crease at 90°), or use the **Corrective Smooth** modifier as a cheap automatic alternative.
- Blender-side drivers on shape keys are only for your own preview — VRChat drives visemes/blinks in Unity.

### 3.4 Export requirements

- Shape keys export through FBX as blend shapes. **Apply Modifiers must NOT apply to shape keys** — a modifier (Mirror, Subsurf) above the shape-key stack silently drops all keys. Apply or move modifiers below, or disable "Apply Modifiers" (§4.3).
- Check **Shape** in the export Geometry panel (on by default).

## 4. Blender ↔ Unity FBX Round-Trip, Panel by Panel

### 4.1 Include panel

- **Selected Objects:** ✅ (select armature + all meshes first)
- **Object Types:** ✅ Armature, ✅ Mesh; everything else ❌ (unless exporting empties as attachment anchors).

### 4.2 Transform panel

- **Scale: 1.00**
- **Apply Scalings: FBX Units Scale** ← this single dropdown is the #1 cause of the "model is 100× too big" bug.
- **Forward: −Z Forward, Up: Y Up**
- **Apply Unit:** ✅ · **Use Space Transform:** ✅
- **Apply Transform:** ⚠️ — leave **OFF for skinned/animated avatars** (applying transforms on skinned meshes historically corrupted skinning). Turn **ON for static props**. If the avatar imports rotated 90°, fix in Unity with **Bake Axis Conversion** instead of fighting Apply Transform.

### 4.3 Geometry & Armature panels

Geometry:
- **Apply Modifiers:** ❌ (protects shape keys — run decimation as a destructive op first instead)
- **Normals:** ✅ · **Tangent Space:** ✅ if using normal maps
- **Smoothing:** Face Metadata (or default "Normals Only")

Armature:
- **Primary Bone Axis: Y, Secondary: X** (default; changing rewrites every bone's local axes and can break auto humanoid mapping)
- **Armature FBXNode Type: Null**
- **Only Deform Bones:** ✅ if using Rigify/control bones; ❌ if you need non-deform physics bones exported
- **Add Leaf Bones:** ❌ for cleanliness; ✅ only if you want ready-made terminal bones for PhysBone Endpoint positioning
- Save settings as an exporter **preset** — mismatched settings between exports silently re-dirty Unity imports.

### 4.4 Unity import settings to match

On the imported FBX's Model tab:
- **Scale Factor should read 1.** If 100/0.01, re-export correctly instead of hand-editing (hand-editing makes PhysBone/outline/physics constants wrong by 2 orders of magnitude).
- **Bake Axis Conversion:** ✅
- Rig tab: **Animation Type: Humanoid** → Configure → verify mapping. Known gotchas:
  - **Chest is mandatory in VRChat** (Unity treats it optional) — the SDK throws "Spine hierarchy missing elements" without it.
  - Map Toes too.
  - **Jaw:** Unity's auto-mapper loves assigning whatever sits under Head to Jaw — often hair bones. Set Jaw to None unless you have one, or your hair flaps during lip sync.
- Materials tab: **Extract Materials…** before assigning shaders.
- Animation tab: import nothing unless bringing animations.

### 4.5 Scale debugging (the 100× case)

1. Blender: armature object scale not 1.0 → `Ctrl+A → All Transforms`, re-export.
2. Check **Apply Scalings = FBX Units Scale**.
3. Unity Model tab Scale Factor: goal **1**. Correct *in Blender*, not Unity — VRChat computes avatar height and physics from real-world scale ("Blender default is off by a factor of 100 and that wreaks havoc with the PhysBone parameters").
4. Emergency: set Scale Factor 1 manually and verify eye height against 1.6 m.

## 5. The VRM Workflow (VRoid → VRChat)

### 5.1 VRM in Blender via the VRM Add-on

Install "VRM format" from Blender Extensions (`Edit → Preferences → Get Extensions → search "VRM"`). Then:
1. `File → Import → VRM (.vrm)` — reconstructs humanoid armature, MToon-style materials, and **all expressions as real shape keys**.
2. Edit freely.
3. `File → Export → VRM` — check the **VRM Humanoid** mapping panel first; VRoid's bone names (`J_Bip_C_Hips`) usually auto-map.

### 5.2 VRM → FBX vs. direct

**Go VRM → Blender → FBX** (the common path) when: editing the model, fixing VRoid's auto-generated weights, stripping its dozens of materials. Recipe:
1. Import VRM with the add-on.
2. Clean the armature: VRoid exports misoriented leaf bones and physics helpers; delete per taste.
3. Delete the `glTF_not_exported` icosphere dummy.
4. Rotate 180° about Z if the model faces +Y (glTF convention) and apply transforms — Blender-FBX models face −Y.
5. Reduce/merge materials (Cats Material Combiner or Tuxedo), re-export FBX (§4.3).

**VRM → Unity directly** only when the model is already perfect and you want to keep MToon — rare for VRChat, which needs FBX humanoid rigs + VRChat shaders/PhysBones anyway.

**When VRM isn't needed:** for a native Blender-built avatar, export FBX directly. VRM's role here is interchange from VRoid, not a VRChat deliverable.

## 6. MMD Workflow: PMX → VRChat

### 6.1 Import

1. Install **MMD Tools** (Get Extensions, Blender 4.2+). Optionally **MMD Tools Append**.
2. `File → Import → MikuMikuDance Model (.pmx/.pmd)` — creates armature (Japanese bone names), meshes, **rigid bodies and joints as Blender objects**, display-frame collections.
3. Keep the default scale (1 MMD unit = 0.08 m) so the model imports at real meters.

### 6.2 Cleanup for VRChat

1. **Fix the armature first**: Cats Unofficial *Fix Model* (renames MMD bones to Unity names, deletes dangling bones) or MMD Tools assembly tools. Verify names per §2.1.
2. Apply transforms, check scale (same §4.5 ladder).
3. **Materials:** rebuild in Unity anyway — just ensure textures are sane (`.sph`/`.spa` sphere maps aren't readable by Unity; bake into diffuse or drop).
4. **Morphs:** MMD morphs import as Japanese-named shape keys — rename mouth morphs to `vrc.v_*`. Cats can auto-generate visemes from MMD's `あ/い/う/え/お` (map beautifully: あ→aa, い→i, う→ou-ish, え→e, お→oh).
5. **Delete the physics objects** — but harvest their structure first (next step).

### 6.3 Physics → PhysBones-ready bones

1. Identify each physics chain (hair strand: root → 3 children, each with rigid body + joint).
2. Give each chain bone real **vertex weights** on its mesh section.
3. Export the bones in the FBX (uncheck "Only Deform Bones" or make them deform); in Unity add **VRC PhysBone** per chain with limits; **Add Leaf Bones** ✅ earns its keep here (terminal bones = free Endpoint positions).
4. Leftover rigid-body objects don't export.

Gotchas: IK bones (足ＩＫ) are control bones — exclude; MMD meshes are often 100k+ tris — decimate (Tuxedo smart mode preserves shape keys); check the model's ToU before distributing.

## 7. Texturing: Blender vs. Substance, and Cycles Baking

### 7.1 Blender painting vs. Substance Painter

| Factor | Blender Texture Paint | Substance Painter |
|---|---|---|
| Cost | Free | Subscription |
| PBR workflow | Manual nodes | Excellent (smart materials, generators) |
| Anime/toon | Excellent (MToon-style nodes) | Against the tool's grain |
| Round-trip | Same app | Export/import each iteration |

Rule of thumb: toon/anime avatars → **Blender**. Realistic/sci-fi/armor → **Substance Painter**. Either way, final VRChat materials are assembled in Unity — both tools are texture *generators*.

### 7.2 Baking AO and normals in Blender (Cycles)

**AO bake:**
1. Clean UVs (no overlaps — overlapping islands double-bake).
2. New image 2048² (4096 for hero faces), **Non-Color**.
3. Insert an Image Texture node with the target image, *unconnected*.
4. Cycles, GPU.
5. Bake Type **Ambient Occlusion**, Influence: Ambient only, **Normalize** ✅, Margin 8 px, Samples 64–128, Non-Color output.
6. Bake → **save the image (`Alt+S`) — Blender doesn't auto-save bakes** (classic trap).

**Normal bake (high → low):**
1. Low poly selected, high poly shift-selected. Bake type **Normal**, ✅ **Selected to Active**, Extrusion ~0.05 m, Max Ray Distance 0.02–0.05 m for character work.
2. Samples: **16** is plenty. Margin 8–16 px.
3. Space: **Tangent Space**, Swizzle: **Y+** — Unity uses +Y (OpenGL-style); Blender 4.x defaults match.
4. Bake skinned meshes in **rest pose** with modifiers applied; "Bake from Multires" if detail came from multires.
5. Verify in Unity before building the shader stack — a flipped green channel is the #1 bake defect.

### 7.3 Atlasing and optimization

**Material Combiner** (Grim-es → Neoneko fork: github.com/teamneoneko/material-combiner-addon) merges materials into atlases; **Tuxedo** bakes everything into one atlas + decimates in one pass. Both preserve shape keys; Tuxedo's smart decimation protects visemes.

## 8. Add-On Catalog (2025–2026)

| Add-on | Status | Blender | Use |
|---|---|---|---|
| **Cats Unofficial** (Neoneko) | EOL announced; successor "Avatar Toolkit" coming | 3.6–5.0 (version-matched) | MMD/XPS import, Fix Model, bone merging, viseme generation |
| **Tuxedo** | feilen v0.4.2 (4.2–5.0) / Neoneko fork v1.2.0 (5.0+) | 4.2+ | Bake + atlas + smart decimation, twistbones |
| **MMD Tools** | v4.x current | 4.2–5.2 | PMX/PMD/VMD import-export |
| **MMD Tools Append** | v4.x current | 4.2+ | MMD helpers, rigid-body→cloth |
| **VRM Add-on** (saturday06) | v4.10.x very active | 2.93–5.2 | VRM import/export/edit, MToon |
| **Material Combiner** | Maintained | 3.x–4.x+ | UV-based atlasing |
| **TexTools** | Maintained | 4.x | UV toolbox, texel density |
| **UVPackmaster** | Commercial | 4.x/5.x | GPU UV packing |

Avoid: the dead `absolute-quantum` Cats repo, Windows-Store Blender, Blender 4.0/4.1 (worst add-on window — use 4.2 LTS).

## 9. Scripting & Automation

Run in the Scripting tab. Save first.

**Batch rename bones (MMD → Unity):**
```python
import bpy
mapping = {"左腕": "LeftArm", "右腕": "RightArm", "左足": "LeftUpLeg",
           "右足": "RightUpLeg", "左ひざ": "LeftLeg", "右ひざ": "RightLeg"}
for b in bpy.context.active_object.pose.bones:
    b.name = mapping.get(b.name, b.name)
```

**Merge weights from one bone into another:**
```python
import bpy
src, dst = "UpperArm_twist", "UpperArm"
for o in bpy.data.objects:
    if o.type != 'MESH': continue
    for v in o.data.vertices:
        for g in v.groups:
            if g.group in o.vertex_groups and o.vertex_groups[g.group].name == src:
                o.vertex_groups[dst].add([v.index], g.weight, 'ADD')
    o.vertex_groups.remove(o.vertex_groups[src])
```

**Fix normals on all selected meshes:**
```python
import bpy
for o in bpy.context.selected_objects:
    if o.type != 'MESH': continue
    bpy.context.view_layer.objects.active = o
    bpy.ops.object.mode_set(mode='EDIT')
    bpy.ops.mesh.select_all(action='SELECT')
    bpy.ops.mesh.normals_make_consistent(inside=False)
    bpy.ops.object.mode_set(mode='OBJECT')
```

**Ensure every vertex has ≥1 vertex group:**
```python
import bpy
obj = bpy.context.active_object
target = obj.vertex_groups.active  # pick target group in UI first
for v in obj.data.vertices:
    if not any(g.group in obj.vertex_groups for g in v.groups):
        target.add([v.index], 1.0, 'REPLACE')
```

**Discipline:** write against the Blender 4.x API — `bone.layers` → `bone.collections`, `use_auto_smooth` gone in 4.1+, old 2.8–3.x scripts frequently die on 4.1+.

## 10. Troubleshooting Gallery

**10.1 "Mesh explodes / 100× too big on Unity import."** FBX exported with Apply Scalings ≠ FBX Units Scale, or armature scale not applied. Fix: `Ctrl+A → All Transforms`, re-export with FBX Units Scale / −Z / Y / Apply Unit ✅. Don't patch via Unity Scale Factor — PhysBones, outlines, physics all inherit the wrong magnitude.

**10.2 Weights wrong in Unity, right in Blender.** Fix ladder: (1) check bone rolls; (2) "Only Deform Bones" didn't drop weighted bones; (3) set rig to *Generic* temporarily — right in Generic + wrong in Humanoid = mapping issue, not weights; (4) FBX exports vertex groups only — Blender's envelope overlay is irrelevant.

**10.3 Shape keys missing after export.** A modifier above the shape-key stack bakes keys away; or "Shape" unchecked; or keys live on a different mesh than the descriptor's lip-sync target. Visemes must be on the ONE mesh the Avatar Descriptor points at.

**10.4 Bones not mapping in Configure.** `.001` suffixes, non-standard names, unmapped Chest (VRChat mandatory), jaw auto-grabbing hair bones, scaled/rotated rest pose. Fix per §2.1/§4.4.

**10.5 Mirrored L/R bones misbehaving.** Mixed conventions (`LeftArm` + `arm.R`) or `.001` duplicates from `Shift+D`. Pick ONE convention (`LeftXxx`/`RightXxx` preferred), enforce with a rename script, use Symmetrize.

**10.6 "Model imports rotated 90°."** Expected with Apply Transform off — fix with Unity **Bake Axis Conversion** ✅.

**10.7 "Hair flaps while I talk."** Unity auto-mapped a hair bone to Jaw. Set Jaw = None.

## Source URLs

- Cats: https://catsblenderplugin.xyz · https://git.neoneko.org/NeoNeko/Cats-Blender-Plugin
- Tuxedo: https://github.com/feilen/tuxedo-blender-plugin · https://github.com/teamneoneko/Tuxedo-Blender-Plugin
- MMD Tools: https://github.com/MMD-Blender/blender_mmd_tools
- VRM Add-on: https://extensions.blender.org/add-ons/vrm/ · https://github.com/saturday06/VRM-Addon-for-Blender
- Visemes: https://wiki.vrchat.com/wiki/Visemes
- FBX export settings: https://bkjam.github.io/posts/2024-10-19-blender-to-unity-settings/ · https://note.com/kikjin/n/n66595f364918
- PhysBone scale: https://ask.vrchat.com/t/phys-bone-being-stiff/48989
- Material Combiner: https://github.com/Grim-es/material-combiner-addon
