---
name: vrchat-world
description: Membuat, mengedit, dan debugging VRChat World dengan Unity + SDK3 + Udon. Gunakan skill ini saat user meminta bantuan membuat world VRChat, scripting Udon/UdonSharp, komponen VRChat (VRC_Station, pickups, video player), optimasi ranking world, atau upload world.
---

# VRChat World Skill

Referensi lengkap: `docs/vrchat-worlds.md` (overview), `docs/udonsharp-cookbook.md` (kode UdonSharp lengkap siap pakai), `docs/udon-frameworks.md` (tools ecosystem: ProTV, QvPen, persistence, VRCJson), `docs/world-design.md` (desain & game state patterns), `docs/world-advanced-systems.md` (mirror, voice API, particles), `docs/shaders.md` §5 (world shaders, LTCGI), `docs/testing-qa.md` (test workflow), `docs/glossary.md` (jargon).

## Environment (2025–2026)

- **Unity 2022.3.22f1** — JANGAN upgrade ke LTS lain. Built-in Render Pipeline only (URP/HDRP tidak didukung).
- Project via **VCC** (VRChat Creator Companion), template Worlds, SDK 3.9.x. Paket di `Packages/` (VPM), bukan `Assets/`.
- UdonSharp sudah bundled. Upload butuh akun VRChat (bukan Steam-only) rank ≥ New User. SDK2 mati total sejak Nov 2025.

## Udon essentials

- Script = `UdonBehaviour` di GameObject; UdonSharp (U#) adalah standar. Single-threaded — hindari kerja berat di `Update` (dibagi dengan semua pemain di instance).
- Event kunci: `Interact()`, `OnPickup/OnDrop/OnPickupUseDown`, `OnTriggerEnter/Exit`, `OnPlayerTriggerEnter(VRCPlayerApi)`, `OnPlayerJoined/Left`, `OnVideoStart/End/Loop/Error`, `OnOwnershipRequest`.
- Sync: `[UdonSynced]` (None/Continuous/Manual) + `RequestSerialization()`; ownership via `Networking.SetOwner(...)` / cek `Networking.IsOwner(...)`.
- Network events: `SendCustomNetworkEvent(NetworkEventTarget.All/Others/Owner/Self, "Method")` — sejak SDK 3.8.1 bisa sampai 8 parameter dengan `[NetworkCallable]` (rate limit 5/detik/event).
- Delay: `SendCustomEventDelayedSeconds("M", f)` / `DelayedFrames`.
- Player: `Networking.LocalPlayer` → `TeleportTo(pos, rot)`, `Respawn()`, `Immobilize(bool)`, `UseAttachedStation()`.

## Komponen kunci

| Komponen | Catatan |
|---|---|
| VRC_Station | Duduk/kursi |
| VRC_Pickup | Objek dipegang; AutoHold checkbox (3.9.0) |
| VRC_ObjectSync | Sync transform fisik |
| VRC_UiShape | Wajib agar world-space canvas bisa diklik |
| VRC_SpatialAudioSource | Spatial audio; bila bisu setelah upload → Enable Spatialize + Spatial Blend 1 + rolloff |
| VRC_AvatarPedestal | Ganti avatar dunia |
| VRC_Mirror | MAHAL — maks 1, selalu default OFF, kasih toggle |
| VRC_PortalMarker/Spawner | Portal |

`VRC_Trigger` (SDK2) tidak ada di SDK3 — semua trigger via Udon.

## Budget performa

- Android/Quest: **100 MB compressed hard cap**, ~**250k tris** seluruh world, tekstur ≤1024 + ASTC.
- Public world: jaga <200 MB. Video player ≤2 per room. Mirror ≤1, default off.
- Post-processing: mati total di Android; di PC hindari SSAO/SSR/CA.
- Target ≥45 FPS dengan 1 user VR di spawn.
- **Bake lighting + occlusion culling** = kemenangan performa terbesar. Lightmap + light probes; minimalkan realtime light.

## Upload pipeline

1. **ClientSim** (VPM) untuk test di editor → **Build & Test** lokal (tanpa rank, sejak 3.9.0)
2. Build & Publish PC → switch build target Android → publish lagi (Quest) → iOS multi-platform (3.8.1+)
3. Upload pertama = private → publish ke **Community Labs** (rank "User"+, 1 world baru/7 hari) → bagus → otomatis Public
4. Content Warnings di Builder tab: Sexually suggestive / Adult language / Graphic violence / Excessive gore / Extreme horror
5. Shader Android: custom boleh (world), rekomendasi `VRChat/Mobile/Lightmapped` / `Toon Standard` (3.8.1+)

## Gotchas

- AVPro video player tidak jalan di editor — hanya Build & Test. Resolusi video max efektif ~800×600 (SVGA); 1080p blur.
- Video URL rate limit: 1 URL baru / 5 detik / user (global antar player) — stagger load untuk late joiner.
- Trusted domains untuk instance publik; URL harus https.
- Shader harus support single-pass stereo (hindari grab-pass/screen-space).
- Jangan import `.blend` langsung — export FBX.
- Pemain bisa spawn item sendiri ke world Udon — desain collision dengan itu pikirkan.

Detail lengkap (event list penuh, sync patterns, versi SDK, Soba/Udon 2 status, tools VPM seperti ProTV/LTCGI/VRWorldToolkit): baca `docs/vrchat-worlds.md`.
