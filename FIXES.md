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

### OALWrapper Find modules: package name mismatch warnings
- **Repo / commit:** OALWrapper `08aa2f2`
- **Files:** `CMakeScripts/FindVorbis.cmake`, `CMakeScripts/FindVorbisFile.cmake`
- **Problem:** configuring printed two CMake author warnings: the modules
  passed `VORBIS` / `VORBISFILE` to `find_package_handle_standard_args`,
  which doesn't match the package names `Vorbis` / `VorbisFile` (reported by
  the owner on Arch Linux).
- **Fix:** pass `Vorbis` / `VorbisFile`. CMake still sets the upper-case
  `VORBIS_FOUND` / `VORBISFILE_FOUND` as well; nothing in the three repos
  reads them.
- **Verified:** a fresh configure (CMake 3.28) shows "Found Vorbis" and
  "Found VorbisFile" with no warnings, and the whole game builds.

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

### Remaining compiler warnings cleared
- **Repos / commits:** HPL1Engine `233e7dc`, OALWrapper `93a253d`, PenumbraOverture `2f72449`, all on `claude/project-thread-msxp3s`
- **Files:** HPL1Engine sources/graphics/Renderer3D.cpp, sources/gui/GuiSkin.cpp; OALWrapper sources/OAL_OggSample.cpp; PenumbraOverture Init.cpp
- **Problem:** warnings Xenan saw in a clean build: `NULL` passed to the `int` argument of `RenderDebugObject` and to the `cVector3f` fill value in `GuiSkin` (`-Wconversion-null`); `size_t` narrowed into `ogg_int64_t` in a brace initialiser (`-Wnarrowing`); `"Initializing "PRODUCT_NAME` read as a literal suffix (`-Wliteral-suffix`).
- **Fix:** `0`, `cVector3f(0)`, an explicit `(ogg_int64_t)` cast, and spaces around the macros. Same values as before. The other warnings in that build were real bugs, logged under "Gameplay and engine bugs".
- **Verified:** a clean Release build (GCC 13, x86_64, default warning flags) of all three repos, vendored code included, now prints **zero warnings**. Not checked with `-Wall`, which would show more.

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

### Newton 1.53 layer: sleep the way the 1.53 library does (replaces the 10-step count)
- **Repo:** HPL1Engine
- **Commit:** `49d2484` on `claude/project-thread-msxp3s`
- **Files:** external/newton-1.53/NewtonBody.h, NewtonBody.cpp, NewtonHPL1Engine.cpp, newtonWorld.cpp, newton-4/source/ndBodyKinematic.h, newton-4/source/ndDynamicsUpdate.cpp, README.md, tests/newton153_test.cpp; external/newton-2.36/README.md
- **Problem:** the 10-step quiet count (`7bb4dc3`, below) came from reading 1.53's documentation. Running the original 32-bit 1.53 library (test only) showed it was wrong. 1.53 ignores `framesCount`. It rests a quiet body at once, but keeps its velocity and skips the step. Newton 4 zeroed the velocity, which is what made the boat lamp stop dead. 1.53 also doesn't wake a resting body when its velocity is set (the layer did), and `NewtonWorldFreezeBody` keeps the velocity (the layer cleared it).
- **Fix:** `ndNewtonBodyDynamic` applies the measured rule. A body is quiet when its start-of-step speed² and omega² are under 0.022 × max(t, 0.1), and its solved accel² and alpha² are under 2.64 × max(t, 0.01), where t is the `NewtonBodySetFreezeTreshold` value (1.53 default 0.01). When its whole island is quiet, it rests at its pre-step position and velocity. `NewtonWorldUnfreezeBody` only wakes. `SetVelocity`/`SetOmega` don't wake a resting body, but make an awake one take its next step. Small core change: `DetermineSleepStates` calls a new virtual `OnIslandRest` instead of zeroing velocities.
- **Feel:** resting and waking now follow 1.53. In the boat's sway, a lamp hanging almost still waits for the tilt to build before moving, as in 1.53: it stands still 42 of 600 steps in the test (1.53: 44, previous layer: 8). Props keep their velocity through a rest instead of losing it. Bodies that HPL sets a velocity on while they're resting stay put, as in 1.53.
- **Verified:** sleep checks rewritten from the library's measured behaviour. 73/73 pass on Newton 4. The original library passes all except the three safety nets this layer adds. The full game builds. In the headless boat scene the two-body lamp chain still swings less lively on Newton 4 than on 1.53, even with sleeping off, so that difference lies elsewhere and is being looked into. Not yet tried in game. The 2.36 option still uses the old count and fails 5 of the new sleep checks (it's now for comparison only).

