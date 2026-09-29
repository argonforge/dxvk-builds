# DXVK builds

Optimized [DXVK](https://github.com/doitsujin/dxvk) (Direct3D 9/10/11 to Vulkan translation layer) built for modern CPUs.

## Requirements

- **Architecture:** amd64
- **CPU:** x86-64-v3 (AVX2) or x86-64-v4 (AVX-512) - pick the matching variant
- **Runtime:** Wine 8.0+ or Proton 8.0+

**Builds will not run** on CPUs without AVX2/AVX-512. Check support:

```bash
grep -o 'avx[0-9_]*' /proc/cpuinfo | sort -u
```

If the output is empty - **do not use these builds**.

## Package

<details>
<summary>Build details</summary>

| Component      | Description                                                                                |
|----------------|--------------------------------------------------------------------------------------------|
| Target         | Windows PE DLLs (x64 + x86), cross-compiled                                                |
| Compiler       | Clang from [llvm-mingw](https://github.com/mstorsjo/llvm-mingw) 20260922 (UCRT)            |
| Linker         | LLD                                                                                        |
| Compiler flags | `-march=x86-64-v3/znver3` or `-march=x86-64-v4/znver4`, `-O3`                              |
| Patches        | Removed upstream AVX build check, `std::tuple()` compatibility for Clang 22                |

Debug symbols are **not included** in releases - they do not affect performance and only take up space.
</details>

## Installation

### 1. Download and extract

```bash
mkdir -p ~/dxvk-opt && cd ~/dxvk-opt
gh release download --repo argonforge/dxvk-builds --pattern '*.tar.zst'

# Unpack the archive matching your CPU:
tar --zstd -xf dxvk-*-x86-64-v4.tar.zst   # AVX-512 (Zen 4/5)
# or
tar --zstd -xf dxvk-*-x86-64-v3.tar.zst   # AVX2 (Zen 1/2/3)
```

Or download the `.tar.zst` file manually from the [Releases](../../releases) page.

### 2. Copy DLLs into Wine prefix

```bash
export WINEPREFIX=/path/to/your/prefix
cd dxvk-*/

# 64-bit DLLs
cp x64/*.dll "$WINEPREFIX/drive_c/windows/system32/"

# 32-bit DLLs
cp x86/*.dll "$WINEPREFIX/drive_c/windows/syswow64/"
```

### 3. Set DLL overrides

Open `winecfg` and set the following DLLs to **"native, then builtin"**:

- `d3d8`
- `d3d9`
- `d3d10core`
- `d3d11`
- `dxgi`

### 4. Clear shader caches

**Mandatory.** Old shader caches are incompatible with the new DXVK version and will cause crashes or rendering artifacts.

```bash
rm -rf ~/.cache/dxvk/* \
       ~/.cache/mesa_shader_cache*
```

For per-game caches, remove `dxvk-cache*` next to the game's `.exe`.

### 5. Verify

Launch the game with the DXVK HUD enabled:

```bash
DXVK_HUD=1 WINEPREFIX=/path/to/your/prefix wine /path/to/game.exe
```

A HUD overlay with FPS and GPU info confirms DXVK is active.

## Rollback

This is a manual DLL installation - no package manager integration. To roll back:

```bash
export WINEPREFIX=/path/to/your/prefix

# Remove DXVK DLLs (64-bit)
rm -f "$WINEPREFIX/drive_c/windows/system32/"{d3d8,d3d9,d3d10core,d3d11,dxgi}.dll

# Remove DXVK DLLs (32-bit)
rm -f "$WINEPREFIX/drive_c/windows/syswow64/"{d3d8,d3d9,d3d10core,d3d11,dxgi}.dll

# Reset DLL overrides in winecfg, then run:
wineboot --update
```

Or simply delete the Wine prefix and recreate it:

```bash
rm -rf "$WINEPREFIX"
WINEPREFIX=/path/to/your/prefix WINEARCH=wow64 wineboot --init
```

## Expected Performance

| Component                                   | Gain   | Comment                                              |
|---------------------------------------------|--------|------------------------------------------------------|
| CPU part of DXVK (D3D9/10/11 translation)   | 1-5%   | Noticeable in CPU-bound scenarios                    |
| GPU-bound games                             | ~0%    | Bottleneck is GPU and memory bandwidth               |

**Honest note:** DXVK is a translation layer, not a graphics driver. Its own CPU cost matters only in scenarios where the game is limited by D3D call overhead. The main FPS gains in games come from the graphics stack (Mesa, RADV), not from DXVK itself.

Source workflow: [`.github/workflows/build.yml`](.github/workflows/build.yml).

## Companion projects

For a complete optimized graphics stack on **AMD Zen (x86-64-v3/v4)**:

| Project                                                                    | Purpose                           |
|----------------------------------------------------------------------------|-----------------------------------|
| [`mesa-builds`](https://github.com/argonforge/mesa-builds)                 | Mesa (radeonsi, RADV)             |
| [`vkd3d-proton-builds`](https://github.com/argonforge/vkd3d-proton-builds) | VKD3D-Proton (D3D12 -> Vulkan)    |
| [`wine-builds`](https://github.com/argonforge/wine-builds)                 | Wine WoW64 (Clang)                |
| [`gamescope-builds`](https://github.com/argonforge/gamescope-builds)       | Micro-compositor for game scaling |

## Important

- Builds are compiled on **Arch Linux** (multilib) against llvm-mingw. The resulting DLLs are Windows PE files that run inside any Wine 8.0+ or Proton 8.0+ prefix, regardless of the host distribution.
- Builds are **not signed**. Verify integrity using SHA-256 from the release description.
- **Do not use v4 builds** if you are unsure about AVX-512 support. Use v3 if your CPU has only AVX2.
- DXVK manipulation in online multiplayer games may be considered cheating. **Use at your own risk.**
- Always keep a working DXVK release from [upstream](https://github.com/doitsujin/dxvk) as fallback.
- Not affiliated with the upstream project. Report build-specific issues in this repository's [Issues](../../issues) tracker.

## License

The build scripts and GitHub Actions workflows in this repository
are licensed under the MIT License. See LICENSE file.

The DXVK source code is distributed under the
[zlib license](https://github.com/doitsujin/dxvk/blob/master/LICENSE).
The compiled DLLs in Releases are redistributions of DXVK under its original license.
