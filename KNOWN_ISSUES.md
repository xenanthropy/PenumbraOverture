# Known Issues and Background

What is already known about building and running Penumbra Overture and
HPL1Engine, collected before any fixes were made. Fixes themselves go in
[FIXES.md](FIXES.md).

## 1. Baseline build (no fixes applied)

Environment: Ubuntu 24.04, x86_64, GCC 13.3, CMake 3.28, assessed 2026-11-26.
All three repos at their current `master`.

Distro packages installed:

```
libsdl1.2-dev (sdl12-compat) libsdl-ttf2.0-dev libsdl-image1.2-dev
libopenal-dev libalut-dev libvorbis-dev libogg-dev libtheora-dev
libglu1-mesa-dev libfltk1.3-dev nvidia-cg-toolkit
```

### 1a. Stock build: `cmake ../PenumbraOverture && make -k`

- CMake configures, with deprecation warnings for `cmake_minimum_required(2.6)`
  and `(2.8)`. CMake 4.x removes support for minimums below 3.5, so this
  configure step will fail outright on newer CMake.
- OALWrapper builds (one `-Wnarrowing` warning in `OAL_OggSample.cpp:155`).
- HPL1Engine and every PenumbraOverture file stop at `angelscript.h` and
  `Newton.h` not found. Those headers lived in the old `../dependencies`
  bundle, which is not in any repo.

### 1b. With AngelScript 2.7.1b supplied

To see the errors hidden behind the missing headers, AngelScript 2.7.1b was
built from source outside the repos (see section 3) and passed in with
`-DCMAKE_CXX_FLAGS="-I<as>/include -DAS_MAX_PORTABILITY"`. Newton was still
missing because no Newton 1.x header is available.

Result:

- **All 59 PenumbraOverture `.cpp` files compile.** The game only fails at
  link, because `libHPL.a` was not produced.
- **HPL1Engine: 92 errors in 16 files.** Missing assets play no part in any of
  them:

| # | Where | Error | Cause |
|---|-------|-------|-------|
| 1 | 7 `impl/*Newton*.h` headers | `Newton.h: No such file` | Newton 1.x is unavailable (section 4) |
| 2 | `sources/graphics/Material_Fallback01_BaseLight.cpp`, `Material_Fallback02_BaseLight.cpp`, `impl/VertexBufferVBO.cpp`, `impl/OcclusionQueryOGL.cpp`, `impl/LowLevelGraphicsSDL.cpp` | ~70x `glBindBufferARB`, `gl*FragmentShaderATI`, `glGenQueriesARB`, `glBlendFuncSeparateEXT`, ... not declared | GLee checks the guard macro `__glext_h_`, which modern Khronos/Mesa `glext.h` no longer defines in the same way. GLee then skips its own declarations. |
| 3 | `sources/graphics/Renderer2D.cpp:898,899,903,904,923` | taking address of rvalue | `AddVertexToBatch(&cVertex(...))` takes the address of a temporary |
| 4 | `sources/sound/SoundHandler.cpp:289` | cannot convert `bool` to `iSoundChannel*` | `return false;` in a function that returns a pointer |
| 5 | `sources/impl/LowLevelSystemSDL.cpp:635` | `symlink` not declared | missing `#include <unistd.h>` |
| 6 | `include/graphics/GraphicsDrawer.h:57` | static assertion: comparison object must be invocable as const | `cGfxBufferCompare::operator()` is not `const` |

Warnings worth knowing about: 173x `-Wattributes` (`__stdcall` is ignored on
x86_64, from `system/Script.h`), 3x `-Wliteral-suffix`, 2x
`-Wconversion-null`, 1x `-Wformat-security`.

Items 3, 4 and 5, plus the warnings, were already fixed in community forks
(section 5).

### 1c. With the original `Newton.h` header supplied too

Using the `Newton.h` from the original `dependencies.zip` (section 3a), all
the Newton backend files compile on x86_64. HPL1Engine drops to **82 errors**,
and the only new one is:

| # | Where | Error | Cause |
|---|-------|-------|-------|
| 7 | `sources/impl/PhysicsBodyNewton.cpp:104` | taking address of rvalue | `&apEntity->GetLocalMatrix().GetTranspose().m[0][0]` takes the address of a temporary matrix |

So once GLee is replaced and items 3–7 are fixed, HPL1Engine should compile
completely. **Linking is a separate problem**: the only Newton 1.53 library
that exists is 32-bit (section 3a), so a 64-bit build cannot link it.

## 2. Runtime bugs reported by players

### AMD / Mesa crash: coin in the vice (Black Plague), fishing boat (Overture)

- **Symptom:** on native Linux with AMD GPUs (r600 and amdgpu, Mesa), *Black
  Plague* crashes when you flatten the coin in the vice in the first room.
  *Overture* can crash in a similar way when leaving the fishing boat at the start.
- **Where:** the reported backtrace is in `r600_dri.so`, under
  `cSDLTexture::CreateFromBitmapToHandle`, called from
  `cParticleEmitterData3D_UserData::LoadFromElement`. A script call triggers
  loading the effect's particle system and textures, and the crash happens
  inside the GL driver.
- **Root cause (community analysis):** the bundled AngelScript calls native
  C++ functions through its own x86 assembly. That code does not keep the
  stack 16-byte aligned. Modern GCC builds of Mesa use SSE instructions that
  require an aligned stack, so a GL call made from inside a script callback
  crashes.
- **Workarounds:** build Mesa without those optimisations (e.g. `-O0`/`-O2`,
  no LTO, run via `LIBGL_DRIVERS_PATH`), or **rebuild the game with
  AngelScript in portability mode or a newer AngelScript.** Portability mode
  (`AS_MAX_PORTABILITY`) uses the generic calling convention, so no
  hand-written assembly runs. The owner has confirmed that rebuilding with a
  newer AngelScript fixes the crash.
- **Relevance here:** HPL1Engine already supports portability mode.
  `include/system/Script.h` generates `asCALL_GENERIC` wrappers when
  `AS_MAX_PORTABILITY` is defined. A source build with that flag should avoid
  the crash entirely. This is not verified in-game yet.
- Black Plague and Requiem were never released as source. They share this
  engine, so the fix carries over if those games can be built against it.

### Other reported problems (forum threads 56750 / 56758, 2020)

- A stock AngelScript build crashes at game start. Both AngelScript and the
  game had to be built with `AS_MAX_PORTABILITY`.
- Shader quality above "Very low" turns everything blue (seen in a VM and on
  Intel). The drop-in shader replacements in the HPL1 repo reportedly only
  work well at the highest setting.
- Zoomed-in view in the starting scene; the mouse is too sensitive in
  fullscreen and offset in windowed mode (possibly VM related).
- AngelScript fails to bind `SetJointControllerPropertyFloat`, which opens the
  mine trapdoor. This would block progress.
- Requiem: crash when picking up Eloff Carpenter's first audio log
  (GamingOnLinux comments).
- An old Arch report blames a system `libvorbis` for a crash; the commercial
  release ships its own `libvorbis.so.0` for this reason.

### Found in our build

- **Segfault after a fatal error.** `FatalError()` calls `exit(1)` from
  wherever it is, with the engine half built. Static destructors and
  `atexit` handlers then run against that state and often segfault. The real
  error is the last line of `hpl.log`. To fix: shut down in order, or
  `_exit` after flushing the log.

## 3. Dependencies: what's needed and where to get it