### Newton layers: NewtonBodySetMatrix drops scale, as 1.53 did (missing boat notebook)
- **Repo:** HPL1Engine
- **Commit:** `607d960` on `claude/project-thread-msxp3s`
- **Files:** external/newton-1.53/NewtonBody.cpp, external/newton-1.53/README.md, external/newton-1.53/tests/newton153_test.cpp, external/newton-2.36/newton153_on_236.cpp
- **Problem:** the notebook in the boat cabin was missing, so the level exit never unlocked. Its map node (`_ref_item_notebook_notebook`) carries a scale of (0.028, 0.223, 0.169), and HPL's Collada loader puts that scale into the entity's matrix. HPL passes the matrix to `NewtonBodySetMatrix`. The original 1.53 library makes the rotation orthonormal there, so the scale disappears. Both our layers kept it, so the notebook's body, and the mesh that follows it, were shrunk to a few millimetres. 2.36 also produced NaNs from it.
- **Fix:** both layers now orthonormalise the matrix exactly as 1.53 does: front normalised, right = front × up (normalised), up = right × front. Worked out and checked by running the original 32-bit 1.53 library on scaled and skewed matrices.
- **Feel:** none for bodies already placed with clean matrices. Bodies placed with a scale now behave as in 1.53.
- **Verified:** for a scaled matrix, a skewed matrix and a non-orthogonal rotation, both layers give the same result as the original library to 4 decimal places. New layer check (69/69 pass on Newton 4 and 2.36; the original 1.53 passes it too). Not yet tried in game.

### Newton 2.36 core as a build option (Phase 6)
- **Repo:** HPL1Engine
- **Commit:** `483cab2` (core and switch), `9d13d76` (test) on `claude/project-thread-msxp3s`
- **Files:** CMakeLists.txt, external/newton-2.36/ (new: core236/ from ScummVM, newton153_on_236.cpp, include/Newton.h, CMakeLists.txt, README.md), external/newton-1.53/tests/newton153_test.cpp
- **Problem:** some of Penumbra's physics can't be matched on Newton 4 without rewriting its contact code. The clearest case is the boat lamp: Newton 4 treats level mesh faces as one-sided, so a push sweeps the chain along the ceiling beam toward the locker, where retail stops it. Newton 2.36 is much closer to 1.53 (two-sided thin plates, 1.x sleep thresholds, solver models and joint readouts), and is the documented fallback in ROADMAP Phase 5.
- **Fix:** `-DHPL1_NEWTON_CORE=236` builds ScummVM's Newton 2.36 (commit `695524fef`) with a shim that provides the 1.53 API instead of the Newton 4 layer. Newton 4 stays the default. Local changes to the core: every `Newton...` name renamed `N236_...` so both APIs can coexist; a const mismatch that left `NewtonJointSetDestructor` undefined; an endless loop in `dgPolyhedra::EdgePenalty` left by a ScummVM rename, which hung loading any optimised level mesh; and 1.53's rule that a body rests only after `framesCount` quiet steps (default 10). The shim keeps the Newton 4 layer's behaviour where HPL depends on it: matrices copied to aligned buffers, `NewtonCollisionCollide` normals pointing from B to A, the NaN guard, the world-box stop, and waking a resting body when its velocity is set. Details in external/newton-2.36/README.md.
- **Test changes (both cores):** test boxes get zero damping, since the cores' defaults differ and HPL sets damping on every entity body. The sleeping-state check uses a fast body, because 1.53's default thresholds rest a slow one too. Joint angle and slider position also accept the reading from one step earlier, which is how 1.x and 2.x report them.
- **Feel:** nothing changes unless the option is set. On 2.36 the boat lamp matches ScummVM's own 2.36 build for the same solver setting. With HPL's default "High" accuracy (exact solver), a push of 2 m/s tilts the chain about 7° toward either wall. With "Medium" (adaptive solver) the bed side still stops at about 7° and the locker side reaches about 29°. Newton 4 reaches 30° both ways.
- **Verified:** 68/68 layer checks pass on both cores. The full game builds and links with each core. Headless boat lamp scene (map geometry, 40 s of wave gravity with the per-frame wake, then a push each way): on 2.36 the lamp stands still on 4 to 6 of 2400 steps, against 4 to 8 on Newton 4. Not yet tried in game, and the missing boat notebook was not reproduced headless: its .ent and the table's .ent weren't available.

