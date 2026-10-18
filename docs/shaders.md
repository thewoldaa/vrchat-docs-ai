# VRChat Shader Deep-Dive: lilToon, Poiyomi, Toon Standard, and World Shaders (2025–2026)

*Based on live web research in October 2026 (GitHub repos, changelogs, official docs) plus established community knowledge. Unverified claims flagged.*

---

## 1. lilToon

**Repository:** https://github.com/lilxyzw/lilToon (MIT, ~1.6k stars).

### Version status (as of late 2025 / 2026)

- **1.7.0 (Jan 2024):** UV Tile Discard.
- **1.8.0 (Oct 2024):** SDF Face Shadow, LTCGI support, fur rim lighting.
- **1.9.0 (Apr 2025):** Shadow SDF mode extended (B channel blends SDF with normal shadows, A = shadow strength); faster inspector.
- **1.10.0 (May 2025):** **VRC Light Volumes** support; `toonstandard`/`toonstandardoutline` as VRCFallback options; ramp baking.
- **2.0 (Jul 2025):** VRC Light Volumes 2.0 (directional support).
- **2.1 (Jul–Aug 2025):** VRCLV rim light; multi-avatar compilation API.
- **2.2 (Aug 2025):** `_UdonForceSceneLighting` support.
- **2.3.x (Oct 2025, latest 2.3.2):** Adaptive Probe Volumes support, Unity 6 fixes.

Install: unitypackage, UPM git URL, or VCC/ALCOM.

### Feature list

- **Shading:** main texture with HSVG (Hue/Sat/Val/Gamma) shift; Main 2nd/3rd color layers for hair/highlights; custom shadow colors + border/blur ("toon ramp" via shadow border+blur+strength); Shadow SDF face shadows (1.8+); backlight; RimShade; **VRCLV rim light** (2.1).
- **Transparency modes:** Opaque, Cutout, Transparent, TransClipping, One-Pass Transparent (optional fade/anti-alias), Two-Pass Transparent (opaque backface + transparent front — the classic anime-hair/eyes trick), Refraction, Foliage, FakeShadow.
- **Emission:** main + 2nd emission, gradation texture, parallax, scrolling/rotation, blinking modes.
- **Matcap:** main + 2nd matcap, matcap mask, "matcap generator" mode.
- **Rim light:** color/border/blur/strength + mask; RimShade adds lit/shade-dependent rim.
- **Outline:** inverse-hull, color, width/width mask, follow-normal-map option. **Outline width must be authored at the avatar's natural scale** (scale bug, §8).
- **Distance fade** toward a color by distance; distance clipping canceller (no longer applies in mirrors since 1.9.0).
- **Vertex light:** `_VertexLightStrength`, `_LightMinLimit`/`_LightMaxLimit`/`_BeforeExposureLimit`/`_MonochromeLighting`/`_AsUnlit`/`_lilDirectionalLightStrength`/`_LightDirectionOverride` — the "lighting conditioning" suite that makes anime avatars look consistent in dark worlds.
- **AudioLink:** `_UseAudioLink`, UV modes (Radial Distance/UV/Custom, "Blink" via `_AudioLinkDefaultValue`/`_AudioLinkStart`), `_AudioLinkMask`, routing into Main2nd/Main3rd/Emission/Emission2nd (`audioLink2X` properties) and vertex displacement (`_AudioLink2Vertex`); local audio texture (`_AudioLinkAsLocal`, `_AudioLinkLocalMap`).
- **UV Tile Discard** (1.7+), ID-based masking (IDMask system), stencils, dissolving, fur (length/gravity/speed/rim), gem, and a **bake/optimize pipeline** (build-time pass stripping + animation baking into shader variants via NDMF integration, 1.8.0+).

### Setup recipes

