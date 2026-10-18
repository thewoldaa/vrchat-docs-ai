# VRChat Platform Ecosystem Report for World & Avatar Creators (as of 2025–2026)

*Compiled October 2026 from official VRChat documentation, wiki, legal pages, and community sources.*

---

## 1. Platform Basics

### 1.1 Instances

Instances are parallel sessions ("rooms") of a world, identified by World ID + Instance ID. Each has a region (server), a max-user capacity set by the world author, and a join-type:

| Type | Who can join |
|---|---|
| **Public** | Anyone; listed publicly |
| **Friends+** | Your friends and friends-of-friends (transitively) |
| **Friends** | Only your direct friends |
| **Invite+** | Join requests allowed from your friends' networks |
| **Invite** | Explicit join requests/invites only |
| **Group** | Only members (or specific roles) of the hosting group |
| **Group+** | Group members plus friends-of-members |
| **Group Public** | Anyone; group-hosted, with appointed group moderators able to moderate |

Key mechanics: the **Instance Owner** can insta-kick without a vote; everybody else relies on **vote-kick** (New User+ vote; successful kick bans the user from the instance for 1 hour). Instances support an **age gate** flag (`~ageGate`) requiring Verified 18+ status.

### 1.2 Trust / Rank System

Hidden aggregate of behavior signals feeding visible **Trust Ranks**:

- **Visitor** (grey) — starting rank; cannot upload.
- **New User** (blue) — can upload avatars and worlds (worlds stay private). This is the upload gate.
- **User** (green) — can publish worlds to **Community Labs**, can send reports, can receive shop gifts.
- **Known User** (orange) and **Trusted User** (purple) — can toggle displayed rank to "User."
- Special: **Friends** (yellow), **Nuisance** (muted + avatar blocked by default), **VRChat Team**.

Ranking up is by normal play; AFK-idling, mass-friending, spam-uploading do *not* raise trust. VRC+ grants a one-time trust boost (minimum reaching User). Inner workings deliberately opaque.

### 1.3 Age Verification