### AngelScript on 64-bit: bytecode pointers cut in half
- **Repo:** HPL1Engine
- **Commit:** `461bf61` on `claude/project-thread-msxp3s`
- **Files:** external/angelscript/source/as_context.cpp, external/angelscript/README.md
- **Problem:** once the tokenizer was fixed (below), the first script that made an object (any `string`) crashed in `asCScriptEngine::CallAlloc` with a pointer like `0x55612870`: the real 64-bit pointer with its upper half missing. The interpreter's `PTRARG(x)` macro was `asPTRWORD(*(x+1))` with `x` an `asDWORD *`, so it read 32 bits and widened them. Every pointer stored in the bytecode (`BC_ALLOC`, `BC_FREE`, `BC_OBJTYPE` and others) was affected.
- **Fix:** read through the right type, `*(asPTRWORD*)(x+1)`. Item 3 in the vendored README's list of local changes.
- **Verified:** a small test program set up the way HPL does (portability mode, HPL's `scriptstring`, generic functions, message callback through `asCALL_THISCALL`) passes all 15 checks in Debug and Release builds: building and running scripts, string/float/int/bool arguments, float returns, arrays, globals, loops, string + number, string comparison, `Prepare`/`Execute` with a return value, and a syntax error reported through the callback. Not yet run with game data.

### AngelScript on 64-bit: start-up assertion in RegisterArrayObject
- **Repo:** HPL1Engine
- **Commit:** `f703388` on `claude/project-thread-msxp3s`
- **Files:** external/angelscript/source/as_tokendef.h, external/angelscript/source/as_tokenizer.cpp, external/angelscript/README.md
- **Problem:** the game aborted at start-up with `as_arrayobject.cpp:168: void RegisterArrayObject(asCScriptEngine*): Assertion 'r >= 0' failed` (reported by Xenan on CachyOS). `whiteSpace` was declared `const char *` and the tokenizer looped `w < sizeof(whiteSpace)`. That is 4 on 32-bit, which happens to be the length of `" \t\r\n"`, but 8 on 64-bit, so it read past the string and treated `'\0'` and the bytes after it, which included `'i'`, as white space. `int` was split into pieces, so registering anything with an `int` in its declaration failed. Release builds skip the assert and would fail later instead.
- **Fix:** made `whiteSpace` an array and loop over `sizeof(whiteSpace) - 1`. Item 2 in the vendored README.
- **Verified:** engine creation succeeds, then the test program above passes.

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

### Newton 1.53 layer: a body rests only after framesCount quiet steps (Phase 6)
- **Superseded** by the entry above (`49d2484`): measured against the original library, 1.53 doesn't use `framesCount`.
- **Repo:** HPL1Engine
- **Commit:** `7bb4dc3` on `claude/project-thread-msxp3s`
- **Files:** external/newton-1.53/NewtonBody.h, NewtonHPL1Engine.cpp, newtonWorld.cpp, README.md, tests/newton153_test.cpp
- **Problem:** in the boat level the paraffin lamp, swinging with the wave gravity, froze for a split second every second or two. Newton 4 put a body to rest, and zeroed its velocity, on the first step it was below its thresholds, so the lamp stopped dead whenever its slow swing slowed down. 1.53 (per its documentation of `NewtonBodySetFreezeTreshold`) waited `framesCount` steps in a row, and HPL passes 10. `cEffect_WaveGravity` wakes every body each frame (`NewtonWorldUnfreezeBody`), so in retail the count never ran out and the lamp kept swinging.
- **Fix:** the layer's body class keeps `framesCount` and a count of quiet steps. The body may rest only once the count reaches `framesCount`; a step above the thresholds, or `NewtonWorldUnfreezeBody`, starts it again. Bodies that never get per-body thresholds keep Newton 4's behaviour.
- **Feel:** props now come to rest about 1/6 s (10 steps) after slowing down instead of at once, as in 1.53. Anything woken every frame, such as the boat's loose objects, keeps moving smoothly.
- **Verified:** headless boat scene (lamp bodies, joints and beam from the map, 40 s of wave gravity with the per-frame wake): the lamp stood still on 96 steps before, 8 after (the turning points of the swing; ScummVM's 2.36 shows 11 with sleeping off). Two new layer checks (68/68 pass), both fail without the change: a body under its threshold rests only after 10 quiet steps, and a pendulum woken every frame keeps swinging under slowly tilting gravity. Floor, stack and slide checks are unchanged. Not yet tried in game.

### Newton 1.53 layer: level mesh faces a body can leave sideways are skipped (Phase 6)
- **Repo:** HPL1Engine
- **Commit:** `2a91d6c` on `claude/project-thread-msxp3s`
- **Files:** external/newton-1.53/newton-4/source/ndShapeConvexPolygon.cpp, external/newton-1.53/README.md, external/newton-1.53/tests/newton153_test.cpp
- **Problem:** in the boat level the paraffin lamp's chain rested pinned about 30 degrees off vertical and could only slide a little around that rim. The rod's hook sits 6.6 cm inside the ceiling beam it hangs from (`noshadow_bjalkar` in level00_01_boat_cabin.dae). Newton 4 always pushes a penetrating body out along a mesh face's normal, so the beam's underside pushed the rod down off-centre and swung it to its 30 degree cone limit. Newton 2.x treats a face as a thin plate and pushes out the shortest way (EPA); for the beam's underside that way is sideways (4.9 cm across its edge, less than the 6.6 cm depth), and then it makes no contact.
- **Fix:** skip a mesh face when the penetrating body could leave it sideways, across one of its edges, in less than its depth along the normal. Feel: shallow resting contacts never meet that condition unless a body overhangs a face's edge by less than its few mm of penetration, where a neighbouring face carries it, as in 2.x.
- **Verified:** ScummVM's Newton 2.36 was built standalone (scratch only, not committed) and run on the same scene (the lamp's two bodies and joints plus the beam's faces from the map): it leaves the rod hanging straight, while our layer pinned it at 30 degrees; with the fix our layer hangs straight and swings as with no beam. New level mesh section in the layer test (66/66 pass): resting, stacking three boxes and sliding across tile edges on a tiled mesh floor give identical numbers with and without the change, and the hooked bar went from 30.6 to 0 degrees. Not yet tried in game.

### Grab: spin brake limited so small props don't buzz (Phase 6)
- **Repo:** PenumbraOverture
- **Commit:** `bb7e43e` on `claude/project-thread-msxp3s`
- **Files:** PlayerState_Interact.cpp
- **Problem:** small held props (the whiskey bottle, soup cans) buzzed back and forth around their long axis; bigger ones (chair, backpack) were smooth. Retail shows no buzz. `cPlayerState_Grab` brakes the held body's spin with `0.9 * -omega * mass` (or `0.8 * (wanted - omega) * mass` while rotating with the player). A thin body has so little inertia about its long axis that this torque more than stops the spin in one step, so it overshoots every step; the layer's 45 degree cap then holds it at a fast back-and-forth. ScummVM's Newton 2.36, built for comparison, buzzes the same way, so retail's 1.53 must have handled this differently in a way we can't see.
- **Fix:** `LimitGrabBrake` limits the brake torque, per body axis, to what removes the error in one step (`|error| * I / dt`). If the original torque is already below that it is used unchanged. The HaptX grab state is not changed.
- **Feel:** only bodies whose brake overshot are affected; they now stop spinning instead of buzzing. Bodies that were smooth get exactly the original torque.
- **Verified:** game builds. Headless check with the layer (same brake, body at an angle, 120 steps): a bottle-sized and a can-sized box buzzed at about 47 rad/s without the limit and stop within a few steps with it; a chair-sized box gets an identical torque with and without (difference 0). Not yet tried in game.

### Newton 1.53 layer: 45 degree spin cap per step (Phase 6)
- **Repo:** HPL1Engine
- **Commit:** `2d8565d` on `claude/project-thread-msxp3s`
- **Files:** external/newton-1.53/newton-4/source/ndBodyKinematic.cpp, external/newton-1.53/README.md, external/newton-1.53/tests/newton153_test.cpp
- **Problem:** grabbing the boat paraffin lamp in level 1 froze it, then it flew off. While an object is held, `cPlayerState_Grab` adds a damping torque of `-omega * 0.9 * mass` every frame. The lamp's inertia about its long axis is tiny (about 0.002 per kg), so that torque overshoots: each step the spin reverses and grows about 6 times, reaching over 100 rad/s in 5 frames. Newton 2.x (`dgBody::IntegrateVelocity`, and very likely 1.53) caps the rotation per step at 45 degrees; Newton 4 only asserts it in debug builds. Newton 4's gyroscopic term was ruled out: removing it didn't stop the blow-up.
- **Fix:** before integrating, the layer shrinks a body's angular velocity by 0.8 until one step turns it by no more than 45 degrees, as 2.x does. Feel: only bodies spinning faster than about 47 rad/s at 60 Hz are touched, so normal objects are unchanged. A held lamp may still buzz around its vertical axis, as it would under 2.x; to compare with retail.
- **Verified:** a headless copy of the lamp (its two bodies, ball joints and limits from Xenan's .dae/.ent, plus the grab's force, torque and damping) blew up without the cap and with it follows the hand and settles after release. Two new checks in the layer test (61/61 pass); the spin check fails without the cap. Not yet tried in game. Separately found, not changed yet: Newton 4 returns inertia about the shape origin, not the centre of mass as 2.x did, so off-centre bodies turn more sluggishly than the original (to fix after comparing with retail).

### Newton 1.53 layer: world box and NaN guard (Phase 6)
- **Repo:** HPL1Engine
- **Commit:** `3d1f5ee` on `claude/project-thread-msxp3s`
- **Files:** external/newton-1.53/newtonWorld.cpp, newtonWorld.h, NewtonBody.cpp, NewtonBody.h, NewtonDeprecated.cpp, README.md, tests/newton153_test.cpp
- **Problem:** in level 1, Xenan grabbed the boat's paraffin lamp; it flew out of the map and the game hung (sound kept playing, no input). `NewtonSetWorldSize` was a no-op in the layer, so nothing stopped a body leaving the level. 1.53 disables any body that leaves its world box (HPL sets it to the level's bounds plus 10 m).
- **Fix:** the layer keeps the world box (default +-100, as at 1.53 world creation) and after each update disables a dynamic body whose bounding box is outside it: velocity zeroed and put to sleep, each update while it stays out. A body whose matrix or velocity becomes NaN/inf is put back at its last valid matrix and stopped, with one message on stderr. Feel: no change for bodies inside the level; matches 1.53 at the edge.
- **Verified:** 6 new checks in the layer test (59/59 pass): a pushed body stops at the box edge and stays there, a body inside moves normally, and a NaN velocity leaves the body at its last valid position, stopped. Newton 4 on its own was also checked: a body flying for 20,000 steps or with a NaN velocity didn't hang it, so the hang was probably in the engine handling an absurd position, which this prevents. Not yet confirmed in game. The lamp freezing when grabbed and then flying is not explained yet (see the lamp's load errors in the log).

