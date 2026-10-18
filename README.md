# VRChat Docs AI

Dokumentasi open-source terlengkap yang dirancang agar **AI (LLM)** memahami secara mendalam: **model 3D**, **VRChat World**, dan **VRChat Avatar** — dipaketkan menjadi **skills** siap pakai. 17+ dokumen riset (~40k kata), diverifikasi terhadap docs resmi + komunitas per Oktober 2026.

## Struktur

```
docs/                              # Dokumentasi riset lengkap (sumber utama)
  3d-model-pipeline.md             # Fundamental: mesh, UV, PBR, format, VRAM math, shader
  blender-workflow.md              # Blender deep-dive: rigging, viseme, FBX, VRM/MMD, Python
  optimization-deep-dive.md        # Texture memory, quest conversion, decision trees
  vrchat-worlds.md                 # World: Unity/VCC, Udon, komponen, limits, upload
  udonsharp-cookbook.md            # 12 contoh kode UdonSharp + networking patterns
  udon-frameworks.md               # Tools ecosystem: ProTV, QvPen, VUdon, persistence, MIDI
  world-design.md                  # Desain world: kategori, anti-patterns, game state patterns
  world-advanced-systems.md        # Mirror, lighting, voice API, particles, camera tricks
  vrchat-avatars.md                # Avatar: AV3, PhysBone, expressions, rank limits
  av3-recipes.md                   # Resep AV3: toggle, WD, blend trees, contacts, FBT
  ma-vrcfury-ecosystem.md          # Modular Avatar / VRCFury / AAO complete reference
  shaders.md                       # lilToon, Poiyomi, Toon Standard, LTCGI, z3y
  audiolink.md                     # AudioLink: setup, shader API, Udon API, performance
  osc.md                           # OSC: input, parameters, chatbox, VRCFaceTracking, Python
  vrchat-ecosystem.md              # Platform: trust rank, TOS, economy, tools, roadmap
  dev-workflow.md                  # Git untuk Unity, vrc-get/ALCOM, CI, backup/recovery
  learning-resources.md            # Katalog: YouTube, Discord, GitHub orgs, learning paths
skills/
  vrchat-world/SKILL.md            # Skill AI: pembuatan & debugging VRChat World
  vrchat-avatar/SKILL.md           # Skill AI: pembuatan & debugging VRChat Avatar
  vrchat-3d-model/SKILL.md         # Skill AI: pipeline model 3D
```

## Cara pakai

Setiap `SKILL.md` berisi ringkasan padat + quick rules + gotchas; file `docs/` adalah referensi mendalam. Berikan keduanya ke AI, atau instal folder `skills/` ke Claude Code / agent lain.

## Untuk siapa

- AI yang membantu membuat world VRChat (Udon, Unity, SDK3, ProTV, AudioLink, persistence)
- AI yang membantu membuat/mengoptimasi avatar (AV3, Modular Avatar/VRCFury/AAO, Quest)
- Siapa pun yang memahami pipeline model 3D untuk platform real-time/VR

## Pembaruan data

Riset terakhir: Oktober 2026 (Unity 2022.3.22f1, SDK 3.9–3.10.x, AudioLink 3.1.x, lilToon 2.3.x, Poiyomi 9.3–10). Dunia VRChat berubah cepat — cek https://creators.vrchat.com/releases/.

## Lisensi

CC-BY-4.0
