# VRChat Custom Shader Development Guide (2025–2026)

*Writing custom shaders for VRChat from scratch: stereo-safe toon shading, fallbacks, variant control, Udon integration. Based on official docs (creators.vrchat.com shader pages, Unity 2022.3 Built-in RP) + community-verified patterns. Version-specific claims flagged where they may drift.*

---

## 0. Ground rules

- VRChat runs **Unity Built-in Render Pipeline** (BiRP), **single-pass stereo instanced** rendering in VR. URP/HDRP shaders do not work.
- Anything you write must degrade gracefully: Quest cannot run your shader at all (falls back per `VRCFallback`), and shader-blocking users see the fallback too.
- Unity version is pinned by VRChat (2022.3.22f1 at research time — verify). Shader features must match that Unity's compiler.

## 1. A stereo-safe toon shader from scratch

### 1.1 ShaderLab skeleton

```hlsl
Shader "MyName/VRToon"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
        _RampTex ("Toon Ramp", 2D) = "white" {}      // grayscale ramp, sampled by NdotL
        _ShadowTint ("Shadow Tint", Color) = (0.7,0.7,0.85,1)
        _OutlineWidth ("Outline Width", Range(0,0.01)) = 0.002
        _OutlineColor ("Outline Color", Color) = (0.1,0.1,0.1,1)
        [Enum(UnityEngine.Rendering.CullMode)] _Cull ("Cull", Float) = 2
        // Standard names VRChat's fallback system can copy:
        _Cutoff ("Alpha Cutoff", Range(0,1)) = 0.5
        _EmissionColor ("Emission", Color) = (0,0,0,1)
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Geometry" "VRCFallback"="Toon" }
        Pass { ... front faces with toon lighting ... }
        Pass { ... inverted-hull outline ... }
    }
    FallBack "Diffuse"
}
```