### Headless Newton layer test (Phase 5, step 9)
- **Repo / commit:** HPL1Engine `2e2a3d8`
- **Files:** `external/newton-1.53/tests/newton153_test.cpp`,
  `external/newton-1.53/CMakeLists.txt`, layer README; PenumbraOverture
  `CLAUDE.md`, `ROADMAP.md`
- **Change:** the scratch tests used while completing the layer (steps 4–8)
  are now one committed program, `newton153_test` (target not built by
  default). 53 checks: buffer overruns, `NewtonCollisionCollide` normals and
  depths, joint force/angle/position readouts and the stop helpers,
  sleeping state, freeze thresholds, impulses, freeze/unfreeze, CCD flag,
  ball cone limits, HPL1-style hinge/slider/corkscrew limit and friction
  callbacks, materials (per-pair elasticity and friction) with HPL1-style
  begin/process/end callbacks (thread, order, once per update, impact speed,
  contact force, rejected pairs), and joint destroy/collision/stiffness.
  Run instructions are in CLAUDE.md.
- **Verified:** built from a fresh build folder with `make newton153_test`
  (no warnings with `-Wall -Wextra`) and run: 53 of 53 pass, exit code 0.
  The full game still builds. The checks were not each re-proven to fail on
  the old code inside this one program; each was written against a bug or
  behaviour found in steps 4–8 and passed only after its fix there.

