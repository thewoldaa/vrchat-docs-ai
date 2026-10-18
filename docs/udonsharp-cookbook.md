# The VRChat UdonSharp Cookbook (2025–2026 Edition)

A code-heavy field manual for world creators. All API usage verified against `creators.vrchat.com` (Udon/UdonSharp docs), `udonsharp.docs.vrchat.com`, the `vrchat-community/UdonSharp` wiki, and VRChat SDK 3.x class exposure.

---

## 1. Setup & Script Structure

### 1.1 The skeleton

Every Udon script inherits from `UdonSharpBehaviour` (not `MonoBehaviour`):

```csharp
using UdonSharp;        // UdonSharpBehaviour, attributes
using UnityEngine;
using VRC.SDKBase;      // Networking, VRCPlayerApi
using VRC.Udon;         // UdonBehaviour
using VRC.SDK3.Components; // VRCPickup, VRCObjectPool, VRCUrlInputField

public class MyScript : UdonSharpBehaviour
{
    public GameObject target;              // public fields appear in Inspector

    [UdonSynced] private bool _state;      // replicated to all players

    void Start() { }

    public override void Interact() { }
}
```

UdonSharp generates a `UdonBehaviour` + `UdonSharpProgramAsset` automatically. One GameObject can host multiple UdonBehaviours.

### 1.2 `[UdonSynced]` fields

- **Owner-writable only.** Non-owner writes get overwritten at the next sync tick.
- Syncable types: `bool`, `char`, `byte`, `sbyte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`, `Vector2/3/4`, `Quaternion`, `string`, `VRCUrl`, `Color`, `Color32`, **and arrays of these**.
- **Always initialize synced arrays** (even `new int[0]`). An uninitialized synced array causes the *entire behaviour* to silently fail to sync.
- `[UdonSynced(UdonSyncMode.Linear)]` lerps floats/Vector3s; `Smooth` does exponential smoothing.
- `[FieldChangeCallback(nameof(MyProperty))]` routes network changes through a property setter — the cleanest reaction pattern (§2.1).

### 1.3 Sync modes

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]   // attribute-enforced
```

| Mode | Behaviour |
|---|---|
| `Continuous` | Frequent automatic sync; **unreliable** (may drop updates). Good for positions. |
| `Manual` | Syncs only on `RequestSerialization()`. **Reliable** — correct for state. |
| `None` | No variable sync; `SendCustomNetworkEvent` also **won't work**. |
| `NoVariableSync` | No sync but local-only logic allowed. |

**Rule of thumb:** state → Manual + `RequestSerialization()`. Continuous motion → Continuous (+ Linear). One-shot effects → `SendCustomNetworkEvent`.

### 1.4 Common compile errors

| Error | Cause / fix |
|---|---|
| `CS0246: List<> not found` | No generics — use `T[]` arrays, `DataList`, `DataDictionary`. |
| `foreach` on List / LINQ | LINQ not exposed — write `for` loops. |
| `try/catch/throw` fails | No exceptions — guard clauses, null checks. |
| `Cannot await` / no `StartCoroutine` | No async/coroutines — `SendCustomEventDelayedSeconds`. |
| `CS8025` on switch expressions/tuples | Modern C# sugar not implemented — classic syntax. |
| `'UdonBehaviour' does not contain 'Interact'` | Field typed as `UdonBehaviour` — serialize as **your script's type** (`public MyScript other;`). |
| Inspector fields are `None` | Broken prefab serialized-field propagation — reassign on the instance. |
| Synced array never arrives | Array was `null` at sync time — initialize at declaration. |

To check whether a class/method exists in Udon: **VRChat SDK → Utilities → Class Exposure Tree**.

## 2. Complete Examples

### 2.1 Networked toggle (Manual sync + FieldChangeCallback)

The canonical pattern. Sync mode: **Manual**.

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class SyncedToggle : UdonSharpBehaviour
{
    public GameObject toggleObject;

    // Private synced field; network changes routed into the property below.
    [UdonSynced, FieldChangeCallback(nameof(SyncedState))]
    private bool _syncedState;

    public bool SyncedState
    {
        set
        {
            _syncedState = value;
            toggleObject.SetActive(value);
        }
        get => _syncedState;
    }

    public override void Interact()
    {
        // 1. Take ownership — only owners can write synced fields.
        Networking.SetOwner(Networking.LocalPlayer, gameObject);
        // 2. Change via the PROPERTY (writing the field directly bypasses
        //    the callback on remote players).
        SyncedState = !SyncedState;
        // 3. Manual sync mode: flag for the next network tick.
        RequestSerialization();
    }
}
```

