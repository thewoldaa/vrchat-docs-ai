---
name: vrchat-3d-model
description: Pipeline model 3D untuk platform VR/sosial — Blender modeling, rigging, UV, tekstur PBR, format FBX/glTF/VRM, optimasi poly/texture memory, shader. Gunakan skill ini saat user bekerja dengan model 3D untuk VRChat atau platform real-time lainnya.
---

# VRChat 3D Model Skill

Referensi lengkap: `docs/3d-model-pipeline.md` (sumber utama, baca bila butuh detail).

## Quick rules

1. **Scale**: 1 unit = 1 meter. Karakter humanoid 1.6–1.8 m. FBX dari Blender: Scale 1.0, Forward -Z, Up Y.
2. **Polycount avatar VRChat** (tris): PC Excellent ≤32k, PC max 70k; Mobile Excellent ≤7.5k, max 20k.
3. **Texture memory**: PC Excellent ≤40 MB, Good ≤75 MB; Mobile Excellent ≤10 MB, Good ≤18 MB. 2048² BC7 = 5.3 MB (+mips); 2048² RGBA32 = 21.3 MB (jangan biarkan uncompressed!).
4. **Bones**: PC Excellent ≤75, Good ≤150. Mobile 75/90.
5. **Material slots**: Excellent ≤4; 1 SkinnedMesh + ≤3 static meshes. Atlasing adalah optimasi #1.
6. **Bind pose**: T-pose wajib untuk upload VRChat & VRM.
7. **Visemes**: 15 blendshape `vrc.v_aa…vrc.v_sil`; blink = `vrc.blink`. Nama awalan `vrc.` auto-detected SDK.
8. **Quads untuk modeling, triangle untuk export** — triangulate di tahap akhir.
9. **Mobile**: ASTC compression (Quest tidak support BC); PC: BC7 (RGBA) / DXT1 (RGB) / BC5 (normal). Dimensi harus kelipatan 4.
10. **Blend file jangan langsung di-import** ke Unity — export FBX.

## Workflow end-to-end

1. Blender: blockout → model/sculpt → retopo (30–60k tris PC; ≤10k mobile)
2. UV: unwrap, texel density konsisten (~800 px/m), rencanakan atlas (≤4 material)
3. Rig: armature humanoid T-pose, weight paint (max 4 bone/vertex), shape keys `vrc.*`
4. Tekstur: bake high→low (normal MikkTSpace, OpenGL/Y+), channel-pack masks, 1024–2048
5. Export FBX: scale 1.0, apply modifiers
6. Unity (VCC + SDK3): import → rig Humanoid → prefab: VRCAvatarDescriptor, 1 SkinnedMesh, lilToon/Poiyomi, PhysBones, expression menu/params
7. Optimasi: cek SDK Performance panel; merge mesh/material, compress texture (override ASTC di tab Android!), strip blendshape tak terpakai, disable Read/Write
8. Test: visemes, blink, eye-look, first-person (sembunyikan interior kepala), lalu upload

## Keputusan format

| Kebutuhan | Format |
|---|---|
| VRChat avatar/world | FBX (Blender → Unity) |
| VRM (Cluster, VRoid Hub) | VRM 1.0 (z+ forward) / 0.x legacy (z− forward) |
| MMD | PMX via MMD Tools (jangan jadi format interchange) |
| Prop statis | OBJ/FBX |

## Red flags performa (avatar instant Very Poor)

- Mesh Read/Write enabled
- Texture 4096 uncompressed RGBA32
- >8 material slots, banyak SkinnedMeshRenderer
- PhysBones berlebihan + collider banyak
- Light/component realtime di avatar

Detail lengkap (rumus VRAM, tabel kompresi, shader toon, transparansi, animasi): baca `docs/3d-model-pipeline.md`.
