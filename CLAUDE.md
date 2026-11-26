# CLAUDE.md

Guidance for Claude (and any other assistant) working in this repository.

## How we work on this project

This is a collaborative modernization effort. The owner wants to **walk through
the project together, step by step**, rather than have large changes made
autonomously.

- Explain what you found and what you propose **before** changing code.
  Wait for a go-ahead on anything non-trivial.
- Prefer small, focused changes (one concern per commit) so each step can
  be reviewed and understood.
- When something is a judgment call (e.g. which replacement library to use),
  lay out the options with a recommendation and let the owner decide.
- Say plainly what was verified (compiled, ran, tested) and what was not.
- Expect the build to fail for now. That is known and is the point of the
  work: there are real bugs and outdated dependencies to fix.
- **Game assets are not in this repository.** The owner will provide the
  Penumbra Overture data files separately. Never commit them.
- **Never open pull requests for this work**, in this repo, HPL1Engine or
  OALWrapper, and never propose anything upstream to the original Frictional
  Games repositories. Work stays on branches in the owner's forks. If the
  owner wants to send a bug fix upstream, they will make that PR themselves,
  manually.
- **The same goes for Newton Dynamics.** We vendor Julio Jerez's Newton 1.53
  compatibility layer and finish it locally. **Never contribute anything to
  the newton-dynamics repository**: no PRs, issues, discussion posts or
  comments. Our completion of that layer stays local to these forks.

## Repository layout and change log

- **PenumbraOverture is the main repository.** HPL1Engine and OALWrapper are
  its dependencies.
- When a fix belongs in HPL1Engine or OALWrapper, commit it in that repo on
  its own working branch (no PRs, as above).
- **Every fix, in any of the three repos, gets an entry in
  [`FIXES.md`](FIXES.md)** in this repo: which repo, commit, files, the
  problem, the fix, and how it was verified. Update FIXES.md in the same step
  as the fix.
- The ordered plan of work is in [`ROADMAP.md`](ROADMAP.md).
- Background, the baseline build errors, known runtime bugs and the
  dependency situation are catalogued in [`KNOWN_ISSUES.md`](KNOWN_ISSUES.md).

## Goal

The project is called **PenumbraRevival**. Overture comes first; the aim is
to get all three Penumbra games (Overture, Black Plague, Requiem) working.
Avoid names that sound like the retail release. Each game keeps its user
data in its own folder under `PenumbraRevival/` (`Overture/`,
`Black Plague/`, `Requiem/`), mirroring the retail `Penumbra/Overture`
layout, so their `hpl.log`, settings and saves never collide.

Get Penumbra Overture and its engine building cleanly on modern Linux and
Windows (and ideally macOS), fix bugs, and improve performance, while keeping
the original game behaviour.

### Physics feel comes first

The move from Newton 1.53 to Julio's compatibility layer on Newton 4 (Phase 5)
must keep the **feel of Penumbra's physics as close to the original as
possible**. This is the owner's top priority for the physics work. Very slight
differences are acceptable; significant ones (objects sliding, bouncing or
stacking differently, door/lever weight, how the player pushes and throws
things) are not.

- When the layer's Newton 4 defaults and real 1.53 behaviour disagree, choose
  **fidelity to 1.53** (e.g. substeps, clamps, callback order, limits).
- Call out any change that could affect feel, and say whether it was checked.
- If fidelity can't be reached on Newton 4, the documented fallback is the
  Newton 2.36 / ScummVM route (see ROADMAP.md, Phase 5).

## Performance testing

Once the game runs, we profile and optimise it. The baseline to compare
against is the **retail Steam Linux port** of Penumbra Overture, played by
the owner, in whatever areas they can reach in both builds. Compare like
with like: same machine, same settings, same spot in a map. Use frame
time/FPS and load times as the main measures.

Don't set up a 32-bit "original" build environment for this. Retail vs.
our build is the comparison the owner wants, even though the retail port was
built by someone else with unknown settings.

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
- `CMakeLists.txt` links only `HPL` (since Phase 2); the engine target brings
  in everything else.

### HPL1Engine (engine, built as a static lib `HPL`)

- `include/` and `sources/` split by subsystem: `ai`, `game`, `graphics`,
  `gui`, `haptic`, `input`, `math`, `physics`, `resources`, `scene`, `sound`,
  `system`, and `impl/` (the concrete backends).