- **Anime skin:** Opaque; shadow = warm desaturated tone, border ~0.5, blur ~0.25; clamp lighting with `_LightMinLimit` ≈ 0.05–0.1, `_LightMaxLimit` ≈ 1–1.5; SDF Face Shadow if you have an SDF face map.
- **Hair with "nori"** (glossy anime highlight band): Main 2nd with nori texture masked to hair, or Anisotropy/Matcap 2nd with hair-shaped matcap; Two-Pass Transparent for alpha hair on PC.
- **Glass:** Transparent + Refraction (or Gem variant) with cubemap; on Quest no refraction — use Transparent.
- **Emissive eyes:** Emission 1 masked to iris with gradation falloff, or Main3rd additive; "blink in Luminance" modes for pulsing.
- **Audio-reactive parts:** enable AudioLink, choose UV mode, point `_AudioLink2Emission` or `_AudioLink2Vertex` at the target layer.

### Editions / performance tiers

- **lilToon (standard)** — full set; build-time optimizer strips unused passes, so build cost scales with what you use.
- **lilToonLite** — reduced set (fewer layers, no refraction), lower variant count.
- **lilToonMulti** — many features in one shader to cut draw calls/variant memory.
- **lilToonVariant** — runtime look-switching in one shader; used for color-changing outfits.

Cost tiering: **Opaque ≪ Cutout < Transparent < Two-Pass/Refraction/Fur**. Transparency disables early-Z and adds overdraw; fur multiplies fragment cost by shell count.

## 2. Poiyomi Shaders

**Repo/docs:** https://github.com/poiyomi/PoiyomiToonShader, https://www.poiyomi.com/. Built on **ThryEditor** for its inspector and shader-locking system.

### Version status

Latest public: **versions 9.3 thru 10.0** (10.0.25 released Oct 2026; 9.3.x maintained in parallel, docs at `/9.3/…`). A **URP package (`Poi.Toon.URP`)** ships alongside for BasisVR etc.

### Pro vs Free

**Poiyomi Pro** (Patreon $10 tier): **Fur Shader, Tessellation/Geometry effects (Geometric Dissolve), TPS, SSAO, Contact Shadows, Squish**, early features. Note: **TPS is Pro-only**; the free ecosystem commonly pairs with **DPS** (separate shader/material system).

### Module system

Poiyomi 9/10 is modular: each feature (Rendering, Color Adjust, Lighting, Emission ×4, Matcap ×4, Rim ×2, Decals ×4, Details, Dissolve, Glitter, Flipbook, Iridescence, Cubemaps, Clear Coat, Parallax, Outline, Vertex Options, UV Tile Discard…) is a module; disabled modules compile out via ThryEditor locking. The `PoiMods` struct provides **16 global mask channels** (4 mask textures × RGBA).