### Newton 1.53 layer linked into HPL; prebuilt libNewton.a dropped (Phase 5, step 8)
- **Repo / commits:** HPL1Engine `7714694`, `7c968af`, `254d709`
- **Files:** `CMakeLists.txt`, `external/newton-1.53/NewtonHPL1Engine.cpp`,
  `NewtonDeprecated.cpp`, `joints/NewtonJoints.h/.cpp`, `Newton.h`, layer
  README; PenumbraOverture `CLAUDE.md`
- **Problem:** HPL still linked the IMPORTED 32-bit `libNewton.a` from
  `HPL_DEPENDENCIES_DIR`. Linking the layer instead showed 3 more functions
  that only existed inside `#if 0`: `NewtonDestroyJoint`,
  `NewtonJointGetCollisionState`, `NewtonJointGetStiffness` (called from the
  header `PhysicsJointNewton.h`, which the earlier symbol check, done over
  `.cpp` files only, missed). `NewtonJointSetCollisionState` was live but
  only asserted, so a map's "joined bodies collide" setting was ignored.
  `Newton.h` used `size_t` without including `<stddef.h>`.
- **Fix:** HPL links `newton153`; the IMPORTED target, the
  `HPL_DEPENDENCIES_DIR` option and its include path are gone. The joint
  functions are defined: destroy removes the joint (its destructor callback
  runs), collision state maps to Newton 4's per-joint collidable flag
  (default off, as in 1.53), stiffness is stored and reported (default 0.9)
  but has no effect, as Newton 4 has no equivalent. `Newton.h` includes
  `<stddef.h>`.
- **Verified:** a fresh build folder configures with no extra options and
  `make` builds everything with **no errors**: `overture.bin` is a 64-bit
  x86-64 executable. Headless test: joint collision state defaults to 0 and
  toggles; stiffness reads back; destroying a ball joint runs its
  destructor once and the hanging box falls; overlapping jointed bodies
  don't push apart unless collision is enabled. All step 4–7 tests still
  pass, and compile without `<cstddef>`. Started without a display or game
  data, the game logs its module start-up and stops at "Unable to initialize
  display" as expected (it then segfaults on that error path, which is the
  engine's existing behaviour and unrelated; to look at in Phase 6). Not
  run with game data yet.

### Newton 1.53 layer: contact materials, callbacks and world update (Phase 5, step 7)
- **Repo / commits:** HPL1Engine `db0ad1f`, `2cfee41`, `e666f69`, `7a5d954`,
  `f0aa30a`, `758ec5d`, README `f7d6709`
- **Files:** `external/newton-1.53/newtonWorld.cpp`, `CMakeLists.txt`,
  `NewtonMaterialGraph.cpp`, `newton-4/source/ndScene.cpp`,
  `newton-4/source/ndThreadPool.h`, layer README
