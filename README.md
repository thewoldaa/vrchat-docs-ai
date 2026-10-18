# VRChat Docs AI

Dokumentasi open-source terlengkap yang dirancang agar **AI (LLM)** memahami secara mendalam: **model 3D**, **VRChat World**, dan **VRChat Avatar** — dipaketkan menjadi **skills** siap pakai. 24 dokumen riset (~55k kata), diverifikasi terhadap docs resmi + komunitas per Oktober 2026, difact-check oleh agen kritik independen.

## Struktur

```
docs/                              # Dokumentasi riset lengkap (sumber utama)
  FUNDAMENTAL & PIPELINE
  3d-model-pipeline.md             # Fundamental: mesh, UV, PBR, format, VRAM math, shader
  blender-workflow.md              # Blender deep-dive: rigging, viseme, FBX, VRM/MMD, Python
  optimization-deep-dive.md        # Texture memory, quest conversion, decision trees
  sdk-pipeline.md                  # Internals build SDK: NDMF, bundle, validasi, blueprint ID
  dev-workflow.md                  # Git untuk Unity, vrc-get/ALCOM, CI, backup/recovery
  testing-qa.md                    # Av3Emulator/GestureManager/ClientSim, checklist, error catalog
  WORLD
  vrchat-worlds.md                 # World: Unity/VCC, Udon, komponen, limits, upload
  udonsharp-cookbook.md            # 12 contoh kode UdonSharp + networking patterns
  udon-frameworks.md               # Tools ecosystem: ProTV, QvPen, VUdon, persistence, MIDI
  world-design.md                  # Desain world: kategori, anti-patterns, game state patterns
  world-advanced-systems.md        # Mirror, lighting, voice API, particles, camera tricks
  AVATAR
  vrchat-avatars.md                # Avatar: AV3, PhysBone, expressions, rank limits
  av3-recipes.md                   # Resep AV3: toggle, WD, blend trees, contacts, FBT
  ma-vrcfury-ecosystem.md          # Modular Avatar / VRCFury / AAO complete reference
  avatar-design-patterns.md        # Desain avatar, base ecosystem, komisi, FBT, aksesibilitas
  avatar-troubleshooting.md        # Index symptom → penyebab → fix
  CROSS-CUTTING
  shaders.md                       # lilToon, Poiyomi, Toon Standard, LTCGI, z3y
  audiolink.md                     # AudioLink: setup, shader API, Udon API, performance
  osc.md                           # OSC: input, parameters, chatbox, VRCFaceTracking, Python
  platform-features.md             # Impostors, fallbacks, IK 2.0, scaling, groups, cross-platform
  vrchat-ecosystem.md              # Platform: trust rank, TOS, tools, roadmap
  creator-economy.md               # Credits, Tilia, Avatar Marketplace, subscriptions, gifting
  META
  learning-resources.md            # Katalog: YouTube, Discord, GitHub orgs, learning paths
  glossary.md                      # Decoder jargon (AV3, NDMF, WD, FBT, SPS...)
skills/
  vrchat-world/SKILL.md            # Skill AI: pembuatan & debugging VRChat World
  vrchat-avatar/SKILL.md           # Skill AI: pembuatan & debugging VRChat Avatar
  vrchat-3d-model/SKILL.md         # Skill AI: pipeline model 3D
```

## Cara pakai

Setiap `SKILL.md` berisi ringkasan padat + quick rules + gotchas + pointer ke docs terkait; file `docs/` adalah referensi mendalam. Berikan keduanya ke AI, atau instal folder `skills/` ke Claude Code / agent lain. Mulai dari `glossary.md` bila istilah asing.

## Pembaruan data

Riset terakhir: Oktober 2026 (Unity 2022.3.22f1, SDK 3.9–3.10.x, AudioLink 3.1.x, lilToon 2.3.x, Poiyomi 9.3–10). Dunia VRChat berubah cepat — cek https://creators.vrchat.com/releases/. Angka versi spesifik dalam dokumen bisa usang; angka limit resmi (rank, size, parameter budget) terverifikasi terhadap docs resmi per tanggal riset.

## Lisensi

CC-BY-4.0

