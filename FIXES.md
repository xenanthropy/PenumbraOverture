# Fixes and Changes

This file records every change made to Penumbra Overture and its two
dependencies as part of the modernization effort. PenumbraOverture is the
main repository. Changes to HPL1Engine and OALWrapper are committed in those
repositories and logged here too, so this file is the single record of
everything that differs from the original Frictional Games release.

None of this is submitted upstream.

## How to add an entry

Add new entries at the top of the relevant section, one per fix:

```
### Short title
- **Repo:** HPL1Engine | OALWrapper | PenumbraOverture
- **Commit:** <short hash> on <branch>
- **Files:** path/to/file.cpp
- **Problem:** what was wrong (error message, crash, wrong behaviour)
- **Fix:** what was changed and why
- **Verified:** how it was checked (compiles, runs, tested in-game)
```

## Build system

ROADMAP Phase 2.

### Raise CMake minimums to 3.16
- **Repo / commit:** OALWrapper `bd2100c`, HPL1Engine `83f1d15`, PenumbraOverture `9b003eb` on `claude/project-thread-msxp3s`
- **Files:** CMakeLists.txt in each repo
- **Problem:** `cmake_minimum_required(2.6/2.8)` gives deprecation warnings on CMake 3.x and is rejected outright by CMake 4.x.
- **Fix:** Raised to 3.16. HPL1Engine and the game also pin C++11 (`CMAKE_CXX_STANDARD 11`) instead of using the compiler's default (gnu++17 on GCC 13).
- **Verified:** Configures with CMake 3.28 with no deprecation warnings. Build gives the same 70 GLee errors as before Phase 2 and nothing new; all 59 game files compile as C++11. No files are written into the sibling repos. Not linked or run yet.

### Add FindCg.cmake
- **Repo / commit:** HPL1Engine `77c7668` on `claude/project-thread-msxp3s`
- **Files:** cmake/FindCg.cmake
- **Problem:** Cg has no standard CMake find module, so it was hard-coded by name in the link list.
- **Fix:** New find module that locates `Cg/cg.h`, `Cg` and `CgGL` (Linux packages, the NVIDIA toolkit on Windows, the macOS framework) and provides a `Cg::Cg` target.
- **Verified:** Configures with CMake 3.28 with no deprecation warnings. Build gives the same 70 GLee errors as before Phase 2 and nothing new; all 59 game files compile as C++11. No files are written into the sibling repos. Not linked or run yet.

### HPL target finds and links its own dependencies
- **Repo / commit:** HPL1Engine `3b01ef3` on `claude/project-thread-msxp3s`
- **Files:** CMakeLists.txt
- **Problem:** The `HPL` library linked nothing itself; the game hard-coded ~17 library names on its behalf and pointed at `../dependencies` for headers and libs.
- **Fix:** `find_package` for OpenGL, SDL, SDL_image, SDL_ttf and Cg; includes and libraries are `PUBLIC` on the `HPL` target. Unused `alut`, and `ogg`/`vorbis` (OALWrapper already links those), are dropped. AngelScript and Newton still come from the prebuilt bundle, now found through the `HPL_DEPENDENCIES_DIR` cache option (default `../dependencies`) until Phases 4 and 5.
- **Verified:** Configures with CMake 3.28 with no deprecation warnings. Build gives the same 70 GLee errors as before Phase 2 and nothing new; all 59 game files compile as C++11. No files are written into the sibling repos. Not linked or run yet.

### Build output written into sibling repos
- **Repo / commit:** PenumbraOverture `7c9bd45` on `claude/project-thread-msxp3s`
- **Files:** CMakeLists.txt
- **Problem:** `add_subdirectory(../OALWrapper ../OALWrapper)` put the binary dirs next to the build dir. With the build dir in the same parent folder, objects, Makefiles and `libHPL.a` landed inside the HPL1Engine and OALWrapper source trees.
- **Fix:** Binary dirs are now `${CMAKE_BINARY_DIR}/OALWrapper` and `${CMAKE_BINARY_DIR}/HPL1Engine`.
- **Verified:** Configures with CMake 3.28 with no deprecation warnings. Build gives the same 70 GLee errors as before Phase 2 and nothing new; all 59 game files compile as C++11. No files are written into the sibling repos. Not linked or run yet.