- **Problems and fixes:**
  - **Materials were never applied** (`db0ad1f`). `NewtonBodySetMaterialGroupID`
    only stored the group; the core picks a contact's material by shape user
    ids, which were never set. Every contact used the default material, so
    there was no per-material friction or bounce, and none of HPL's contact
    callbacks ran (impact/slide/roll sounds, breaking, enemy damage, the
    player backing off a pushed object). Contacts now look the material up
    by the bodies' groups. Pairs with group 0 (HPL's "Default" material) get
    their own material too, instead of all sharing one.
  - **Callbacks on another thread** (`2cfee41`). Newton 4 ran the update on
    its own thread. The core is now built with `D_USE_THREAD_EMULATION` and
    one thread, so `NewtonUpdate` runs on the game thread like 1.53, and each
    pair's begin → process… → end run back to back (HPL keeps their state in
    statics). The emulation branch of `ParallelExecute` no longer compiled
    and was fixed.
  - **3 substeps** (`e666f69`): now 1 per update, as in 1.53. Callbacks had
    run 3 times per update.
  - **SIMD8 solver bug** (`7a5d954`): it mixed up rows of constraints with
    different row counts in a batch. Seen: a frictionless box made another
    box's contact frictionless, depending on order. Switched to the standard
    solver (no loss: single thread, scalar maths).
  - **Stale contacts / skipped callbacks** (`f0aa30a`): when the begin
    callback returned 0 the pair kept last step's contacts (now dropped, as
    in 1.53). Contacts of moving pairs are now recomputed every step instead
    of being reused while bodies moved less than 1 mm / 0.25°, so callbacks
    run every step for moving bodies, and bodies HPL teleports by setting
    their matrix don't keep stale contacts.
  - **Elasticity clamp** (`758ec5d`): 0.01..2 as in 1.53 (was 0..1).
- **Verified:** headless test with HPL-style begin/process/end callbacks and
  four material groups: callbacks run on the main thread, strictly
  B P… E per pair, once per pair per update; impact normal speed 5.89 m/s for
  a 1.75 m drop (positive, as HPL expects); `NewtonMaterialGetContactForce`
  sums to 9.81 N for a 1 kg box; a bouncy pair rebounds to 1.16 m while a
  dead pair doesn't bounce; a frictionless pair slides while a μ=0.6 pair
  decelerates at ~0.6 g; a rejected pair falls through the floor at once.
  All earlier step 4–6 tests still pass. With 1 substep a ±0.5 rad hinge
  limit overshoots by up to 0.044 rad and a slider stop by 3 cm at 60 Hz
  (was 0.009 rad / 6 mm with 3 substeps); 1.53 also stepped once per
  update, so this should be closer, but it is a feel item for Phase 6.
- **Known differences:** when both bodies of a pair are completely still,
  Newton 4 skips the pair, so no callbacks fire for it (1.53 kept calling
  until the bodies froze). HPL's uses of the callbacks need motion, so this
  should not matter. In 1.53, `end` may also have been called for a pair
  with no contact points; HPL's `end` ignores that case either way. A
  process callback returning 0 does not reject the point (HPL always
  returns 1). Not checked in-game.

### Newton 1.53 layer: joints obey HPL's limit callbacks (Phase 5, step 6)
- **Repo / commit:** HPL1Engine `4434e4a`
- **Files:** `external/newton-1.53/joints/NewtonHinge.cpp`, `NewtonSlider.cpp`,
  `NewtonCylinder.cpp`, new `joints/NewtonHPL1Joints.h`, layer README
- **Problem:** the hinge, slider and corkscrew wrappers called HPL1's
  `LimitCallback` but discarded its return value and the row it filled in.
  Every door, lever, drawer and screw limit (and one-sided stop) would have
  been ignored. The default force bounds were also wrong (±1e-20, i.e. no
  force at all, instead of unbounded).
- **Fix:** as in Newton 2.36 (the reference for 1.53): base rows first, then,
  if the callback returns non-zero, one row along the child's pin, angular
  for the hinge and linear for the slider. For the corkscrew, bit 0 adds the
  linear row (`desc[0]`) and bit 1 the angular row (`desc[1]`). The row
  uses the callback's `m_accel` as motor acceleration and its min/max
  friction as force bounds (unbounded unless the callback sets them). The
  joint's angle and position are refreshed before the callback, so it sees
  current values.
- **Verified:** headless test with callbacks copied from HPL1's hinge and
  slider `LimitCallback`: a hinge limited to ±0.5 rad overshoots by at most
  0.009 rad and rests at −0.500 (unlimited it swings to −π); pushed away it
  leaves the stop freely (one-sided). A bounded stop-motor row holds the arm
  at 20 N·m and slips at 4 N·m (gravity torque 9.81 N·m). Slider and
  corkscrew limited to ±1 m stop at −1.000 (lowest −1.006) and, pushed up,
  leave the stop and halt at +1.000. Not yet checked in-game (door/lever
  weight is a Phase 6 feel check). Note: the callback runs 3 times per
  update because of Newton 4's 3 substeps; step 7 goes to 1 substep.

### Newton 1.53 layer: stubs and behaviour fixes (Phase 5, step 5)
- **Repo / commits:** HPL1Engine `50900b6` (sleep query), `8caf8b1` (impulse,
  freeze), `84cce93` (joint stop helpers), `e130544` (ball cone axis),
  `e5e1480` (continuous collision flag), `d5d67da` (layer README)
