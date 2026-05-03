# Agent notes for local Windows builds

This file records local build lessons so a new agent/conversation can resume without rediscovering the same issues.

## Project and artifact paths

- Project root used here: `C:\Users\87933\project\screenpipe\screenpipe`
- Tauri app: `apps\screenpipe-app-tauri`
- Tauri Rust crate: `apps\screenpipe-app-tauri\src-tauri`
- Successful app exe:
  `apps\screenpipe-app-tauri\src-tauri\target\release\screenpipe-app.exe`
- Successful NSIS installer:
  `apps\screenpipe-app-tauri\src-tauri\target\release\bundle\nsis\screenpipe - Development_2.4.124_x64-setup.exe`

## Branch strategy for this fork

- Keep `main` as the clean upstream-tracking baseline for `screenpipe/screenpipe`.
- Use `build` as the long-lived fork branch for local/self-distributed builds, Windows packaging fixes, i18n work, and future release/updater experiments.
- Prefer new feature branches off `build`, for example `feature/i18n-zh-cn` or `feature/github-updater`, then merge them back into `build`.
- To follow upstream later, update `main` from `upstream/main`, then merge `main` into `build` and resolve conflicts there.
- The old temporary branch name `build/windows-ci` was only for the initial Windows build experiment; prefer `build` going forward.

## Successful local build command

Run from PowerShell. This keeps the heavy ASR default features off while preserving `custom-protocol`, which production Tauri builds need.

```powershell
$app = 'C:\Users\87933\project\screenpipe\screenpipe\apps\screenpipe-app-tauri'
Set-Location -LiteralPath $app

$env:Path = "C:\Windows\System32;C:\Windows;$env:USERPROFILE\scoop\shims;$env:USERPROFILE\scoop\persist\bun\bin;$env:USERPROFILE\.cargo\bin;C:\Program Files\CMake\bin;C:\Program Files\LLVM\bin;$env:Path"
$env:LIBCLANG_PATH = 'C:\Program Files\LLVM\bin'
$env:CMAKE_GENERATOR = 'Ninja Multi-Config'
$env:CMAKE_CONFIGURATION_TYPES = 'MinSizeRel;Release;RelWithDebInfo;Debug'
$env:CMAKE_POLICY_VERSION_MINIMUM = '3.5'
$env:CARGO_BUILD_JOBS = '2'
$env:CMAKE_BUILD_PARALLEL_LEVEL = '2'

bun tauri build -- --no-default-features --features custom-protocol
```

## Optional full-feature local build command

If local ASR/model features are needed, enable the default ASR feature set explicitly while keeping the same Windows/CMake environment above:

```powershell
bun tauri build -- --no-default-features --features custom-protocol,qwen3-asr,parakeet
```

This is closer to the default project feature set (`custom-protocol`, `qwen3-asr`, and `parakeet`) but can take significantly more disk space and build time than the lighter verified command. Keep at least 50 GB free before trying it; 80 GB is safer for a clean or mostly uncached build.

For a future self-distributed build with in-app updates, investigate the existing `official-build` feature first before enabling it. It may be tied to the upstream update flow and should not be used blindly for a forked release channel.

## Problems encountered and fixes

### 1. `libsamplerate-sys` and CMake generator mismatch

Failures seen:

```text
ninja: error: loading 'build-MinSizeRel.ninja': The system cannot find the file specified.
```

and later:

```text
CMake Error: generator : Ninja
Does not match the generator used previously: Ninja Multi-Config
```

Fix:

- Use `Ninja Multi-Config`.
- Also set `CMAKE_CONFIGURATION_TYPES` to include `MinSizeRel`.
- If generator settings were changed between attempts, delete only the stale `libsamplerate-sys-*` CMake cache directories instead of wiping the whole target directory.

```powershell
$tauri = 'C:\Users\87933\project\screenpipe\screenpipe\apps\screenpipe-app-tauri\src-tauri'
$releaseBuild = Join-Path $tauri 'target\release\build'
Get-ChildItem -LiteralPath $releaseBuild -Directory -Filter 'libsamplerate-sys-*' -ErrorAction SilentlyContinue |
  Remove-Item -Recurse -Force
```

Do not blindly run `cargo clean` unless you are willing to rebuild many GB of cached dependencies.

### 2. `libsamplerate-sys` could not find `samplerate.lib`

Failure seen when using single-config Ninja:

```text
error: could not find native static library `samplerate`, perhaps an -L flag is missing?
```

Cause: `libsamplerate-sys` expects the MSVC output under a profile/config directory such as `out\build\MinSizeRel`. Single-config Ninja did not place it where the crate expected.

Fix: use the successful multi-config settings above:

```powershell
$env:CMAKE_GENERATOR = 'Ninja Multi-Config'
$env:CMAKE_CONFIGURATION_TYPES = 'MinSizeRel;Release;RelWithDebInfo;Debug'
```

### 3. Tauri Windows resource glob for FFmpeg DLLs

Failure seen:

```text
glob pattern ffmpeg\bin\*.dll path not found or didn't match any files.
```

Cause: the downloaded `ffmpeg.exe` was a static/full build, so `src-tauri\ffmpeg\bin` contained `ffmpeg.exe`, `ffprobe.exe`, and `ffplay.exe`, but no DLL files.

Local fix applied in `apps\screenpipe-app-tauri\src-tauri\tauri.windows.conf.json`: remove this resource entry:

```json
"ffmpeg\\bin\\*.dll": "./"
```

Keep these entries:

```json
"ffmpeg\\bin\\ffmpeg.exe": "./",
"ffmpeg\\bin\\ffprobe.exe": "./"
```

Before changing this again, check the actual FFmpeg package layout. If using a shared FFmpeg build with DLLs, the wildcard may be valid.

### 4. Disk space

The final installer was only about 100+ MB, but the build needed tens of GB of temporary/intermediate space. This is normal for Rust + Tauri + native/CMake dependencies.

Reasons:

- Rust `target` contains `.rlib`, `.rmeta`, `.obj`, build scripts, and CMake/Ninja intermediates.
- Native dependencies include FFmpeg, ONNX Runtime, OpenBLAS, Whisper-related crates, NSIS tooling, and VC runtime files.
- Failed retries leave additional cache/intermediate files.
- NSIS compresses the final app, so the installer is much smaller than the build tree.

Recommended free space before a fresh/retry build: at least 20-30 GB; more is safer. During the successful run, free space dropped to roughly 2-3 GB.

Avoid deleting these while trying to finish a build:

```text
apps\screenpipe-app-tauri\src-tauri\target
target-ninja-release
```

Delete them only after copying out the installer, or if you accept a long rebuild.

## Final successful result

The successful build ended with:

```text
Finished `release` profile [optimized]
Built application at: ...\src-tauri\target\release\screenpipe-app.exe
Finished 1 bundle at: ...\src-tauri\target\release\bundle\nsis\screenpipe - Development_2.4.124_x64-setup.exe
exit_code: 0
```

If the goal is only to keep/install the app, copy out the NSIS setup executable from `target\release\bundle\nsis`.