### Game links only to HPL
- **Repo / commit:** PenumbraOverture `ce569b0` on `claude/project-thread-msxp3s`
- **Files:** CMakeLists.txt
- **Problem:** Long hard-coded link list and `link_directories(../dependencies/lib/linux)`.
- **Fix:** `target_link_libraries(Overture HPL)`; everything else comes through HPL. Output name stays `overture.bin`.
- **Verified:** Configures with CMake 3.28 with no deprecation warnings. Build gives the same 70 GLee errors as before Phase 2 and nothing new; all 59 game files compile as C++11. No files are written into the sibling repos. Not linked or run yet.

### Remove FLTK
- **Repo / commit:** HPL1Engine `927c470` on `claude/project-thread-msxp3s`
- **Files:** sources/impl/LowLevelSystemSDL.cpp, CMakeLists.txt
- **Problem:** A whole GUI toolkit was linked for a single `fl_alert()` error popup on Linux/macOS.
- **Fix:** The message is printed to stderr instead (converted with `cString::To8Char`). Windows keeps `MessageBox`. An SDL2 message box can replace it later.
- **Verified:** Builds with `libfltk1.3-dev` uninstalled. Not run yet, so the stderr message itself is untested.

### Remove the GLaux.lib pragma
- **Repo / commit:** HPL1Engine `7f311e0` on `claude/project-thread-msxp3s`
- **Files:** sources/impl/LowLevelGraphicsSDL.cpp
- **Problem:** `#pragma comment(lib, "GLaux.lib")` made MSVC builds look for a deprecated library that isn't in the dependency bundle (Frictional forum).
- **Fix:** Removed; nothing uses GLaux.
- **Verified:** Compiles on GCC (the pragma only matters to MSVC). Not verified on Windows yet.


## Compiler / C++ fixes

ROADMAP Phase 1. After these, the only HPL1Engine errors left are the GLee
ones (Phase 3), and all PenumbraOverture files compile.

### Sort comparators must be const (Renderer2D, GuiSet)
- **Repo:** HPL1Engine
- **Commit:** `7382405` on `claude/project-thread-msxp3s`
- **Files:** include/graphics/Renderer2D.h, sources/graphics/Renderer2D.cpp, include/gui/GuiSet.h, sources/gui/GuiSet.cpp
- **Problem:** `static assertion failed: comparison object must be invocable as const` for `cRenderObject2DCompare`, `cRenderTransObjectCompare` and `cGuiRenderObjectCompare`. These were hidden behind earlier errors in the baseline.
- **Fix:** Marked each `operator()` as `const`. They only read their arguments, so behaviour is unchanged.
- **Verified:** Compiles with GCC 13.3 (Ubuntu 24.04, x86_64): the error no longer appears in the HPL1Engine build. Not yet run (needs the rest of the roadmap).

### Ambiguous pow() in SDLTexture (MSVC)
- **Repo:** HPL1Engine
- **Commit:** `a8a12da` on `claude/project-thread-msxp3s`
- **Files:** sources/impl/SDLTexture.cpp
- **Problem:** `(int)pow((float)2,(int)mlSizeLevel)` is an ambiguous overload on MSVC (reported on the Frictional forum).
- **Fix:** Replaced with `1 << mlSizeLevel`. This gives the same value for the texture size divisor, without floating point.
- **Verified:** Compiles with GCC 13.3 (GCC never flagged it). The MSVC side is not verified yet (Phase 7).

