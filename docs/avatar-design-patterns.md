# VRChat Avatar Design Patterns & the Base-Avatar Ecosystem, 2025–2026

*Sources: creators.vrchat.com, docs.vrchat.com, wiki.vrchat.com, jinxxy.com, gumroad.com, booth.pm via VRCFinder, Reddit r/VRchat, commission cards (carrd.co), note.com creator guides, panora.tokyo, VRCLibrary*

---

## 1. What Top Avatars Do Differently: Performance, Toggles, Parameters, Saved States

### Performance discipline as a product feature

The single biggest thing separating respected avatar packages from amateur work: performance rank is treated as a **design constraint, not a post-hoc cleanup**. Headline PC numbers: **32k tris Excellent, 70k ceiling above Very Poor, 150 MB texture memory at Poor, 1 skinned mesh Excellent, 4 material slots Excellent, 4 PhysBone components + 16 affected transforms Excellent**. Mobile: 20k tris max before Very Poor, 40 MB texture memory, 2 skinned meshes, PhysBones hard-capped at 8 components.

VRChat counts **everything, including disabled GameObjects and components** — "An object toggle changes runtime visibility; it does not remove meshes, materials, PhysBones, Contacts, or other components from the uploaded avatar's performance totals" (VRCreators). Top creators therefore design toggle-heavy avatars with merged meshes (one skinned mesh with material-swap animations), atlased textures, and shader-toggle tricks rather than relying on the optimizer to hide bloat.

Concrete field examples: **Novabeast** ships "~43k tris with 7 materials, Medium rank" (kittomatic.gumroad.com/l/Novabeast); **Nardoragon** "50,408 (Basic) / 67,512 (Expanded)" tris at Medium; **Hyenid** "72k polys with all accessories, below 70k for the quest version". The pattern: the *Basic* version deliberately stays under 70k PC / 20k Quest; the *Expanded* version consciously spends the remaining budget. Optimization has been **outsourced to tooling**: the standard stack is **Modular Avatar + AAO: Avatar Optimizer + VRCFury + d4rkAvatarOptimizer**, with NDMF as the glue.

### Toggle organization and menu taxonomy

The menu is 8 controls per page — taxonomy is a genuine design discipline. De-facto best practice: a **top-level menu of 6–8 category submenus**:

```
Outfits (Cloth/Wardrobe)
Cosmetics (hair, glasses, accessories — per-item toggles)
Hair / Hair physics toggles
Body (sliders, chest scale, toe grab, breathing)
Effects (dissolves, particles, glow, material swaps)
Dynamics (per-zone PhysBone on/off)
System (GoGo Loco, ASF, mirror/TPS toggles, reset)
```