Key `Tags` decisions:
- `VRCFallback="Toon"` tells VRChat which mobile-safe material to synthesize for Quest/shader-blocked viewers. Combinable base tags: `Unlit`/`Toon`/`VertexLit`/`Matcap` × `Transparent`/`Cutout`/`Fade` (+ `DoubleSided` for Toon). Non-combinable specials: `toonstandard`/`toonstandardoutline` (copies same-named properties to VRChat's Toon Standard), `Hidden` (hide the mesh — for raymarch-only objects).
- Standard property names (`_MainTex`, `_Color`, `_Cutoff`, `_EmissionColor`, `_BumpMap`) get copied to the fallback material — use them wherever possible so your fallback actually looks like your shader.

### 1.2 Stereo instancing (the part that breaks in VR)

Every custom vertex/fragment shader that will ever be seen in VR needs the stereo macros, or the effect renders in one eye only / at wrong offsets:

```hlsl
Pass
{
    CGPROGRAM
    #pragma vertex vert
    #pragma fragment frag
    #pragma multi_compile_instancing

    #include "UnityCG.cginc"

    struct appdata
    {
        float4 vertex : POSITION;
        float3 normal : NORMAL;
        float2 uv : TEXCOORD0;
        UNITY_VERTEX_INPUT_INSTANCE_ID
    };

    struct v2f
    {
        float4 pos : SV_POSITION;
        float2 uv : TEXCOORD0;
        float3 worldNormal : TEXCOORD1;
        float3 worldPos : TEXCOORD2;
        UNITY_VERTEX_INPUT_INSTANCE_ID
        UNITY_VERTEX_OUTPUT_STEREO
    };

    v2f vert (appdata v)
    {
        v2f o;
        UNITY_SETUP_INSTANCE_ID(v);
        UNITY_TRANSFER_INSTANCE_ID(v, o);
        UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
        o.pos = UnityObjectToClipPos(v.vertex);
        o.uv = TRANSFORM_TEX(v.uv, _MainTex);
        o.worldNormal = UnityObjectToWorldNormal(v.normal);
        o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
        return o;
    }

    fixed4 frag (v2f i) : SV_Target
    {
        UNITY_SETUP_INSTANCE_ID(i);
        // ... lighting ...
    }
    ENDCG
}
```

The trio that must all be present: `UNITY_SETUP_INSTANCE_ID` in both stages, `UNITY_TRANSFER_INSTANCE_ID` in the vertex stage, `UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO` on the interpolator. Miss any one → classic "works in Game view, broken/one-eyed in VR."

### 1.3 Toon lighting fragment

```hlsl
sampler2D _MainTex, _RampTex;
fixed4 _Color, _ShadowTint, _EmissionColor;

fixed4 frag (v2f i) : SV_Target
{
    fixed4 albedo = tex2D(_MainTex, i.uv) * _Color;
    float3 N = normalize(i.worldNormal);
    float3 L = normalize(_WorldSpaceLightPos0.xyz);   // directional light dir
    // Remap NdotL to 0..1 then sample the ramp: the toon quantizer.
    float ndl = saturate(dot(N, L) * 0.5 + 0.5);
    fixed3 ramp = tex2D(_RampTex, float2(ndl, 0.5)).rgb;
    fixed3 lit = albedo.rgb * lerp(_ShadowTint.rgb, 1.0, ramp);
    lit += _EmissionColor.rgb;
    // Ambient so shadow side isn't pure black in dark worlds:
    lit += albedo.rgb * UNITY_LIGHTMODEL_AMBIENT.rgb;
    return fixed4(lit, albedo.a);
}
```

Ramp texture design: 2–4 hard bands for cel, wide smooth gradient for soft toon. lilToon's "Generate Ramp" (right-click a lilToon material) produces compatible ramps you can borrow to match looks.

### 1.4 Inverted-hull outline pass

```hlsl
Pass
{
    Name "Outline"
    Cull Front                    // render backfaces only
    CGPROGRAM
    #pragma vertex vert
    #pragma fragment frag
    #pragma multi_compile_instancing
    #include "UnityCG.cginc"
    float _OutlineWidth; fixed4 _OutlineColor;

    struct v2fo { float4 pos : SV_POSITION; UNITY_VERTEX_OUTPUT_STEREO };
    v2fo vert (appdata_base v)
    {
        v2fo o;
        UNITY_SETUP_INSTANCE_ID(v);
        UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
        // World-space constant width: displace along normal, compensate camera distance:
        float3 wpos = mul(unity_ObjectToWorld, v.vertex).xyz;
        float3 wnormal = UnityObjectToWorldNormal(v.normal);
        float dist = distance(_WorldSpaceCameraPos.xyz, wpos);
        wpos += wnormal * _OutlineWidth * dist * 0.1;   // screen-constant-ish thickness
        o.pos = mul(UNITY_MATRIX_VP, float4(wpos, 1));
        return o;
    }
    fixed4 frag (v2fo i) : SV_Target { return _OutlineColor; }
    ENDCG
}
```

Rules: author `_OutlineWidth` at the avatar's final scale (scaled avatars get wrong thickness — the classic bug); non-uniform scale varies thickness per axis; normal-mapped outline needs the sampled normal's perturbation or it follows raw vertex normals.

## 2. ORL Shader Framework / ORL Shader Creator

orels1's ORL Shader Framework (github.com/orels1/ShaderSet, docs orl.readthedocs.io) is the main **node/text hybrid shader authoring framework for VRChat BiRP**: a library of modular HLSL includes (ORL Base, Toon shading stack, fallback-aware, AudioLink/LTCGI/VRC Light Volumes hooks built in) plus the **ORL Shader Creator** (node graph that generates ORL-format HLSL you can hand-edit afterwards). It handles the stereo/fallback boilerplate of §1 automatically and is the recommended starting point over raw ShaderLab. Status: actively maintained as of 2025; verify current version in its VPM listing. ("shader.tips" by Raif fell out of maintenance several years ago — do not build on it.)

## 3. Animatable properties & toggle conventions

- Anything in the `Properties` block can be animated by a VRChat animation clip (material float/color/texture on the MeshRenderer). VRChat **animates material properties directly** — MaterialPropertyBlocks are irrelevant for avatar toggles (and break batching guarantees differently).
- `[Toggle(_MYKEYWORD)]` drawer: animating the float toggles the shader keyword. Keyword changes create variant switches at runtime — fine for toggles, terrible for continuous animation.
- Continuous effects (hue shift, emission pulse): animate plain floats/colors, not keywords. lilToon's `_MainTexHSVG` (Vector4 H/S/V/Gamma multiplier) is the standard hue-shift lever; Poiyomi exposes its own documented set. UDIM discard tiles (`_UDIMDiscardRowX_Y` in Poiyomi) are per-tile float toggles — individually animatable.
- Texture swaps animate `_MainTex` (ObjectReferenceKeyframe) — the material-swap pattern.
- Keep property names stable; shader updates that rename animated properties break every toggle silently.

## 4. GrabPass alternatives (stereo-safe effects)

`GrabPass{}` breaks in single-pass instanced stereo (per-eye-wrong texture, plus a full-frame copy). Stereo-safe alternatives:

1. **Refraction via view-space normal offset:** don't sample the actual screen behind; fake it — sample the albedo/scene approximation and shift UVs by view-space normal:
   ```hlsl
   float2 refractUV = i.uv + N_view.xy * _RefractionStrength;
   fixed3 col = tex2D(_BackgroundApprox, refractUV);  // or distort your own albedo
   ```
   Good enough for glass/gems/eyes in practice.
2. **`_CameraDepthTexture`:** available in VRChat worlds (declare `sampler2D_float _CameraDepthTexture;` and sample with `Linear01Depth`). Works per-eye correctly because it's a per-eye texture. Enables depth-fade (soft intersection with floors), fog by depth, edge detection. For avatars, depth-texture access is available only if the world provides one — gate with `#ifdef` or accept a plain fallback.
3. **Additive/fresnel instead of transparency:** many "glass" looks survive as additive rim + specular with alpha cutout — zero sorting problems.

## 5. VRCFallback deep dive

When a Quest user or shader-blocked PC user sees your avatar, VRChat synthesizes a fallback material per material:

- Tags choose the base: `VRCFallback="Toon"` etc. Unrecognized → Standard.
- Properties copied to the fallback: `_MainTex`, `_Color`, `_Cutoff`, `_EmissionColor`, `_BumpMap` and other standard slots.
- `toonstandard` / `toonstandardoutline` (non-combinable): copies properties/keywords to VRChat's mobile Toon Standard — the highest-fidelity fallback since SDK 3.8.1 (lilToon ≥1.10 emits these automatically).
- Known traps: Toon+Transparent → **Transparent Unlit** (flat glow — design cutout hair instead); the Outline variant of Toon Standard doesn't exist on mobile; fallback preview exists in Poiyomi (Tools → Avatar → Fallback Shaders) — always check it before upload.

Design rule: give every material an intentional fallback. A "hidden" (`Hidden` tag) fallback for effect-only meshes avoids weird rectangles on Quest.

## 6. Variant explosion control

- `shader_feature` (stripped unused at build per material) vs `multi_compile` (all variants compiled). Prefer `shader_feature_local` (local keywords don't eat the global keyword limit — 64 global / 64 local in Unity 2022).
- Every `[Toggle(KEY)]` × every other toggle = multiplied variants. Keep toggles orthogonal and few; continuous effects should be uniforms, never keywords.
- Variant count hits: compile time in editor, build size, runtime keyword-switch cost, and on Quest, memory. Rule of thumb: <100 variants per shader is comfortable; thousands is a bug.
- `LOD` / `LODCrossFade` (dither crossfade): useful for world shaders; avatars don't use LODs.

## 7. Common compile errors & meaning

| Symptom | Meaning |
|---|---|
| Renders in one eye / offset in VR | Missing `UNITY_VERTEX_OUTPUT_STEREO` / `UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO` / instance ID macros (§1.2) |
| Animation on a property does nothing | Property not in the `Properties` block (animating it is silently dropped), or it's a section header in ThryEditor (Poiyomi) rather than a real property |
| Texture looks washed out / too dark | sRGB mismatch: color textures need `sRGB=on`, data textures (masks, ramps-as-data, normal maps) `sRGB=off`. Sampling a linear-marked texture with lighting math doubles the error |
| Pink after Unity/SDK update | Shader compiler error — check the Console; often a deprecated API from an older Unity idiom |
| Outline scales wrong | `_OutlineWidth` authored at a different mesh scale than runtime (§1.4) |
| Works in editor, wrong in VRChat build | Strip keywords: `shader_feature` variants stripped per-material at build; if your C# sets a keyword at runtime that no material used at edit time, the variant doesn't exist — declare it in the shader's `#pragma` and ensure at least one material enables it, or use `multi_compile` |
| Z-fighting with transparent outlines | Outline pass renders in Geometry queue under transparents — use `Queue`="Geometry+1" for the outline pass or per-layer offsets |

## 8. Udon ↔ shader interaction

**Globals from Udon** — worlds can push data into *all* shaders (including every avatar's) via `VRCShader` (`VRC.SDK3.Rendering`):

```csharp
using VRC.SDK3.Rendering;   // VRCShader
using UnityEngine;

public class DayNightController : UdonSharpBehaviour
{
    [UdonSynced] private float _timeOfDay;   // 0..1

    public override void OnDeserialization() { Apply(); }
    public override void Interact()          // owner changes time
    {
        Networking.SetOwner(Networking.LocalPlayer, gameObject);
        _timeOfDay = (_timeOfDay + 0.25f) % 1f;
        RequestSerialization();
        Apply();
    }

    private void Apply()
    {
        // Drive a global vector every shader can sample:
        Vector4 sunDir = Quaternion.Euler(_timeOfDay * 360f - 90f, 30f, 0) * Vector3.forward;
        VRCShader.SetGlobalVector(Shader.PropertyToID("_MySunDir"), sunDir);
        VRCShader.SetGlobalFloat(Shader.PropertyToID("_MySunIntensity"),
            Mathf.Clamp01(Mathf.Sin(_timeOfDay * 2f * Mathf.PI) * 2f));
    }
}
```

Shader side:
```hlsl
float4 _MySunDir;    // no Properties entry needed for globals
float _MySunIntensity;
// float ndl = saturate(dot(N, normalize(_MySunDir.xyz)) * 0.5 + 0.5);
```

This is the same mechanism AudioLink uses (`_AudioTexture`). Available setters include `SetGlobalFloat/Int/Vector/Matrix/Texture` (verify exact surface in the SDK's Class Exposure Tree — it drifts).

**Per-material from Udon:** `Material.SetFloat/SetColor/SetVector` on renderer materials is exposed to Udon (worlds). Caveats: it mutates the shared material unless you instantiate it first (`new Material(Shader)` or `Material[] mats = renderer.materials` — the `materials` property getter instantiates copies); combined with animation, Udon writes fight animation curves — pick one driver per property.

**Avatar side:** avatars cannot call `VRCShader.SetGlobal*` (world-only) — avatar-shader reactivity must come from world-provided globals (AudioLink) or the avatar's own animated material properties.

## 9. Checklist before shipping a custom shader

- [ ] Stereo macros present in every pass (test in VR, both eyes)
- [ ] `VRCFallback` tag set per SubShader; standard property names used so the fallback inherits
- [ ] Fallback preview checked (Poiyomi tool or upload-and-check on Quest)
- [ ] `shader_feature_local` used; variant count sane
- [ ] No GrabPass; transparency either cutout or deliberately queued
- [ ] Animatable properties documented for toggle authors; names stable
- [ ] GPU Instancing pragma included (worlds/mobile expect it)
- [ ] Tested with: shader-blocked client (safety), Quest, mirror reflection (your shader renders inside mirrors!)

## Sources

- https://creators.vrchat.com/avatars/shader-fallback-system/ · https://creators.vrchat.com/worlds/ (VRCShader/graphics docs) · SDK Class Exposure Tree
- ORL: https://github.com/orels1/ShaderSet · orl.readthedocs.io
- lilToon ramp generation & fallback emission: https://github.com/lilxyzw/lilToon
- Poiyomi fallback preview & module docs: https://www.poiyomi.com/
- Unity 2022.3 Built-in RP shader docs: https://docs.unity3d.com/2022.3/Documentation/Manual/Shaders.html
