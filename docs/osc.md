# VRChat OSC and External Integration Tools — Comprehensive Report (2025–2026)

---

## 1. OSC Basics

VRChat ships a built-in OSC server and client, exposed as two launch APIs: **Input** (simulating controller input) and **Avatar Parameters** (reading/writing your own avatar's animator parameters). OSC is a lightweight UDP protocol: an address pattern (like `/input/Jump`) + type-tag string + typed values.

Supported type tags: `i` (32-bit int), `f` (32-bit float), `T`/`F` (bool), `s` (ASCII string).

**Ports.** VRChat **listens on UDP 9000** and **sends outbound on UDP 9001**. Tools send *to* 9000 and listen *on* 9001 for feedback (parameter echoes, `/avatar/change`, chatbox events). Both configurable:

```
VRChat.exe --osc=inPort:senderIP:outPort
```

Default ≈ `--osc=9000:127.0.0.1:9001`. To drive VRChat from a LAN device (TouchOSC tablet at 192.168.1.42): `--osc=9000:192.168.1.42:9001`.

**Enabling OSC:** off by default. Enable in-game: Action Menu → **OSC → Enabled**. The **OSC Debug** screen (OSC → OSC Debug) visualizes incoming messages in real time — invaluable for verifying packets arrive. VRCFaceTracking tries to flip the toggle automatically, but enable it manually to avoid a game restart.

Sources: https://creators.vrchat.com/worlds/osc/ · https://wiki.vrchat.com/wiki/OSC

## 2. Input OSC

All input endpoints live under `/input/…` and are **write-only**.

**Axes** take floats **−1.0 to 1.0** — and *persist* until you send `0`: send `/input/Horizontal 1` and never zero it → walks right forever.

**Buttons** take `1`/`0` (or `T`/`F`) — send the release before the next press counts: `/input/Jump 1` twice with no release = one jump.

### Axes (`,f`, −1.0 … 1.0)

| Address | Behavior |
|---|---|
| `/input/Vertical` | +1 forward, −1 backward |
| `/input/Horizontal` | +1 right, −1 left |
| `/input/LookHorizontal` | Look L/R. Smooth on Desktop; in VR snaps a full turn at value 1 (if Comfort Turning enabled) |
| `/input/LookVertical` | Look up/down (via OSCQuery in client; community-wiki documented) |
| `/input/UseAxisRight`, `/input/GrabAxisRight` | Analog use/grab of held item (works unreliably; missing from client OSCQuery per wiki) |
| `/input/MoveHoldFB` | Move held object forward/back |
| `/input/SpinHoldCwCcw`, `SpinHoldUD`, `SpinHoldLR` | Spin held object |

### Buttons (`,T` or int 0/1)

- `/input/MoveForward|Backward|Left|Right` — act while held
- `/input/LookLeft`, `/input/LookRight` — turn while held
- `/input/Jump`, `/input/Run`
- `/input/ComfortLeft|Right` — snap-turn, **VR only**
- `/input/GrabRight|UseRight|DropRight` (+ `…Left`)
- `/input/Voice` — mic control; behavior depends on "Microphone Behaviour": with **Toggle Voice** on, `false → true` toggles mute (while held at 1, blocks controller/keyboard mute); off = push-to-mute (0 = muted, 1 = unmuted)
- `/input/PanicButton` — Safe Mode
- `/input/QuickMenuToggleLeft|Right`, `/input/AFKToggle`, `/input/ToggleSitStand`
- `/input/ShowDebugInfo0`–`9` — debug overlays (0: UiShapes, 2: build/FPS, 3: output log, 7: PhysBones/Contacts)

Examples:

```
/input/Vertical ,f 1.0        # walk forward
/input/Horizontal ,f -0.5     # strafe left at half speed
/input/Jump ,T 1              # press jump
/input/Jump ,T 0              # release (required)
```

Official TouchOSC demo layout: https://github.com/vrchat-community/osc/blob/main/files/touch-osc/vrc-input.tosc

## 3. Avatar OSC Parameters

**Writing.** Send to `/avatar/parameters/{ParameterName}` to set any custom (or aliased built-in) animator parameter on *your* avatar: `/avatar/parameters/VRCEmote ,i 3`.

**Reading.** VRChat echoes parameter values out on port 9001 whenever they change. When an avatar loads with OSC enabled, VRChat emits `/avatar/change` with the avatar ID.

### Custom parameter limits

| Type | Range | Sync cost |
|---|---|---|
| int | 0–255 | 8 bits |
| float | −1.0 … 1.0 | 8 bits (signed 8-bit fixed point over network) |
| bool | true/false | 1 bit |

- Synced custom parameters max **256 bits total**; up to **8192 total** custom parameters (synced+unsynced).
- Synced floats travel in **1/127** steps (−1.0, 0.0, 1.0 exact). Locally, OSC-written floats are full 32-bit — a value can look right locally yet quantize remotely.
- Sync types: **Speech** (visemes), **Playable** (0.1–1 s cadence; usual for synced custom params), **IK** (~10 Hz with remote interpolation; Puppet-controlled), **None** (local-only).
- **Cross-platform (PC/Quest) sync is by list position and type, not by name.**
- Don't use Trigger-typed parameters to store state; they de-sync.

**Reserved/built-in (read-only via OSC):** `VRCEmote` (int 1–16), `VRCFaceBlendH/V`, `GestureLeft/Right` (0–7), `GestureLeft/RightWeight`, `VelocityX/Y/Z`, `VelocityMagnitude`, `AngularY`, `Upright`, `Grounded`, `Seated`, `InStation`, `AFK`, `VRMode`, `TrackingType`, `MuteSelf`, `Earmuffs`, `Viseme` (0–14), `Voice`, `ScaleFactor` family, `AvatarVersion`, `IsOnFriendsList`, `IsAnimatorEnabled`, `PreviewMode`.

One writable extra: **`/avatar/eyeheight`** (float meters; 0.01–10000, official 0.1–100; ignored unless `/avatar/eyeheightscalingallowed` is true), with read-only `/avatar/eyeheightmin` (0.2) / `max` (5.0).

Sources: https://creators.vrchat.com/worlds/osc-avatar-parameters/ · https://creators.vrchat.com/avatars/animator-parameters/

## 4. Avatar Switching — `/avatar/change`

```
/avatar/change ,s "avtr_26187637-0c30-4a09-86e1-bc928c07309e"
```

Restrictions (2025.4.x openbeta): only avatars in your **Favorites**, **Recent**, **own uploads**, and **purchased** — no arbitrary public-avatar switching. Outbound: whenever a new avatar loads, VRChat sends `/avatar/change` back with its ID — how automation tracks the current avatar.

**OSC as automation:** `/avatar/change` + parameter writes + input OSC + chatbox = VRChat scriptable from anything that sends UDP: Stream Deck macros, OBS scene hooks, Twitch bots, Python daemons. Common pattern: automation listens on 9001 for parameter changes (a "Streaming" bool on your avatar) and reacts by sending input/chatbox/avatar-switch messages on 9000 — VRChat is both sensor and actuator.

## 5. VRCFaceTracking (VRCFT)

https://github.com/benaclejames/VRCFaceTracking (Apache 2.0, docs https://docs.vrcft.io) — the de-facto Windows OSC bridge for eye/face-tracking hardware (webcams, Tobii/Pupil eye trackers, Quest Pro via `QuestProOpenXRTrackingModule`, phone AR). Normalizes hardware signals into the **Unified Expressions**-based standard and republishes as OSC avatar parameters.

**Modules:** hardware support via community modules installed through the in-app **Module Registry**; developers write against the Tracking Module SDK feeding `UnifiedTracking`.

**FT v2 parameter spec:** parameters under `/avatar/parameters/…/v2/{Name}` — the `v2` prefix is detected at any nesting level. Floats 0–1 (−1–1 bidirectional, e.g. `v2/EyeLeftX`). Catalog spans:
- **Eye gaze:** `EyeLeftX/Y`, `EyeRightX/Y` (+ averaged `EyeX`, `EyeY`)
- **Eyes:** `EyeLid*`, `EyeSquint*`, `PupilDilation*`
- **Brow/Nose/Cheek:** `BrowInnerUp`, `NoseSneer*`, `CheekSquint*`
- **Jaw/Mouth/Lips:** `JawOpen`, `JawX/Z`, smile/frown/pucker, `MouthX/Y`
- **Tongue:** `TongueX/Y/Z` · **Neck** rotation params

Full reference: https://docs.vrcft.io/docs/tutorial-avatars/tutorial-avatars-extras/parameters

## 6. External Tools: TouchOSC, Chatbox OSC

**TouchOSC** (https://hexler.net/touchosc) — canonical OSC control surface: multi-page button/fader/XY-pad layouts, mapped to addresses over UDP. Official `vrc-input.tosc` layout ships in the vrchat-community/osc repo. Typical mapping: XY pad → `/input/Horizontal`+`Vertical`; impulse-mode buttons (send 1 then 0) → `/input/Jump`; faders → avatar parameters.

**Chatbox OSC:**

```
/chatbox/input  ,sT[T]  "Hello world" True False
/chatbox/typing ,T      True
```

- `/chatbox/input`: **text (max 144 chars, max 9 displayed lines)**, bool (**True = send immediately; False = open keyboard pre-filled**), optional bool (default True; **False suppresses the notification SFX**). Newlines whitelisted (2024.1.1), subject to the 9-line cap.
- `/chatbox/typing`: toggles the "typing…" indicator.
- Design caveat: the chatbox is for ephemeral, voice-like proximity communication, not a general chat channel — no spam, no way-around-mute.

## 7. Practical Recipes

**A. Mute-toggle hotkey:** bind a key/TouchOSC button to `/input/Voice` pulse (`1` then `0`). With Toggle Voice: a single button becomes a mute toggle. Read `MuteSelf` from 9001 to display true state.

**B. Avatar switcher macro** (AutoHotkey+Python, Stream Deck, TouchOSC):

```python
AVATARS = {
    "F1": "avtr_26187637-0c30-4a09-86e1-bc928c07309e",  # casual
    "F2": "avtr_9d58037b-xxxx-xxxx-xxxx-xxxxxxxxxxxx",  # streaming
}
```
On keypress: `/avatar/change ,s <id>`. IDs from `Avatars/*.json` filenames or `/avatar/change` messages on 9001.

**C. Python control:**

```python
# pip install python-osc
from pythonosc import udp_client, dispatcher, osc_server
import threading, time

client = udp_client.SimpleUDPClient("127.0.0.1", 9000)

disp = dispatcher.Dispatcher()
disp.map("/avatar/change", lambda a, *args: print("Avatar:", args))
disp.map("/avatar/parameters/*", lambda a, *args: print(a, args))
server = osc_server.ThreadingOSCUDPServer(("127.0.0.1", 9001), disp)
server.daemon_threads = True
threading.Thread(target=server.serve_forever, daemon=True).start()

client.send_message("/input/Vertical", 1.0)
time.sleep(2)
client.send_message("/input/Vertical", 0.0)
client.send_message("/input/Jump", True)
client.send_message("/input/Jump", False)

client.send_message("/avatar/parameters/OutfitColor", 0.5)
client.send_message("/chatbox/input", ["Hello from Python!", True, False])
client.send_message("/avatar/change", ["avtr_26187637-0c30-4a09-86e1-bc928c07309e"])
```

Note the list form for multi-argument messages, and mandatory release messages for held inputs.

## 8. Common Problems & Troubleshooting

1. **Localhost firewall.** OSC is UDP — Windows Firewall can silently drop 9000/9001 even for loopback in some configs, and definitely blocks LAN. Allow `VRChat.exe` through, or explicit inbound UDP 9000 / outbound UDP 9001 rules.
2. **Parameter "not found."** Name is case/spelling-sensitive and must exist on the *currently worn* avatar; built-ins like `VelocityX` are read-only. Check the auto-generated JSON at `%LocalAppData%Low\VRChat\VRChat\OSC\{userId}\Avatars\{avatarId}.json`. **Configs are not generated for Build & Test avatars** — publish (or use a published test copy).
3. **Stuck inputs.** Axes and buttons latch — always zero axes on script exit, pair presses with releases. Port mixups (send to 9001, listen on 9000) produce "nothing happens."
4. **Float ranges.** Custom floats −1.0…1.0 (clamped); synced floats quantize to 1/127 remotely; ints 0–255.
5. **Synced vs local.** Sync type **None** = local only. Playable params update on 0.1–1 s cycles. The 256-bit budget means adding one synced bool can silently push older params out of sync.
6. **Config-file fragility.** The per-avatar JSON rerouting is officially "a stop-gap … may be removed"; edits require VRChat restart and exact `"type"` casing (`"Int"`, `"Float"`, `"Bool"`).
7. **`/avatar/change` rejects the ID** — public/unfavorited avatar.
8. **Chatbox limits** — 144 chars/9 lines truncates; SFX bool only applies to immediate sends.

Sources: https://creators.vrchat.com/worlds/osc/ · /osc-as-input-controller/ · /osc-avatar-parameters/ · https://docs.vrchat.com/docs/local-vrchat-storage · https://wiki.vrchat.com/wiki/OSC · https://github.com/vrchat-community/osc · https://docs.vrcft.io · https://hexler.net/touchosc