`FieldChangeCallback` fires the setter on network deserialization — remote players get `SetActive` applied automatically.

### 2.2 Door with animation + sync

Setup: door with an `Animator` ("Open"/"Close" trigger), collider, Manual sync.

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class SyncedDoor : UdonSharpBehaviour
{
    public Animator doorAnimator;
    public AudioSource doorSound;

    [UdonSynced] private bool _isOpen;

    public override void Interact()
    {
        Networking.SetOwner(Networking.LocalPlayer, gameObject);
        _isOpen = !_isOpen;
        ApplyState(_isOpen);          // apply locally for responsiveness
        RequestSerialization();
    }

    public override void OnDeserialization()
    {
        ApplyState(_isOpen);          // late joiners + everyone else
    }

    private void ApplyState(bool open)
    {
        if (doorAnimator != null)
            doorAnimator.SetTrigger(open ? "Open" : "Close");
        if (doorSound != null)
            doorSound.PlayOneShot(doorSound.clip);
    }
}
```

### 2.3 Teleport pad with cooldown (local only)

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.NoVariableSync)]
public class TeleportPad : UdonSharpBehaviour
{
    public Transform destination;
    public float cooldownSeconds = 3f;

    private float _lastTeleportTime = -999f;

    public override void OnPlayerTriggerEnter(VRCPlayerApi player)
    {
        if (!player.isLocal) return;
        if (Time.time - _lastTeleportTime < cooldownSeconds) return;

        _lastTeleportTime = Time.time;
        player.TeleportTo(
            destination.position,
            destination.rotation,
            VRC_SceneDescriptor.SpawnOrientation.AlignPlayerWithSpawnPoint,
            lerpOnRemote: false
        );
    }
}
```

### 2.4 Respawn / kill floor

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.NoVariableSync)]
public class KillFloor : UdonSharpBehaviour
{
    public Transform respawnPoint; // optional custom point

    public override void OnPlayerTriggerEnter(VRCPlayerApi player)
    {
        if (!player.isLocal) return;
        if (respawnPoint != null)
            player.TeleportTo(respawnPoint.position, respawnPoint.rotation);
        else
            player.Respawn();
    }
}
```

### 2.5 Pickup launcher with cooldown + haptics

Setup: GameObject with `VRCPickup` (Auto Hold on), `VRCObjectPool` of projectiles.

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.SDK3.Components;
using VRC.Udon;
using VRC.Udon.Common.Interfaces;

[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class BallLauncher : UdonSharpBehaviour
{
    public VRCObjectPool projectilePool;
    public Transform muzzle;
    public float projectileSpeed = 10f;
    public float shotCooldown = 0.5f;

    private float _lastShotTime = -999f;

    public override void OnPickupUseDown()
    {
        if (Time.time - _lastShotTime < shotCooldown) return;
        _lastShotTime = Time.time;

        var pickup = (VRCPickup)GetComponent(typeof(VRCPickup));
        Networking.LocalPlayer.PlayHapticEventInHand(
            pickup.currentHand, 0.1f, 0.5f, 0.5f);

        // Broadcast spawn so everyone sees the shot.
        SendCustomNetworkEvent(NetworkEventTarget.All, nameof(NetworkShoot));
    }

    public void NetworkShoot()
    {
        GameObject ball = projectilePool.TryToSpawn();
        if (ball == null) return;   // pool exhausted

        ball.transform.SetPositionAndRotation(muzzle.position, muzzle.rotation);
        var rb = (Rigidbody)ball.GetComponent(typeof(Rigidbody));
        if (rb != null)
        {
            rb.velocity = Vector3.zero;
            rb.AddForce(muzzle.forward * projectileSpeed, ForceMode.VelocityChange);
        }
    }
}
```