### cGfxBufferCompare must be const
- **Repo:** HPL1Engine
- **Commit:** `b865d58` on `claude/project-thread-msxp3s`
- **Files:** include/graphics/GraphicsDrawer.h, sources/graphics/GraphicsDrawer.cpp
- **Problem:** `static assertion failed: comparison object must be invocable as const` (modern libstdc++ requires const comparators for std::multiset).
- **Fix:** Marked `cGfxBufferCompare::operator()` as `const`. It doesn't modify anything.
- **Verified:** Compiles with GCC 13.3 (Ubuntu 24.04, x86_64): the error no longer appears in the HPL1Engine build. Not yet run (needs the rest of the roadmap).

### symlink not declared
- **Repo:** HPL1Engine
- **Commit:** `7a5f7d9` on `claude/project-thread-msxp3s`
- **Files:** sources/impl/LowLevelSystemSDL.cpp
- **Problem:** `'symlink' was not declared in this scope` (line 635).
- **Fix:** Added `#include <unistd.h>` in the non-Windows include block.
- **Verified:** Compiles with GCC 13.3 (Ubuntu 24.04, x86_64): the error no longer appears in the HPL1Engine build. Not yet run (needs the rest of the roadmap).

### PlayStream returns false instead of NULL
- **Repo:** HPL1Engine
- **Commit:** `bed2163` on `claude/project-thread-msxp3s`
- **Files:** sources/sound/SoundHandler.cpp
- **Problem:** `cannot convert 'bool' to 'hpl::iSoundChannel*' in return` (line 289).
- **Fix:** `return NULL;` when the stream can't be loaded. Old compilers silently turned `false` into a null pointer, so behaviour is the same.
- **Verified:** Compiles with GCC 13.3 (Ubuntu 24.04, x86_64): the error no longer appears in the HPL1Engine build. Not yet run (needs the rest of the roadmap).

### Address of a temporary matrix in PhysicsBodyNewton
- **Repo:** HPL1Engine
- **Commit:** `ec8578a` on `claude/project-thread-msxp3s`
- **Files:** sources/impl/PhysicsBodyNewton.cpp
- **Problem:** `taking address of rvalue` (line 104): `&apEntity->GetLocalMatrix().GetTranspose().m[0][0]`.
- **Fix:** Store the transposed matrix in a local variable and pass its address. Newton copies the matrix, so behaviour is unchanged.
- **Verified:** Compiles with GCC 13.3 (Ubuntu 24.04, x86_64): the error no longer appears in the HPL1Engine build. Not yet run (needs the rest of the roadmap).

### Address of temporary vertices in Renderer2D
- **Repo:** HPL1Engine
- **Commit:** `b066bc1` on `claude/project-thread-msxp3s`
- **Files:** sources/graphics/Renderer2D.cpp
- **Problem:** `taking address of rvalue` (lines 898–923): `AddVertexToBatch(&cVertex(...))` in the 2D shadow code.
- **Fix:** Build each vertex in a local variable first. `AddVertexToBatch` copies the data straight away, so behaviour is unchanged. Matches Kristian-Popov's 2017 fix.
- **Verified:** Compiles with GCC 13.3 (Ubuntu 24.04, x86_64): the error no longer appears in the HPL1Engine build. Not yet run (needs the rest of the roadmap).

### Include <functional> for MSVC 2012+
- **Repo:** PenumbraOverture
- **Commit:** `47e2f0a` on `claude/project-thread-msxp3s`
- **Files:** TriggerHandler.h, MainMenu.cpp
- **Problem:** Visual Studio 2012+ fails with ~600 errors because `std::` functional pieces moved headers (Frictional forum, 2013).
- **Fix:** Added `#include <functional>` to both files. Harmless on GCC/Clang.
- **Verified:** Compiles with GCC 13.3. The MSVC failure itself is not verified yet (Windows comes in Phase 7).


## Dependencies

