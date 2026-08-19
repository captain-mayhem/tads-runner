# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

An open-source fork of the TADS Runner (interpreter/compiler for TADS 2 and TADS 3 interactive fiction), extracted from the [garglk](https://github.com/garglk/garglk) repository under GPL-2.0. The focus is the Glk port (GlkTADS), but changes for non-Glk interpreters are welcome too. This is largely inherited C/C++ code from the original TADS distribution (TADS2, TADS3) plus CMake build plumbing added on top; most `.c`/`.cpp` files under `tads2/` and `tads3/` should be treated as vendored upstream code — prefer minimal, targeted changes over refactors.

## Build system

CMake (>= 3.17), builds with Ninja by default via presets. Key top-level targets, built via `add_subdirectory`, live in `tads2/`, `tads3/`, `glk/`, and `tests/`.

```
cmake --preset default          # configure (Ninja, build/default)
cmake --build --preset default  # build
```

Presets (`CMakePresets.json`):
- `default` — Ninja, single-config, `build/default`
- `ninja-multi` — Ninja Multi-Config
- `windows-only` — inherits `default`, forces the MSVC toolchain file (`toolchain/Windows.MSVC.toolchain.cmake`), only usable on Windows

On Windows without a preset, the MSVC toolchain files under `toolchain/` (`Windows.MSVC.toolchain.cmake`, `VSWhere.cmake`, `Windows.Kits.cmake`, `WSL.cmake`) locate Visual Studio/Windows SDK; use the `windows-only` preset rather than hand-configuring these.

### Emscripten (WASM) builds

Passing `-DEMSCRIPTEN=...` (or configuring with the Emscripten toolchain) enables WASM-specific paths: it pulls in `curl/` as a subdirectory (used for TADS networking support), and packages compiled `.t3` game files into Emscripten `.data`/`.js` blobs via the `em_package` CMake function (see `TadsFunctions.cmake`).

### Sibling `htmltads` directory

If a `../htmltads` directory exists next to this repo (see `tads-runner.code-workspace`, which opens both as one VS Code workspace), CMake auto-detects it and, on Windows or Emscripten only, sets `WITH_HTMLTADS=ON` and builds it as a subdirectory, enabling the HTML TADS targets (`t3htm`, `t3htm_d`) in `tads3/CMakeLists.txt`. Without that sibling directory present, HTML TADS targets are skipped entirely.

`htmltads` is an independent git repository with its own restrictive (non-GPL) license and its own [CLAUDE.md](../htmltads/CLAUDE.md) — read that before working on anything under `htmltads/`. It currently hosts the active `guit3` migration (porting the legacy Win32 `htmlt3` GUI client to cross-platform Dear ImGui/GLFW), whose plan and working notes live in `htmltads/htmltads/imgui/migration.md`.

## Testing

Tests live in `tests/` and run against a Glk implementation (Remglk), driven by `regtest.py` (from the [plotex](https://github.com/erkyrath/plotex) project). CI (`.github/workflows/build-and-test.yml`) runs:

```
./tests/prepare.sh   # downloads regtest.py + builds Remglk into tests/remglk (git-ignored)
./tests/runtests.sh  # builds glk/ against Remglk, then runs regtest.py against both test games
```

`runtests.sh` builds the `glk/` directory standalone (`cmake -DGlkLibPath=remglk -S../glk/ -Bbuild/`), producing `tadsr`, then replays `.regtest` transcripts against it:
- `tests/ditch.gam.regtest` — TADS 2 test game (`ditch.gam`)
- `tests/ditch3.t3.regtest` — TADS 3 test game (`ditch3.t3`)

To run a single test manually after `prepare.sh`, use `python regtest.py -i <path-to-tadsr> <name>.regtest -t 20` from `tests/`. Only run `prepare.sh`/`runtests.sh` on Linux/WSL (they assume a POSIX shell, `curl`, `tar`, `make`, and `nproc`).

## Architecture

Three main pieces compose the runtime:

- **`tads2/`** — the legacy TADS 2 VM/compiler runtime (`run.c`, `dat.c`, `voc.c`, `execmd.c`, etc.), including OS-abstraction layers under `tads2/msdos/` (Windows) and `tads2/unix/`. Builds static libs `tr32h` (release runner) and `trd32h` (debugger-enabled runner), plus the `maketrx32` tool on Windows.
- **`tads3/`** — the TADS 3 VM/compiler (`vm*.cpp` = VM runtime, `tc*.cpp` = compiler front end). Produces several executables/libs:
  - `t3make` — the TADS 3 compiler
  - `t3res` — resource bundler
  - `mkchrtab` — character mapping table generator
  - `t3core` — static lib with the core VM (used by `t3run`)
  - `t3run` — standalone command-line interpreter
  - `t3htm` / `t3htm_d` — HTML TADS VM libs (release/debug), only built when `WITH_HTMLTADS` is on
- **`glk/`** — a Glk binding on top of `tads2`+`tads3` sources, producing the `tadsr` executable (the Glk-based interpreter used for both TADS 2 and TADS 3 games). Expects a compiled Glk library (e.g. Remglk) passed via the `GlkLibPath`/`GlkLibName` cache variables.

Cross-cutting notes:
- Platform selection is via `WIN32`/`EMSCRIPTEN`/else-Unix branches throughout the CMakeLists files, each pulling in different OS-abstraction (`os*.c`/`osunix*.c`/`oswin*.c`) source sets and compile definitions (`MICROSOFT`/`MSDOS`/`T_WIN32` vs `UNIX`/`OSANSI`/`LINUX_386`).
- `VM_WITH_NETWORK` (auto-on for Windows, or when libcurl is found elsewhere) toggles TADS's built-in networking (`vmnet*.cpp`, `osifcnet.cpp`); this pulls in a different builtin-function registration file (`vmbifregn.cpp`/`vmbifregxn.cpp` vs `vmbifreg.cpp`/`vmbifregx.cpp`).
- `tads_settings` (in `tads3/CMakeLists.txt`) is a shared `INTERFACE` target carrying common include dirs and compile definitions consumed by all `tads3` executables/libs — check it first when tracking down platform-specific defines.
- `TadsFunctions.cmake` provides helpers used by downstream consumers of this repo (e.g. game projects): `make_t3r`, `make_trx`, `em_package`, `build_game`.