**Gotcha:** network events carry only syncable value types (max 8 params, `[NetworkCallable]`, since SDK 3.8.1) — not `VRCPlayerApi` or object references. Here no params are needed because pool + muzzle are scene references identical on every client.

### 2.6 Global synced slider → light intensity

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.Continuous)]
public class GlobalLightSlider : UdonSharpBehaviour
{
    public Light targetLight;
    public UnityEngine.UI.Slider slider;

    [UdonSynced] private float _intensity = 1f;

    // Wire: Slider OnValueChanged -> this behaviour -> "OnSliderChanged"
    public void OnSliderChanged()
    {
        if (!Networking.IsOwner(gameObject))
            Networking.SetOwner(Networking.LocalPlayer, gameObject);
        _intensity = slider.value;
        Apply(_intensity);
    }

    public override void OnDeserialization() => Apply(_intensity);

    private void Apply(float v)
    {
        if (targetLight != null) targetLight.intensity = v;
        if (slider != null && !Networking.IsOwner(gameObject))
            slider.value = v;   // watchers see the synced position
    }
}
```

### 2.7 Random item spawner (self-rescheduling timer, no Update)

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.NoVariableSync)]
public class RandomItemSpawner : UdonSharpBehaviour
{
    public VRCObjectPool itemPool;
    public Transform[] spawnPoints;
    public float minInterval = 15f;
    public float maxInterval = 45f;

    void Start() => ScheduleNext();

    private void ScheduleNext()
    {
        float interval = Random.Range(minInterval, maxInterval);
        SendCustomEventDelayedSeconds(nameof(TrySpawn), interval);
    }

    public void TrySpawn()
    {
        if (itemPool != null && spawnPoints != null && spawnPoints.Length > 0)
        {
            GameObject item = itemPool.TryToSpawn();
            if (item != null)
            {
                Transform point = spawnPoints[Random.Range(0, spawnPoints.Length)];
                item.transform.SetPositionAndRotation(point.position, point.rotation);
            }
        }
        ScheduleNext();
    }
}
```

The core performance idiom: **a self-rescheduling delayed event replaces `Update()` polling entirely**.

### 2.8 Simple per-player score game

```csharp
// TargetHit.cs — on the target, Manual sync
[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class TargetHit : UdonSharpBehaviour
{
    public ScoreBoard scoreboard;

    public override void Interact()
    {
        scoreboard.AddPoint(Networking.LocalPlayer.playerId);
    }
}
```

```csharp
// ScoreBoard.cs — one object, Manual sync, owner persists the score
[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class ScoreBoard : UdonSharpBehaviour
{
    public UnityEngine.UI.Text display;

    // Parallel arrays — Udon has no Dictionary<T,K>.
    [UdonSynced] private int[] _playerIds = new int[0];  // ALWAYS initialize!
    [UdonSynced] private int[] _scores = new int[0];

    public void AddPoint(int playerId)
    {
        SendCustomNetworkEvent(NetworkEventTarget.All, nameof(RecordPoint), playerId);
    }

    [NetworkCallable]
    public void RecordPoint(int playerId)
    {
        if (!Networking.IsOwner(gameObject))
            Networking.SetOwner(Networking.LocalPlayer, gameObject);

        int idx = IndexOfPlayer(playerId);
        if (idx < 0)
        {
            int n = _playerIds.Length;
            int[] newIds = new int[n + 1];
            int[] newScores = new int[n + 1];
            _playerIds.CopyTo(newIds, 0);
            _scores.CopyTo(newScores, 0);
            newIds[n] = playerId;
            newScores[n] = 1;
            _playerIds = newIds;
            _scores = newScores;
        }
        else
        {
            _scores[idx] += 1;
        }

        RequestSerialization();
        UpdateDisplay();
    }

    public override void OnDeserialization() => UpdateDisplay();

    private int IndexOfPlayer(int id)
    {
        for (int i = 0; i < _playerIds.Length; i++)
            if (_playerIds[i] == id) return i;
        return -1;
    }

    private void UpdateDisplay()
    {
        if (display == null) return;
        string s = "";
        for (int i = 0; i < _playerIds.Length; i++)
        {
            VRCPlayerApi p = VRCPlayerApi.GetPlayerById(_playerIds[i]);
            string name = (p != null) ? p.displayName : "left";
            s += name + ": " + _scores[i] + "\n";
        }
        display.text = s;
    }
}
```

