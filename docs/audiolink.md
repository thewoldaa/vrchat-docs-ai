# AudioLink (VRChat Audio-Reactive System) — Comprehensive Research Report (2025–2026)

*All property names, function signatures, region macros, and changelog entries read directly from the repository master branch. Performance figures qualitative.*

---

## 1. What AudioLink Is and How It Works

AudioLink is an open-source Unity package (llealloo, with pema99, float3, cnlohr, fundale, techanon and others) that analyzes in-world audio in VRChat and exposes it as reactive data streams to **Udon scripts, world shaders, and avatar shaders**. Also compatible with ChilloutVR and PoligonVR.

**The data pipeline:**

1. An `AudioSource` in the world plays audio.
2. Per-frequency amplitude data is read on the CPU into Udon using Unity's `GetOutputData` (via the `VRCSDK3.Rendering`/`VRCShader` API).
3. Samples are sent to the GPU where a shader performs signal processing (FFT/DFT, band splitting, ColorChord note detection, autocorrelation, chronotensity/beat-tracking) into a **CustomRenderTexture**.
4. That CRT is broadcast as the **global shader texture `_AudioTexture`** (128-wide). Because it's global (`Shader.SetGlobalTexture`), it's sampleable by **world shaders and every avatar's shaders in the instance** — the core trick: VRChat's global shader properties propagate from world to avatars.

The texture is a multi-region data atlas (`AudioLink.cginc`, `ALPASS_*` constants):

| Region | Macro | Location | Size |
|---|---|---|---|
| 4-band audio (bass/low-mid/high-mid/treble) | `ALPASS_AUDIOLINK` | (0,0) | 128×4 |
| DFT (log-frequency spectrum) | `ALPASS_DFT` | (0,4) | 128×2 |
| Waveform | `ALPASS_WAVEFORM` | (0,6) | 128×16 |
| 4-band history (128 samples) | `ALPASS_AUDIOLINKHISTORY` | (1,0) | 127×4 |
| VU/general (network/unix time, player info, media state) | `ALPASS_GENERALVU` | (0,22) | various |
| Theme colors 0–3 | `ALPASS_THEME_COLOR0..3` | (0–3,23) | 1×1 each |
| ColorChord colors/strip/lights | `ALPASS_CCCOLORS/CCSTRIP/CCLIGHTS` | (25,22)/(0,24)/(0,25) | — |
| Autocorrelator (BPM detection) | `ALPASS_AUTOCORRELATOR` | (0,27) | 128×1 |
| Filtered audio — 4 bands × **4 delay lines** | `ALPASS_FILTEREDAUDIOLINK` | (0,28) | 16×4 |
| Chronotensity (beat-mapped timers) | `ALPASS_CHRONOTENSITY` | (16,28) | 8×4 |
| Global strings (local player, master, 2 custom, 32 chars) | `ALPASS_GLOBAL_STRINGS` | (40,28) | 8×4 |

**Version status:** **2.1.0 (Mar 2025)** phase info via DFT alpha channel, Unity 6 fixes · **3.1.0 (Jan 2026)** **URP support**, theme colors in AudioReactiveLight, Persistent ColorChord Colors, Media Playback Reporting API · **3.1.2 (Feb 2026)** current. Package id `com.llealloo.audiolink` (VCC Curated, or `vrc-get i audiolink`). Unity 2022.3+ required since 1.2.1 (async GPU readbacks on mobile).

## 2. Setup in Worlds

1. Install via VCC (Curated) — note: package files don't show in Project-search by default; press "In Packages".
2. Drag the **AudioLink prefab** into the scene (or `Tools > AudioLink > Add AudioLink Prefab`).
3. Assign an **AudioSource** to `audioSource`. Tooltip warning: *use with AudioLinkInput unless the source is 2D — a spatialized source attenuates by distance*, weakening reactivity as you walk away.
4. Optional `optionalRightAudioSource` for **dual-mono** (AVPro players) — supported since 2.0.0.
5. Optional: "Link all sound reactive objects to this AudioLink" auto-wires all `AudioReactive*` components.
6. Optional **AudioLinkController prefab** — world-UI panel adjusting gain, bass, treble, band crossovers, thresholds, fade, theme colors, media controls. Since 2.0.0 configurable which parts are network-synced. Settings are `[UdonSynced]` and take over local values.

**Which sources feed it:** any AudioSource — bundled AudioLinkMiniPlayer/ytDlpPlayer, third-party players (ProTV, USharpVideo via adapter), plain clips. `autoSetMediaState` (on by default) publishes "is playing" to shaders; disable if your player drives media state itself.