### AngelScript 2.7.1b built from source (portability mode)
- **Repo / commits:** HPL1Engine on `claude/project-thread-msxp3s`: `4e6e58b` (vendor, unmodified),
  `a60f781` (GCC fix), `062df1a` (CMake target), `eaa0dde` (stdcall warnings)
- **Files:** new `external/angelscript/` (2.7.1b `include/` + `source/`,
  `LICENSE`, `README.md`, `CMakeLists.txt`); `CMakeLists.txt`;
  `include/system/Script.h`
- **Problem:** the engine depended on a prebuilt 32-bit `libangelscript.a`
  from the old bundle. The stock 32-bit build's native-call assembly
  (`as_callfunc_x86.cpp`) pushes arguments without keeping the stack 16-byte
  aligned. That is the root cause of the AMD/Mesa "coin in the vice" crash
  (see KNOWN_ISSUES.md).
- **Fix:** vendored the original 2.7.1b source from AngelScript's SourceForge
  SVN (`branches/2.7.1`, r90; zlib licence). One local change: the
  `return 0;` → `return asUPtr();` GCC fix in `angelscript.h`, marked
  with an `HPL1Engine:` comment as the zlib licence requires. It is built as a
  static library with `AS_MAX_PORTABILITY` as a **public** definition, so
  the engine and the game also register script functions through generic
  wrappers (`Script.h` already had them). This is required on 64-bit, where
  2.7.1b has no native calls, and avoids the misaligned-stack bug on 32-bit.
  The prebuilt AngelScript is no longer used. Also: `Script.h` now only uses
  `__attribute__((stdcall))` on 32-bit x86 (173 warnings gone).
- **Verified:** builds on GCC 13 x86_64 with zero errors; the game reaches the
  final link, and the only unresolved part is the 32-bit `libNewton.a`.
  `AS_MAX_PORTABILITY` is present in both the HPL and game compile flags.
  Not run yet. In portability mode scripts use `scriptstring` instead of
  `stdstring` (see KNOWN_ISSUES.md), so watch for script compile or bind
  errors in the log on first run.

### Newton 1.53 layer: matrix copies overran by 8 floats on GCC/Clang
- **Repo / commit:** HPL1Engine `d5282e1`
- **Files:** `external/newton-1.53/newton-4/source/ndMatrix.h`,
  `external/newton-1.53/NewtonBody.cpp`, `external/newton-1.53/NewtonCollision.cpp`
- **Problem:** memory corruption. Under the Itanium C++ ABI (GCC, Clang)
  `ndMatrix` is 96 bytes with `m_front` at offset 16, because it and its
  `ndVector` members share the empty base class `ndClassAlloc`; MSVC (which
  upstream is tested on) makes it 64 bytes. Three places copied
  `sizeof(ndMatrix)/sizeof(float)` = 24 floats instead of 16:
  building a matrix from a float array (used by almost every API call that
  takes a matrix), `NewtonBodyGetMatrix` and
  `NewtonConvexHullModifierGetMatrix`. Each wrote 32 bytes past its buffer.
  glibc's `_FORTIFY_SOURCE` (on by default on Ubuntu) aborted with "buffer
  overflow detected" / "stack smashing detected".
- **Fix:** copy exactly 16 floats. Checked the rest of the layer and core for
  the same pattern; only these three.
- **Verified:** a guard-word test around `NewtonBodyGetMatrix` shows no
  overrun; the step-4 test program runs cleanly with fortify enabled.

### Newton 1.53 layer: the 12 missing functions (Phase 5, step 4)
- **Repo / commits:** HPL1Engine `997f6cf` (per-body freeze thresholds),
  `605b994` (the functions)
- **Files:** new `external/newton-1.53/NewtonHPL1Engine.cpp`; `Newton.h`
  (declarations block at the end); `NewtonBody.h`, `NewtonBody.cpp`;
  `newton-4/source/ndBodyDynamic.h` (one method private → protected)