Introduced January 2025 via third-party **Persona**: government ID photo + liveness selfie check; VRChat receives only extracted text (only birth-date-derived hash kept). Optional badges "Age Verified" / "Verified 18+". Unlocks **Age-Gated instances**. In **H1 2026** VRChat will reintroduce **Content Gating** tied to Age Verification: public/non-verified spaces must be general-audience; some tagged content will require Age Verification (https://hello.vrchat.com/blog/trust-safety-update-2025).

### 1.4 Safety Settings & Shield Levels

Per trust rank, each user configures: voice, avatar display, user icons/emoji, avatar audio, lights & particles, shaders (blocked shaders revert to Standard), custom animations. Presets: **Maximum**, **Normal** (default; malicious avatar features suppressed automatically), **None**, **Custom**. One-click "Safe Mode" = Maximum. Friends always show full features at Normal.

### 1.5 Content Gating & Content Warnings

Uploads can carry **Content Warning labels**: **Sexually suggestive, Adult language and themes, Graphic violence, Excessive gore, Extreme horror**. Applied at upload or via the website; users filter per-label; users under 18 have all filters forced on.

### 1.6 Fallback Avatars & Performance Ranks

**Avatar Performance Ranking System** grades Excellent → Good → Medium → Poor → Very Poor (static analysis): PC Excellent ≤32k tris / 40 MB texmem; Mobile Excellent ≤7,500 tris / 10 MB. Mobile default block rank **Medium**, cannot be fully disabled.

**Avatar Fallback System**: when an avatar is blocked/missing, a simple fallback avatar displays (default grey robot). Custom fallback: must be Good/Excellent on Android, uploaded first from an Android-configured project with Fallback flag, then re-uploaded to same blueprint ID from Windows/iOS. "Show Avatar" on perf-blocked user forces the real avatar. Missing *worlds* can't be joined.

## 2. Terms of Service / Creator Rules

Documents: TOS (Feb 9, 2026), Community Guidelines (Sept 2025), Creator Guidelines (Apr 2025), SDK License, Copyright Policy, Creator Economy Rules (https://hello.vrchat.com/legal).

### 2.1 Allowed / not allowed

- **Hateful/harassing content: never allowed.**
- **Crashers** (full-screen shaders, full-world audio, oversized models): forbidden.
- **Public content** must avoid controversial/sensitive/intimate/extreme material — "like a big event, a parade, a shopping center, or a family-friendly beach." Such content may exist on **private** avatars/instances with explicit consent — but **minors can never consent**.
- **World moderation systems** (Udon) allowed but regulated: rules must be posted, actions explained; no forced avatar changes, crashing, trapping, or pre-emptive block lists.
- **Misleading thumbnails** violate guidelines.

### 2.2 Copyright, cloning, ripping

- Own or be licensed for every element uploaded; DMCA process exists.
- **Public avatar cloning** is a platform feature; disable via "Allow Avatar Cloning" or set avatar Private.
- **Ripping** (extracting others' assets) explicitly banned by SDK License & TOS.
- **Marketplace avatars** cannot be cloned/made public unless purchased by the cloner.
- Uploading your own purchased model is fine, but base-model licenses (Booth/Gumroad) dictate public/private/redistribution terms — your legal obligation.

### 2.3 Moderation

User reports (User rank+), automated filtering, Trust & Safety team. Consequences: content removal → Nuisance flag → suspensions → permanent bans. Users may block specific avatars locally.

## 3. Upload Logistics

1. Install **VCC** → correct Unity (**2022.3.22f1** — do not upgrade) → Avatar or World project.
2. `VRChat SDK > Show Control Panel > Authentication` → sign in with a **VRChat account** (Steam/Meta accounts can't upload).
3. Build & Publish; avatars need New User rank + latest SDK (older SDKs blocked since Oct 2025).
4. **Pipeline Manager** blueprint ID (`avtr_*` / `wrld_*`) identifies content; re-upload updates in place — use Git for version history.
5. Worlds begin private → submit to **Community Labs** (User rank; 1 new world/7 days) → good worlds graduate to Public automatically.

### Cross-platform builds

Targets: **Windows**, **Android** (Quest 2+, PICO 4, phones), **iOS** (iOS 17+, 6 GB+ RAM — iPhone 14+). Switch build target, upload to the **same blueprint ID** per platform. **Per-platform overrides** (SDK 3.8.1+) allow different avatar variants in one publish. iOS falls back to the Android build with shader replacement. **Armature rules**: PC and Quest/iOS versions must share identical essential bone paths and root-bone scale/rotation.

## 4. Economy

### 4.1 VRChat Plus (VRC+)

$9.99/month or $99.99/year. Perks: more groups, one-time trust boost, stickers/emoji, drones, Age Verification eligibility. Gift drops into instances (rank-gated User+).

### 4.2 Creator Economy

Application-gated; approved creators **sell in VRChat** for **VRChat Credits** (payouts via Tilia/PayPal):

- **Listings in Stores** (group + in-world stores)
- **Avatar Marketplace** (2025): moderation-reviewed avatar sales; NSFW not allowed; purchased avatars can't be cloned/made public
- **Group Subscriptions** (100–10,000 credits/month, ~15.3% fee): subscription group role with permissions
- **Udon products**: unlock world features
- Listing gifting shipped late 2025; world analytics on 2026 roadmap

### 4.3 Groups, communities, events

Groups underpin group instances and events. Seasonal events (Pride, Holiday, Spookality), **Hype Trains**, **Jams**, Inventory/Items/Content Drops. Streaming VRChat is permitted with platform monetization.

## 5. Tools Ecosystem

### 5.1 Official

- **VCC + VPM** (community packages: https://vpm-catalog.vercel.app/packages); third-party CLI `vrc-get`/ALCOM.
- **Creator Docs**: https://creators.vrchat.com
- **Udon / UdonSharp** (C# → Udon).

### 5.2 SDK milestones (3.8–3.9)

- **3.8.0** (Mar 2025): Drone Udon interface, PhysBone improvements.
- **3.8.1** (May 2025): **Toon Standard** mobile shader, iOS Build & Test, per-platform avatar overrides.
- **3.8.2** (Jun 2025): spawn radius, Item layer sync, nested-armature validation.
- **3.9.0** (Oct 2025): Camera Dolly, Pickup AutoHold, Network ID Utility; older SDKs blocked for new uploads.

### 5.3 Community frameworks (VCC packages)

**Avatar side:**
- **VRCFury** (vrcfury.com): non-destructive toggle/gesture/prop system; builds animators/menus at upload; auto-fixes WD problems, Quest compat.
- **Modular Avatar** (nadena.dev): assemble avatars from prefab modules non-destructively; on **NDMF** pipeline.
- **AAO: Avatar Optimizer** (anatawa12): NDMF optimization (Trace & Optimize, mesh merging, blendshape removal, atlasing); **d4rkAvatarOptimizer** = popular alternative. Companions: TexTransTool, lilAvatarUtils.
- **GoGoLoco**: de-facto locomotion system (sit/crawl/fly poses).
- **VRLabs** (github.com/VRLabs): Avatars 3.0 Manager, Marker, Followers, Framebreaker, etc.
- **Shaders**: **lilToon** (JP dominant), **Poiyomi Toon** (EN dominant), official Standard Lite / Toon Standard (mobile).

**World side:**
- **ProTV** (feature-rich media player; supersedes VideoTXL), **USharpVideo** (simple), KineL Video Player.
- Varneon VUdon, CyanTrigger, QvPen, ClientSim (world testing without VR), Gesture Manager (avatar testing), EasyQuestSwitch, VRCQuestTools.

### 5.4 AI-assisted creation

No official VRChat AI-generation features as of early 2026. AI textures/voice feed standard Unity workflows; AI content bound by same TOS (you must own everything you upload).

## 6. Community Norms

**Avatar licensing culture:** model creator's license > VRChat TOS > platform mechanics. JP bases (Booth): purchase = personal use; public versions (公開アバター) have explicit cloning/mod rules; commissions = **private** avatars. EN (Gumroad/Jinxxy/Marketplace): "public"/"private"/"clone-allowed" variants. Making a paid base public without permission is a serious community violation.

**Commissioning:** custom avatar commissions (model + outfit + toggles, often VRCFury/MA) form a large gig economy; "Quest conversions" are a common specialty.

**Base models:** rigged humanoid with standard 3.0 setup. JP: various cat/anime bases; EN: **Rexouium** (furry staple), Flexion, free bases (Lyuma's, etc.).

**Etiquette:** avoid Very Poor in crowded publics, optimize for Quest, label questionable content, respect world rules boards.

## 7. 2025–2026 Platform State

**Client**: 2025.1.x (Main Menu 2.5, OSC avatar switching), 2025.2.3 (Inventory/Items), 2025.4.x (world networking checks, Steam Audio experimentation).

**SDK/Unity**: SDK 3.9.x; Unity **2022.3.22f1** (upgrading breaks uploads).

**iOS**: full launch (App Store), uploads since SDK 3.8.1.

**Roadmap** (Mar 2026):
- **Soba** — successor to Udon (C#-feature-parity; closed beta upcoming)
- **Official Avatar Optimizer** — automatic mesh/material merging & atlasing at build (competes with AAO/d4rk)
- **Steam Audio** open beta
- Box-shaped Contacts, avatar **Raycast** component, **global PhysBone colliders**, **VRCTween**, world analytics

**Big picture 2026**: Content Gating + Age Verification rework (H1 2026), Creator Economy maturation, official avatar optimizer, Soba groundwork, iOS closing the platform triangle → cross-platform-first workflows.

### Key sources

- https://docs.vrchat.com/docs/vrchat-safety-and-trust-system · https://wiki.vrchat.com/wiki/Trust_Rank · https://wiki.vrchat.com/wiki/Instances
- https://hello.vrchat.com/creator-guidelines · https://hello.vrchat.com/legal · https://hello.vrchat.com/legal/sdk · https://hello.vrchat.com/copyright
- https://hello.vrchat.com/blog/age-verification · https://hello.vrchat.com/blog/trust-safety-update-2025
- https://creators.vrchat.com/sdk/ · https://creators.vrchat.com/platforms/ · https://creators.vrchat.com/releases/
- https://docs.vrchat.com/docs/avatar-fallback-system · https://docs.vrchat.com/docs/public-avatar-cloning
- https://creators.vrchat.com/economy/ · https://hello.vrchat.com/legal/economy · https://wiki.vrchat.com/wiki/VRC+/en
- https://vcc.docs.vrchat.com/ · https://vrcfury.com/ · https://vpm-catalog.vercel.app/packages
- https://ask.vrchat.com/t/vrchat-creators-roadmap-march-2026/48077
- https://help.vrchat.com/hc/en-us/articles/45880051153939-iOS-Platform-FAQ