Main inspector fields: `gain`, `bass`, `treble` (0–2 EQ), band crossovers `x0–x3`, thresholds `threshold0–3`, `fadeLength`/`fadeExpFalloff`, `autogain`, `themeColorMode` (ColorChord/Custom/Persistent), four `customThemeColor`s, two synced **custom global strings** (32 UTF-32 chars — e.g., DJ messages readable on all avatars), `audioDataToggle` (experimental CPU readback).

**Avatar testing in-editor:** the **AudioLinkAvatar** prefab plays a track and publishes `_AudioTexture` in-editor — no VRChat needed.

## 3. Setup on Avatars

Avatars do **not** include the AudioLink prefab. What matters:

- The avatar shader must be **AudioLink-enabled** — VRChat identifies ready shaders via the **`_AudioLinkAddon`** property (`[Toggle]` + AudioLink header annotation; the mechanism Poiyomi/lilToon use for their AL toggles). Enabled at runtime → the shader reads the world-broadcast `_AudioTexture`.
- Shaders with built-in support: Poiyomi (module per effect), lilToon, orels1's AudioLink Shader, Silent Cel Shading, Mochie, Fire Lite, **VRSL** (club fixtures standard), plus more.
- Test locally: import AudioLink into the avatar project and use the **AudioLinkAvatar** prefab.

**Bands and delay lines:** four canonical bands — bass, low-mids, high-mids, treble (`ALPASS_AUDIOBASS/LOWMIDS/HIGHMIDS/TREBLE`) split by adjustable crossovers. Each band has **4 delay lines** in `ALPASS_FILTEREDAUDIOLINK` — delays let effects "chase" the beat (trails, traveling pulses). C# side: `GetBandAsSmooth(band, delay, smooth)`.

## 4. Shader-Side Usage

```hlsl
#include "UnityCG.cginc"
#include "Packages/com.llealloo.audiolink/Runtime/Shaders/AudioLink.cginc"
```

`_AudioLinkAddon` is the *discovery* mechanism; actual sampling is via global `_AudioTexture` + `_AudioTexture_TexelSize`. If you sample raw without the cginc it works anywhere; cginc helpers require version match (upgrader: `Tools > AudioLink > Update AudioLink Compatible Shaders`).

**Core accessors:**

```hlsl
float4 AudioLinkData(uint2 xycoord);      // raw texel fetch
float4 AudioLinkDataMultiline(uint2 xy);  // wraps x overflow onto next row
float4 AudioLinkLerp(float2 xy);          // lerp between adjacent columns
bool   AudioLinkIsAvailable();
```

**Minimal fragment shader:**

```hlsl
// Column 0 = current value; columns 1..127 = history. uv.y*4 picks a band row.
return AudioLinkData(ALPASS_AUDIOLINK + int2(0, i.uv.y * 4.)).rrrr;
```

**Useful helpers:** `AudioLinkGetAmplitudeAtFrequency(hz)` (13.75–14080 Hz), `AudioLinkGetAmplitudeAtNote(octave, note)`, `AudioLinkGetChronoTime(index, band)` (beat-locked `_Time.y`), `AudioLinkGetTimeOfDay()`, `AudioLinkGetLocalPlayerNameChar(i)`, `AudioLinkGetCustomString1Char(i)`, `AudioLinkGetAudioSourcePosition()`, `AudioLinkDecodeDataAsUInt/Seconds(indexloc)`, `AudioLinkCCtoRGB(bin, intensity, rootNote)`.

**Emissive pulse with delay line:**

```hlsl
float bass = AudioLinkData(ALPASS_AUDIOBASS).r;
float delayed = AudioLinkData(ALPASS_FILTEREDAUDIOLINK + uint2(4*0 + 2, 0)).r;
float t = AudioLinkGetChronoTimeNormalized(0, 0, 0.25);
float3 emis = _EmissionColor * (bass + 0.5 * delayed) * sin(t * 6.2831);
```

## 5. Udon Integration

AudioLink is itself a UdonSharpBehaviour. Reading to CPU requires the **`audioDataToggle`** (async GPU readbacks since 1.2.1 — cheap on PC and Quest; inspector labels it "experimental, performance hit").

