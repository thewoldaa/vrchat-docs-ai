# VRChat World Design Patterns, 2025–2026: A Field Report

*Categorical design logic corroborated by fetched primary sources (creators.vrchat.com, wiki.vrchat.com, hello.vrchat.com, vrcreators.net); details about specific famous worlds partly from community knowledge, flagged where so.*

---

## 1. Anatomy of Successful World Categories

Worlds are categorized by what the player's *default loop* is:

**Hangout worlds** (The Great Pug, The Black Cat, Japanese lounges/はなれ retreats) optimize for **low-effort conversation that survives any group size**:
- A small, acoustically warm spawn area (bar, café, living room) where being idle doesn't look idle.
- Ambient audio (rain, record crackle, fireplace) loud enough to mask silence, quiet enough to never compete with voice.
- **Multiple small conversation "nooks"** within voice range of each other — fails if 20 people share one audio bubble; succeeds when four 5-person bubbles coexist.
- Conversation-generating props: jukeboxes, video screens, pool tables, fireplace seats.
- Essentially zero "gameplay" — the job is making *people* the content.

Japanese lounges add: extremely high environmental fidelity, team-built, strong baked lighting; often paired with paid cosmetics under the Creator Economy.

**Game worlds** (trivia, murder, golf, Among Us-likes) optimize for **round-based replayability and in-group coordination**: spawn isn't the social heart — the *game lobby* is. Everything architected around a game-state machine (§4). Capacity deliberately low (4–16): per-player interaction over crowd density. Genre specifics:
- **Trivia:** multi-team simultaneous play, answer anonymity until reveal, per-round score persistence.
- **Murder games:** *information asymmetry* — per-player role assignment is the hardest networking problem; worlds that can't hide roles per-client get abandoned.
- **Golf:** *parallel* play — several players each on their own ball, nobody waits. Waiting is the #1 kill factor for sports worlds.
- **Among Us-likes:** clean lobby → role assignment → play → discussion → results cycle, with voice behavior changing per phase.

**Showcase/dance/club worlds** optimize for **peak-moment spectacle under extreme load**: one stage, one massive screen, teleport/move the crowd rather than let them scatter, Udon-driven lighting sequences synced to a start timestamp. Capacity can be manually raised by VRChat staff for events (normal cap 80).

**Art galleries** optimize for **paced attention**: corridor or cloister loop, breathing room per piece, local light/frame per work, trivially legible navigation.

**Horror worlds** optimize for **tension curves**, tightly constrained by guidelines: "extreme horror or 'shock' content" may not go to Community Labs/Public. Successful public horror uses **jump-scare economy** instead of gore — dark corridors, Udon-triggered audio, scares that fire only for the local player.

## 2. Common Architectural Patterns

**Spawn/lobby design.** "Profile the spawn area first because that is where first impressions and Community Labs checks often begin" (vrcreators.net world-optimization checklist):
- Spawn faces the *most photogenic and most explanatory* view.
- Multiple spawn points with nonzero **Spawn Radius** so rapid-join players don't stack; Spawn Order can randomize or sequence.
- A readable "you are here" cue: sign, world name visible in-world, rules near the entrance (legally required if you run Udon moderation — §3).
- Game worlds spawn you *outside* the game area, in a lobby — late joiners don't interrupt running rounds.

**Instance capacity:** two-tier — **Recommended Capacity** (reached → stops being listed publicly) and **Maximum Capacity** (hard join cap, max 80; effective max 82 with reserved slots). Adjustable on the website without reupload; long-running instances auto-reset with 5-minute (and optional 60/180-minute) warnings. Design implications: hangouts set Recommended ≈ Maximum; games set Recommended well below Max so full games show "full."

**Mirror placement:** official Labs guidance is emphatic — ">1 mirror in your room… bad things happen. If you have 1 mirror, make sure to set it to toggle. We STRONGLY SUGGEST not enabling mirrors by default." Community pattern: one big mirror at the selfie spot, default-off, toggled by switch or proximity; small Udon-clipped "cheap mirror" planes for outfits. Mirrors multiply avatar cost exactly where crowds gather.

**Video players:** ">2 video players in a room — bad things." One primary big screen facing social seating + visualizer elsewhere; never two audible players in one voice bubble. URL allowlists are a moderation surface (authors responsible for linked content).

**Seating/stations:** hangouts seat densely at conversation angles (facing, ~1–2 m) so seated players don't drift; sit-to-stand as instant as possible. Games use stations as *role assignments* — a seat is how you join the lobby. Dance worlds avoid floor stations; station the audience on risers.

