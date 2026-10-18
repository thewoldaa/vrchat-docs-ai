# VRChat SDK Build Pipeline Internals (2025–2026)

*What happens between clicking "Build & Publish" and content going live. Exact SDK internals partly inferred from stack traces in public forum posts and community reverse engineering — VRChat does not publish the full builder source.*

---

## 1. What Happens on Build & Publish

The SDK (packages `com.vrchat.base` + `com.vrchat.avatars`/`com.vrchat.worlds`) runs a staged pipeline (`VRCSdkControlPanelAvatarBuilder.cs` / `VRCSdkControlPanelWorldBuilder.cs`):

1. **Pre-build validation** — the builder enumerates "SDK Builder Validations" (checklist UI): multiple avatars in scene, missing descriptor, unsupported mobile shaders, mesh Read/Write, platform component limits, expression parameter overflow. Any blocking validation aborts with `BuilderException: Avatar validation failed`. Console output is notoriously unhelpful — the *real* reason is in the SDK panel's Validation list, not the console ([ask.vrchat.com example](https://ask.vrchat.com/t/avatar-validation-failed/25422)).
2. **Scene/object preparation** — SDK clones the avatar/world root, strips EditorOnly objects, applies build-time components (Dynamic Bone conversion, descriptor finalization, world spawner setup). Third-party transformations run here via **NDMF** (§2).
3. **Asset collection & bundle build** — `BuildPipeline.BuildAssetBundles` packs the cloned content into an AssetBundle for the build target (StandaloneWindows64/Android/iOS) with LZMA compression. Build statistics go to the Editor log — for Android, search the log for "statistics" for the pre-compressed size.
4. **Size checks** — SDK measures compressed + uncompressed sizes and refuses upload above platform limits. **Build & Test does not enforce size limits** — local tests can succeed while publishing is rejected.
5. **Thumbnail generation** — isolated temp scene capture (avatars) or panorama (worlds).
6. **API upload** — calls the VRChat web API (async `VRC.SDKBase.Editor.Api.VRCAvatar/VRCWorld` with `CancellationToken`): creates/updates the blueprint record, uploads the bundle as a file version (delta/multipart for large files), links the file version to the blueprint. Errors surface as `Upload Error` events via the Public SDK API.
7. **Server-side post-processing** — the backend rescans the bundle: extracts component stats, recalculates performance rank, can modify the bundle (stripping scripts historically; scanning serialized meshes/textures server-side since ~2024). Since the **Aug 2025** dev update, **the client uses server-determined performance stats** — better VRAM detection for material swaps, ranks can change post-upload ([ask.vrchat.com](https://ask.vrchat.com/t/new-avatar-performance-rank/46308)).

**Error handling:** each stage can throw; the builder wraps the async chain and calls `HandleBuildError`, logging `[Always] Failed to build avatar!`. The Public SDK API exposes `Build Start/End`, `Upload Success/Error` events for tool authors.

## 2. NDMF (Non-Destructive Modular Framework)

[NDMF](https://github.com/bdunderscore/ndmf) ("なでもふ", by bd_) is the de-facto foundation under every modern avatar tool (Modular Avatar, AAO, etc.). The SDK's avatar build invokes NDMF agents against the avatar root before final serialization.

**Architecture:** the unit of extension is a **Plugin** (`Plugin<T>` with `QualifiedName`/`DisplayName`), declaring **Passes** — callbacks at build points — organized into **Sequences** within **Build Phases**. Ordering uses `BeforePlugin`/`AfterPlugin`/`BeforePass`/`AfterPass` constraints — cross-tool conflicts resolved declaratively.

**Phases** (in execution order):
- **FirstChance** — before platform init; plugins replacing the avatar wholesale. EditorOnly objects *not yet* removed.
- **PlatformInit** — platform backend init (`ExtractCommonAvatarInfo` runs just before).
- **Resolving** — very early: rehydrating string-serialized references, cloning animation controllers. Contains built-in `RemoveEditorOnlyPass`.
- **Generating** — generating components later plugins consume.
- **Transforming** — general-purpose avatar transformation (most of Modular Avatar).
- **Optimizing** — late, semantics-preserving (mesh merging, texture atlasing in AAO).
- **PlatformFinish** — platform cleanup and **validation** (e.g., VRChat expression-parameter bit limit).

**Other facilities:** `BuildContext` (AvatarRootObject/Path, `AssetSaver` for generated assets with auto-cleanup, `GetState` for cross-pass state), play-mode support (transforms apply on entering Play), animation-path retargeting after objects move, and `ObjectRegistry.RegisterReplacedObject` so downstream plugins/error reports map cloned objects back to originals.

**Error reporting** (ErrorReport.cs): plugins call `ErrorReport.ReportError(Localizer, ErrorSeverity, key, args...)` with **localized** messages, attaching Unity object references so the NDMF error window selects the offending object. Severity: Informational → Error → **InternalError**; a throwing pass produces a `StackTraceError`. `ErrorReport.WithContextObject` associates errors with the processed object; `ObjectRegistry` indirection lets errors reference the *original* prefab object even though the build operates on a clone. Best practices: split features across phases, register replaced objects, attach related objects to errors.

NDMF abstracts the platform: `PlatformInit`/`PlatformFinish` and platform backends keep it VRChat-agnostic; the VRChat backend registers parameter-limit validation at PlatformFinish.

## 3. Asset Bundle Format & Size

**What's in the bundle:** the cloned scene serialized as Unity scene objects, all meshes, textures re-encoded to GPU formats (DXT1/DXT5/BC7 PC; ASTC/ETC2 Android/iOS), audio (Vorbis/PCM per import settings), animation clips, compiled shader variants, materials, descriptor data. **Scripts are not shipped** — VRCSDK components are stubbed; gameplay is Udon programs.

**Compression:** the bundle is LZMA-compressed; textures individually compressed to GPU formats beforehand. **Crunch** is a separate pre-bundle step shrinking **download size only** — crunch textures decompress to DXT/BC7/ETC2 after download (CPU hitch on first load; **does not reduce VRAM** — "Crunch compression does not help with in-memory size").

**Download vs uncompressed:** download = compressed bundle; uncompressed = after LZMA decompression, before GPU texture conversion. Limits (`VRC.ValidationHelpers.GetAssetBundleSizeLimit`):

| Platform | Download | Uncompressed |
|---|---|---|
| PC avatar | 200 MB | 500 MB |
| Android avatar | 10 MB | 40 MB |
| Android world | 100 MB | — (PC worlds effectively unlimited) |

Historical: PC limits (previously 500 MB/1.2 GB) tightened mid-2024, enforced from SDK 3.7.0; Android uncompressed enforced 3.5.2. Rule of thumb for Android: 5–8 MB total.

## 4. Validation Rules (What the SDK Blocks)

- **Mesh Read/Write disabled** — triangle stat can't be computed server-side; SDK shows `2147483647` triangles (Int32.MaxValue) + "Mesh Read/Write Disabled", forces **Very Poor**, and *blocks upload until fixed*. Known UX bug: warning doesn't appear for `.mesh` assets (only FBX).
- **Platform component limits** — Android/iOS enforce mobile tables (particles, lights, cameras, shader whitelist — only `VRChat/Mobile/*` on Quest). Hard-limit excess fails validation; soft limits cause runtime rank downgrade/stripping.
- **Expression parameter overflow** — 256-bit synced budget is build-blocking; under NDMF checked at PlatformFinish. Modular Avatar shows a "parameter bit usage" bar in advance.
- **Blueprint ID issues** — uploading with someone else's ID fails ("Attempted to load the data for a world we do not own"). Prefab-embedded Pipeline Managers triggered bugs in 3.6.1.
- **Size limits** (§3) — post-build check, blocks publish.
- **Missing/invalid descriptor, multiple uploadable objects, non-humanoid misconfiguration** — pre-build Validation panel.

**Reading errors:** console output is deliberately generic; the actionable list is the SDK Control Panel's validation area (red octagons, clickable "select object" links) + the NDMF Error Report window. Full diagnostics: `%LocalAppData%\Unity\Editor\Editor.log`.

## 5. Performance Calculation

A **static analysis** of the serialized avatar — no shaders/animators/runtime behavior considered. All GameObjects/components count, **including disabled ones**.

**Stats:** Triangles, Bounds Size, Texture Memory, Skinned/Basic Meshes, Material Slots, PhysBones (components, affected transforms, colliders, collision check count), Contacts, Constraint count & depth, Animators, Bones, Lights, Particle Systems, Total Particles Active, Mesh Particle Active Polys, Particle/Trail/Line renderers, Raycasts, Cloths, Audio Sources. Exceeding a "bolded" stat fully blocks under the viewer's Minimum Displayed Rank; non-bolded (except Bounds) = partial blocking (components stripped at load).

**Texture memory formula:** per texture ≈ `width × height × bytes-per-pixel(format) × 1.33 (mip chain)` at the *runtime* GPU format. 2048² BC7 ≈ 5.5 MB; 4096² RGBA32 ≈ 85 MB. Crunch contributes nothing. Since Aug 2025, server processing detects **material-swap animations** and counts those textures too — the in-client rank is more accurate than the local SDK estimate.

**PhysBone counting:** "Affected Transforms" = transforms driven by PB components; "Collision Check Count" sums per collider how many transforms it can touch — one transform affected by multiple colliders counts multiple times.

**Server-side recalculation:** the backend rescans and computes authoritative stats; the client displays these (Aug 2025). Known mismatch bugs exist (server-side material-slot counting over-counting on mesh-less renderers) — a locally Medium avatar can show Very Poor in-game until patched.

## 6. Blueprint / Pipeline Manager

The `VRCPipelineManager` component (auto-added alongside the Avatar Descriptor / Scene Descriptor) stores:
- **Unity version** that last built the content (forward-compat warnings).
- **Blueprint ID** — prefixed UUID: `avtr_…` / `wrld_…`. Blank until first upload; server assigns on first publish.

**Identity & versioning:** one blueprint ID holds **platform variants** (PC + Android + iOS uploads to the same ID = one cross-platform avatar/world; the client downloads the matching bundle per platform; missing variants → impostor/fallback). Every publish creates a **new file version** server-side; VRChat's `/api/1/file` pipeline keeps previous versions — content can be **rolled back** by re-linking an older version (web tooling; SDK UI exposes only the current version). Server-side deltas store unchanged assets once.

**Rollback/deletion in practice:** the SDK's **Content Manager** tab lists uploads with Copy ID / Delete. `Detach` clears the local ID (next upload = new separate content); pasting a known ID + `Attach` re-binds (overwrite a published world with a tested variant). Deletion removes access; IDs are never reissued. **Distributing assets with a blueprint ID attached causes upload failures for buyers — detach before sharing.**

## 7. Build Caching & Slowness

- The SDK builds a single bundle for the whole avatar/world, defeating Unity's per-bundle incremental rebuild — nearly everything rebuilds each time. Slowness sources: **texture re-encoding** (crunch and ASTC are extremely slow), **shader compilation** (`Library/ShaderCache`; corrupted caches cause hard failures fixed by deleting `Library`), **lightmap recompression** (worlds), **NDMF passes** (generated assets rebuilt each build; NDMF plans cross-build caching), **platform switches** (PC↔Android re-imports everything — Unity's local **Cache Server** makes subsequent switches far faster).
- **Cache locations:** Unity import caches `Library/`; shader cache `Library/ShaderCache/`; the *client's* bundle cache under `%LocalAppData%Low\VRChat\VRChat\` (hashed dirs per asset ID + version). Build & Test avatars → `%LocalAppData%Low\VRChat\VRChat\Avatars`.
- **Stale builds:** a rebuilt world still loading the old version usually = broken shader or corrupted cache; remedies: delete `Library/`, reinstall SDK, clear the VRChat cache.

## 8. Multi-Platform Build Internals

- **Targets:** Windows (StandaloneWindows64, DX11, BC), Android (Quest, Vulkan, ASTC/ETC2), iOS (Vulkan/Metal-derived, ASTC). Switching target re-imports the whole Library; the SDK's "Switch Platform" mirrors Build Settings.
- **Per-platform build differences:** materials/shaders differ (Quest requires `VRChat/Mobile/*` — unsupported-shader materials fail validation); lightmaps, mesh compression, component whitelists differ. Tools: EasyQuestSwitch (legacy), MA/AAO object-per-platform NDMF paths, SDK 3.8.1+ per-platform avatar overrides.
- **Same blueprint, separate bundles:** each platform publishes its own bundle under one blueprint ID. Current SDKs publish multiple selected platforms in sequence from one panel — each resolves validation independently.
- **Client behavior:** Android/iOS bundles exclude PC shader variants, are texture-format-swapped, size-capped; the client picks variants by platform. If a platform variant is absent, Quest/iOS shows fallback/Impostor — the PC bundle is not loaded on Quest.

### Sources

- NDMF: https://github.com/bdunderscore/ndmf · https://ndmf.nadena.dev/execution-model.html · ErrorReport.cs · best-practices.md
- https://creators.vrchat.com/avatars/avatar-performance-ranking-system/ · https://ask.vrchat.com/t/new-avatar-performance-rank/46308
- https://creators.vrchat.com/avatars/avatar-size-limits/ · https://creators.vrchat.com/platforms/android/quest-content-optimization
- https://creators.vrchat.com/sdk/vrcpipelinemanager/ · https://github.com/vrchat-community/creator-docs (public-sdk-api.md)
- https://ask.vrchat.com/t/avatar-validation-failed/25422 · https://help.vrchat.com/hc/en-us/articles/360062658873