**Note:** `VRCPlayerApi.GetPlayerById` returns `null` for departed players — always null-check (no try/catch to save you!).

### 2.9 Join/leave messages

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.NoVariableSync)]
public class JoinLeaveMessages : UdonSharpBehaviour
{
    public UnityEngine.UI.Text feed;

    public override void OnPlayerJoined(VRCPlayerApi player)
    {
        // On YOUR join, this fires once for every player already in the
        // instance (including yourself); afterwards, once per new joiner.
        Append("joined", player.displayName);
    }

    public override void OnPlayerLeft(VRCPlayerApi player)
    {
        Append("left", (player != null) ? player.displayName : "someone");
    }

    private void Append(string action, string name)
    {
        if (feed != null) feed.text = name + " " + action + "\n" + feed.text;
    }
}
```

### 2.10 Late-joiner sync pattern (the most important section)

**The rule:** *Variables are updated for late joiners; events are not.* Network events fired before a late joiner arrived are lost forever.

1. **Persistent state → synced variables.** Never encode durable state in "who last fired an event."
2. **Cosmetic one-shots → network events.** Particles, sounds.
3. Always implement `OnDeserialization` to apply synced state.

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class LateJoinSafeState : UdonSharpBehaviour
{
    public GameObject[] lightsToEnable;
    public MeshRenderer indicator;

    [UdonSynced] private bool _partyMode;
    [UdonSynced] private Color _lightColor = Color.white;

    public override void Interact()
    {
        Networking.SetOwner(Networking.LocalPlayer, gameObject);
        _partyMode = !_partyMode;
        _lightColor = Random.ColorHSV();
        RequestSerialization();
        ApplyState();
    }

    public override void OnDeserialization() => ApplyState();

    private void ApplyState()
    {
        foreach (var go in lightsToEnable)
            if (go != null) go.SetActive(_partyMode);
        if (indicator != null) indicator.material.color = _lightColor;
    }
}
```

**Elapsed-time sync:** owner syncs an *offset* against `Time.realtimeSinceStartup` (which differs per client); receivers compute `offset + theirRealtime`. The `OnDeserialization(DeserializationResult result)` overload exposes `sendTime`/`receiveTime` on the same clock.

**Ownership handoff on late join:** master owns everything at world load by default. In `OnPlayerJoined`, the joiner can claim objects — remember *every existing client* runs `OnPlayerJoined`, so gate on `player.isLocal`.

### 2.11 Audio zone crossfade (step-based, no Update)

```csharp
[UdonBehaviourSyncMode(BehaviourSyncMode.NoVariableSync)]
public class AudioZoneCrossfade : UdonSharpBehaviour
{
    public AudioSource zoneA;
    public AudioSource zoneB;
    public float fadeSeconds = 1.5f;

    private bool _inZoneA = true;
    private bool _fading = false;

    void Start()
    {
        zoneA.volume = 1f; zoneB.volume = 0f;
        zoneA.Play(); zoneB.Play();
    }

    public override void OnPlayerTriggerEnter(VRCPlayerApi player)
    { if (player.isLocal) StartFade(true); }

    public override void OnPlayerTriggerExit(VRCPlayerApi player)
    { if (player.isLocal) StartFade(false); }

    private void StartFade(bool enteringA)
    {
        _inZoneA = enteringA;
        if (!_fading)
        {
            _fading = true;
            SendCustomEventDelayedSeconds(nameof(FadeStep), 0.05f);
        }
    }

    public void FadeStep()
    {
        float step = 0.05f / fadeSeconds;
        float targetA = _inZoneA ? 1f : 0f;
        float targetB = _inZoneA ? 0f : 1f;

        zoneA.volume = Mathf.MoveTowards(zoneA.volume, targetA, step);
        zoneB.volume = Mathf.MoveTowards(zoneB.volume, targetB, step);

        if (!Mathf.Approximately(zoneA.volume, targetA))
            SendCustomEventDelayedSeconds(nameof(FadeStep), 0.05f);
        else
            _fading = false;
    }
}
```

