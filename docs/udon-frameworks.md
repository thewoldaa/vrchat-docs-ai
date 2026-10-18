# VRChat Udon Game Frameworks & World Tooling Ecosystem — Research Report (2025–2026)

---

## 1. The "Big Three" Udon authoring tools: UdonToolkit, CyanTrigger, UdonRabbit

### UdonToolkit (attribution note)
Persistent naming confusion worth clearing up: **UdonToolkit is authored by orels1, not MerlinVR or Varneon**. MerlinVR created UdonSharp and USharpVideo; Varneon created VUdon. UdonToolkit: https://github.com/orels1/UdonToolkit (MIT, ~190 stars).

**What it offers:** attribute-driven custom inspectors (`ListView`, `Popup`, `TabGroup`), trigger behaviours with access control (whitelist/blacklist by name), `SyncedTrigger`, teleporters, player modifiers, velocity followers, plus **Flight System** and **Presentation System** (slide decks). v1.x works with UdonSharp 1.x/VCC; v2.0 (split into 4 VPM packages, listing `https://orels1.github.io/UdonToolkit/index.json`) only reached `v2.0.0-dev.2` (Apr 2024).

**Status:** effectively dormant — last push April 2024. Works, but not updated for SDK 3.8.1 networked event parameters or Soba. "Works, but unmaintained."