The [VRChat Wiki Toggles page](https://wiki.vrchat.com/wiki/Toggles) formalizes deeper patterns: **exclusive toggles** (one active outfit) via integer selectors or parameter drivers rather than fighting bools; **radial puppets** for hue-shift/scale; **dissolve toggles** via shader cutoff float (exponential-smoothing dissolves cost roughly half a standard layer toggle).

Parameter naming has converged on **`Category/Feature` slash names** (`Props/Glasses`, `Clothing/Hoodie`, `TailDynamics`), typed consistently, spelled identically across Expression Parameters, FX controller, and menu — `Props/Glasses`, `Props/glasses`, and `Glasses` are three different parameters, and one typo silently breaks the toggle. Serious packages keep PC and Quest parameter lists **in the same order and types**.

### Saved-state philosophy

The governing rule (VRCreators): **Saved is for the wearer's identity choices (outfit, colors, hair); Synced is for what others must see; the two are independent.** Mature philosophy: save what the wearer would hate to re-pick each session; sync booleans affecting *other players' perception* (dissolve state, sitting pose); never waste sync budget on local-only FX. The 256-bit synced budget is why "expanded" avatar tiers often *drop* Saved on cosmetic FX to make room.

## 2. The Base Model Catalog, 2025–2026

Three markets: **furry bases (Gumroad/Jinxxy, $25–45)**, **anime/humanoid JP bases (Booth, ¥5,000–6,000)**, **in-client VRChat Marketplace (2025.2.2+)**.

### Furry bases (Gumroad / Jinxxy)

| Base | Creator | Price | Notes |
|---|---|---|---|
| **Rexouium** | Rezillo Ryker (2021) | $45+ | 4.8★ with **2,000+ Gumroad ratings**; effectively "the default furry avatar" (wiki.vrchat.com/wiki/Community:Rexouium); open species; digitigrade/plantigrade switch; separate commercial license |
| **Rexouium Evolved / EvoRex HD** | Tosca | Gumroad | Split-UV rework for higher texture resolution; free upgrade for prior owners |
| **Rezuium** | Hirozu | Gumroad | Built-in face-tracking variant; credit required |
| **Hyenid** | Alber | $40 | 876 ratings, 5.0★; VRChat *and* Resonite pack; strict commercial ToS |
| **Novabeast** | Kittomatic | $35 | 1,419 ratings, 4.9★; dragon; FT version included; lilToon |
| **Nardoragon** | Nardo (nardoiri) | $30 Basic / $45 Expanded | 1,148 ratings; ARKit + Vive FT blendshapes; **mandatory public clonable demo before purchase** |
| **Wickerbeast** | open species (Daemon_Lady/PineWarden; Jin A) | varies | Species managed via the Wicker Cave; Jin A's Wickerbeast King 900+ ratings |
| **Maxigen (Protogen)** | sablewulf828 | $40 | Jinxxy top-25 |
| **Free bases** | various | free | Ryuki/Amaterasu kemono, Stellara, Mouzer, Aquaticat |

Reddit's community census: "Rex, Nardoragon, Wickerbeast, protogen, novabeast, hyenid/yeen, taidum, avali."

**License model:** purchase-per-user, no redistribution, upload-only-by-buyer; commercial/streaming use gated behind a higher tier or written permission (Hyenid requires explicit permission; Rexouium has a separate commercial SKU). Two enforcement levers: the **public clonable demo avatar** (try before buy) and license-key/Discord-gated downloads.

### Anime/humanoid JP bases (Booth)

Highest-volume segment. From VRCFinder's wishlist-ranked catalogue and the VTuber Kazumi's community poll (panora.tokyo/archives/100253):

- **Kipfel** (もち山金魚 mukumi, Jun 2024, ¥5,500) — #1 in the 2024 poll; SD/chibi; shared **"Mame Friends"** common-body project means assets fit 12+ other avatars (Mamehinata, Nuku, Kuronka, Ortua…).
- **Shinano** (Ponderonium, Sep 2024, ¥6,000) — #2 poll; famous high facial-morph freedom.
- **Manuka** (STUDIO JINGO, Aug 2023, ¥6,000) — the most outfit-supported avatar on Booth: **11,207 compatible Booth items**, ~50,500 wishlist saves; "VRChat avatars are divided into before and after Manuka."
- **Rurune** (IKUSIA, shark-girl, ¥6,000) — #3 poll, 138k-wishlist creator.
- **Milltina** (DOLOS art, Jan 2025, ¥6,000) — 658 expression shapekeys; "new standard for busty avatars."
- **Chocolat** (komado/あまとうさぎ, Dec 2024, ¥6,000) — 2025's breakout; shares a base body with Chiffon, Lime, Plum — the asset-inheritance mechanic that made **Kikyou (桔梗)** Booth's three-year #1.

Market analysis (hibit, note.com): the decisive economic factor is **compatible outfit count compounding over time** — new avatars win not on raw quality but on body-compatibility with an established ecosystem; hence "shared 素体 (base body)" projects.

### The EN marketplace layer

**Jinxxy** — community-led marketplace; its popular page is dominated by $20–45 finished avatars tagged `PC / QUEST / FT / GOGO / SPS` — those tags *are* the taxonomy: Face Tracking (VRCFT), GoGoLoco, SPS (Salty Petting Suite), Limb Scaling. **Gumroad** remains the furry-base home; **VRChat Marketplace** (live June 2025) adds try-before-buy in client plus vendor review; **Booth** remains JP-payments-first.

## 3. The Commissioning Pipeline

Synthesized from live commission cards (Terrorized, USBFIG, Jinxit3D, Kirther, Earldense, Parallel Art, streamskins.net):

1. **Brief** — reference sheet (mandatory; most artists refuse to design from scratch or charge extra), species/base preference, platform targets (PC/Quest), FBT requirement, SFW/NSFW scope, toggles wanted.
2. **Base selection or scratch** — *base edit* (Rexouium/Nardoragon retexture + outfit fit, **$25–300**) vs *custom from scratch* (**$550–1,000+**; Kirther $550 start, Parallel $1,000, Jinxit €1,200, streamskins $890–6,900 tiers). Base edits dominate volume.
3. **Outfit modeling** — priced à la carte (the main cost driver): socks/gloves $30, pants $50, shirts $100, jackets $200; wings $100–300/pair; horns $50–100 (Parallel's card).
4. **Rigging** — humanoid rig + visemes + eye look in every base package; PhysBones billed per section ($30+/section at USBFIG); ARKit FT is a line item (+$200); FBT compat promised by good rigs but explicitly untestable by some artists.
5. **Toggles & menu build** — the Unity-side specialist market: Terrorized's base Unity setup **$25** (14 expressions, visemes, PhysBones, VRAM optimization), dissolve toggles $5–15, contacts $3 each, **volume pricing: 20+ toggles $5, 150+ toggles $30** — toggle count is now a headline metric. Clothing toggles at USBFIG $100+.
6. **Delivery** — Unity package (SDK3), Blender + Substance/PSD sources on premium tiers, upload instructions, written license. Revision windows contractually bounded (Parallel: 5 revisions, then $10/change, 1-month bug window).

**A "full" commission includes:** rigged model, visemes + blinking + eye look, 3–14 gesture expressions, PhysBones in 2+ zones, basic toggles, one outfit, PC build; Quest decimation (+$200), FT (+$200), VRM export, AudioLink/body sliders at top tiers.

## 4. Avatar Showcase Strategy

- **Try-before-buy as marketing** — the single most effective pattern: a public clonable demo avatar. Nardoragon's page literally requires it. Jinxxy sellers ship "SERVER FREEBIE" versions doubling as Discord-growth engines.
- **Thumbnails:** in-client Marketplace enforces 1:1, 64–2048px, ≤10 MB + six 1920×1080 gallery images, style tags, content warnings. Booth/Gumroad winners: full-body neutral-pose renders with strong silhouette + close-up face shots.
- **Instance testing:** upload a **private** test build first; verify in an empty instance and with a friend on a different platform (synced-parameter behavior and FT only reveal in multiplayer); re-test after every patch.
- **Upload naming & iteration:** `Name (vX.Y)` / `Name [PC]/[Quest]`; version bumps in place (Manuka ver1.02, Binah ver1.0.71); Booth pages carry changelogs like software ("VERSION LOG" on Novabeast). Marketplace updates upload against the **same avatar ID**, re-reviewed, "all owners see the new version" — patch notes become user-facing.
- **Discoverability:** VRCFinder ranks by compatible-Booth-item count; wishlists are social proof; 2025.3.3 added marketplace filtering by tags/performance/platform — sellers optimize the *tag set* (`FT`, `GOGO`, `SPS`, `Quest`) as hard as the art.

## 5. FBT-First Design

FBT compatibility is monetizable ("Full Body Tracking Compatible!" on virtually every premium page). VRChat's rig requirements ([full-body-tracking docs](https://docs.vrchat.com/docs/full-body-tracking)) define the checklist:

- **Human-proportional limbs** — "FBT works best when the avatar is proportional to a typical human… disproportionate limbs → 'arm pull' (arms too short), 'stilts' (legs too long)." Users fix broken bases with Immersive Scaler via CATS.
- **Hip placement:** hip bone must sit **above** the line between the upper-leg bones (or the hip "flips" during locomotion); no zero-length bones in Hips→Spine→Chest→Neck→Head; slight forward knee bend / rearward elbow bend in rest pose; zero uniform roll on all IK bones; no "full-body hacks" (flipped hips, extra leg bones, unweighted necks) that IK 2.0 progressively breaks.
- **Calibration UX as a feature:** IK 2.0's saved calibrations, Lock Hip/Head/Both spine modes, arm-span vs height measurement (height preferred for FBT). Good packages calibrate on the first try; bad ones require per-user height fudging — why "FBT compat" is a selling point.
- **The GoGo Loco layer:** desktop/3-point users get GoGoLoco; "FBT + GoGo" coverage is the complete locomotion story.
- **The chibi exception:** SD bases (Kipfel, Mamehinata) deliberately accept FBT awkwardness — deformed proportions fight the IK solver; full-height anime bases (Manuka, Shinano) are FBT darlings. vrchat-fbt.com reviews avatars by FBT behavior.

## 6. Accessibility

Asymmetric: **the client provides filters; the avatar provides toggles.**

- **Colorblind:** client-side filters exist (Protanopia/Deuteranopia/Tritanopia/Grayscale). Avatar-side patterns: **color-variant toggles and hue-shift puppets**; preset high-contrast colorways (Novabeast's two variants). World-side: "strongly opposing colors, avoid same-hue gradients" (VRCLibrary Visual Accessibility).
- **Deaf-friendly:** client added muted/deafened nameplate icons (2026.2.3 beta). Avatar-side convention: **visual status indicators** — "Social Battery Pin — 7 Mood Levels (VRCFury)" products, ASL hand-gesture toggles as a commission line item, text/emoji communication props.
- **Photosensitivity:** client has only Bloom intensity — no strobe limiter (feature request still open). De-facto standard: **particles/FX off by default** + a prominent "disable world post-processing/AudioLink" toggle.
- **Scaling:** size-scaling puppet controls are a priced commission feature ($50+ at Parallel) and a Jinxxy tag ("Limb Scaling").

## 7. Dual-Format Workflows: One Model, VRM *and* VRChat

"One character, every platform" is now explicit in pricing (streamskins Studio: "PC + Quest builds · VRM / Warudo version included").

**Direction 1 — VRM → VRChat:** VRoid models via **VRM Converter for VRChat** (esperecyan) through VCC; MToon10 caveat (missing MToon → black eye areas). Official 6-step flow documented in VIVE's 2025 beginner guide.

**Direction 2 — VRChat → VRM** (harder): manual path — clean Unity project → UniVRM first → import avatar → zero blendshapes → convert Poiyomi/lilToon materials to **MToon** → Manual Bake (Modular Avatar) with outfits toggled on → export VRM 0.x → set up VRM BlendShape Proxy clips (A/I/U/E/O, blink, look) → re-export (note.com/tolpaz). The 2025 shortcut: the **NDMF VRM Exporter** automates this for Modular Avatar + lilToon projects and strongly recommends Avatar Optimizer to prune accumulated blendshapes; can even pull metadata from the VRChat API. Pain points: un-unified meshes float in VNyan, non-MToon shaders stripped, blendshape look differences between viewers.

## 8. Case-Study Teardowns

**Novabeast** — $35, 1,419 ratings, 4.9★. ~43k tris, 7 materials, lilToon, Medium. Gestures + visemes incl. MMD, 2 color variants, FT version in the same package, PhysBones across mane/ears/cheeks/chest fluff/tail/tongue/toes, toggleable mane/fluff/goggles/collar, FBT compatible. Ships .SPP/.PSD/.Blend sources. Why it spread: dragon silhouette that reads in a crowd, near-Quest-ready weight, sources included (fueling the edit community), two-year version log.

**Nardoragon** — $30/$45 tiered. 50.4k/67.5k tris, Medium, Quest in two flavors. Basic = body controls, color shift, female variant, visemes + 10 gestures, PhysBones; Expanded adds antlers, two-hair-style custom hair, glasses, shorts, hoodie. Mandatory public clonable demo; usable in ChilloutVR/NeosVR/VSeeFace. Why it spread: protogen-adjacent aesthetic, aggressive tier pricing, decade-spanning accessory ecosystem.

**Hyenid** — $40, 876 ratings, 5.0★. 72k tris with accessories (sub-70k Quest cut), Blender sculpt + Substance, VRChat *and* Resonite packages, Mochie shader preset, strict commercial ToS. Hyena niche + cross-platform pack + Alber's FBT-community reputation.

**Rexouium** — $45+, 2,000+ ratings. The volume king: open species (community lore and derivative bases — Evolved, EvoRex HD, Rezuium — compound it), canine-enough-to-be-anything face, huge blendshape customization, digitigrade toggle, largest accessory ecosystem in the furry segment. Its commercial-license SKU institutionalized the "base edit commission" economy.

**Manuka** — ¥6,000, ~50,500 wishlists, 11,207 compatible Booth items. The anime-side reference case: value = ecosystem, not the file. 28 expression animations, 4 AFK motions, extreme face-morph freedom (the default for game/anime cosplay), deepest outfit catalog. Post-2023 releases are benchmarked against it.

**Kipfel** — ¥5,500. Won the 2024 community poll as a *chibi* — proof the shared-body strategy (Mame Friends fits 12+ avatars with one asset pool) beats raw scale for SD models.

## Synthesis

Four forces define 2025–2026 avatar design:
1. **Performance rank is a marketing surface** — sellers print "Medium rank, 43k tris" on the product page; tooling made one-skinned-mesh, shader-toggle-heavy avatars the default architecture.
2. **Ecosystems beat models** — compatible-outfit count and shared base-body projects are the actual product, furry (Rexouium) and anime (Manuka, Mame Friends) alike.
3. **The marketplace layer fragmented then re-consolidated** — Booth (JP) + Gumroad (EN furry) + Jinxxy (EN community) + in-client VRChat Marketplace.
4. **Feature tags are the spec language** — `PC/QUEST/FT/GOGO/SPS/Limb Scaling`; FBT compatibility and toggle count are the headline numbers; accessibility (FX-off defaults, color variants, visual status indicators) moved from afterthought to listed feature.

## Sources

- https://creators.vrchat.com/avatars/avatar-performance-ranking-system/ · https://docs.vrchat.com/docs/avatar-optimizing-tips · https://www.vrcreators.net/docs/vrchat/avatar-creation/toggle-object · /saving-toggles
- https://wiki.vrchat.com/wiki/Toggles · https://wiki.vrchat.com/wiki/Community:Rexouium · /wiki/Community:Wickerbeast · /wiki/Community:Furry_culture_in_VRChat
- https://kittomatic.gumroad.com/l/Novabeast · https://nardoiri.gumroad.com/l/Nardoragon · https://alber.gumroad.com/l/Hyenid
- https://vrcfinder.net/en/model/ · https://avatar-catalog.com/ranking · https://panora.tokyo/archives/100253 · https://note.com/hibit_at/n/n2c8ec7908a6a · https://note.com/aji8818/n/n2d5a0ae9e0a9
- https://docs.vrchat.com/docs/full-body-tracking · https://docs.vrchat.com/docs/ik-20-features-and-options · https://vrchat-fbt.com/manuka/
- https://www.reddit.com/r/VRchat/comments/1c9mpfj/ · https://www.reddit.com/r/VRchat/comments/1p0gziu/ · https://jinxxy.com/popular · https://jinxxy.com/leabee/lists/3039605250961966508
- https://wiki.vrchat.com/wiki/Settings/en (colorblind filters) · https://docs.vrchat.com/docs/vrchat-202623-openbeta (deaf icons) · https://vrclibrary.com/wiki/books/accessibility-in-vrchat-worlds/page/visual-accessibility
- https://blog.vive.com/us/how-to-make-a-vrchat-avatar-in-2025-beginners-guide/ · https://note.com/tolpaz/n/nb6ba55b25506 · https://note.com/kuromitsutou7/n/n4b822a142e44
- https://creators.vrchat.com/economy/products/avatar/ · https://docs.vrchat.com/docs/vrchat-202522
