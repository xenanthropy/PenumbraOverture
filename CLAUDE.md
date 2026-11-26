# CLAUDE.md

Guidance for Claude (and any other assistant) working in this repository.

## How we work on this project

This is a collaborative modernization effort. The owner wants to **walk through
the project together, step by step**, rather than have large changes made
autonomously.

- Explain what you found and what you propose **before** changing code.
  Wait for a go-ahead on anything non-trivial.
- Prefer small, focused changes (one concern per commit / PR) so each step can
  be reviewed and understood.
- When something is a judgment call (e.g. which replacement library to use),
  lay out the options with a recommendation and let the owner decide.
- Say plainly what was verified (compiled, ran, tested) and what was not.
- Expect the build to fail for now. That is known and is the point of the
  work: there are real bugs and outdated dependencies to fix.
- **Game assets are not in this repository.** The owner will provide the
  Penumbra Overture data files separately. Never commit them.

## Goal

Get Penumbra Overture and its engine building cleanly on modern Linux and
Windows (and ideally macOS), fix bugs, and improve performance, while keeping
the original game behaviour.

## The three repositories

They are expected to be checked out **side by side** in one parent folder:

```
parent/
  PenumbraOverture/   <- this repo (the game)
  HPL1Engine/         <- the engine
  OALWrapper/         <- audio wrapper used by the engine
  dependencies/       <- (historical) prebuilt libs + headers bundle, NOT in any repo
```

`PenumbraOverture/CMakeLists.txt` pulls the other two in with
`add_subdirectory(../OALWrapper ...)` and `add_subdirectory(../HPL1Engine ...)`.

### PenumbraOverture (game code)

- ~60 flat `.cpp/.h` pairs at the repo root: player states, enemies
  (`GameEnemy_Dog/Spider/Worm`), inventory, notebook, menus, save system,
  map handling, game scripts glue, etc. `Main.cpp` / `Init.cpp` are the entry.
- Project files: `CMakeLists.txt` (Linux), `Penumbra.sln/.vcproj`
  (Visual Studio 2003), `PenumbraOverture.xcodeproj` (old Xcode, uses
  `SDLMain.m`).
- `*HaptX*` files are support for the old Novint Falcon haptic device; the
  engine side is stubbed / optional.
- Last real upstream change: 2010 (Frictional Games open-source release).
- `CMakeLists.txt` hard-codes link names: `HPL OALWrapper Newton angelscript
  fltk GL GLU Cg CgGL ogg vorbis vorbisfile openal alut SDL SDL_ttf SDL_image`,
  and looks for libs in `../dependencies/lib/linux`.

### HPL1Engine (engine, built as a static lib `HPL`)

- `include/` and `sources/` split by subsystem: `ai`, `game`, `graphics`,
  `gui`, `haptic`, `input`, `math`, `physics`, `resources`, `scene`, `sound`,
  `system`, and `impl/` (the concrete backends).
- Backends in `impl/`:
  - Window / input / 2D: **SDL 1.2** (+ SDL_image, SDL_ttf).
  - Rendering: fixed-function + extension OpenGL through **GLee**
    (`include/GL/GLee.h`, `sources/impl/GLee.c`), shaders in **NVIDIA Cg**
    (`CGProgram.cpp`, `.cg` files under `assets/core/programs`). Also ATI
    fragment-shader fallback materials (`Material_Fallback01/02_BaseLight`).
  - Physics: **Newton Game Dynamics 1.x** (old, closed-source-era API:
    `NewtonCreateBox(world, x,y,z, matrix)`, `NewtonBodySetAutoFreeze`, ...).
  - Scripting: **AngelScript ~2.7–2.12** era API (engine-level
    `AddScriptSection/Build/ExecuteString`, `asBEHAVE_ASSIGNMENT`, global
    operator behaviours). Modern AngelScript (2.3x) removed these.
  - Sound: OpenAL via OALWrapper.
  - Video: Theora (`VideoStreamTheora.cpp`, excluded from the CMake build).
  - Message boxes: **FLTK** (`fl_alert` in `LowLevelSystemSDL.cpp`, the only use).
  - XML: bundled TinyXML. Collada + `.msh` mesh loaders.
