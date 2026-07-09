# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Mod Loader is a Windows plugin (`modloader.asi`) for GTA III, Vice City and San Andreas that lets players drop mod files into a `modloader/` folder and have them injected into the game at runtime, with no changes to original game files. It's a native C++14 project targeting the Windows XP toolset (v141_xp) so it runs on the same old engines the games use. It hooks the game process via injected code/detours and file-system interception, not a launcher or asset repacker.

The core (`modloader.asi`) is a small event-driven plugin host. All actual file-type handling (models, audio, scripts, streams, etc.) lives in separate plugin DLLs loaded by the core.

## Build

Requires Premake 5, Visual Studio 2017+ with the Windows XP Platform Toolset (v141_xp) component, and submodules checked out (dependencies live in `deps/`, several as vendored/pinned sources rather than actual git submodules — check `deps/*.txt` for version notes).

Generate a Visual Studio solution:
```
premake5 vs2022
```
This produces a solution under `build/` (default `--outdir`). Build it from Visual Studio or via `msbuild`.

Install built binaries straight into a game directory:
```
premake5 install "C:/Program Files (x86)/Rockstar Games/GTA San Andreas"
```

Auto-install on every build (adds a postbuild copy step):
```
premake5 vs2022 "--idir=C:/Program Files (x86)/Rockstar Games/GTA San Andreas"
```

Other premake actions (see `premake5.lua`):
- `premake5 clean` — removes `bin/`, `build_temp/`, `release/`
- `--final-release` — defines `MODLOADER_FINAL_RELEASE` solution-wide (public release build)
- `--outdir=<path>` — change the generated build files location (default `build`)

There is no unit test suite in this repo — verification is done by building and running the plugin against an actual game install.

### Producing a full public release

`release.lua` (invoked via `premake5 --file=release.lua prepare --toolset=vs2022[--final-release]`, or via `release.bat` on Windows) cleans the workspace, regenerates build files into `build_temp/`, builds with msbuild (or gcc/mingw32-make for the `gcc` toolset), installs into `release/binaries/`, and packages stripped PDB symbols into `release/symbols/` via `pdbcopy`. This mirrors what `.github/workflows/build.yml` does in CI for tagged (`v*.*.*`) releases, which also zips the output and attaches it to a GitHub Release using the matching section of `doc/CHANGELOG.md`.

## Architecture

### Core vs. plugins

- `src/core/` builds `modloader.asi` — the loader core. It patches the game (`Patch()` in `loader.cpp`/`loader.hpp`), scans the `modloader/` directory tree, tracks mods/files/profiles, and dispatches file-install events to whichever plugin claims a given file extension/behaviour.
- `src/plugins/gta3/std.*` are the built-in plugins, each compiled as its own `SharedLib` project by `addplugin()` in `premake5.lua` and installed to `modloader/.data/plugins/gta3/`. Each plugin owns one domain, e.g.:
  - `std.data` — game data files (handling.cfg, carcols.dat, timecyc.dat, ide/ipl, weapon.dat, etc.) — by far the largest plugin, handled via the `data_traits` framework (one `.cpp` per data file format under `std.data/data_traits/`).
  - `std.stream` — CD image streaming (models/textures loaded from `.img` archives), including a hand-written asm backend (`std.stream/asm/backend.cc`).
  - `std.asi` — loads and manages other people's `.asi` plugins, including an argument-translator shim (`args_translator/`) for path translation between ASI/CLEO/EXE contexts.
  - `std.scm`, `std.sprites`, `std.fx`, `std.movies`, `std.dmaudio`, `std.bank`, `std.tracks` — scripts, sprites, FX, movies, DirectMusic audio, wave banks, and radio tracks respectively.
  - Build order in `premake5.lua`'s `gta3_plugins` list is deliberately sorted by compile time (slowest last) to parallelize CI/local builds well.
- `src/shared/` holds code shared between core and plugins but not part of the public plugin API: game structure definitions (`game/gta3/`), the `datalib` serialization framework used heavily by `std.data`, ini/regex/fxt parsers, per-game trait tables (`traits/gta3/{iii,vc,sa}.hpp` — the three supported games have different binary layouts), and the precompiled-header setup (`stdinc/`).
- `include/modloader/` is the **public plugin API** — `modloader.h` (C ABI) and `modloader.hpp` (C++ binding on top of it). This is what a third-party plugin author includes; treat changes here as API-surface changes.

### Plugin protocol

Plugins are DLLs that must **not** implement `DllMain`. They register via `REGISTER_ML_PLUGIN(instance)` in `modloader.hpp`, which exports `GetPluginData`/`GetLoaderVersion` as the C ABI entry points the core calls. A plugin implements `modloader::basic_plugin` and its event methods:

- `GetInfo()` — name/version/author/priority/extension table.
- `GetBehaviour(file&)` — decide `MODLOADER_BEHAVIOUR_{NO,YES,CALLME}` for a file; `YES` means claiming it (files sharing a `behaviour` value are mutually exclusive — only the highest-priority one installs).
- `InstallFile` / `ReinstallFile` / `UninstallFile` — lifecycle for a claimed file. Any file passed to `InstallFile` stays valid until it's passed back to `UninstallFile`; plugins may safely cache the pointer.
- `Update()` — optional, called after a batch of install/uninstall/reinstall calls settles.

The full protocol and object lifetime rules are documented in `doc/Creating Your Own Plugin.md` — read it before adding or modifying a plugin.

### Core data model (`src/core/loader.hpp`)

`Loader` (singleton `extern class Loader loader`) owns:
- `FolderInformation` — a scanned `modloader/`-style directory tree, containing `ModInformation` (one mod folder) → `FileInformation` (one file, `modloader::file` subclass) mappings, plus a `Profile` system (inheritance, per-mod priority overrides, ignore/include globs, exclusivity) loaded from ini config.
- `PluginInformation` — a loaded plugin DLL, tracking which `behaviour` values it currently owns.
- `ExtMap` — extension → candidate-plugins index, rebuilt via `RebuildExtensionMap()`, used to route files to handlers.
- Status tracking (`Added`/`Updated`/`Removed`/`Unchanged`) drives incremental rescans (`ScanAndUpdate`, journal-based updates) so only changed files get re-installed — this is what makes hot-swapping mods while the game runs work.

### Multi-game support

III, VC and SA share the plugin/core codebase but differ in binary structures and behavior; game-specific differences are isolated behind `src/shared/traits/gta3/{iii,vc,sa}.hpp` trait headers and conditional logic in plugins (notably `std.data`), rather than separate builds per game.

### Dependencies (`deps/`)

Vendored: Boost (subset), `injector` (memory patching/hooking), `cereal` (serialization), `tinympl`, `utf8-cpp`. These are pinned/vendored, not fetched by a package manager — check `deps/boost.txt` etc. for versioning notes before assuming upstream behavior.
