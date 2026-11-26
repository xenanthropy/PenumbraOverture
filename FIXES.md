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

### Add CLAUDE.md, FIXES.md, KNOWN_ISSUES.md, ROADMAP.md
- **Repo:** PenumbraOverture
- **Files:** CLAUDE.md, FIXES.md, KNOWN_ISSUES.md, ROADMAP.md
- **Change:** Added collaboration notes, an analysis of the three repos,
  this change log, the catalogue of known issues, and the roadmap.
