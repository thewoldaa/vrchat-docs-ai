---
name: vrchat-avatar
description: Membuat, mengedit, dan debugging VRChat Avatar — Avatar 3.0, FX animator, PhysBone, expressions, optimasi rank, Quest/iOS. Gunakan skill ini saat user meminta bantuan avatar VRChat, rigging, toggles, gestur, atau masalah upload avatar.
---

# VRChat Avatar Skill

Referensi lengkap: `docs/vrchat-avatars.md` (sumber utama) dan `docs/vrchat-ecosystem.md` (platform/rules).

## Environment (2025–2026)

- Unity **2022.3.22f1** (JANGAN upgrade), VCC + template Avatars, SDK 3.10.x. Upload butuh rank New User+; Build & Test gratis.
- Instal Unity dengan Android Build Support (+ iOS) untuk build Quest.

## Pipeline

1. Blender: humanoid rig (55 bone standar), T-pose, 15 visemes `vrc.v_*` + blink, 1 skinned mesh, ≤70k tris PC / ≤10k mobile
2. Tekstur: mask packing, ≤2K body, mobile target 10–40 MB texture memory
3. Unity: FBX → Rig Humanoid → **VRC_AvatarDescriptor** (playable layers, expressions, lip sync = Viseme Blend Shape, eye look)
4. Animasi/menu → optimasi → upload PC + Android/iOS ke **blueprint ID sama**

## Avatar 3.0 essentials

- **5 playable layers**: Base (locomotion), Additive, Gesture (per tangan, GestureLeft/Right 0–8), Action (emote), **FX** (non-transform: blendshape, material, toggle). Generic avatar = hanya Base/Action/FX (root Animator Controller dibiarkan kosong).
- **256-bit budget**: bool 1 bit, int/float 8 bit. Saved-but-unsynced = gratis. **Sync lintas platform positional** — urutan parameter harus identik antara varian PC/Quest.
- Menu: maks **8 kontrol per halaman** (Button, Toggle, Sub-menu, Two/Four-Axis & Radial Puppet).
- **Write Defaults: OFF konsisten di semua state**, selalu buat animasi off-state. Direct Blend Trees menghindari banyak masalah WD.
- Blink sync otomatis via descriptor. Gestur: layer Gesture + `GestureWeight` untuk analog.

## Komponen Avatar Dynamics

- **PhysBone** (bukan DynamicBone — legacy): pull/spring/gravity/limits per chain. Biaya dihitung: components, affected transforms, colliders, collision checks — terpisah.
- **Contacts**: Sender (tag+radius) / Receiver (tulis float 0–1 ke param FX). Receiver Local Only tidak dihitung rank.
- **VRChat Constraints**: Parent/Rotation/Aim/LookAt dll — lebih baik dari Unity constraints untuk avatar.

## Rank limits (ringkas — detail di docs)

| | PC Excellent | PC Poor | Mobile Excellent | Mobile Poor |
|---|---|---|---|---|
| Tris | 32k | 70k | 7.5k | 20k |
| Texture Mem | 40 MB | 150 MB | 10 MB | 40 MB |
| Skinned Mesh | 1 | 16 | 1 | 2 |
| Material Slots | 4 | 32 | 1 | 4 |
| Bones | 75 | 400 | 75 | 150 |
| PhysBones | 4 | 32 | 0 | 8 |
| Lights | 0 | 1 | 0 | 0 |

- Read/Write mesh enabled = **instant Very Poor** (upload diblokir). Komponen disabled tetap dihitung.
- Rank dihitung server-side sejak 2025 — bisa beda tipis dari preview SDK.

## Tools non-destruktif (standar 2025–2026)

Stack: **Modular Avatar** (assembly prefab) → **TexTransTool** (atlas) → **AAO Avatar Optimizer** (Trace & Optimize) → **VRCFury** (toggle/gesture tanpa clip, Fix WD) → **Av3Emulator** (test play-mode, aktifkan "Run Preprocess Avatar Hook"). Semua di atas **NDMF**. Setelah optimasi, rank SDK tidak reliabel — pakai "Actual Performance Window".

## Shader

- PC: **lilToon** (JP standar, lebih ringan) atau **Poiyomi** (EN standar, fitur melimpah) — keduanya PC-only.
- Mobile: hanya shader native VRChat — `Standard Lite`, `Toon Lit`, **Toon Standard** (SDK 3.8.1; ramp/normal/emission/matcap). Konversi otomatis: **VRCQuestTools**.
- PC upload otomatis dapat VRCFallback tags agar degrades di Quest.

## Gotchas

- Blueprint ID (`avtr_*`) = konten sama; re-upload menggantikan. Pakai Git untuk versi.
- Fallback avatar: upload dulu dari project Android dengan flag "Is a Fallback" (harus Good+ mobile), lalu re-upload PC ke ID sama.
- Animator tambahan (di luar playable layers) memakan rank.
- Jangan blanket-merge mesh multi-outfit — bisa memperburuk performa; merge per-state.
- Param mustahil melebihi 256 bit synced → avatar gagal build; geser ke unsynced-saved bila perlu.
- Menjual avatar? Avatar Marketplace melarang NSFW; avatar terbeli tak bisa di-clone/public.

Detail penuh (tabel rank lengkap, event, contoh stack, roadmap Soba/optimizer resmi): baca `docs/vrchat-avatars.md` dan `docs/vrchat-ecosystem.md`.
