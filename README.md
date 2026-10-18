# VRChat Docs AI

Dokumentasi open-source yang dirancang agar **AI (LLM)** memahami secara mendalam: **model 3D**, **VRChat World**, dan **VRChat Avatar** — dan dipaketkan menjadi **skills** siap pakai.

## Struktur

```
docs/                          # Dokumentasi riset lengkap (sumber utama)
  3d-model-pipeline.md         # Pipeline model 3D: Blender, UV, PBR, format, optimasi, shader
  blender-workflow.md          # Blender deep-dive: rigging, viseme, FBX, VRM/MMD, troubleshooting
  vrchat-worlds.md             # VRChat World: Unity/VCC, Udon, komponen, limits, upload
  udonsharp-cookbook.md        # Kode UdonSharp lengkap: 12 contoh + networking patterns
  vrchat-avatars.md            # VRChat Avatar: AV3, PhysBone, expressions, rank limits
  av3-recipes.md               # Resep AV3 hands-on: toggle, WD, blend trees, contacts, FBT
  optimization-deep-dive.md    # Texture memory math, quest conversion, decision trees
  vrchat-ecosystem.md          # Platform: trust rank, TOS, economy, tools, roadmap
skills/
  vrchat-world/SKILL.md        # Skill AI: pembuatan & debugging VRChat World
  vrchat-avatar/SKILL.md       # Skill AI: pembuatan & debugging VRChat Avatar
  vrchat-3d-model/SKILL.md     # Skill AI: pipeline model 3D
```

## Cara pakai

Setiap `SKILL.md` berisi ringkasan padat + quick rules + gotchas; file `docs/` adalah referensi mendalam. Berikan keduanya ke AI, atau instal folder `skills/` ke Claude Code / agent lain.

## Untuk siapa

- AI yang membantu membuat world VRChat (Udon, Unity, SDK3)
- AI yang membantu membuat/mengoptimasi avatar (Avatar 3.0, PhysBone, expressions, Quest)
- Siapa pun yang memahami pipeline model 3D untuk platform real-time/VR

## Pembaruan data

Riset terakhir: Oktober 2026 (Unity 2022.3.22f1, SDK 3.9–3.10.x). Dunia VRChat berubah cepat — cek https://creators.vrchat.com/releases/ untuk versi terbaru.

## Lisensi

CC-BY-4.0