- **Files (all in `external/newton-1.53/`):** `NewtonBody.cpp`,
  `NewtonBody.h`, `newtonWorld.cpp`, `NewtonDeprecated.cpp`,
  `joints/NewtonBallSocket.cpp`, `README.md`
- **Problems and fixes** (1.53 behaviour, from the Newton 2.36 source where
  1.53's isn't available):
  - `NewtonBodyGetSleepingState` returned 1 for a *sleeping* body; 1.53
    returns 1 for *active*, which HPL's `GetEnabled` relies on (used by the
    character controller, mesh entities and save games). Inverted.
  - `NewtonAddBodyImpulse` did nothing (throwing, hits, explosions). Its
    argument is the change in velocity wanted *at the given point*, not an
    impulse; it now solves for the impulse that gives exactly that, applies
    it to linear and angular velocity and wakes the body.
  - `NewtonWorldFreezeBody` did nothing (HPL disables bodies with it, e.g. a
    door settling at its limit). Now clears velocity and sleeps the body.
  - `NewtonHingeCalculateStopAlpha`, `NewtonSliderCalculateStopAccel`,
    `NewtonCorkscrewCalculateStopAlpha/Accel` returned 0. HPL's door, lever,
    drawer and screw limits are built on them. Now 1.53's formula:
    100 × error − 1.01 × speed / timestep, ignoring speed towards the stop,
    with the hinge's ×16 factor below its stop.
  - `NewtonBallSetConeLimits` ignored the cone axis, so ragdoll/chain/lamp
    limits were centred on the wrong axis (twist limited as cone and vice
    versa). Now re-aims the joint frame to the given axis, with 1.53's
    thresholds and clamps.
  - `NewtonBodyGetContinuousCollisionMode` always returned 0, so save games
    stored the wrong value. The flag is now stored and reported back (no
    simulation effect, as in 1.53).
- **Verified (scratch test programs, Newton only):** an impulse at the centre
  of mass changes velocity by exactly the requested amount; off-centre, the
  point's velocity changes by exactly the request and the body spins; frozen
  bodies report 0 and stay put, unfrozen ones report 1 and move again; the
  stop helpers return the 1.53 values for a hinge at 0.5 rad / 1 rad/s and a
  slider at 0.25 m / 0.5 m/s (past the stop, short of it, exactly on it); a
  ball joint with cone axis +X, 30° cone and 10° twist now stops a twist at
  10.1° (it reached 30° before the fix) and a tilt at 30.0°; the continuous
  collision flag round-trips. The step-4 tests still pass. **Not verified:**
  the corkscrew helpers (same code path as the slider/hinge ones, not run
  separately), and anything in-game. The stop helpers have no effect yet,
  because the joint wrappers ignore the callbacks' results (step 6).

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

### Choppy streamed music in the main menu
- **Repos / commits:** PenumbraOverture `cddb660`, OALWrapper `d6498f3`, on `claude/project-thread-msxp3s`
- **Files:** PenumbraOverture Init.cpp; OALWrapper sources/OAL_Device.cpp
- **Problem:** menu music (and the rain loop) cut in and out rapidly, a phaser-like stutter. Retail has it too (Xenan, also under Proton). Penumbra asked for 64-byte stream buffers, which OALWrapper raises to its 4 KB minimum, with 4 buffers: about 93 ms of 44.1 kHz stereo. The stream thread refills every 100 ms (`StreamUpdateFreq` 10), so the source ran dry on almost every refill. Lower-rate streams (22 kHz mono) have 4x the slack, which probably explains why it's mostly noticed in the menu (not checked against the game data).
- **Fix:** Penumbra's default stream buffer is 16 KB (4 x 16 KB, about 370 ms of 44.1 kHz stereo). OALWrapper also enlarges the buffers so the queue holds at least twice the refill interval of 48 kHz 16-bit stereo, whatever the settings. Only buffering changes; the audio itself doesn't.
- **Verified:** a test program streams a 44.1 kHz stereo Ogg through OALWrapper with OpenAL Soft and counts underruns over 4 s. Before: 34 with the game's settings. After: 0 with the game's old settings, with 16 KB buffers, and with a 5 Hz refill. Not yet heard on Xenan's machine.

### Our build shared the retail game's settings, saves and logs
- **Repo:** PenumbraOverture
- **Commit:** `bb8268d` on `claude/project-thread-msxp3s`
- **Files:** Init.h, Init.cpp
- **Problem:** the build used the same user folder as the retail game (`~/.frictionalgames/Penumbra/Overture` on Linux). Xenan's first run overwrote the retail `hpl.log`, and it would read the retail settings (which is how the -1 resolution below got in) and could overwrite retail saves.
- **Fix:** settings, saves and logs now go to `PenumbraRevival/Overture/` under `~/.local/share` (Linux), `Documents` (Windows) and `~/Library/Application Support` (macOS). Removed the retail code that moved an old `Penumbra Overture/Episode1` folder, which would now look in the wrong place. `XDG_DATA_HOME` isn't read yet; the path is always relative to `$HOME`.
- **Verified:** a run under Xvfb wrote `hpl.log` and `settings.cfg` to `~/.local/share/PenumbraOverture/` (renamed to `PenumbraRevival/Overture/` afterwards, same code path) and created no `.frictionalgames` folder. Not built on Windows or macOS.

