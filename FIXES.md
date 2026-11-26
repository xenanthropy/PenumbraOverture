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

_No entries yet._

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
- **Verified:** Compiles with GCC 13.3 (Ubuntu 24.04, x86_64): the error no longer appears in the HPL1Engine build. Not yet run (needs the rest of the roadmap).

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

_No entries yet._

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