### 2.12 Simple video player queue

Setup: `VRCAVProVideoPlayer` (or Unity player), `VRCUrlInputField`, UI text. **Owner manages the queue**; `OnVideoEnd` advances it.

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.SDK3.Components;
using VRC.SDK3.Components.Video;
using VRC.Udon;

[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class VideoQueue : UdonSharpBehaviour
{
    public VRCAVProVideoPlayer videoPlayer;
    public VRCUrlInputField urlInput;
    public UnityEngine.UI.Text queueDisplay;

    [UdonSynced] private VRCUrl[] _queue = new VRCUrl[0]; // initialized!
    [UdonSynced] private int _head = 0;
    [UdonSynced] private bool _playing = false;

    // Wire to VRCUrlInputField's OnValueChanged -> "EnqueueSubmitted"
    public void EnqueueSubmitted()
    {
        if (!Networking.IsOwner(gameObject))
            Networking.SetOwner(Networking.LocalPlayer, gameObject);

        VRCUrl submitted = urlInput.GetUrl();
        if (submitted == null || submitted.ToString().Length == 0) return;

        int n = _queue.Length;
        VRCUrl[] newQueue = new VRCUrl[n + 1];
        _queue.CopyTo(newQueue, 0);
        newQueue[n] = submitted;
        _queue = newQueue;
        urlInput.SetText("", false, false);

        if (!_playing) { _playing = true; PlayHead(); }
        RequestSerialization();
        UpdateDisplay();
    }

    public override void OnVideoEnd()
    {
        if (!Networking.IsOwner(gameObject)) return;
        if (_head + 1 < _queue.Length)
        {
            _head++;
            PlayHead();
        }
        else _playing = false;
        RequestSerialization();
    }

    public override void OnVideoError(VideoError videoError)
    {
        Debug.Log("Video error: " + videoError);
        if (Networking.IsOwner(gameObject)) OnVideoEnd();  // skip on error
    }

    public override void OnDeserialization()
    {
        if (_playing && !videoPlayer.IsPlaying) PlayHead();
        UpdateDisplay();
    }

    public override void OnPlayerJoined(VRCPlayerApi player)
    {
        if (player.isLocal && _playing && _queue.Length > 0) PlayHead();
    }

    private void PlayHead()
    {
        if (_head < 0 || _head >= _queue.Length) return;
        videoPlayer.PlayURL(_queue[_head],
            VRC.Udon.Common.Enums.VideoResolutionControl.Auto);
    }

    private void UpdateDisplay()
    {
        if (queueDisplay == null) return;
        string s = "";
        for (int i = _head; i < _queue.Length; i++)
            s += (i == _head ? "▶ " : "") + _queue[i].ToString() + "\n";
        queueDisplay.text = s;
    }
}
```

**Sync note:** the video player component syncs current URL/playback state automatically; the queue itself needs your own synced state (above). Remember the **5-second per-user URL rate limit** across all players.

## 3. Networking Patterns Explained

### 3.1 Ownership model

- Every GameObject has exactly one owner. At instance start, **master** owns everything not explicitly assigned.
- Only the owner's `[UdonSynced]` writes are authoritative.
- `Networking.SetOwner(player, gameObject)` requests ownership; `Networking.IsOwner(gameObject)` checks.
- `OnOwnershipRequest(requester, newOwner)` → return `false` to reject (master-only systems). VRChat may still force-transfer when the owner leaves.

### 3.2 Master vs owner

| | Master | Owner |
|---|---|---|
| Who | First player in instance; re-elected on leave | Per-GameObject |
| Find | `Networking.LocalPlayer.isMaster` | `Networking.IsOwner(go)` |
| Use for | Instance-global decisions (global spawners, cleanup) | Per-object state writes |

### 3.3 When to use what

| Situation | Use |
|---|---|
| Persistent world state | Synced var, **Manual** + `RequestSerialization()` |
| Continuously moving object | Synced Vector3 + Linear/Continuous, or `VRCObjectSync` |
| One-shot cosmetic for everyone | `SendCustomNetworkEvent(All, ...)` |
| Tell the owner to do something | `SendCustomNetworkEvent(Owner, ...)` |
| Local-only effects (haptics, teleport) | Plain method call |
| Fresh tracking data | `PostLateUpdate` + `GetTrackingData` |

### 3.4 Event targeting

`NetworkEventTarget.All` (sender executes locally+immediately), `.Others` (excludes sender), `.Owner`, `.Self` (loopback, bypasses rate limits). `[NetworkCallable(maxEventsPerSecond: X)]` rate-limits.

### 3.5 Common desync bugs and fixes

1. **Non-owner writes a synced var** → change snaps back. Fix: `SetOwner` before writing.
2. **Forgot `RequestSerialization()` in Manual** → owner sees it, nobody else does.
3. **Late joiners see default state** → apply state in `OnDeserialization`; never rely on events for state.
4. **Uninitialized synced array** → *nothing* on the behaviour syncs. Initialize!
5. **Ownership race** (two players interact simultaneously) → `OnOwnershipRequest` validation, or design state as a function of a single synced value.
6. **FieldChangeCallback bypassed** → writing the backing field directly skips the setter; write through the property.
7. **`SendCustomNetworkEvent` does nothing** → target's sync mode is `None`.
8. **Works in ClientSim, not in VRChat** → ClientSim is single-player; ownership paths differ. Test with two clients.

## 4. Events Reference Table

All are `public override` methods on `UdonSharpBehaviour`.

### Lifecycle
`Start`, `Update` (expensive — avoid), `LateUpdate`, `PostLateUpdate` (after IK — freshest tracking), `FixedUpdate`, `OnEnable/OnDisable`, `OnDestroy`, `OnSpawn`.

### Interaction & pickups
`Interact()`, `OnPickup()/OnDrop()`, `OnPickupUseDown()/OnPickupUseUp()`, `OnOwnershipRequest(VRCPlayerApi, VRCPlayerApi) → bool`, `OnOwnershipTransferred(VRCPlayerApi)`.

### Players
`OnPlayerJoined/Left/Respawn/Restored(VRCPlayerApi)`, `OnPlayerTriggerEnter/Stay/Exit(VRCPlayerApi)`, `OnPlayerCollisionEnter/Stay/Exit(VRCPlayerApi)`, `OnPlayerParticleCollision(VRCPlayerApi)`, `OnControllerColliderHitPlayer(ControllerColliderPlayerHit)`.

### Stations & video
`OnStationEntered/Exited(VRCPlayerApi)` (needs VRCStation), `OnVideoEnd()`, `OnVideoError(VideoError)`, `OnVideoLoop()`, `OnVideoPause()/OnVideoPlay()/OnVideoStart()/OnVideoReady()`.

### Networking
`OnPreSerialization()`, `OnPostSerialization(SerializationResult)`, `OnDeserialization()`, `OnDeserialization(DeserializationResult)`.

### Input (each: `void InputX(bool value, UdonInputEventArgs args)`)
`InputJump`, `InputUse`, `InputGrab`, `InputDrop`, `InputMoveHorizontal/Vertical`, `InputLookHorizontal/Vertical` — use `args.eventType`/`args.handType`.

### MIDI
`MidiNoteOn/Off(channel, number, velocity)`, `MidiControlChange(channel, number, value)`.

## 5. Performance

**Why `Update()` is expensive:** Udon is a bytecode interpreter, not native C#. Every `Update()` behaviour executes every frame on every client, and interpreter overhead multiplies across behaviours in a busy 40-player instance. Over-budget worlds throttle ALL Udon.

**Alternatives:**
1. **`SendCustomEventDelayedSeconds`/`Frames`** — self-rescheduling work instead of polling. The single biggest Udon optimization.
2. **Event-driven everything**: `OnPlayerTriggerEnter` instead of distance checks; `OnVideoEnd` instead of polling `IsPlaying`.
3. **Animator-driven animation** — Unity's Animator is native code; Udon just fires triggers. Animator doors cost near-zero Udon time.
4. **Disable what you don't need** — `SetActive(false)` stops Update; stopped delayed-event chains cost nothing.
5. **`PostLateUpdate` only when you need fresh tracking data.**
6. **No per-frame string building** — update UI only when values change.
7. **Rate-limit network events** on spammy handlers.

**Also:** `OnPlayerTriggerStay`/`OnPlayerCollisionStay` fire every frame per player — treat like Update. Heavy one-shot work can be time-sliced across frames with delayed events.

## 6. Debugging

**Logging:** `Debug.Log/LogWarning/LogError` — string interpolation `$"..."` supported. Logs appear in the Unity console (ClientSim) and in `%LOCALAPPDATA%..\..\LocalLow\VRChat\VRChat\output_log_*.txt` for test builds.

**ClientSim** runs your world in editor Play Mode with a fake player. It's single-player — ownership races and event ordering behave differently than live, so final testing needs **two VRChat clients**.

**Common runtime errors:**

| Symptom | Cause → fix |
|---|---|
| NullReference on a player | `GetPlayerById(id)` returns null after departure — null-check everything player-related. |
| Synced change "doesn't fire" remotely | Checking the field in Update with no side effects — use `OnDeserialization` or `FieldChangeCallback`. |
| Delayed event never runs | Event name misspelled (use `nameof(...)`) or behaviour disabled. |
| Object doesn't interact | Missing collider, wrong layer, or disabled UdonBehaviour. |
| Editor works, build doesn't | Broken prefab serialized fields, or editor-only API (check Class Exposure Tree). |

## 7. Extern Availability Gotchas

Udon is a **restricted VM** — your C# compiles to Udon assembly; anything without a Udon instruction cannot run.

### Not available
- **`async`/`await`** — use `SendCustomEventDelayedSeconds` chains.
- **`try`/`catch`/`throw`** — guard clauses; `TryGetValue`-style APIs.
- **Coroutines/`yield`** — delayed events, frame-splitting.
- **Generics** (`List<T>`, `Dictionary<K,V>`) — `T[]` arrays, `DataList`, `DataDictionary` (values via `DataToken`, `TryGetValue` with cast).
- **LINQ** — `for` loops.
- **Delegates/C# events at runtime** — `SendCustomEvent`, Inspector-wired UI events.
- **Interfaces** — `UdonSharpBehaviour` inheritance + virtual override works.
- **File/network I/O, threads** — supported subset: `VRCStringDownloader`, `VRCImageDownloader`, `VRCJson`.
- **Player camera access** — `localPlayer.GetTrackingData(TrackingData.Head)`.
- **Non-Udon `MonoBehaviour` inheritance.**
- **Method overloading** — recent UdonSharp supports it, but verify with your SDK.

### Available (often assumed missing)
- **Properties** (`get`/`set`) — `[FieldChangeCallback]` depends on them.
- **String interpolation `$"..."`**, `string.Format/Split/Substring`.
- **Jagged arrays**, `out`/`ref` externs, `params`, extension methods, static methods, `[RecursiveMethod]`.
- **`??` null-coalescing**, ternary, full flow control.

### Other sharp edges
- **Numeric casts are overflow-checked** — `(byte)300` throws at runtime rather than wrapping.
- **Field initializers are compile-time** — scene-dependent init goes in `Start()`.
- **Prefab serialized-field propagation** is unreliable for U# — prefer scene objects.
- **Class Exposure Tree** (SDK → Utilities) is the single source of truth for callable APIs.

## Sources

- https://creators.vrchat.com/worlds/udon/udonsharp/
- https://udonsharp.docs.vrchat.com/udonsharp/ · https://udonsharp.docs.vrchat.com/events/
- https://creators.vrchat.com/worlds/udon/networking/ · /networking/network-components/ · /networking/events/
- https://creators.vrchat.com/worlds/udon/graph/event-nodes/
- https://github.com/vrchat-community/UdonSharp/wiki/events
- https://docs.vrchat.com/docs/udon-networking
- https://udonsharp.docs.vrchat.com/frequently-asked-questions/
