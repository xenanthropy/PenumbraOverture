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

## 5. Community forks and references

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

- Frictional forum, "OK.. here's a quick building guide" (Urkle, 2010):
  https://forum.frictionalgames.com/thread-3360.html
- Frictional forum, "Compilation instructions for Ubuntu 12.04 32 bit" (2020):
  https://forum.frictionalgames.com/thread-56758.html
- Frictional forum, "Let's make this working on Android" (2020):
  https://forum.frictionalgames.com/thread-56750.html
- GamingOnLinux, "A look at the Penumbra Collection on Linux with Mesa in
  2020" and comments:
  https://www.gamingonlinux.com/2020/05/a-look-at-the-penumbra-collection-on-linux-with-mesa-in-2020
