# The Curated Catalog: Best VRChat Creation Learning Resources (2025–2026)

*A field guide to where the VRChat creator community actually learns — official docs, YouTube, Discords, GitHub, courses, tool docs, learning paths, and how to stay current.*

---

## 1. Official VRChat Resources

### creators.vrchat.com — the Creator Portal
**URL:** https://creators.vrchat.com · **Lang:** EN · **Freshness:** continuously updated

The single most important site for any VRChat creator, in four major sections:

- **SDK / Getting Started** (`/sdk/`) — installing VCC, first Avatar/World project, supported Unity version page, platform setup (PC/Android/Quest/iOS). The canonical "day one" document.
- **Avatars** (`/avatars/`) — *Creating Your First Avatar*, Avatars 3.0 (expression menus, playable layers, parameters), PhysBones, optimization tips, whitelisted components list.
- **Worlds** (`/worlds/`) — *Creating Your First World*, optimization & Community Labs guidance, scene components, Unity layers, the **Udon Example Scene** (a treasure chest of copy-paste-able prefabs).
- **Udon** (`/worlds/udon/`) — Node Graph vs UdonSharp, networking/sync, `VRCGraphics`, debugging.

**Best for:** ground truth. Every third-party tutorial eventually disagrees with the SDK; this site wins. **Weakness:** reference, not a course — it tells you *what*, community videos teach *how*.

### ask.vrchat.com — official forums
Discourse forum: Help! (avatar/world), Tutorials Tools & Resources, Tool Creators, Dev Updates. When your avatar fails to upload with an obscure Unity error, someone here has hit it too. **Best for:** troubleshooting edge cases and official dev updates as they drop.

### hello.vrchat.com/blog — official blog
News and Trust & Safety announcements. Posts that matter to creators: mobile launch (why iOS builds became mandatory), age assurance & content gating (changed how uploads work), Developer Update summaries, community events.

### wiki.vrchat.com — community wiki
Tracks releases closely (front page shows current build with changelog). Covers Getting Started, Controls, Avatars (fallbacks, impostors, expressions, Avatar Dynamics), VRC+, Creator Economy. **Best for:** plain-language conceptual explanations, faster to navigate than official docs.

## 2. YouTube Channels (verified active 2024–2026)

1. **VRChat Learning Channel** (official) — official VCC walkthrough series (~76K views). Safest first stop, kept aligned with the SDK. EN.
2. **Sipp's Avatar Box** (~25.6K subs) — *the* most-recommended avatar tutorial channel on r/VRchat in 2025. 54 videos: Basic Avatar Setup (271K views), AV3 toggles (413K), facial expressions (253K), PhysBones (335K), AudioLink-reactive avatars, Unity 2022 migration. EN.
3. **Iconoclass** — "Create Your First VRChat World" playlist (15 videos, ~234K views): modeling, materials, lighting, mirrors, Udon, pickups, video players, PC+Quest upload. The best structured free world-building course. EN (2023–2024, still the standard reference).
4. **PlayerBush001** — actively uploading Udon/SDK3 scripting: clickable doors, multi-key doors, object pools, toggles, sharing Udon graph code. One of the few consistently active Udon-specific channels. EN.
5. **Kadeko** — world tutorials including **VRC Light Volumes** (May 2025), the modern baked-lighting replacement. KO/JP with EN notes.
6. **Akira / VRC Akira** — 2025 world-building shorts: materials, albedo/normal maps, Quest-friendly textures, with inspectable demo worlds. EN.
7. **Crispy Carrots** — "Crispy Avatars Tutorials 2026 update": avatar-base setup with VRCFury, Avatar Optimizer, Gesture Manager, face-tracking templates. Few covering the *modern non-destructive toolchain*. EN.
8. **HTC VIVE (official)** — "How to Make Your Own Custom VRChat Avatar in 2025" (VRoid → VRM converter → upload pipeline). EN, 2025.
9. **Samaye** — "How to Make a VRChat Avatar in 2025 | Blender" — current two-part guide on the western kitbash workflow (Jinxy/Gumroad bases, CATS, weight transfer). EN.
10. **Tupper's VRChat Tutorials** — long-running (former VRChat community manager); older but orients newcomers well. EN.
11. **禅zenアネキ (Zen Aneki)** — legendary in JP; the "ワニでもできる" Blender/Unity series is what nearly every Japanese beginner cites. JP.

Honorable mentions: **Vowgan** (Udon logic breakdowns), **CyanLaser** (world triggers), **HIYU** (lighting micro-tutorials).

## 3. Discord Communities

- **Official VRChat Discord** — dedicated creator help channels for Avatars, Worlds, Udon + SDK announcements. Fastest sanity check for an upload error.
- **VRLabs Discord** — support for Av3Emulator, Avatars 3.0 Manager, Marker, Follower.
- **Udon/UdonSharp community** — compiler quirks and networking questions.
- **Modular Avatar / nadena.dev support** — de-facto support for MA and non-destructive workflows; anatawa12 responsive here.
- **VRCFury Discord** — VRCFury, TPS ecosystem.
- **Tool servers** — AudioLink, Poiyomi, CATS/Avatar Toolkit (Team Neoneko), VRC Wiki Discord.

Tip: the "Need help?" boxes on creators.vrchat.com link directly to the official Discord and ask.vrchat.com — use those as entry points.

## 4. GitHub Organizations to Watch