- Backends in `impl/`:
  - Window / input / 2D: **SDL 1.2** (+ SDL_image, SDL_ttf).
  - Rendering: fixed-function + extension OpenGL through **GLEW** (GLee was
    replaced in Phase 3), shaders in **NVIDIA Cg**
    (`CGProgram.cpp`, `.cg` files under `assets/core/programs`). Also ATI
    fragment-shader fallback materials (`Material_Fallback01/02_BaseLight`).
  - Physics: **Newton Game Dynamics 1.x** (old, closed-source-era API:
    `NewtonCreateBox(world, x,y,z, matrix)`, `NewtonBodySetAutoFreeze`, ...).
  - Scripting: **AngelScript ~2.7–2.12** era API (engine-level
    `AddScriptSection/Build/ExecuteString`, `asBEHAVE_ASSIGNMENT`, global
    operator behaviours). Modern AngelScript (2.3x) removed these.
  - Sound: OpenAL via OALWrapper.
  - Video: Theora (`VideoStreamTheora.cpp`, excluded from the CMake build).
  - Message boxes: FLTK was removed in Phase 2; non-Windows prints to stderr.
  - XML: bundled TinyXML. Collada + `.msh` mesh loaders.
- Extras: `assets/` (engine core shaders, textures, fonts), `tests/`,
  `tools/` (map editor, model viewer, particle editor ...), not part of the
  CMake build.
- Last real code change: 2014 ("Updated code for new OALWrapper");
  2017 commits are whitespace-only.
- Since Phase 2 the `HPL` target finds and links its own dependencies
  (`find_package` + `cmake/FindCg.cmake`). AngelScript 2.7.1b is vendored in
  `external/angelscript` (Phase 4, always `AS_MAX_PORTABILITY`). Newton is
  Julio Jerez's 1.53 compatibility layer on the Newton 4 core, vendored and
  completed in `external/newton-1.53` (Phase 5; local changes listed in its
  README). Since Phase 6, `-DHPL1_NEWTON_CORE=236` builds ScummVM's Newton
  2.36 core (`external/newton-2.36`, with a shim for the same 1.53 API)
  instead; Newton 4 is still the default. Nothing comes from a prebuilt
  bundle any more.

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
     missing. The original 2.7.1b source is on SourceForge SVN
     (`branches/2.7.1`) and builds with a one-line fix (see KNOWN_ISSUES.md). Either vendor the matching old version (needs 64-bit
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

## Building

Since Phase 5 the whole game builds and links on Linux (GCC 13, x86_64)
with no prebuilt bundle: AngelScript and Newton are built from the vendored
sources in `HPL1Engine/external`. It has not been run with game data yet
(Phase 6). Use a build folder that is **not** inside any of the three repos.

```sh
mkdir build && cd build
cmake /path/to/PenumbraOverture -DCMAKE_BUILD_TYPE=Release
make -j"$(nproc)"
```

The game binary is `overture.bin` in the build folder.
It keeps its settings, saves and `hpl.log` in
`~/.local/share/PenumbraRevival/Overture/` (Linux;
`Documents/PenumbraRevival/Overture/` on Windows), not in the retail game's
`~/.frictionalgames` folder, so the two never overwrite each other.

### Physics test

A headless test checks the Newton 1.53 layer the way the engine uses it
(82 checks: collisions, joints and their limit callbacks, sleeping,
impulses, materials and contact callbacks, the world box, the NaN guard,
the spin cap, level meshes and continuous collision). It draws nothing, so it gives
the same result on any machine; it doesn't replace playing the game. It is
not part of the normal build. From the build folder:

```sh
make newton153_test && ./HPL1Engine/external/newton-1.53/newton153_test
```

Exit code 0 means every check passed. Run it after any change to
`HPL1Engine/external/newton-1.53`. In a build configured with
`-DHPL1_NEWTON_CORE=236`, the same checks run against the 2.36 core with
`make newton236_test && ./HPL1Engine/external/newton-2.36/newton236_test`;
run that after any change to `HPL1Engine/external/newton-2.36` (it fails
five sleep checks, see its README, and three checks for fixes made only in
the Newton 4 layer: push-out, beam side faces, and bodies resting at a joint
limit). The original 32-bit 1.53 library can
run the same test as a reference (test only; see the newton-1.53 README).
The source is
`HPL1Engine/external/newton-1.53/tests/newton153_test.cpp`.

System packages (Ubuntu names): `libsdl1.2-dev libsdl-image1.2-dev
libsdl-ttf2.0-dev libopenal-dev libvorbis-dev libogg-dev libglu1-mesa-dev
libglew-dev nvidia-cg-toolkit`. FLTK and ALUT are no longer needed. On Arch
(and CachyOS), `nvidia-cg-toolkit` is in the official `extra` repo.
