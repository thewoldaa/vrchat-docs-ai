# VRChat Platform: Avatar & Client Features Report (2025–2026)

*Based on official docs + wiki. A few topics (OSC FT v2 official docs, some iOS minutiae) lean on secondary/community documentation.*

---

## 1. Avatar Impostors

**What they are:** "avatar body doubles" — simplified billboard-style renditions generated automatically from the original mesh, bridging platform and performance gaps: instead of a gray robot or fallback, a recognizable, performance-friendly approximation ([creators.vrchat.com/avatars/avatar-impostors](https://creators.vrchat.com/avatars/avatar-impostors/)). Introduced 2023.4.2 (Nov 2023).

**Generation:** server-side, automatic on upload/update; creators can regenerate from the avatar's vrchat.com page ("Generate Impostors", ~10/day cap, owner only). The system renders body parts into sprites packed onto a shared texture atlas — why per-part customization matters. Impostor versions iterate (Impostors 1.2.0 with 2025.1.1p1, Feb 2025); VRChat may auto-regenerate to newer versions. **Only humanoid avatars supported**; generic unsupported.

**Customization — the `VRCImpostorSettings` component** (placeable multiple times on different bones):
- **Resolution Scale** — how much of the texture atlas this part gets (head gets more).
- **Transforms To Ignore** — hide specific transforms from capture.
- **Extra Child Transforms** — force separate sprites for wings/tails (atlas space cost).
- **Re-parent Here** — fold another bone's geometry into this part's sprite (wing bones → chest).

**When shown:** (1) avatar preview in menu; (2) **performance blocking** (Very Poor blocked by viewer's Minimum Displayed Rank); (3) **platform mismatch** (PC avatar seen on Android/iOS). The wearer can toggle "Use Impostor as Fallback". Nameplates show an impostor badge.

**2025 relevance:** **fallbacks are being deprecated in favor of impostors** — 2025.2.2 release notes (June 2025): "Fallbacks are being deprecated slowly over time in favor of impostors." Impostors are now the primary way strangers see each other across platform/performance boundaries.

## 2. Fallback Avatars

Simple performant avatars shown when the real one can't display. Deprecated-but-functional ([docs.vrchat.com/docs/avatar-fallback-system](https://docs.vrchat.com/docs/avatar-fallback-system)).

**When shown:** performance blocked · file-size blocked · missing platform asset · load error · content-gating filtered.

**Custom fallback requirements:**
1. Built for **all available platforms** (Windows + Android minimum);
2. Rank **Good or better** on the respective platform — judged at upload time on the Android target;
3. **"Is a Fallback" flag** set from the Android upload.

**Upload flow:** build/upload on Android target, check the fallback-candidate button at the naming/thumbnail screen, switch to Windows, upload again to the **same blueprint ID**. Uploading a non-compliant avatar on either platform clears the flag.

The separate **Shader Blocking & Fallback System** ([docs](https://creators.vrchat.com/avatars/shader-fallback-system/)) is different: when a viewer blocks custom shaders, materials fall back to mobile-safe shaders via `Tags{"VRCFallback"="Toon"}`-style tags.

**Restrictions:** fallbacks can't be made public; pre-Sept-2021 fallbacks are grandfathered (may show a perf-blocked robot if not re-uploaded); a humanoid fallback worn over a Generic avatar loses IK; VRChat scales fallbacks to roughly match the wearer. "View Fallback Details" UI + blue-feather nameplate badge indicate a fallback.

## 3. IK 2.0

Shipped 2022.2.1, replacing the legacy solver ([announcement](https://hello.vrchat.com/blog/vrchat-ik-2), [features doc](https://docs.vrchat.com/docs/ik-20-features-and-options)).

**What changed vs IK 1.0:**
- Complete solver rebuild — more accurate tracking in twisted/upside-down poses, less "twisty spine."
- **Elbow anti-clip** — automatic prevention without elbow trackers.
- **Avatar measurement choice:** match by arm span (`--custom-arm-ratio`) or by height (better FBT fit).
- **Calibration saving** — calibrate once, switch avatars endlessly (trackers proportionally remapped).
- **Per-tracker IK combinations** — asymmetric setups (one left elbow + one right foot), graceful degradation on disconnect, automatic restoration.
- FBT users start in 3-point IK, not T-pose.

**FBT improvements:** up to **11 tracked points** (chest, knees, upper arms added). Chest tracking twists the chest; upper-arm trackers drive shoulder+elbow. Quick Menu: **Lock-hip** (prioritize hip tracking), **Lock-head** (prioritize viewpoint), **Lock-all**. `TrackingType` animator parameter updates live per-tracker (3/4/6 while valid; 0–2 are transient init states animators must not dead-end on).

**Avatar compatibility requirements** ([rig-requirements](https://creators.vrchat.com/avatars/rig-requirements)): humanoid rig; head/hands/feet mapped; pelvis/spine/chest/neck/shoulders required, shoulders+neck direct children of chest; LowerArm first child of UpperArm (legs analogous); hips ancestor of all humanoid bones; eye bones Y-up/Z-forward. For FBT: pelvis–thigh bone angle near 180° in T-pose. Poorly proportioned rigs remain the main FBT complaint source.

**Status:** default since 2022.2.1; legacy IK toggle on removal path. 2025 tracking news was **Selfie Expression** (webcam face/head/eye/hand tracking for desktop, 2025.1.3, VRC+ Early Access → free on PC by Sept 2025).

## 4. Eye & Face Tracking on the Avatar Side

**No native hardware integration** — everything flows through OSC or legacy headset integrations.

- **SRanipal (legacy):** HTC's runtime for Vive Pro Eye/Facial Tracker; VRChat's native SRanipal integration is deprecated in favor of OSC pipelines — VRCFaceTracking now carries the load, with SRanipal as a VRCFT module.
- **OSC eye tracking (native):** VRChat natively accepts eye data via `/tracking/eye/*` addresses (`EyesClosedAmount`, `CenterPitchYaw[Dist]`, `CenterVec[Full]`, `LeftRightPitchYaw|Vec`), 10-second timeout back to auto-look/auto-blink. Any Av3 avatar with eye bones supports it.
- **Face tracking "v2" (Unified Expressions / VRCFT):** the modern standard — `v2/`-prefixed float OSC avatar parameters (`v2/JawOpen`, `v2/EyeLid`, `v2/BrowInnerUpLeft`, `v2/TongueX`), ~100+ parameters mapping ARKit-like and extended blendshapes ([docs.vrcft.io](https://docs.vrcft.io/docs/tutorial-avatars/tutorial-avatars-extras/parameters)). Avatar side: add as Expression Parameters / FX animator inputs; v1 parameters remain emulated. VRCFT bridges essentially all hardware (Quest Pro, Vive Pro Eye, Varjo, Pimax Crystal, PSVR2, webcams, iPhone, Project Babble, EyeTrackVR).
- **Selfie Expression** (2025): drives built-in visemes and eye look via webcam — "works for most avatars with no additional effort."

## 5. Avatar Scaling

References: [avatar-scaling](https://creators.vrchat.com/avatars/avatar-scaling/), [player-avatar-scaling](https://creators.vrchat.com/worlds/udon/players/player-avatar-scaling/).

- **Mechanics:** scaling changes the avatar root's `localScale`. Terms: **Prefab Height** (eye height at default scale, from SDK View Position), **Target Eye Height**, **Avatar Scale** = target/prefab ratio. Scale is **enforced** — animations cannot touch root scale (this killed GoGoLoco-style scale prefabs in 2023.2.4, intentional). Scale changes trigger a "remeasure" (viewpoint, voice position). Scale syncs as quantized eye height (3 decimals), smoothed on remote clients. The Action Menu dial previews in mirrors only; confirming applies network-wide.
- **Ranges:** player dial **0.2–5.0 m** eye height. Udon: clamped 0.1–100 m (harder limits 0.01–10,000 m with HUD warning). Prefab height unlimited — a giant avatar circumvents the dial, so worlds that care must re-apply limits on `OnAvatarEyeHeightChanged`. "Match Eye Height" matches another user. Worlds can disable scaling via website toggle or **world-authoritative mode** (`SetManualAvatarScalingAllowed(false)` + Udon-set min/max).
- **Parameters:** `ScaleModified`, `ScaleFactor`, `ScaleFactorInverse`, `EyeHeightAsMeters`, `EyeHeightAsPercent`. OSC: `/avatar/eyeheight` (read/write), `/eyeheightmin|max|scalingallowed`.
- **Physics implications:** scaling **does not change environment collision** — a 5× avatar collides at the same capsule size; worlds must account for it explicitly.

## 6. Client Features 2025–2026 Relevant to Creators

**Inventory, Content Drops, Items (2025.2.3, June 2025):**
- **Inventory**: Main Menu system (chest icon) managing Emoji, Stickers, Items with Action Menu bindings; emoji/sticker slots 9→18; stickers free (custom still VRC+).
- **Content Drops**: claimable seasonal bundles ("not loot boxes").
- **Items**: a new spawnable object class (Pride Ribbon, REFERENCE CUBE, Groan Tube, Companions). Items run their own Udon + VRC components — but **world Udon cannot interact with Items yet**. Items live on the Item layer (formerly reserved3), collide only with other items. Moderation: world can ban spawning entirely; users hide individual items; blocked users' items invisible; group moderators can disable spawning in Group instances. Creator-made Items were "probably not until 2026"; Dec 2025 added "Item Dynamic components" hinting at expansion.

**Main Menu 2/2.5** (2025.1.2, Feb 2025): major rework — new header bar, labeled bottom tabs, streamer-mode credit censoring, reworked tooltips; 2025.1.3 added instance-creation revamp + redesigned Safety settings (merged into regular settings).

**OSC changes:** 2025.3.2p1 added **OSC Camera endpoints** (programmatic camera control); OSCQuery discovery on port 54802.

**Safety/shield:** Shield levels (Maximum/Normal/None/Custom) now cover voice, avatar, photo icons, **custom images**, **drone visibility**, audio, lights & particles, shaders, custom animations, animated emoji — per trust rank. **Content Gating** as a new layer: creators label content warnings, users filter; mislabeling is reportable. Late 2025: repeated Trust & Safety updates, new reporting flow (Nov 2025), Tech Coalition membership (Oct 2025).

**Other:** Avatar Marketplace (open beta May 2025, live June 2025), Camera Drone & Dolly, Steam Audio (open beta 2025.1.2 → live 2025.4.2), instance naming, events calendar + event-linked group instances, Discord login/linking, Very Poor avatars on Mobile under settings (2025.1.1p1), SDK 3.9.0 blocking uploads from older SDKs, SDK2 fully deprecated (Aug 2025).

## 7. Group Features

**Group instances** — three types: **Group** (members only, optionally role-restricted), **Group+** (friends of members), **Group Public** (publicly listed). Role-gating admits only selected roles; members holding a role above the lowest selected moderation-capable role always get in. **Group Queues** handle full instances via waitlist with 60-second join windows and queue priority as a role permission. 2025 additions: **age-gated instances** (18+ Age Verification, from "Create Age Gated Instances" permission), **linked instances** (tied to live/upcoming events), **instance naming** (Sept 2025), **Group+ portal permissions**. Groups max 100,000 members, 50 roles.

**Group moderation:** the **Moderate Group Instance** permission enables warn/mute/kick/ban inside group instances. **Group bans** (2023.2.3) work even on non-members, blocking all group instances until unbanned (an invite back unbans). Other permissions: Manage Group Bans, Remove Group Members, Manage Group Instances, View Audit Log, and notably **Bypass Avatar Performance Requirements** — designated members ignore the group instance's minimum avatar performance setting.

**Group announcements:** **group announcements** ("Manage Group Announcement") — group-wide notification; **instance announcements** ("Create Instance Announcement") — pushed to everyone in a group instance via HUD pop-up + notification inbox. Sent from Quick Menu "Here" tab or Instance Details; 10-message history; recipient-side "Mute Announcements" resets on world change.

## 8. Cross-Platform Display (PC ↔ Quest ↔ iOS)

- **Platform mismatch:** avatars are built per-platform; if the wearer has no build for your platform, you see their **impostor** (or fallback if impostor disabled) — never the gray robot if an impostor exists. Cross-platform uploads must share a blueprint ID, and **armatures must be essentially identical** (essential bone paths, root bone scale/rotation).
- **Tags:** the nameplate badge carries an inset icon: Performance Blocked, File Size Blocked, Impostor, Filtered (content gating), Missing Asset, Error. "Show Avatar" overrides performance blocking but can't conjure a missing platform asset.
- **iOS:** shares the Android (mobile) content pipeline — same mobile asset requirements, impostors, fallbacks. Milestone: **October 23, 2025 — official iOS + Android phone release** (2025.3.4). Mobile ranks stricter; Feb 2025 allowed Very Poor on mobile under settings. PC-only avatars appear as impostors by default. iOS behaves as "Android-class" for avatar display purposes.

## Sources

- https://creators.vrchat.com/avatars/avatar-impostors/ · https://docs.vrchat.com/docs/vrchat-202342 · https://wiki.vrchat.com/wiki/Fallbacks/en
- https://docs.vrchat.com/docs/avatar-fallback-system · https://creators.vrchat.com/avatars/shader-fallback-system/
- https://hello.vrchat.com/blog/vrchat-ik-2 · https://docs.vrchat.com/docs/ik-20-features-and-options · https://creators.vrchat.com/avatars/rig-requirements
- https://docs.vrchat.com/docs/osc-eye-tracking · https://docs.vrcft.io/docs/tutorial-avatars/tutorial-avatars-extras/parameters
- https://creators.vrchat.com/avatars/avatar-scaling/ · https://docs.vrchat.com/docs/osc-avatar-scaling
- https://ask.vrchat.com/t/developer-update-26-june-2025/44933 · https://wiki.vrchat.com/wiki/2025_in_VRChat_updates · https://docs.vrchat.com/docs/vrchat-202523-openbeta
- https://docs.vrchat.com/docs/vrchat-safety-and-trust-system · https://wiki.vrchat.com/wiki/Settings/en
- https://help.vrchat.com/hc/en-us/articles/11711571101587-Managing-a-Group · https://wiki.vrchat.com/wiki/Groups/en · /wiki/Instances/en
- https://creators.vrchat.com/platforms/android/cross-platform-setup