- **github.com/vrchat-community** — **UdonSharp** (C#→Udon compiler), **ClientSim**, **osc** docs, **EasyQuestSwitch**, **template-package** (build your own VPM package), creator-companion issue tracker.
- **github.com/VRLabs** — **Avatars-3.0-Manager**, **Marker**, **World-Constraint**, **Follower**, **Contact-Tracker**, Av3Emulator.
- **github.com/anatawa12** — **AAO: Avatar Optimizer** (the standard non-destructive optimization suite), **ContinuousAvatarUploader** (batch-upload avatars), **git-vrc**; VPM repo at vpm.anatawa12.com.
- **github.com/nadena-dev** — **Modular Avatar** (docs at nadena.dev).
- **github.com/lilxyzw** — **lilToon** (most-used JP shader, now supporting VRC Light Volumes), **lilAvatarUtils**, **lilycalInventory**, **lilFurGenerator**, **Shader-MEMO** (free JP shader-dev reference).
- **github.com/poiyomi** — **PoiyomiToonShader** (V9.x, Pro tier, VRC Light Volumes support).
- **github.com/Varneon** — **VUdon** (UdonSharp prefab ecosystem), **UdonExplorer**, **UdonEssentials**.
- **github.com/TeamNeoneko** — **CATS Blender Plugin** continuation, **Avatar-Toolkit** (multi-platform avatar prep). Primary dev now at git.neoneko.org.

## 5. Paid / Free Courses and Schools

**Japanese ecosystem (richest):**
- **note.com** — JP community's tutorial medium. にわとりP's 5-part "from avatar creation to selling on BOOTH" (free); Eucrid's キメラアバター講座 2025年10月版 (free, current); regularly-updated tool/shader guides. Search "VRChat アバター 作り方".
- **BOOTH (booth.pm)** — avatar marketplace *and* tutorial marketplace: paid PDF/video guides for modding, rigging, tools.
- **Kotobukiya official avatar guides** (avatar.kotobukiya.co.jp) — free JP video series.
- **Zen Aneki's video series** (free, JP) — the informal "school" most JP learners attend.

**English ecosystem:** no dominant paid academy; learning is free-tier: official docs + YouTube (Iconoclass, Sipp's) + Discord help. Iconoclass's world playlist effectively functions as a free English "world school."

## 6. Tool Documentation Sites

| Site | URL | Best for | Lang |
|---|---|---|---|
| VRCFury | https://vrcfury.com | Non-destructive toggles/gestures via GUI; "for Artists" section | EN |
| Modular Avatar | https://nadena.dev | Avatar assembly; manual doubles as a non-destructive tutorial | EN/JP |
| anatawa12 VPM + AAO | https://vpm.anatawa12.com | AAO, CAU, gists pack; optimization components | EN/JP |
| UdonSharp docs | https://udonsharp.docs.vrchat.com | Official U# reference | EN |
| AudioLink | https://audiolink.dev | Audio-reactive prefabs, world/avatar shaders | EN |
| ProTV | https://protv.dev | Media player system setup/API | EN |
| TexTransTool | https://ttt.kaikoga.info + GitHub wiki | Texture painting/decals/ATX | JP/EN |
| lilToon | github.com/lilxyzw/lilToon wiki | Shader features, migration, Light Volumes notes | JP/EN |
| Poiyomi | github.com/poiyomi/PoiyomiToonShader | V9 features, modules, Pro licensing | EN |
| VRCLightVolumes (REDSIM) | github.com/REDSIM/VRCLightVolumes | HowToUse + BestPractices — essential 2025 lighting reading | EN |

## 7. Learning Paths

### Path A: "I know nothing → my first avatar is uploaded"
1. Account & rank: VRChat account, play to "New User" trust rank.
2. (Optional) No-Unity warm-up: VRoid Studio.
3. Install toolchain: creators.vrchat.com/sdk/ + official VCC video. Create an "Avatar" project.
4. First upload: "Creating Your First Avatar" + Sipp's "Basic Avatar Setup".
5. Test before upload: Gesture Manager or Av3Emulator in-editor.
6. Expressions & toggles: Sipp's AV3 videos, or skip animator pain with **VRCFury** / **Modular Avatar**.
7. Optimize: official Optimizing Tips + **AAO** docs.
8. Shaders: install **lilToon** or **Poiyomi** via VCC.
9. When something breaks: ask.vrchat.com "Help! avatar" or official Discord.

### Path B: "I know nothing → my first world is live"
1. VCC → **World** project.
2. "Creating Your First World" (creators.vrchat.com).
3. Iconoclass's 15-video playlist end-to-end.
4. Greybox: ProBuilder + **VRWorld Toolkit** for mistake-checking.
5. Lighting: official world docs → HIYU/Akira → **VRCLightVolumes** for 2025-quality lighting.
6. Interactivity: Udon Node Graph first (official docs + Udon Example Scene), then **UdonSharp** with PlayerBush001's videos.
7. Video/audio: **ProTV**, **AudioLink**.
8. Test & optimize: ClientSim, official Community Labs tips; submit through Labs.

## 8. Staying Current

- **SDK releases & deprecations:** creators.vrchat.com changelog, creator-companion GitHub issues, official Discord SDK-announcements.
- **Developer updates:** ask.vrchat.com "Dev Updates" (roughly biweekly), summarized on wiki.vrchat.com front page; major items on hello.vrchat.com/blog. Policy shifts (age assurance, content gating) land here first.
- **Tool updates:** GitHub releases of §4 orgs; VCC shows updates in-app; vpm.anatawa12.com and vrcfury.com announce breaking changes.
- **Community pulse:** r/VRchat for what beginners actually hit.
- **Unity version changes:** "Current Supported Unity Version" page — always check before migration.

### Honesty notes on verification
Activity dates come from listings fetchable during research; some search calls hit rate limits, so not every Discord invite or upload date was deep-verified. The consistently recommended names in 2025–2026 community discussions are Sipp's Avatar Box, Iconoclass, and Zen Aneki. Where freshness couldn't be confirmed, the tool's GitHub wiki is the fallback.