### Audio never opened: device "NULL"
- **Repo:** HPL1Engine
- **Commit:** `1aebde8` on `claude/project-thread-msxp3s`
- **Files:** sources/impl/LowLevelSoundOpenAL.cpp
- **Problem:** start-up stopped with `Failed to inialize Audio Trying fallback driver` / `FATAL ERROR: Failed! Check your OpenAL installation` (Xenan, CachyOS/PipeWire). Penumbra's settings default `DeviceName` to the string `"NULL"`, meaning no particular device, and the fallback asked for `"NULL"` again. OpenAL Soft has no device by that name: `alcOpenDevice("NULL")` fails, while `alcOpenDevice(NULL)` and `alcOpenDevice("")` open the default device.
- **Fix:** `"NULL"` is treated as an empty name (default device); if a named device fails, the fallback opens the default device.
- **Verified:** under Xvfb with OpenAL Soft's null backend the log shows `Trying to open audio device... Success!` with the default settings. Not yet confirmed on Xenan's machine.

### -1 x -1 screen size dropped to a 640x480 window
- **Repos / commits:** HPL1Engine `33d8097`, PenumbraOverture `8c10835`, on `claude/project-thread-msxp3s`
- **Files:** HPL1Engine sources/impl/LowLevelGraphicsSDL.cpp; PenumbraOverture Init.cpp
- **Problem:** the retail settings had `Width="-1" Height="-1"` (the retail Linux port's "desktop size"). `SDL_SetVideoMode(-1, -1, ...)` fails, so the game showed "Could not set displaymode and 640x480 is used instead!".
- **Fix:** the engine uses SDL's current desktop size when the width or height is 0 or less (800x600 if SDL can't tell). The game then reads back the size actually set, so the options menu and saved settings show it. This also saves the real numbers in place of -1, so a later desktop resolution change isn't followed automatically.
- **Verified:** under Xvfb (1280x1024 screen) with `-1 x -1` the log shows `Using desktop resolution 1280 x 1024` and the mode is set.

Bugs found while clearing compiler warnings. Each was a real fault, not just
noise; the warning pointed at it.

### Log writer used each message as a printf format
- **Repo:** HPL1Engine
- **Commit:** `9565367` on `claude/project-thread-msxp3s`
- **Files:** sources/impl/LowLevelSystemSDL.cpp
- **Problem:** `cLogWriter::Write` called `fprintf(mpFile, asMessage.c_str())`. A log line containing `%` (a file path, a script error) was read as format directives, giving garbage or a crash. GCC: `-Wformat-security`.
- **Fix:** `fputs(asMessage.c_str(), mpFile)`.
- **Verified:** compiles with no warning. Log output for normal lines is identical.

### Frame number buffer overflow in ImageEntityData
- **Repo:** HPL1Engine
- **Commit:** `5c6aeb6` on `claude/project-thread-msxp3s`
- **Files:** sources/graphics/ImageEntityData.cpp
- **Problem:** `char buff[5]; sprintf(buff,"%d",lNum)` overflows for animation frame numbers of 10000 or more. GCC: `-Wformat-overflow`.
- **Fix:** 16-byte buffer and `snprintf`. Same text for every frame number.
- **Verified:** compiles with no warning.

### Ignored results of system() and fread()
- **Repo:** HPL1Engine
- **Commit:** `136c139` on `claude/project-thread-msxp3s`
- **Files:** sources/impl/LowLevelSystemSDL.cpp, sources/impl/SqScript.cpp
- **Problem:** `OpenBrowserWindow` ignored whether `system()` succeeded; `cSqScript::LoadCharBuffer` ignored `fread`, so a short read compiled a partly filled buffer. GCC: `-Wunused-result`.
- **Fix:** log a warning when opening a URL fails; return NULL on a short read, which the caller already reports as "Couldn't load script".
- **Verified:** compiles with no warning. The URL is still passed to the shell unquoted, as in the original; worth revisiting with the launcher work.

### Missing or short WAV file crashed
- **Repo:** OALWrapper
- **Commit:** `23a28ca` on `claude/project-thread-msxp3s`
- **Files:** sources/OAL_WAVSample.cpp
- **Problem:** `CreateFromFile` called `fseek` on the result of opening the file without checking it, so a missing file crashed, and it ignored `fread`'s result. GCC: `-Wunused-result`.
- **Fix:** return false if the file can't be opened or isn't read in full.
- **Verified:** compiles with no warning.

## Performance

_No entries yet._

## Documentation

### Arch package note for the Cg toolkit
- **Repo:** PenumbraOverture (this commit)
- **Files:** CLAUDE.md
- **Change:** noted that on Arch-based systems (Xenan uses CachyOS) `nvidia-cg-toolkit` is in the official `extra` repo, not the AUR.

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