```csharp
using AudioLink;

public class ReactiveDoor : UdonSharpBehaviour
{
    public AudioLink audioLink;
    public AudioLinkBand band = AudioLinkBand.Bass;

    void Update()
    {
        if (!audioLink.AudioDataIsAvailable()) return;
        Vector4 v = audioLink.GetBandAsSmooth(band, 0, true);
        float bass = AudioLink.ToGrayscale(v);
        // theme0 = audioLink.GetDataAtPixel(AudioLink.GetALPassThemeColor0());
        // hist = audioLink.GetDataAtPixelMultiline(30, 0);
    }
}
```

Verified static helpers: `GetALPassDft()`, `GetALPassAudioBass()`, …, per-pass `*Size()` getters; instance methods `AudioDataIsAvailable()`, `GetDataAtPixel(int,int)`, `GetBandAsSmooth(band, delay, smooth)`, `GetDataAtPixelMultiline(...)`. Note from 1.3.0: some old static *properties* became static *functions* (UdonSharp miscompilation workaround) — prefer function forms. Also: `ToggleAudioLink()`, `SetAudioLinkState(bool)`, `UpdateSettings()`, `UpdateThemeColors()`, `UpdateCustomStrings()`.

**Reactive gameplay:** beat-matched triggers (bass vs threshold over history), rhythm-game scoring (DFT/autocorrelator onset detection), DJ boards writing `customString1` (synced, appears on all avatars), proximity gating via `AudioLinkGetAudioSourcePosition()`, synced light shows with `AudioLinkZone` (per-zone splitter; purely local since 3.1.0).

## 6. AudioReactive Prefab Components

From `Runtime/Scripts/`:

- **AudioReactiveLight** — drives a Light's intensity/color. Fields: band, colorMode (STATIC/THEME0–3), smooth, delay (0–127 = history), affectIntensity, intensityMultiplier, hueShift. Theme color support + custom editor GUI in 3.1.0 (some fields renamed — breaking).
- **AudioReactiveObject** — scales/transforms GameObjects by band amplitude.
- **AudioReactiveSurface / SurfaceArray** — mesh surface displacement by band (the classic "wave pool" floor).
- **AudioReactiveBlendshapes** (2.0.0+) — drives SkinnedMeshRenderer blendshapes.
- **AudioLinkZone** — spatialized local audio source within a trigger zone feeds AudioLink (in-world instruments); local-only since 3.1.0.
- **AudioLinkController/ControllerHandle** — the in-world control panel.
- **AudioLinkMiniPlayer / ytDlpPlayer** — test video players feeding AudioLink.
- **ThemeColorController** — manages four global theme colors (since 2.0.0: `SetCustomThemeColors`/`GetCustomThemeColors`).
- **PlayerAPI** — publishes player data into the texture.

## 7. Performance

- **Core pipeline is cheap**: one CPU read + one CRT shader pass writing a 128×~30 texture, then global broadcast. Sampling `_AudioTexture` in shaders ≈ one texture fetch — effectively free.
- **The expensive part is CPU readback** (`audioDataToggle`): historically `ReadPixels` (stall-prone); since 1.2.1 async GPU readbacks. If you only need shader reactivity, leave it off.
- Autogain/ColorChord/DFT run inside the CRT pass — fixed modest GPU cost regardless of consumer count. Hundreds of reactive materials cost less than one extra realtime light.
- **Quest:** supported; avoid per-frame Udon data reads (async readback is the mitigations since 1.2.1).
- Prefer one shader-based reactive surface over many component-driven ones.

## 8. Common Problems

- **Avatar doesn't react in-world:** shader must be AL-enabled (`_AudioLinkAddon` on / module enabled) **and** the world must contain the AudioLink prefab. Old cginc shaders need the upgrader. Use `AudioLinkIsAvailable()` to degrade gracefully.
- **Reactivity fades with distance:** you fed it a **spatialized (3D) AudioSource**. Mark it 2D, route via AudioLinkInput, or use AudioLinkZone. Dual-mono for AVPro.
- **No audio from video players:** wire the player's direct audio output; USharpVideo adapter; disable `autoSetMediaState` if your player drives media state.
- **`GetBandAsSmooth` does nothing:** `audioDataToggle` off, or the 1.3.0 property→function rename.
- **Controller changes don't apply:** the controller syncs over network and takes over prefab values; after upgrades delete both prefabs and re-add from the package folder.

## Sources

- https://audiolink.dev/ · https://github.com/llealloo/audiolink · https://github.com/llealloo/audiolink/releases
- `AudioLink.cginc`, `AudioLink.cs`, `AudioLink.DataAPI.cs`, `AudioReactiveLight.cs` (repo master)