- Extras: `assets/` (engine core shaders, textures, fonts), `tests/`,
  `tools/` (map editor, model viewer, particle editor ...), not part of the
  CMake build.
- Last real code change: 2014 ("Updated code for new OALWrapper");
  2017 commits are whitespace-only.
- The engine `CMakeLists.txt` declares Newton / angelscript / SDLmain as
  IMPORTED static libs from `../dependencies/lib/linux/` and does not link
  anything itself; the game target does all linking.

### OALWrapper (audio, static lib)

- Thin C++ layer over OpenAL + Ogg/Vorbis: sources, buffers, streams,
  EFX effects/filters, WAV/Ogg loaders. `tests/` has small sample programs.
- Most modern of the three (Edward Rudd, 2014–2015): uses `find_package` for
  Ogg/Vorbis/OpenAL/SDL, has a `USE_SDL2` option, iOS/Emscripten paths.
- Compiles on GCC 13 today with only warnings.

## Known build state (Ubuntu 24.04, GCC 13.3, CMake 3.28)

Assessed with distro packages installed (SDL 1.2 via sdl12-compat, SDL_ttf,
SDL_image, OpenAL, Vorbis, GLU, FLTK 1.3, ALUT, Theora, nvidia-cg-toolkit).

- **CMake configure succeeds** with deprecation warnings
  (`cmake_minimum_required(VERSION 2.6/2.8)`). CMake 4.x will refuse these
  minimums outright, so they must be raised (>= 3.10).
- **OALWrapper builds.**
- **HPL1Engine fails.** Blockers, roughly in order of size:
  1. **Newton 1.x** is not available on any modern distro and only ever
     shipped as prebuilt (mostly 32-bit) binaries. `Newton.h` missing.
     Options: bundle an open-source Newton 1.53-compatible core (ScummVM's
     `engines/hpl1` port did exactly this), or port to a newer physics lib.
  2. **AngelScript** too old for any packaged version; `angelscript.h`
     missing. Either vendor the matching old version (needs 64-bit
     calling-convention support) or port the script bindings to a modern
     2.3x release (ScummVM again is a working reference using 2.35).
  3. **GLee vs. modern Mesa headers**: `glBindBufferARB`, `gl*FragmentShaderATI`,
     `glGenQueriesARB`, etc. "not declared". The system `GL/gl.h`/`glext.h`
     defines the extension guard macros first, so GLee skips its own
     declarations. Fix: replace GLee with GLEW / glad / libepoxy.
  4. **Cg** is discontinued by NVIDIA (no updates since 2012, x86 only, no
     ARM / Apple Silicon). It works today on x86 Linux via
     `nvidia-cg-toolkit`, but long term the shaders should move to GLSL.
  5. Plain C++ errors under modern GCC (easy):
     - `Renderer2D.cpp:898-923`: taking the address of a temporary
       (`&cVertex(...)`).
     - `SoundHandler.cpp:289`: `return false;` from a function returning a
       pointer.
     - `cGfxBufferCompare` (GraphicsDrawer) comparator `operator()` is not
       `const`, which libstdc++ now static-asserts on for `std::set`.
     - More are likely hidden behind the missing-header failures.
- **PenumbraOverture**: every game file includes `angelscript.h` via the
  engine's `system/Script.h`, so nothing compiles until (2) is resolved.
- Windows: only a Visual Studio 2003 project exists; CMake would need to
  become the cross-platform build (and `win32.cpp` / `<windows.h>` paths
  revisited).
- SDL 1.2 still works via sdl12-compat, but moving to SDL2 (or SDL3) is the
  right long-term step; OALWrapper already has a `USE_SDL2` switch.

## Useful reference

ScummVM merged an HPL1 / Penumbra Overture port (`engines/hpl1` in the
ScummVM repo, GPLv3) that already solved Newton, AngelScript 2.35 and many
64-bit / modern-compiler issues. It is a good place to compare fixes, while
keeping in mind it targets ScummVM's own platform layer, not SDL directly.

## Building (current, expected to fail)

```sh
# from the parent folder that contains all three repos
mkdir build && cd build
cmake ../PenumbraOverture
make -k -j"$(nproc)"
```