- **Problem:** 12 functions HPL1Engine calls were declared by the layer but
  never defined (their old code is inside `#if 0`), so the engine couldn't
  link. Four weren't even declared any more.
- **Fix:** each one follows what real 1.53 did, using the Newton 2.36 source
  (ScummVM) where 1.53's isn't available:
  - `NewtonCollisionCollide`: shape-vs-shape contacts via Newton 4's
    `ndContactSolver`, the same way Newton 4 does its own shape queries.
    Normal points from B to A and depth is positive, which is what HPL's
    `CheckShapeCollision` push-out expects (character controller, ladders,
    areas, weapon hits).
  - `NewtonBall/Hinge/Slider/CorkscrewGetJointForce`: the net linear force
    the joint applies to the child body, excluding the slider/corkscrew
    pin direction, which matches 1.53's sum of its base-row forces. HPL
    compares its length to each joint's break force.
  - `NewtonBallGetJointAngle/Omega`: as 2.36 (relative Euler angles; relative
    angular velocity in the joint frame).
  - `NewtonMaterialGetContactForce`: normal + friction forces of the contact
    point, as 1.53's own code did.
  - `NewtonBodySetFreezeTreshold`: per-body speed²/omega² limits, applied by a
    small body subclass in Newton 4's sleep test. HPL's default (0.01, i.e.
    0.1 m/s) is much looser than Newton 4's own (0.032 m/s), so without this
    objects would keep creeping where 1.53 stopped them.
  - `NewtonSetSolverModel`: exact → 16 solver iterations, adaptive → 8,
    n passes → n. These two numbers are tuning knobs for feel testing.
  - `NewtonSetFrictionModel`, `NewtonMaterialSetContinuousCollisionMode`:
    no Newton 4 equivalent; no-ops.
- **Verified:** `newton153` builds on GCC 13; every Newton function HPL calls
  is now defined in the library (symbol check), and HPL's 11 Newton files
  compile against the layer's `Newton.h`. A scratch test program (not
  committed yet; it becomes step 9) passes: sphere/box, box/box and
  sphere/level-mesh collisions give the expected normals (+Y) and depths
  (0.200 / 0.101 / 0.150); a 1 kg body hanging from a hinge, a ball joint and
  a slider reports 9.81 N on each; a resting ball joint reports ~0 angle and
  omega; a box sliding at 6 cm/s freezes with HPL's threshold and keeps
  sliding without it. **Not verified:** contact force (needs the contact
  callbacks, step 7), and anything in-game.

### Newton 1.53 layer vendored and building (Phase 5, steps 1–3)
- **Repo / commits:** HPL1Engine on `claude/project-thread-msxp3s`: `42a464d` (vendor,
  unmodified), `34c2977` (CMake target), `c608a72` (GCC fixes)
- **Files:** new `external/newton-1.53/` (Julio Jerez's 1.53 layer from
  JulioJerez/newton-dynamics `294f27f6cde10c3c269041c3c4944cefb0480dbf`,
  `newton-4.00/applications/toolsAndWrapers/newton-1.53`, without `demos/`
  and the MSVC-only CMake; includes its Newton 4 core, about 147k lines;
  plus `LICENSE` (zlib), `README.md`, our own `CMakeLists.txt`);
  `CMakeLists.txt`; in the layer: `NewtonUtils.h`, `NewtonVehicle.cpp`,
  `joints/NewtonJoints.cpp`
- **Problem:** Newton 1.53 only exists as a prebuilt 32-bit `libNewton.a`
  with no source, so a 64-bit build can't link.
