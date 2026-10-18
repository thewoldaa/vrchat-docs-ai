# VRChat Developer Workflow & Automation Report (2025–2026)

*How VRChat creators and studios manage Unity projects, packages, builds, tests, assets and backups. VRChat content creation is a Unity project like any other — but shaped by the VPM ecosystem and an SDK upload pipeline that is still not officially scriptable end-to-end.*

---

## 1. Version Control for VRChat Unity Projects

### 1.1 Unity settings first

Project Settings → Editor: **Version Control: Visible Meta Files**, **Asset Serialization: Force Text** (defaults in 2022.3, but verify — legacy `.unitypackage` imports may not have them).

### 1.2 What to commit: the VCC/VPM answer

Official guidance ([Using Source Control with the VPM](https://vcc.docs.vrchat.com/vpm/source-control/)):

**Include:** `Assets/`, `ProjectSettings/`, `Packages/` (UPM `manifest.json`, `vpm-manifest.json`, `vpm-dependencies.json`), `Packages/com.vrchat.core.vpm-resolver/` — the resolver can reinstall VPM packages **without** the VCC.

**Exclude:** all other VPM packages under `Packages/`, `Library/`, `Temp/`, `Logs/`, `obj/`, `UserSettings/`, build output.

**The catch:** the VCC **auto-writes** `Packages/.gitignore` containing

```
/*/
!com.vrchat.core.*/
```

every time you add/remove/update a package — silently un-committing community packages including local dev packages. Tracked in [creator-companion issue #416](https://github.com/vrchat-community/creator-companion/issues/416), unresolved as of 2025. Accepted workaround: keep VCC's two lines and re-add your own inclusions:

```gitignore
# Packages/.gitignore — VCC keeps rewriting the first two lines
/*/
!com.vrchat.core.*/
!at.*/
!com.anatawa12.*/
!dev.*/
!tlp.*/
# locally-authored or vendor-locked packages that can't be re-resolved
dev.yourname.yourpackage/
```

For fresh clones: open Unity and the resolver offers to restore missing packages — or run `vrc-get resolve` headlessly (faster).

### 1.3 Git LFS via `.gitattributes`

Community baseline: [Obey138/Unity-VRChat-Git-Config](https://github.com/Obey138/Unity-VRChat-Git-Config):

```
# Binary art — LFS
*.psd   filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text
*.fbx   filter=lfs diff=lfs merge=lfs -text
*.png   filter=lfs diff=lfs merge=lfs -text
*.tga   filter=lfs diff=lfs merge=lfs -text
*.wav   filter=lfs diff=lfs merge=lfs -text
*.mp3   filter=lfs diff=lfs merge=lfs -text
*.ogg   filter=lfs diff=lfs merge=lfs -text

# Unity YAML — text, diffable
*.unity  text
*.prefab text
*.mat    text
*.asset  text
*.meta   text
*.anim   text
*.controller text
```

`git lfs install` once per machine. Adopt two hooks from that repo: **pre-commit** failing if an asset is added without its `.meta` (or vice versa), and a >100 MB file guard (GitHub's hard limit). Meta-file sync is the #1 way Git-corrupted Unity projects happen.

### 1.4 Scene/prefab/controller merges

`.unity`/`.prefab`/`.controller`/`.anim` are YAML — Git *can* merge them, but two people editing one scene almost always conflicts. Strategies:

1. **UnityYAMLMerge** — ships with the editor, semantically merges scenes/prefabs:

```ini
# .git/config
[merge "unityyamlmerge"]
    name = Unity SmartMerge
    driver = '"C:/Program Files/Unity/Hub/Editor/2022.3.22f1/Editor/Data/Tools/UnityYAMLMerge.exe"' merge -p --force %O %B %A %A
```
```
# .gitattributes
*.unity   merge=unityyamlmerge eol=lf
*.prefab  merge=unityyamlmerge eol=lf
```

Note the `--force` — without it the tool prompts interactively and hangs CI. **Always open merged assets to verify before pushing.**

2. **Prefab decomposition** — scenes as thin shells, content in prefabs; two artists rarely touch the same file. The real long-term fix.
3. **File locking** — `git lfs lock Assets/Scenes/main.unity` (LFS lockable patterns or external lock server). Often overkill for commission work.

### 1.5 The alternative: folder-backup conventions

Many commission artists don't use Git at all. De-facto conventions:

- Versioned project copies: `AvatarName_v1/`, `AvatarName_v2/` — crude but works with 50+ GB projects.
- A `_ClientAssets/` folder **outside** the Unity project holding Blender sources, PSDs, downloaded store packages.
- A `PACKAGES/` folder of raw `.unitypackage`/`.zip` downloads kept forever (store re-downloads can be lost).
- `.zip` snapshots to external drives; VCC project backup / `vrc-get` project backup in ALCOM produce portable zips.

Honest reality: for a solo artist shipping one avatar per client, folder backups are often simpler than Git. Git becomes worthwhile when two people touch one project, or when you want CI builds.

## 2. VCC Project Management Deep Dive

### 2.1 The official `vpm` CLI

```console
dotnet tool install --global vrchat.vpm.cli

vpm new MyWorld World -p C:\Projects     # create from template (Base/World/Avatar/UdonSharp)
vpm add project C:\Projects\MyWorld      # register with VCC GUI list
vpm list projects
vpm resolve project                      # restore packages from vpm-manifest.json
vpm add package com.vrchat.worlds -p <path>
vpm migrate legacy <path>                # 2019 unitypackage SDK → VPM
vpm migrate 2022 <path> [--inplace]      # Unity 2019 → 2022 migration
vpm check package com.vrchat.avatars
```

`vpm migrate 2022` copies the project by default (`--inplace` only with your own backup). Migration is mostly historical now — everyone is on 2022.3.22f1.

### 2.2 vrc-get / ALCOM (anatawa12) — the community standard

[vrc-get](https://github.com/anatawa12/vrc-get) is a Rust reimplementation of the VPM client; **ALCOM** is its GUI — widely treated as a faster, cross-platform VCC replacement.

```pwsh
winget install anatawa12.vrc-get    # Windows
brew install vrc-get                # macOS
```

```console
vrc-get install com.anatawa12.avatar-optimizer [version]   # alias: vrc-get i
vrc-get resolve                       # install everything in vpm-manifest.json
vrc-get remove com.some.package       # alias: rm
vrc-get outdated
vrc-get upgrade                       # upgrade all
vrc-get search "optimizer"
vrc-get repo list
vrc-get repo add <NAME> <url>
vrc-get repo remove <NAME>
vrc-get repo cleanup                  # clear cached repo json
vrc-get repo packages <NAME|url>      # list what a repo offers
```

Why it's the automation standard: single static binary, Linux/macOS/Windows, no .NET runtime, resolves `vpm-manifest.json` exactly like VCC.

**Reproducibility contract:** add repos once (`vrc-get repo add Poiyomi https://repo.poimandres.com/vpm.json`, `vrc-get repo add AAO https://vpm.anatawa12.com/vpm.json`), then **pin versions in `vpm-manifest.json` and commit that file**. Fresh clone + `vrc-get resolve` reproduces the exact package set. ALCOM also makes authoring VPM packages first-class.

### 2.3 GitHub Actions integration

vrc-get ships first-party actions:

```yaml
- uses: anatawa12/sh-actions/setup-vrc-get@master
- uses: anatawa12/sh-actions/resolve-vpm-packages@master
```

`resolve-vpm-packages` reads `vpm-manifest.json` and installs every package into the CI checkout — the standard first step of any VRChat CI job.

## 3. Multiple Unity Versions, Unity Hub, CI Licensing

VRChat pins exact editor versions per SDK release. Unity Hub installs them side by side under `C:\Program Files\Unity\Hub\Editor\<version>\`.

- **Unity Hub CLI** (`Unity Hub.exe -- --headless install --version 2022.3.22f1 --module android`) installs editors/modules from scripts.
- **game-ci docker images** (`unityci/editor:ubuntu-2022.3.22f1-windows-mono-3.2.2`, `-android` suffix for Quest support) give a pinned editor per CI job.
- Keep project↔editor mapping in `ProjectSettings/ProjectVersion.txt` (committed); let CI select the image from it.

**License activation for CI** ([GameCI guide](https://game.ci/docs/github/activation/)):
- *Personal*: activate once locally in Unity Hub (click "Add" so the `.ulf` actually gets written), store file contents as `UNITY_LICENSE` secret + `UNITY_EMAIL`/`UNITY_PASSWORD`. Locations: Windows `C:\ProgramData\Unity\Unity_lic.ulf`, Linux `~/.local/share/unity3d/Unity/Unity_lic.ulf`. Personal-license CI activation is unofficial and flaky — many VRChat devs build locally for this reason.
- *Pro*: `UNITY_SERIAL`, `UNITY_EMAIL`, `UNITY_PASSWORD`.
- *Floating*: `unityLicensingServer: <url>` on the builder step.

## 4. Automated Builds & Uploads

### 4.1 Unity command-line builds

```console
Unity.exe -quit -batchmode -nographics -projectPath . \
  -executeMethod MyEditor.BuildScript.PerformBuild \
  -logFile build.log
```

Caveat for avatars: `-nographics` breaks assets needing GPU during build (Read/Write-disabled meshes in play-mode pipelines) — drop it if your build script touches avatar tooling.

### 4.2 SDK upload automation status (2025–2026)

Honest state: **VRChat has never shipped an official headless upload CLI.** What exists:

1. **`-executeMethod` calling into SDK build APIs.** Best-known public example: [jazzy348/automated-vrchat-world-build](https://github.com/jazzy348/automated-vrchat-world-build/) — an `AutoVRCUploader.cs` in `Assets/Editor/` invoked:

```console
Unity.exe -projectPath /path/to/project \
  -executeMethod AutoVRCUploader.UploadWorldCLI -- \
  --scene=Assets/Scenes/main.unity \
  --thumbnail=Assets/Editor/thumbnail.png \
  --name="World Name" --id="wrld_xxxxxxxx" \
  --platform=pc \
  -logFile /path/to/log
```

   Runs in production behind a Git webhook with a Discord bot for logs. Works, but depends on SDK internals — re-verify per SDK major update.
2. **VRChat's stance:** automated/CI uploads happen but no supported interface; the "user must be logged in" check is the usual blocker. Practical setups pre-authenticate once and cache the session.
3. **Login/2FA:** headless flows typically pre-authenticate and cache the session — the fragile part; expect maintenance.

**Realistic recommendation:** automate *everything up to* the upload (package resolve, compile check, bundle build, rank checks) in CI; keep the final SDK upload manual or on one dedicated build machine running an uploader like jazzy348's. Several studios do exactly this — single upload point "to ensure everyone is working from the same world ID."

### 4.3 CI on GitHub Actions: real examples

**VRCFury** ([deploy.yml](https://github.com/VRCFury/VRCFury/blob/main/.github/workflows/deploy.yml)) is the most complete public VRChat CI:

```yaml
strategy:
  fail-fast: false
  matrix:
    vrcsdkUnityVersion:
      - [ 3.10.3, 6000.4.4f1 ]
      - [ 3.10.3, 2022.3.22f1 ]
      - [ 3.7.0,  2022.3.6f1 ]
    platform: [ windows, android ]
    sdkKind:  [ avatars ]
```

Steps: `download_vrcsdk.sh` fetches the SDK; `unityci/editor` images run `game-ci/unity-test-runner@v4` EditMode tests with license secrets; disk-cleanup steps because `ubuntu-latest` runners overflow with the editor image.

**vrchat-community/template-package** — official starter for publishing **your own VPM package**: package in `Packages/com.yourname.package-name/`, `PACKAGE_NAME` repo variable, GitHub Pages via Actions; workflows build a `.zip`, a `.unitypackage` (via `pCYSl5EDgo/create-unitypackage`), a listing page, and a Release per version.

**Minimal compile-check CI:**

```yaml
name: CI
on: [push, pull_request]
jobs:
  compile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anatawa12/sh-actions/setup-vrc-get@master
      - uses: anatawa12/sh-actions/resolve-vpm-packages@master
      - name: Clean disk for editor image
        run: sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc
      - uses: game-ci/unity-test-runner@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
        with:
          unityVersion: 2022.3.22f1
          testMode: EditMode
          customImage: unityci/editor:ubuntu-2022.3.22f1-windows-mono-3.2.2
```

## 5. Testing Automation

### 5.1 Av3Emulator + Unity Test Framework

[Av3Emulator](https://github.com/lyuma/Av3Emulator) emulates the AV3 animator system in-editor on the PlayableGraph API. Two uses:

- **Manual**: Tools → Avatar 3.0 Emulator drops a runtime object into the scene; drive menus, gesture weights, non-local clones.
- **Automated**: write `UnityTest`/EditMode integration tests that instantiate an avatar, attach the emulator, drive parameters, assert animator outcomes. VRCFury's CI runs exactly this against multiple SDK/Unity combos; AAO supports being triggered by Av3Emulator in Play Mode.

Setup: `com.unity.test-framework`, tests in `Assets/Tests/Editor` asmdef referencing `UnityEngine.TestRunner` + `UnityEditor.TestRunner`, headless with `-runTests -testPlatform EditMode -testResults results.xml`.

### 5.2 ClientSim

Official **ClientSim** package (com.vrchat.clientsim): run worlds in editor play mode — interactables, pickups, UI — without the game client. Standard way to smoke-test worlds in CI play-mode tests. Limits: no networking, no IK — complements, not replaces, in-VRChat testing.

### 5.3 Texture memory / rank checking in CI

- **AAO** rewrote its texture-usage estimation in 1.9.16 (own logic instead of VRCSDK's — fixing RenderTexture inclusion, cubemap 1/6-size miscalculation). "Trace and Optimize" + "Optimize Texture" performs automatic atlasing/VRAM reduction non-destructively at build time, on the **NDMF** pipeline (the de-facto non-destructive build hook system).
- A CI "rank check" = a small editor script walking the avatar's renderers/materials, summing mipmapped GPU texture sizes (format-aware: BC7/ASTC/DXT), failing the build above threshold. Reference math from PLAYTEX AI's texture memory calculator and Thry's VRAM calculator; **LAC** (Limitex Avatar Compressor) ships in-project `Texture Compressor` presets.

## 6. Asset Management

**Where sources live.** Blender/PSD sources in `Assets/` means Unity re-imports multi-GB PSDs on every change and everything enters LFS. Growing convention — separate concerns:

```
MyAvatarProject/
├── Assets/
│   ├── _Project/
│   │   ├── Avatars/MyAvatar/        # FBX, prefabs, materials, animators
│   │   ├── Textures/                # exported PNGs/TGA (LFS)
│   │   └── Scenes/main.unity
│   └── Tests/
├── Packages/                        # vpm-manifest.json committed
├── ProjectSettings/
└── _source/                         # OUTSIDE Assets, NOT imported by Unity
    ├── blender/MyAvatar.blend
    ├── psd/MyAvatar_body.psd
    └── downloads/                   # raw Booth/Gumroad packages, kept forever
```

**Blueprint IDs across variants.** Each uploaded avatar/world gets an immutable blueprint ID (`avtr_…`/`wrld_…`) stored in the scene/prefab descriptor:
- **Never** Ctrl+D a scene/prefab to make a "variant" — it copies the blueprint ID → update collisions between two uploads. Duplicate-then-clear-ID, or NDMF/AAO's ID handling.
- Multi-variant avatars: each blueprint in its own scene/prefab, mapping recorded in a committed `blueprints.md` — the ID exists **only** in that project, so the project is the source of truth.
- Worlds teams converge on "single upload point" to avoid two projects fighting over one ID.

## 7. Multi-Project Strategies

**One project per avatar** (commission default): isolated package versions, no cross-contamination; but every project re-downloads SDK/packages (GBs).

**Shared library project**: one project holding all avatars/worlds; works when everything shares pinned tool versions; breaks when a client needs an older SDK or conflicting shader version. Common hybrid: one "sandbox" project + one project per *shipping* deliverable.

**Authoring your own VPM packages** — the scalable shared library. Package with a `package.json`:

```json
{
  "name": "com.yourname.rig-tools",
  "displayName": "Rig Tools",
  "version": "1.2.0",
  "unity": "2022.3",
  "vpmDependencies": { "com.vrchat.avatars": ">=3.7.0" }
}
```

Reference locally (local folder repo via `vrc-get repo add`, or place in `Packages/` un-ignored) or publish via the template-package Actions flow (zip + unitypackage + hosted VPM listing on GitHub Pages). This is how anatawa12 distributes AAO and how VRCFury ships.

## 8. Backup / Recovery

**Blueprint ID loss.** The ID lives in the project's scene/prefab descriptor; servers keep the *content*, but the linkage is local:
- Any copy of the project or built bundle metadata can recover the ID (search project folder for `avtr_`/`wrld_`; also check SDK cache under `%LOCALAPPDATA%\..\LocalTemp\VRChat\`).
- If truly gone: no "claim this ID" flow exists — the upload is orphaned; re-upload as new ID and update links/whitelists. **The strongest argument for committing scenes/prefabs to version control.**
- Worlds: route all uploads through one pipeline so exactly one place holds the live ID.

**"Asset not found" / missing-script errors** after a clone are almost always resolve-order: assets imported before the resolver restored SDK packages, leaving broken script GUID references. Fixes:
1. `vrc-get resolve` (or `vpm resolve project`) **before** first Unity open of a fresh clone.
2. If already imported: reimport (delete `Library/`, reopen) so scripts load before prefab references bind.
3. Renamed/moved `.cs` file: restore the original GUID from the `.cs.meta` in Git history — never casually regenerate meta files.

**Full recovery checklist**: Git clone (or backup zip) → `vrc-get resolve` → open in pinned Unity from `ProjectVersion.txt` → resolver restores SDK → `_source/` folder from separate backup. Everything reproducible except store-purchased assets — the raw download archive in `_source/downloads/` is non-negotiable.

## Sources

- https://vcc.docs.vrchat.com/vpm/source-control/ · https://vcc.docs.vrchat.com/vpm/cli/
- https://github.com/anatawa12/vrc-get
- https://github.com/vrchat-community/creator-companion/issues/416
- https://github.com/Obey138/Unity-VRChat-Git-Config
- https://github.com/jazzy348/automated-vrchat-world-build/
- https://github.com/VRCFury/VRCFury/blob/main/.github/workflows/deploy.yml
- https://github.com/vrchat-community/template-package
- https://github.com/32ba/VRC-upload-notifier/blob/main/.github/workflows/release.yml
- https://game.ci/docs/github/getting-started/ · https://game.ci/docs/github/activation/
- https://github.com/lyuma/Av3Emulator
- https://vpm.anatawa12.com/avatar-optimizer/en/docs/changelog/
- https://www.playtex.ai/vrchat-texture-memory-calculator · https://lac.limitex.dev

*Honesty notes:* the official SDK still lacks a supported headless upload API; §4.2 relies on SDK internals and community tooling that must be re-verified per SDK release. Personal-license CI activation is unofficial and flaky. Package/version numbers move quickly — treat `vrc-get upgrade` output, not this report, as ground truth.