### CyanTrigger
(https://github.com/CyanLaser/CyanTrigger, docs https://cyanlaser.github.io/CyanTrigger/) — an **Udon compiler with in-scene editing**, event/action-list interface resembling SDK2's `VRC_Trigger`. Compiles to real Udon, interops with UdonGraph/UdonSharp, includes an SDK2 migrator.

**Status — the important one:** **maintenance mode/deprecation** — no new features, critical fixes only; author (Oct 2024) doesn't plan to update for **Soba**. Last release v4.4.8 (Oct 2024) added minimal networked-event-parameter support. Doesn't fully support newer SDK features. License: proprietary closed-source, free-to-use. Advice: don't build long-lived production worlds on it without a migration plan.

### UdonRabbit
"UdonRabbit" usually means **UdonRabbit.Analyzer** (esnya, https://github.com/esnya/UdonRabbit.Analyzer, MIT) — a **Roslyn analyzer** for UdonSharp flagging unsupported syntax at edit time (URA0001 method not exposed, URA0011 try/catch, etc.). **DEPRECATED/ARCHIVED**: "U#1.0 not supported". For U# 1.x there's no direct successor; SDK-side validation absorbed part of its role.

**Context:** UdonSharp (https://github.com/vrchat-community/UdonSharp, MIT) remains the standard (1.1.9 in VCC). The July 2025 creators roadmap confirms **Soba** (Udon successor runtime) in progress with closed beta expected + migration tooling. Also shipped in 2025: parameterized network events, `VRCGraphics`/`VRCCameraSettings`, Drones API, and **World Dynamics** (Nov 2025, 2025.4.1) — PhysBones/Contacts/Constraints usable in worlds.

## 2. Ready-made systems

### Video players: ProTV vs VideoTXL vs USharpVideo vs iwaSync

| | ProTV 3 | VideoTXL 2.x | USharpVideo | iwaSync3 |
|---|---|---|---|---|
| Author | ArchiTechAnon | vrctxl (Texelsaur) | MerlinVR | ikuko / Hoshino Labs |
| Where | protv.dev, GitLab techanon/protv, VCC | github.com/vrctxl/VideoTXL | github.com/MerlinVR/USharpVideo | github.com/hoshinolabs-vrchat/iwaSync3 |
| License | MIT/ISC open core | MIT | MIT | MIT |
| Sync | Full play/pause/seek/loop, sub-second cross-user time sync, auto ownership, URL params, multi-TV swap, pub/sub plugins, ACL | Sync + local-only players, event-driven manager, advanced access control | Master-lock, playlist, stream player | Lightweight, idles when unused |
| Status | **Actively maintained**, most feature-complete | Maintained, solid MIT alternative | Dormant (2021-era fixes) but still used | **ARCHIVED** |

Guidance: **ProTV 3** for new event-heavy/monetization-adjacent worlds; **VideoTXL** for the best local-player story; USharpVideo fine for simple worlds but not developed. Remember the **global 5-second per-user URL rate limit** — ProTV's "autoplay delay offsets" exist for staggered late-joiner loads.

### QvPen
(https://github.com/ureishi/QvPen, package `net.ureishi.qvpen`, **v3.3.15**) — the standard networked drawing pen. v3 is a rewrite with cleaner sync (Ink/Eraser/Manager behaviours). Community extensions (Exporter/Importer) let you persist/load drawings.

### Locomotion
There is **no world-side GoGo Loco equivalent** — GoGo Loco is an *avatar* prefab (franada). World-side: **VUdon Noclip** and **VUdon Simple Player Settings** (flight/gravity/jump), UdonToolkit Flight System, teleport/QuickTP prefabs. Friction point: GoGo Loco historically hijacks `VRCStation` animator controllers, breaking vehicle/seated systems; design stations defensively.

### Games: trivia, murder, golf, combat
- **Trivia:** daalta's **Infinite Trivia Prefab** (github.com/daalta/infinite-trivia-prefab) downloads questions from online trivia APIs via string loading. JP BOOTH: QuizSystem (sunnyf, ¥400), fa0311/Udon-Quiz. Jinxxy: paid quiz products (Quiz Dice Ver.2 — in-world commercial OK, no resale).
- **Murder/social deduction:** no open-source Murder 4 kit — Murder 4 (Jar) is closed, now using persistence for 52 achievements. Nearest buildable kits: BOOTH's **VRC CombatGimmick** (¥3,000, VN3 license — explicitly does NOT do per-player-role games), UdonKnucklebones-style board-game kits.
- **Golf:** **Golf Game SGB** by spookyghostboo (free, via vr.space/Patreon archives; used in Hidden Heights) — ring-target golf, up to 4 players.
- **Combat:** VRC CombatGimmick (subgames/respawn/health sync), ArcheryKit (Vowgan), MS-VRCSA-Billiards (late-joiner sync). Catalog: https://vrcreators.net/docs/vrchat/prefabs/games
- **Economy:** **UdonChips** (Lura & Kubiwak, BOOTH 3060394) — de-facto cross-gimmick currency; **UdonChips Plus** (https://github.com/JLChnToZ/udonchips-plus, MIT) is a VPM drop-in replacement with singleton enforcement, auto-binding, built-in persistence. Creator Economy item gating is separate, first-party.

## 3. Varneon's VUdon ecosystem

**VUdon** (https://github.com/Varneon/VUdon, MIT) — family of individually-versioned UdonSharp prefab packages: Simple Player Settings, Noclip, Playerlist, Logger (runtime in-world console), Array Extensions, **Udonity** (a runtime Unity Editor in Udon — invaluable for live debugging), Common, Event Dispatcher, Quick Menu, Menus, Events. In dev/closed: Music Player, Player Tracker, Roles, World Markers, Footsteps. Editor tools: **UdonExplorer** (scene-wide UdonBehaviour inspector), UdonEssentials, WorldCreatorAssistant (discontinued). Development pace has slowed since ~2023.

## 4. VRCObjectPool and object-sync patterns

**VRCObjectPool** (SDK3): fixed array of GameObjects — `TryToSpawn()` returns unused (or null), `Return(GameObject)` frees, active state syncs to late joiners, ownership follows spawner. Primitive for projectiles/pickups/round props. Caveats: no "who owns it" metadata; synced vars on pooled objects must guard `OnDeserialization` firing before assignment (the problem CyanPlayerObjectPool solves).

**CyanPlayerObjectPool** (CyanLaser, MIT, archived-but-stable) — canonical per-player pool: master assigns unique object per player via synced int array, race-condition-safe under simultaneous joins, verifies on master crash, defines the `_OnOwnerSet`/`_OnCleanup` contract. Set pool size to **2× world capacity**. In 2025+ worlds, **PlayerObjects** (§6) are the official replacement for "one object per player".

**Projectile patterns:** (a) VRCObjectPool + VRCObjectSync bullets with `FlagDiscontinuity`/`TeleportTo` on fire, shooter ownership; (b) manual-sync state machine — only impact events cross the network (cheapest, local prediction needed); (c) post-2025, PlayerObject-based projectiles avoid mid-flight ownership transfer. Set VRCObjectSync's `AllowCollisionOwnershipTransfer` **off** for projectiles.

**Round state:** single "game manager" UdonBehaviour with manual-synced state enum + round seed, owner = master or designated host, `RequestSerialization` on transition, idempotent `OnDeserialization` so late joiners reconstruct. UdonKnucklebones (github.com/wipeseals/UdonKnucklebones) documents a clean version: per-turn owner, late-joiners request full state resync via `OnPlayerJoined`.

## 5. Instance management

- **Master-only logic:** `Networking.LocalPlayer.isMaster` gates game-start/cleanup/pool assignment. Master migrates on leave — re-verify state on master change.
- **OnPlayerJoined bookkeeping:** collect joiners into arrays/DataLists, synced player count + per-player index; pair with `OnPlayerLeft` for slot frees. Ready-made: VUdon-Playerlist.
- **Join queues:** queue DataList + manual-synced position array; players claim slots via owner-transfer of a "ticket" object, or master-validated assignment. Most big event worlds roll their own on top of CyanPlayerObjectPool.
- **Whitelists via string arrays:** `public string[] allowedUsers;` compared exactly against `VRCPlayerApi.displayName` (case-sensitive). Ready-made: Kreyadian/Simple-Whitelist, VideoTXL ACL module, UdonToolkit access control, rhaamo's AdminManager.
- **Remote admin lists:** host a Pastebin/Gist list of admin names, re-poll periodically (Vowgan's tutorial "Creating Admin Lists in VRChat" is canonical). **VRChat Lazy Admin** (SketchFoxsky, Jinxxy) productizes dynamic admin access from an online text list. Trusted string domains: GitHub/GitHub Pages, gists, Pastebin, Disbridge, VRCDN.
- Avatars 3.0 Manager is an *avatar* tool, not world tooling.

## 6. Persistence patterns (2025–2026 status)

**Persistence is live** since VRChat 2024.4.1p3 / SDK 3.7.4 (Nov 2024): **PlayerData** (per-player key-value store) and **PlayerObjects** (auto-instantiated per-player GameObjects whose synced variables persist when marked with `VRCEnablePersistence`). Docs: creators.vrchat.com/worlds/udon/persistence/ (+ /player-data/ + /player-object/).

Key facts:
- **Limits:** 100 KB PlayerData + 100 KB PlayerObject data per player per world (compressed). No key deletion yet; no cross-world sharing; no built-in save slots (roll your own via keys); **nothing can be saved in OnPlayerLeft** — data must be saved before the local player leaves.
- **Timing:** always wait for `OnPlayerRestored(player)` before reading/writing; it fires for every player's data on everyone's client (including empty first visits; community testing found inconsistencies). Guard remote events with `player.isLocal`.
- **Events:** `OnPlayerDataUpdated` (key states Unchanged/Added/Removed/Changed/Restored), `OnPersistenceUsageUpdated`, storage-exceeded/warning events, `Networking.GetPlayerDataStorageLimit/Usage`, `RequestStorageUsageUpdate`.
- **Bandwidth model:** PlayerData behaves like one manual-sync UdonBehaviour; batch keys; split hot small counters (PlayerData) from large blobs (PlayerObject).
- **2025 hardening:** VRChat 2025.4.1 (Nov 2025) added **network-layout compatibility enforcement** — joining an instance of an older/incompatible world version is blocked, protecting persistence data. Be careful updating live event worlds.
- Local testing: Build & Test stores locally per test client; ClientSim as project JSON.

Typical patterns: save-on-change with debounce; PlayerData for scores/wallets/settings (UdonChips Plus does exactly this); PlayerObject for per-player health/loadouts/cosmetics. Murder 4's achievements are a shipped example.

## 7. VRCJson, DataList/DataDictionary

**Data Containers** — `DataList` (ordered), `DataDictionary` (keyed), holding `DataToken`s of any Udon-compatible value including nested containers. **VRCJson**: DataList ↔ JSON array; DataDictionary (string keys) ↔ JSON object. Dependable since SDK 3.7.4-era fixes.

Canonical save-state pattern:
```csharp
var dict = new DataDictionary();
dict["round"] = new DataToken(roundNumber);
var scores = new DataList();
scores.Add(new DataToken(p1 + ":" + s1));
dict["scores"] = new DataToken(scores);
string json = VRCJson.SerializeToJson(dict); // check JsonError out-arg
```
Parse with `TrySerializeToJson`/`TryDeserializeFromJson` — malformed input is common from remote hosts. **Udon has no DateTime API** — use `Networking.GetNetworkDateTime()` or incremented sync clocks (Jetdog8808's DateTime_UI). Tokens must wrap every value; `TryGetValue` returns bool; handle `\r`/`\n` explicitly for cross-platform text files.

## 8. VRCImageDownloader & string loading

**Image loading** (creators.vrchat.com/worlds/udon/image-loading/ + official example repo github.com/vrchat-community/examples-image-loading):
- Construct `new VRCImageDownloader()` once and **keep it in a field** (else GC kills your textures).
- `DownloadImage(url, material, this, textureInfo)`; handle `OnImageLoadSuccess/Error`; **Dispose** old textures or long-session worlds OOM.
- Limits: 2048×2048 max, one download per 5 seconds globally (queued, max 1000), no redirects, 32 MB buffers, **allowlisted domains only** (imgur, GitHub Pages, i.ibb.co, reddit, pbs.twimg.com, VRCDN, assets.vrchat.com…) — off-list needs user's "Allow Untrusted URLs".
- Uses: dynamic galleries/posters, seasonal banners, marketplace displays, remote-config textures.

**String loading:** `VRCStringDownloader.LoadUrl(url, this)` + `OnStringLoadSuccess/Error` with `IVRCStringDownload.Result` (UTF-8) or `ResultBytes`. Trusted domains: GitHub Pages, gists, Pastebin, Disbridge, VRCDN. Uses: remote admin lists, announcements, **remote leaderboards** (JSON/CSV on a gist, parse with VRCJson, render to TextMeshPro), remote avatar/world lists (adding pedestals/portals without re-upload), trivia feeds (daalta's Infinite Trivia).

## 9. MIDI, haptics, VRSL, LTCGI integration

- **MIDI:** realtime via `VRCMidiListener` (`MidiNoteOn/Off`, `MidiControlChange`, `--midi=<name>`) and file playback via `VRCMidiPlayer` + `.mid` assets with BPM matching to an AudioClip (`MidiData/MidiTrack/MidiBlock` for pre-analysis). VRSL supports MIDI output from its grid node; instrument-driven visuals are the flagship use.
- **Haptics:** `VRCPlayerApi.PlayHapticEventInHand(hand, duration, amplitude, frequency)`; pickup-side `GenerateHapticEvent`. On Index controllers the **frequency parameter is ignored** — don't design frequency-coded feedback.
- **VRSL (VR Stage Lighting)** (AcChosen, MIT, v2.8.1 Jan 2026): professional stage lighting via DMX512-over-video — Artnet GridNode render texture decoded by shaders; AudioLink mode needs no grid. DMX screen prefabs for USharpVideo/VideoTXL/ProTV; URP since 2.8.0; shaders MIT but the **Artnet GridNode editor asset is paid/closed** (open alternative: HNode). Integrate: include `VRSLDMX.cginc` to make any property DMX-controllable.
- **LTCGI:** realtime area-lighting for screens/emitters; **free with attribution** (include the attribution prefab or credit line). Integrates with ProTV 3+, AudioLink (band-reactive emitter colors), VRC Light Volumes since 1.7.0; requires compatible receive shaders (ORL, Filamented, Mochie, Poiyomi, z3y…).

## 10. Licensing & watch-outs for commercial/economy use

- **Permissive (safe for monetized worlds):** MIT dominates — UdonSharp, VUdon, USharpVideo, VideoTXL, QvPen, CyanPlayerObjectPool, UdonChips Plus, VRSL. Keep license headers when redistributing.
- **Attribution-required:** LTCGI (must credit in-world), UdonVR scripts (link-back), UserMovingFloor (Zlib — optional but convention is credit).
- **Proprietary/free-to-use:** CyanTrigger (closed, deprecated — plan migration), VRSL's Grid Node (paid), Gumroad add-ons (VRSL GI).
- **BOOTH/Jinxxy (VN3-style):** read each listing. Common pattern: use in your own world allowed (including monetized worlds), **redistribution forbidden**; some forbid sharing even modified. Truth or Dare needs a separate **commercial license** if the world is sold.
- **General:** video content subject to allowlist + rate limits regardless of player license; image/string hosts outside trusted lists silently fail for most users; persistence data is account-bound — never design economies assuming cross-world transfer; with Soba on the horizon, prefer actively maintained packages (ProTV, QvPen, AudioLink, VUdon) for anything long-lived.

### Key sources

- https://github.com/orels1/UdonToolkit · https://github.com/CyanLaser/CyanTrigger · https://wiki.vrchat.com/wiki/Community:CyanTrigger · https://github.com/esnya/UdonRabbit.Analyzer · https://github.com/vrchat-community/UdonSharp
- https://protv.dev · https://github.com/vrctxl/VideoTXL · https://github.com/MerlinVR/USharpVideo · https://github.com/ureishi/QvPen
- https://github.com/Varneon/VUdon · https://github.com/llealloo/audiolink · https://github.com/AcChosen/VR-Stage-Lighting · https://ltcgi.dev
- https://github.com/CyanLaser/CyanPlayerObjectPool · https://github.com/JLChnToZ/udonchips-plus · https://github.com/daalta/infinite-trivia-prefab · https://github.com/Kreyadian/Simple-Whitelist
- https://creators.vrchat.com/worlds/udon/persistence/ · /data-containers/ · /image-loading/ · /string-loading/ · /midi/ · https://docs.vrchat.com/docs/vrchat-202541