**Navigation & wayfinding:** "Many VRChat worlds fail because players do not know where to go… If a simple social world needs constant explanation, the spatial design is still doing too little." Working patterns: one strong landmark per zone; sightlines connecting zones (see your destination before walking there); lighting-temperature changes signaling transitions (warm = social, cool = transit); portals framed in one "portal row" rather than scattered. Large worlds converge on a central hub with radial wings rather than a linear maze.

**Lighting mood:** bake everything ("Baking your lighting is exceedingly important"). Design rule: light where voices come from — readability around faces. Real-time lights rationed to dynamic accents; everything structural baked. Post-processing treated as a hazard (no CA/SSR/SSAO for VR users).

## 3. Udon-Driven Systems in Popular Worlds

The Creator Guidelines define what world-side moderation may do: **rules must be posted at/near the entrance; moderated users must be told what action was applied and why; creators may teleport a user to a separate area, block interaction with world systems or voice, and use an "allow list" to enable features — but may NOT forcibly change avatars, crash or disrupt the client, prevent leaving, or "preemptively moderate users by using a static or dynamic 'block list'"** ([creator-guidelines](https://hello.vrchat.com/creator-guidelines)).

This boundary shapes the prefab ecosystem:
- **Reactive moderation:** **VRC RMS** (github.com/UzerTekton/vrc-rms) — named moderators apply per-instance jail/mute/immobilize actions, rules panel auto-filled near the entrance, notice follows the moderated player. Ban lists framed as "agreed rules" checked by a human; bans expire when the user leaves. Community threads confirm the line: automated "ban on sight" is disallowed; moderator-triggered actions are not.
- **Per-instance whitelisting:** Lokness9's **Per-Instance Whitelister** (jinxxy.com/Lokness9/PerInstanceWhitelister) — dynamically grant a guest special access per-instance (VIP room, unlocked items), optional rejoin-sync. Whitelist = "allow list to enable features" — explicitly TOS-legal where a blacklist is not.
- **Vote-kick and join messages:** VRChat has native Vote-to-Kick; worlds layer Udon join announcements, first-visit tips, supporter name displays.
- **Web-backed UI:** SistersInGaming's **Web UI pack** (jinxxy.com/SistersInGaming/WebUI) — worlds fetch remote text/images so creators update supporter lists, changelogs, rules, even portal destinations without re-uploading. Ships a Banned-User-List prefab (predates blocklist enforcement; now gray-zone — reactive moderation is the compliant modern shape).
- **Avatar pedestals with whitelists:** pedestal rows for staff/performers gated by allowlist + open pedestals for anyone.
- **HUDs:** local-only world info, voice-range indicators, scoreboards, golf/trivia overlays. Local HUDs are free; synced ones respect the ownership model.

## 4. Game Frameworks and State Patterns

Core rules: only an object's **owner** can modify synced variables; **manual sync** requires `RequestSerialization()` (scores, phase); **network events** are one-shot and *never replayed for late joiners*, while **synced variables** deliver latest values to late joiners. Hence the universal pattern: **sync the decision, animate it locally.**

**Round-based state machine:** the **Mugi** framework (jam.vrg.party/mugi/) makes the standard machine explicit: Lobby → Countdown (3 s) → Running (clock, live scoreboard) → Ended (3 s cooldown) → Lobby, with too-few-players bouncing back. Its **Game Master pattern**: the first player to join owns the game object; only the Game Master starts/stops games and processes scores; on leave, ownership auto-promotes the lowest-ID active player, else any remaining player — solves the "host left and the game soft-locked" failure.

**Per-instance state sync strategies:**
- **One authoritative controller.** Beer-pong tutorial's rule: "The ball can change owners as players pick it up, but the score needs one authority. Send scoring attempts to the game controller owner; only that owner should accept a cup, advance the turn, and serialize the new state." Store enough synced state to *rebuild* the table for a late joiner (phase, turn, cup state, score, a round/shot ID to reject stale reports) — never replay events.
- **Route-to-owner vs transfer-ownership:** transferring per interaction is simpler; routing a bounded request to `NetworkEventTarget.Owner` avoids ownership churn. Serialize result + phase + revision; derive animation/sound locally.
- **Trivia pattern:** lobby join-pads → question phase (questions from text/VRCUrl), per-player answers hidden until reveal → owner validates → increment score → leaderboard. Hidden answers = per-player local state serialized only at reveal.
- **Murder pattern:** owner deals roles; each client learns only its own role locally (owner-targeted events or filtered reveal). Voice discipline per phase — proximity while hiding, all-talk during meetings.
- **Golf pattern:** *parallel, not turn-based* — each player owns their ball pickup, stroke counting local, hole triggers report to the scoreboard owner, per-hole bests synced manually.

## 5. World Monetization Setups

- **Paid items / world-linked cosmetics** for Gems; supporting a world grants perks the world can detect.
- **In-world supporter stores:** the Web UI pack's "Supporter List Reader" pattern (remote list of supporters with numeric ranks, color-coded names, "supporters here now" display) — post-economy, fed by actual purchase data.
- **Monetization-friendly design:** visible supporter board near spawn; purchasable cosmetic *personal effects* rather than functional advantages (functional pay-to-win guts trust in public game worlds); supporter-only rooms via the per-instance whitelist pattern (TOS-clean).

## 6. What Makes Worlds Graduate Labs Fast

Per the Community Labs submission page ([docs](https://creators.vrchat.com/worlds/submitting-a-world-to-be-made-public/)):
- **FPS at spawn:** "Aim for at least 45 FPS with a single VR user at the spawn." Reviewers test at spawn first.
- **The first 30 seconds:** spawn must be readable, performant, and show what the world is for. The spawn view is the de facto review shot.
- **Thumbnail craft:** content must "accurately represent what a user will experience… not be misleading." Fast graduates use the money shot — the bar/stage/vista — with characters in it, lit as in-world, not a Unity screenshot of an empty room.
- **Android/Quest parity:** roughly doubles the audience; checked in Labs.
- **Technical hygiene:** single-pass stereo shaders, supersampled UI, ≤1 toggleable mirror, ≤2 video players, minimal real-time lights.
- **Content gating done right:** honest content warnings and tags.

## 7. Anti-Patterns: Why Worlds Fail

- **Empty or dead spawn** — land facing a blank wall; loses players in seconds. The most common Labs rejection driver.
- **Confusing layouts** — mazes with no landmarks, scattered portals, areas invisible from spawn.
- **Over-complex UI** — walls of buttons with tiny text, 12 nested panels. Winners expose three verbs max at spawn (sit, toggle mirror, start game) and put depth behind physical objects.
- **No social loops** — nothing to do while waiting for a game, no idle-friendly props. The best game worlds double as hangouts (the golf course has a café).
- **Performance self-sabotage** — default-on mirrors, 4+ video players, unbaked lighting, always-on particles. "Fine alone but bad with people" fails its real test.
- **TOS-violating moderation** — automated blocklist jailing gets worlds pulled (ask.vrchat.com/t/bos-litsts-and-more/37923).
- **Soft-locked games** — Game Masters leaving without ownership handoff, states that don't rebuild for late joiners, turn systems two clients can advance independently.
- **Stale metadata** — misleading thumbnails, tag spam.

## 8. Concrete Creator-Posted Metrics

- **Capacity:** Recommended ≤ Maximum ≤ 80 (effective 82 with reserved slots); website-adjustable without reupload. Practical: hangouts 20–40 recommended / 40+ max; games 8–16 / 16–24; clubs rely on group priority queues.
- **FPS targets:** ≥45 FPS with one VR user at spawn; profile spawn, main social area, and worst-case angle separately.
- **Spawn settings:** multiple spawn points + Spawn Radius > 0; Respawn Height above all walkable geometry. Creator-posted strategy: respawn point *inside* the main area for hangouts (falling returns you to the fun), *outside* the play space (or per-game-zone respawn volumes) for games so death isn't a teleport to the lobby.
- **Heavy-feature budgets:** ≤1 mirror (default-off + toggle), ≤2 video players, baked lighting default, real-time lights rationed.
- **Instance lifecycle:** auto-reset within 7 days max, 5-minute (always) + optional 60/180-minute warnings — event worlds schedule around this.

## Sources

- https://creators.vrchat.com/worlds/submitting-a-world-to-be-made-public/ · /creating-your-first-world/ · /worlds/udon/networking/
- https://wiki.vrchat.com/wiki/Instances/en · https://hello.vrchat.com/creator-guidelines
- https://ask.vrchat.com/t/moderation-in-worlds/46913 · https://ask.vrchat.com/t/bos-litsts-and-more/37923
- https://vrcreators.net/docs/vrchat/world-building · /world-optimization-checklist · /beer-pong-for-cyantrigger · /spin-the-bottle-vrchat-udon-cyantrigger-tutorial
- https://jam.vrg.party/mugi/ (Mugi framework) · https://github.com/UzerTekton/vrc-rms
- https://jinxxy.com/Lokness9/PerInstanceWhitelister · https://jinxxy.com/SistersInGaming/WebUI

*Honesty note: interior details for The Great Pug, The Black Cat, and named Japanese lounges come from general community knowledge (search was rate-limited mid-research); the categorical design logic is corroborated by fetched guidance, but treat specific claims about individual worlds as unverified.*