| Dependency | Version used originally | Available today | Notes |
|---|---|---|---|
| AngelScript | **2.7.1b** | **Source exists** on SourceForge SVN, `branches/2.7.1` (https://sourceforge.net/p/angelscript/code/HEAD/tree/branches/2.7.1/). It is not tagged; the main angelcode.com site is the usual download. | Builds on GCC 13 with one fix (below). Needs `AS_MAX_PORTABILITY` on x86_64; 64-bit pointers (`AS_64BIT_PTR`) are switched on automatically for LP64. |
| Newton Game Dynamics | **1.53** | **No source.** Only prebuilt libraries: the old dependencies zip and a community Linux 1.53 build posted on Discord. Newer Newton (2.x+) is open source (zlib), but its API is different. | The hardest blocker (section 4). |
| Cg toolkit | 2.x/3.x | `nvidia-cg-toolkit` 3.1 (Ubuntu, x86 only); NVIDIA discontinued it in 2012 | The shaders (`.cg`) need it at runtime. |
| GLee | 5.4 | Original site gone | Breaks with modern headers. Replace with GLEW, glad or libepoxy. |
| SDL 1.2, SDL_ttf, SDL_image | 1.2 | Distro packages (sdl12-compat) | Works. SDL2 is the long-term step. |
| FLTK | 1.1/1.3 | Distro packages | Used only for one `fl_alert`; can be removed (a community fork already did this). |
| OpenAL, ALUT, Ogg/Vorbis, Theora | — | Distro packages | Fine. |
| TinyXML | bundled | — | Fine. |

### 3a. The original dependency bundles (examined 2026-11-26)

Two bundles, supplied by the owner and kept outside the repos (never commit
them):

- **`dependencies.zip`**: the original 2010 Frictional bundle, now only on
  archive.org.
  - `include/`: `angelscript.h` (2.7.1b), `Newton.h`, plus AL, Cg, FL, GL
    (GLee **5.2**; the engine repo carries 5.4), SDL, ogg, vorbis, theora headers.
  - `lib/linux/`: **every binary is 32-bit i386**, built with GCC 4.0.2
    (2006). There are static `libNewton.a`, `libangelscript.a`, `libSDLmain.a`
    and shared Cg, CgGL, GLU, SDL, SDL_image, SDL_ttf, alut, fltk 1.1, jpeg,
    ogg, openal, png12, vorbis, vorbisfile, and even `libstdc++.so.6.0.7`.
  - `lib/win32/`: 32-bit `.lib`/`.dll` for the same set, including
    `Newton.dll`/`newton.lib` (**no `GLaux.lib`**; see section 6).
  - `lib/macosx/`: `libnewton32.a`, `libas.a`, `libfltk.a` and
    Ogg/Vorbis/Theora/ALUT frameworks.
- **`dependencies-Linux.tar.gz`** (onager, 2020): only `Newton.h`,
  `angelscript.h`, `libNewton.a` and `libangelscript.a`, all 32-bit.

**Newton is in both bundles.** Its header and `libNewton.a` are
byte-identical between the two (the 2020 bundle just re-ships the 2010
library). It is Newton 1.5x, 32-bit x86 only, with no source. The Linux
archive is 60 object files (`Newton.o`, `NewtonClass.o`, `NewtonRagdoll.o`,
...).

**The bundled AngelScript header matches our source build.** The
`angelscript.h` from SourceForge `branches/2.7.1` is identical to the bundled
one, apart from the one-line GCC fix below. So the SourceForge copy really is
the original 2.7.1b. onager's `libangelscript.a` is his 2020 32-bit
`AS_MAX_PORTABILITY` rebuild; the 2010 one is the stock (native-call) build.

**What this means:** the original 32-bit setup could probably be rebuilt
today with a multilib (`-m32`) toolchain and these blobs. That is useful
for comparing behaviour, but it is a dead end for a modern 64-bit,
cross-platform build. Newton is the only dependency that truly can't be
rebuilt from source.

### 3b. AngelScript notes

**AngelScript 2.7.1b on modern GCC.** `include/angelscript.h:784` has
`return 0;` in the generic `asSMethodPtr<N>::Convert`, which returns an
`asUPtr` union. Old GCC never checked that body because it was never
instantiated; GCC 13 rejects it. Changing it to `return asUPtr();` is enough.
With that change, every source file compiles with
`g++ -O2 -DAS_MAX_PORTABILITY` into a static `libangelscript.a`.

**Building the original way (2010, 32-bit).** Put the three repos side by
side, unpack `dependencies.zip` from the old HPL1Engine download page into
`../dependencies`, then `cmake ../PenumbraOverture && make`. The 2020 Ubuntu
12.04 guide adds a few steps: rebuild AngelScript 2.7.1b with
`-DAS_MAX_PORTABILITY`, use a known-compatible OALWrapper commit, and put the
game assets plus `materials.cfg`/`resources.cfg` in a `redist` folder next to
`overture.bin`. Settings and logs go to `~/.frictionalgames/Penumbra/Overture`.

## 4. Newton 1.53: the big decision

Options, to discuss before choosing:

1. **Keep prebuilt Newton 1.53** (from the old bundle). It works, but it is a
   binary blob, probably 32-bit only on Linux, and cannot be rebuilt.
2. **Port to Newton 2.x/3.x (open source).** A `Newton2` branch exists in
   several forks; it dates from 2010 and its collision was broken ("walking
   into any object results in instant death"). ScummVM's HPL1 port did this
   properly: `engines/hpl1/engine/libraries/newton` is a Newton core with the
   2.x-style API (e.g. `NewtonCreateBox(..., shapeID, offsetMatrix)`),
   vendored as source.
3. Port to a different physics library (Bullet, Jolt). This is far more
   work, and physics behaviour would change.

### 4a. New: the Newton author's 1.53 compatibility layer (2026)

In newton-dynamics discussion #435 (Aug–Oct 2026), Julio Jerez confirmed the
original 1.53 source is gone. He started a backport that implements the 1.53
API on top of Newton 4, aimed at HPL1/Penumbra:
`newton-4.00/applications/toolsAndWrapers/newton-1.53` in
https://github.com/JulioJerez/newton-dynamics. Checked 2026-11-26 (master
`294f27f`):

- Covers the full original 1.53 function list, including all 90 functions
  HPL1 calls.
- Five that HPL1 uses are still stubs: `NewtonWorldFreezeBody`,
  `NewtonAddBodyImpulse`, `NewtonHingeCalculateStopAlpha`,
  `NewtonSliderCalculateStopAccel`, `NewtonCorkscrewCalculateStopAccel`.
- Windows/MSVC-only for now (MSVC flags in CMake; `_finite`/`_isnan` break
  GCC).

See ROADMAP.md Phase 5 for the options.

## 5. Notes from the Windows guides (2010–2013)

From JKO's guide (2010, Visual Studio 2010) and the "compiling under Visual
Studio 2012" thread. They are older than onager's 2020 Linux guide, so where
they conflict, onager's is more current. Still relevant when we get to
Windows:

- **The VS project files were never really maintained.** They were hand-edited
  by a non-Windows developer. OALWrapper had no usable project at first, and
  output folders (`OALWrapper/lib`, `HPL1Engine/lib`) had to be created by hand.
- **`Penumbra.sln` Debug config is broken:** both Debug and Release use the
  Release settings. Making a real Debug build hits `_ITERATOR_DEBUG_LEVEL`
  mismatches unless HPL, OALWrapper and the game are all built in Debug
  (`HPLd.lib`), and even then people hit a Debug assertion in OALWrapper.
- **`GLaux.lib`** is missing from the dependency bundle. It is pulled in by
  `#pragma comment(lib, "GLaux.lib")` at
  `HPL1Engine/sources/impl/LowLevelGraphicsSDL.cpp:22`. GLaux is long
  deprecated and the pragma can likely just go. Someone also hit
  **`TaskKeyHook.lib` not found**; it isn't in the bundle either, and no
  answer was posted.
- **`#include <functional>`** must be added to `TriggerHandler.h` and
  `MainMenu.cpp` for VS2012+ (about 600 errors otherwise). GCC 13 happens to
  compile these today through transitive includes, but MSVC won't.
- **`SDLTexture.cpp:611` `pow` ambiguity** on MSVC: `pow((float)2,(int)mlSizeLevel)`.
  The guide's suggested fix is `(int)pow((float)2,(int)mlSizeLevel)` with a
  float cast, or better, use an integer shift `1 << mlSizeLevel`.
- **OALWrapper version mismatch:** newer OALWrapper made ALUT optional, so the
  stock game needed `WITH_ALUT` defined, the bundle's AL headers swapped for
  OALWrapper's, or an older OALWrapper. Our current OALWrapper + HPL1Engine
  pair no longer uses ALUT at all (no `alut` references in either), so this
  doesn't apply to us. Older `OV_CALLBACKS_DEFAULT` errors came from the old
  vorbis headers in the bundle.
- onager's 2020 guide pins OALWrapper to commit `ae59c6d7bd47…` (via
  jorgeas80's fork) as the one compatible with stock HPL1. **That commit is not
  in our OALWrapper fork's history.** Our HPL1Engine was updated in 2014 for the
  newer OALWrapper, and our OALWrapper compiles, so no pin should be needed.
  This is not verified at runtime.
- **GLee errors in that thread are a different problem from ours.**
  Vasator's VS2008 `GLee.c(746) syntax error : identifier
  'GLee_Lazy_glBlendColor'` was an include-path mistake: `GLee.c` picked up
  the wrong `GLee.h`. He fixed it by pointing `GLee.c` at the right header.
  Our GCC errors come from GLee's `__glext_h_` guard clashing with modern
  Khronos `glext.h`, which no include path will fix. One overlap is worth
  remembering: the bundle ships GLee **5.2** and the engine repo has **5.4**,
  so mixing them up is easy.
- Runtime: run from `redist/`, and copy `materials.cfg` and `resources.cfg`
  alongside the asset folders. Missing them gives errors like "Couldn't load
  image 'effect_white.jpg'" or "Unable to load PointLight2D".

## 6. Community forks and references

- **Kristian-Popov/HPL1Engine**: has a `fix_compilation` branch (2017),
  taking-address-of-temporary, `unistd.h`, overload resolution, bool to
  pointer. Its `improvements` branch adds warning fixes and FLTK removal
  (2020). Kristian-Popov/PenumbraOverture: `improvements` branch.
- **ace17/hpl1engine** (Sebastien Alaiwan, 2017): modern C++ cleanups, build
  instructions, "Enabled AngelScript portability mode" (2020).
- **ArchyInf/HPL1Engine**: newer AngelScript port and rendering refactor (2013).
- **ScummVM `engines/hpl1`** (GPLv3): a complete working port of Penumbra
  Overture with AngelScript 2.35 and open-source Newton. It is the best
  reference for fixes, though it targets ScummVM's own platform layer
  rather than SDL directly.
- Our HPL1Engine fork already includes the 2017 whitespace commits from
  ace17's history.

### Sources

Copies of the forum threads below (as Markdown) and both dependency bundles
were supplied by the owner and are kept outside the repos.

- Frictional forum, "OK.. here's a quick building guide" (Urkle, 2010):
  https://forum.frictionalgames.com/thread-3360.html
- Frictional forum, "Compilation instructions for Ubuntu 12.04 32 bit" (2020):
  https://forum.frictionalgames.com/thread-56758.html
- Frictional forum, "JKO's Guide to compiling Penumbra" (2010, Windows):
  https://forum.frictionalgames.com/thread-3592.html
- Frictional forum, "compiling under Visual Studio 2012" (2013):
  https://www.frictionalgames.com/forum/thread-22527.html
- Frictional forum, "Let's make this working on Android" (2020):
  https://forum.frictionalgames.com/thread-56750.html
- GamingOnLinux, "A look at the Penumbra Collection on Linux with Mesa in
  2020" and comments:
  https://www.gamingonlinux.com/2020/05/a-look-at-the-penumbra-collection-on-linux-with-mesa-in-2020