- **Fix (so far):** vendored the layer and added a `newton153` static library
  target (C++17 for that target only; `D_SCALAR_VECTOR_CLASS` on GCC/Clang as
  upstream; warnings off; optimised even in Debug; links Threads). Fixed 3
  GCC errors with no behaviour change: MSVC `_finite`/`_isnan` →
  `std::isfinite`/`std::isnan`; an extra class qualifier on
  `NewtonVehicleNotify::PostTransformUpdate`; a temporary `ndSharedPtr`
  bound to a non-const reference in `NewtonJointUserData::GetUserData`.
  The target is `EXCLUDE_FROM_ALL` and **not linked into HPL yet**.
- **Verified:** `make newton153` builds a 64-bit (`elf64-x86-64`) static
  library on GCC 13. The rest of the build is unchanged (still fails only at
  the link, on the 32-bit `libNewton.a`). A symbol check against every
  Newton function HPL calls finds exactly the 12 expected missing
  definitions (see ROADMAP Phase 5, step 4). Not run: no physics has been
  simulated yet.

### Replace GLee with GLEW
- **Repo / commits:** HPL1Engine on `claude/project-thread-msxp3s`: `a45a8c5` (includes), `8bc2418`
  (glewInit), `eb2e44c` (extension checks), `f1646cf` (CMake),
  `a80cdcc` (remove GLee), `73c5ab8` (GLX/WGL headers)
- **Files:** `include/impl/LowLevelGraphicsSDL.h`, `include/impl/PBuffer.h`,
  `include/impl/CGProgram.h`, `include/impl/SDLTexture.h`,
  `sources/impl/LowLevelGraphicsSDL.cpp`, `sources/impl/VertexBufferVBO.cpp`,
  `sources/impl/VertexBufferOGL.cpp`, `sources/impl/OcclusionQueryOGL.cpp`,
  `sources/impl/SDLTexture.cpp`, `sources/graphics/Material_Fallback01/02_BaseLight.cpp`,
  `CMakeLists.txt`; deleted `include/GL/GLee.h`, `sources/impl/GLee.c`
  (about 36k lines).
- **Problem:** about 70 "not declared" errors (`glBindBufferARB`,
  `gl*FragmentShaderATI`, `glGenQueriesARB`, ...). GLee checks the
  `__glext_h_` guard, which modern Khronos/Mesa `glext.h` no longer sets the
  same way, so GLee skipped its own declarations. GLee is also abandoned.
- **Fix:** include `<GL/glew.h>` instead. `GLeeInit()` becomes `glewInit()`
  (logs the GLEW version, or GLEW's error string on failure). The 16 `GLEE_*`
  extension checks are renamed one-to-one to `GLEW_*`. The vsync checks
  become `GLXEW_SGI_swap_control` / `WGLEW_EXT_swap_control`. The headers
  that use GLX/WGL types now include `glxew.h` / `wglew.h`, which GLee used to
  pull in implicitly. CMake finds and links `GLEW::GLEW` (needs
  `libglew-dev`).
- **Verified:** HPL1Engine now compiles with **zero errors** on GCC 13
  (x86_64), and the game links `libHPL.a`. The only remaining link errors
  come from the 32-bit prebuilt `libNewton.a` (and the 32-bit
  `libangelscript.a`, if that is used). With a scratch 64-bit AngelScript
  build, Newton is the only thing missing. Not run yet: the extension
  detection still needs checking against the retail game's log at runtime.


## Gameplay and engine bugs

_No entries yet._

## Performance

_No entries yet._

## Documentation

### Physics feel priority in CLAUDE.md
- **Repo / commit:** PenumbraOverture `8380476`
- **Files:** CLAUDE.md
- **Change:** recorded that the physics must feel as close to Newton 1.53 as
  possible, and that 1.53 behaviour wins over Newton 4 defaults.

### Add CLAUDE.md, FIXES.md, KNOWN_ISSUES.md, ROADMAP.md
- **Repo:** PenumbraOverture
- **Files:** CLAUDE.md, FIXES.md, KNOWN_ISSUES.md, ROADMAP.md
- **Change:** Added collaboration notes, an analysis of the three repos,
  this change log, the catalogue of known issues, and the roadmap.