Key modules and costs:
- **AudioLink:** 10+ reacting modules — Color Adjust, Alpha, Decals, Matcap, Rim, Outlines, Dissolve, Flipbook, Emission, Glitter, Pathing, Voronoi, Truchet, Constellation, Vertex Basics/Fun/Glitching/LookAt, plus **AL Spectrum** and an **Anim Toggle** (must be animated directly, not via section header).
- **UV Tile Discard:** 16-tile (4×4 UDIM) discard with **Vertex mode (default, faster)** vs **Pixel mode (exact, `clip()`)**; per-tile `_UDIMDiscardRowX_Y` independently animatable (interop issue with VRCFury/MA auto-discovery — enter names manually). Also **Face Discard** (selective backface discard). Known side effect: ghosting under heavy world SSAO.
- **RGB/RGBA masking:** 4 Global Mask textures → 16 channels; RGBA Color Masking for per-channel tints.
- **Rendering:** Cull/ZTest/ZWrite/Stencil/Render Queue/**Early Z** (fixes stacking transparency), fallback preview tool.
- **GrabPass:** only as a separate shader variant (grabpass can't share variants).

### Common recipes

- Avatar skin: Rendering Preset Opaque + Lighting Toon preset + Color Adjust tint.
- Clothing color swap: RGBA Color Masking or Global Mask channels, animated via material swaps.
- Sound-reactive: AudioLink toggle + enable AL on Emission/Vertex modules; use the Anim Toggle in toggles.
- Clothing removal: UV Tile Discard vertex mode on a dedicated UV channel (UDIM wardrobe tools).

### Known performance costs

Transparency presets, GrabPass variant (very expensive, breaks VR stereo), pixel-mode UDIM, particle-heavy Flipbook/Truchet/Voronoi, 4× matcaps + 4× emission simultaneously; materials can't share a shader unless ThryEditor locking is used (variant explosion). 10.x targets DX11 primarily; Vulkan experimental.

## 3. Comparison: lilToon vs Poiyomi

| Aspect | lilToon (2.3.x) | Poiyomi (9.3 / 10.x) |
|---|---|---|
| License | MIT, fully free | Free core MIT; Pro paywalled ($10/mo) |
| Inspector | lilInspector (fast since 1.9.0) | ThryEditor (heavy, search, presets) |
| Toon shading | Anime-first: Main2nd/3rd, SDF face shadows, RimShade | Toon presets + ramps; strong toon AND realistic |
| Transparency | Opaque/Cutout/Transparent/Two-Pass/Refraction/Foliage/FakeShadow | Presets; GrabPass as separate variant |
| AudioLink | Rich: layers, emission, vertex displacement, masks | Very rich: 10+ modules incl. Spectrum, Voronoi, Truchet |
| UV Tile Discard | Yes (1.7+) | Yes, vertex/pixel modes + Face Discard |
| TPS/DPS | Not built-in | TPS in Pro; DPS separate |
| LTCGI / VRC Light Volumes | Yes / Yes (1.10+/2.x) | Yes / Yes |
| Global masking | IDMask + stencils | globalMask 16 channels |
| Fur | Free | Pro |
| Adoption | **Dominant in JP** (Booth, anime avatars) | **Dominant in EN/West**, feature-maximalist avatars |
| Quest | Doesn't run; falls back via VRCFallback (incl. `toonstandard`) | Same |

Heuristic 2025-2026: JP anime avatars → lilToon; EN "juiced" avatars → Poiyomi; Quest-crossing → either, but design the fallback.

## 4. VRChat Toon Standard (official mobile shader)

`VRChat/Mobile/Toon Standard`, shipped SDK **3.8.1 (May 2025)**; extended 3.10.3 with **Color Mask** (RGB mask driving per-channel Tint, optionally emissive). Exists because Quest only permits SDK-bundled shaders.

Feature set: custom **Shadow Ramp** textures (Custom mode) + built-in ramps; Detail/Emission maps from UV0 or UV1; detail modes AlphaBlended/Additive/Multiply/MultiplyX2; **maskable Hue Shift** (Albedo/Emission/Detail); normal + tilable maskable detail normals; **Matcap** (Additive/Multiplicative) with mask; **Rim lighting** with Albedo Tint + Environmental option; simplified isotropic GGX specular; configurable backface culling (double-sided allowed — first in the mobile family); vertex colors toggle; "Limit Brightness" clamping; `Min Brightness` (~0.05 recommended to match lilToon); GPU Instancing recommended.

**Mask packing:** each mask slot independently selects which RGBA channel it reads (click the `>` arrow) — pack multiple grayscale masks into one texture, point each feature at its channel. Standard Lite reference packing (Toon Standard defaults match): *Texture 1: Albedo RGB + Detail Mask A; Texture 2: Metallic R, Occlusion G, Smoothness A*. Author the packed texture yourself (lilEditorToolbox can pack).

**Ramp interop trick:** right-click a lilToon material → **lilToon → Generate Ramp** produces a ramp approximating that material's shadow look — drop into Toon Standard's Custom shading slot. Without a ramp, shadows stay black (reads badly for anime).

**Limitations vs PC shaders:**
- **No transparency** (opaque only) — the biggest gap.
- No realtime shadows received.
- **Outline variant is PC-only** (auto-falls back to non-outline on mobile); prefer plain `toonstandard` as fallback (outline mask is lost via fallback).
- No refraction, fur, layering, AudioLink, UV tile discard, stencil/ID masks; limited shadow color control (Shadow Boost/Tint).
- When to use: all Quest-targeting avatars; PC avatars wanting cheap-good toon + graceful degradation for shader-blocked users.

## 5. World shaders

### Built-ins
`VRChat/Mobile/Lightmapped` (lightmap-only diffuse), `Skybox`, `Supersampled UI`, Toon Lit, Matcap Lit, Particles/Additive/Multiply on Quest; `VRChat/Standard` family + Unity Standard on PC.

### z3y shaders / Graphlit
[z3y/shaders](https://github.com/z3y/shaders) (archived → [z3y/Graphlit](https://github.com/z3y/Graphlit)) — Standard/Surface replacement for BIRP: Filament-based shading, Bakery features, specular occlusion, **LTCGI + AreaLit**, box projection on Quest, alpha-to-coverage, geometric specular AA, texture packing config. **Graphlit** is the successor: node shader editor for Toon and PBR (BIRP+URP), clustered BIRP lighting, VRC Light Volumes, PCF point/spot shadows, inverse-square falloff, coat, thin-film iridescence, live preview, variant reduction. VPM listing: `https://z3y.github.io/vpm-package-listing`.

### LTCGI (image-based lighting for worlds)

https://ltcgi.dev/ (PiMaker) — plug-and-play **realtime area light** system using linearly transformed cosine: screens/emitters light and reflect on receivers, including **realtime video lighting**. Setup:
1. Prereq: project already builds/uploads; back up. Install via VCC from PiMaker's VPM repo.
2. **Shaders:** receivers need compatible shader — ORL Shader Family, Silent's Filamented, Mochie, Hekky, z3y, Poiyomi, lilToon (1.8+), or included test shaders; some need an LTCGI toggle enabled.
3. **Controller:** drag `LTCGI_Controller.prefab` into scene root — exactly one per scene, editor-only. "Force Update" when materials desync; bake shadowmap + lightmap.
4. **Screens:** `LTCGI_Screen` components on quads/TVs; configure Affected Renderers (Exclude/Only Listed/Distance), Diffuse vs Specular.
5. **Video:** auto-adapters for USharpVideo and ProTV 3+ (native); AVPro manual via `LTCGI_AvProBlit_Material` + `LTCGI_BlitCRT`; one realtime video max (additional **static** textures allowed).
6. **Performance:** cost scales by *screens per receiving object* — avoid transparent receivers (double cost); merge multi-panel TVs into one Screen; mark non-moving screens non-Dynamic. **Quest:** fast sampling forced, **only one screen supported** — visuals differ from desktop.
7. Since v1.7.0, supports **VRCLightVolumes**.

### Other
- **AreaLit** (z3y): area-light baking pairing with LTCGI-compatible shaders.
- **AudioLink world shaders:** dance-floor/stage lighting sampling `_AudioTexture` (Poiyomi on world objects, z3y, ORL, VRSL).
- **VRSL** (VR Stage Lighting, AcChosen): the standard for club lighting fixtures.
- Gradient skybox shaders — staple of stylized worlds; `VRChat/Mobile/Skybox` for Quest.

## 6. Shader development basics: what breaks in VR

- **GrabPass breaks in single-pass stereo instanced rendering** — per-eye-wrong/misaligned, plus a full-frame copy (very slow). Never ship grabpass on something users see in VR.
- **Screen-space effects** misbehave per-eye (UV math assuming a 2D viewport). UDIM ghosting under world SSAO is a real example of screen-space interference.
- **Single-pass stereo gotchas:** `UNITY_VERTEX_OUTPUT_STEREO` / `UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO` / `UNITY_SETUP_INSTANCE_ID` must be correct, or effects render in one eye only. The classic "works in game view, broken in VR."
- **Safe rules:** avoid GrabPass; no per-pixel dependent reads on huge textures; standard transparency queues; don't rely on renderer order across materials; keep vertex displacement in the vertex stage (VR reprojection hates view-dependent per-pixel displacement); test in-game; respect `_MainTex`-style standard property names so fallbacks work; support GPU instancing; keep variant counts low (each keyword combo is a compile + memory cost on Quest).
- **Transparency sorting:** Unity sorts back-to-front per material queue — no per-pixel sorting. Multiple transparent meshes on one avatar will pop. Mitigations: cutout instead of blend, One-Pass/Two-Pass (lilToon), Early Z (Poiyomi), per-material render queue, minimize nested transparent shells.

## 7. Custom shader properties + animation conventions

VRChat animates materials by animating material float/color/texture properties on the MeshRenderer — properties must exist in the shader's `Properties` block.

- **lilToon:** everything animatable; stable names (e.g., `_MainTexHSVG` (Vector4 H/S/V/Gamma shift), `_MainColor`, `_ShadowColor`/`_ShadowBorder`/`_ShadowBlur`, `_EmissionColor`, `_UseAudioLink`, `_AudioLink2Emission`, `_OutlineColor`/`_OutlineWidth`). NDMF-aware optimization bakes animated properties into variants (1.8+), so animating is safe.
- **Poiyomi:** all module sliders are properties; UDIM tiles = `_UDIMDiscardRowX_Y`; animate the named property, not section headers.
- **Texture swaps** animate `_MainTex` (ObjectReferenceKeyframe on the material's texture slot).
- General rule: any float/range/color/texture in Properties can go into an AnimationClip.

## 8. Common shader bugs

- **Pink/magenta:** shader failed to compile or absent (URP material in BIRP project, corrupted import, blocked shader). In VRChat it also means shader-blocked users see your fallback.
- **Outline thickness wrong:** width is a world-space constant but authored relative to mesh scale — scaled avatars get uniformly wrong thickness; non-uniform scale varies it per axis. Fix: author width at final avatar scale, avoid non-uniform scaling of outline meshes.
- **Transparency on Quest:** community shaders forbidden — everything falls back per `VRCFallback` tag; the fallback for Toon+Transparent is **Transparent Unlit** (alpha-blended hair turns flat-glowy). Design Quest hair with cutout geometry or opaque planes.
- **Fallback tags:** declare `Tags{"Queue"="Geometry" "VRCFallback"="ToonCutout"}` — combinable: Unlit/Toon/VertexLit/Matcap × Transparent/Cutout/Fade (+DoubleSided for Toon), plus `Hidden` (hide mesh — raymarch effects) and non-combinable `toonstandard`/`toonstandardoutline` (copy same-named properties/keywords). Standard params (`_MainTex`, `_Color`, `_Cutoff`, `_BumpMap`, `_EmissionColor`) are copied to the fallback. Unrecognized tag → Standard fallback. lilToon ≥1.10 emits `toonstandard` fallbacks itself.
- **One-eye rendering** (missing stereo macros), **AudioLink not reacting** (shader AL only works when the world provides AudioLink; Poiyomi needs the Anim Toggle), **UDIM discard + MA/VRCFury property discovery failures** round out the most-reported 2025 issues.

## Sources

- lilToon: https://github.com/lilxyzw/lilToon · CHANGELOG.md · lilMaterialProperties.cs
- Poiyomi: https://poiyomi.com/ · https://www.poiyomi.com/9.3/intro · /9.3/audio-link · /9.3/rendering · /special-fx/uv-tile-discard · /9.3/modular-shader-system
- VRChat: https://creators.vrchat.com/avatars/shader-fallback-system/ · https://creators.vrchat.com/platforms/android/quest-content-limitations · https://creators.vrchat.com/releases/release-3-8-1/
- LTCGI: https://ltcgi.dev/ (+ Installation/Controller/Realtime Video/Performance pages)
- z3y: https://github.com/z3y/shaders (archived) · https://github.com/z3y/Graphlit
- Toon Standard JP analysis: https://note.com/mkb0326/n/n0f08e3461a1c · AudioLink: https://audiolink.dev/

*Caveats: Pro-only Poiyomi features summarized from the public README only (private repo). UdonPalette status unverified. Some lilToon wiki claims corroborated via changelog/properties source.*
