# Balatro with AnyPS5 on Windows

A setup guide and a small VideoOut queue-depth patch for testing Balatro with [AnyPS5](https://github.com/boykopovar/AnyPS5). This is not a game download, installer, or emulator bundle. It contains no Balatro game data, PS5 executable/modules, runtime PRXs, or compiled binaries.

## Requirements

- Windows x86-64 and a Vulkan 1.1-capable graphics driver.
- A legally obtained, clean Balatro PS5 ELF, its bundled modules, and its application assets.
- Current AnyPS5 source, Windows PRX libraries, and matching MinGW runtime DLLs.
- AnyPS5's current Windows build requirements: MinGW-w64 GCC 15.2.0, CMake, Ninja, and initialized submodules. See [build instructions](https://github.com/boykopovar/AnyPS5/blob/main/docs/dev/BUILD.md) and [runtime usage](https://github.com/boykopovar/AnyPS5/blob/main/docs/user/USAGE.md).

The v0.1.1 PRX bundle alone did not load this local Balatro build. This guide uses current upstream source to provide the newer system-library exports it needed.

## Build AnyPS5

Clone this repository and the official AnyPS5 repository into sibling folders. From their parent folder, run:

```powershell
git clone https://github.com/RamiChaaben8/Balatro_Anyps5.git
git clone https://github.com/boykopovar/AnyPS5.git
cd AnyPS5
git submodule update --init --recursive
git apply ..\Balatro_Anyps5\balatro-anyps5.patch
```

With the MinGW-w64 GCC 15.2.0 toolchain on `PATH`:

```powershell
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++
cmake --build build --parallel
cmake --build build --target libs --parallel
```

The `libs` target may stop at `libSceAudioIn.prx: no exported names`. In that case, keep the AnyPS5 Windows release bundle's `libs` directory and matching MinGW DLLs, then replace its libraries with the newer patched modules produced by the build. Run this from the AnyPS5 source root after the build:

```powershell
$built = (Resolve-Path .\build\core\libs\libs).Path
$unpatched = Join-Path $built 'unpatched'
$nid = '.\build\core\libs\nid_patcher.exe'
$libc = Join-Path $built 'libc.prx'
Get-ChildItem -LiteralPath $unpatched -Filter '*.prx' |
  Where-Object { $_.Name -notin @('libc.prx', 'libSceAudioIn.prx') } |
  ForEach-Object {
    $out = Join-Path $built $_.Name
    Copy-Item -LiteralPath $_.FullName -Destination $out -Force
    & $nid $_.BaseName --preserve-exports $libc $out
    if ($LASTEXITCODE -ne 0) { throw "NID patch failed: $($_.Name)" }
  }
```

Copy the patched `.prx` files from `build\core\libs\libs\` into your runtime `libs\` directory, replacing matching release files. Also copy the current build's `libc.prx`. Do not copy the `unpatched\` subfolder or original PS5 modules into runtime `libs\`.

## Convert your own Balatro files

Put the clean ELF beside its bundled module directory (`sce_module`, `sce_modules`, or `prx`):

```text
source\
  eboot.bin
  sce_module\   (use the directory name present in your dump)
```

From the AnyPS5 source root, convert it with the just-built Windows relinker (add `--to-intel` only if needed by an Intel host):

```powershell
.\build\core\relinker\relinker.exe --windows .\source\eboot.bin .\Balatro.exe
```

Use the exact converted module path printed by Relinker under `app0\sce_module\`. Copy the game's own application assets into `app0\`, including `conf.lua`, the resource folders, and `sce_sys\param.json`. The metadata file is required by the runtime. Do not replace converted title modules with original PS5 modules.

The runtime directory should look like this:

```text
Balatro.exe
libgcc_s_seh-1.dll
libstdc++-6.dll
libwinpthread-1.dll
libs\                 AnyPS5 Windows PRXs
app0\                 game assets and converted title modules
```

Place the three DLLs beside the executable from the same MinGW toolchain used to build the PRXs. Run from this directory:

```powershell
.\Balatro.exe
```

## Patch and test status

`balatro-anyps5.patch` lowers the VideoOut GPU queue wait depth to two while keeping the API queue capacity at sixteen. Apply it before building the AnyPS5 PRXs. A local Windows smoke test reached rendered game content on a GTX 1660 Ti; this is experimental and is not upstream compatibility certification. Results may vary by AnyPS5 revision, GPU, driver, and game build.

## Game files stay local

Only publish this guide and patch. Keep your PS5 ELF, original modules, `app0` assets, saves, generated `.exe`, and runtime binaries local. The included `.gitignore` excludes the local game and generated runtime files.
