# Roadmap

The ordered plan for getting Penumbra Overture building and running on modern
systems. Each task is a small, separately reviewable step. Nothing here is
started until the owner gives the go-ahead (see [CLAUDE.md](CLAUDE.md)).
Completed tasks get logged in [FIXES.md](FIXES.md). Background and error
lists are in [KNOWN_ISSUES.md](KNOWN_ISSUES.md).

**Target:** native 64-bit builds on Linux first, then Windows, then macOS.
The game should behave the same as the original release.

Legend: **Repo** = where the commit lands. **Verify** = how we know the step
worked. **Decision** = something the owner picks before we start.

---

## Phase 1 — Small compiler fixes (HPL1Engine) ✅ done 2026-11-26

Quick, low-risk fixes that modern GCC/Clang/MSVC reject. Each one is a
one-line or few-line change that keeps behaviour the same. Doing them first
means later steps show only real problems in their build output.

| # | Fix | Files | Change |
|---|-----|-------|--------|
| 1.1 | Address of temporary vertex | `sources/graphics/Renderer2D.cpp:898-923` | Store `cVertex(...)` in a local variable, then pass `&local` |
| 1.2 | Address of temporary matrix | `sources/impl/PhysicsBodyNewton.cpp:104` | Store `GetLocalMatrix().GetTranspose()` in a local, pass `&m.m[0][0]` |
| 1.3 | `return false` from pointer function | `sources/sound/SoundHandler.cpp:289` | `return NULL;` |
| 1.4 | `symlink` undeclared | `sources/impl/LowLevelSystemSDL.cpp` | `#include <unistd.h>` on non-Windows |
| 1.5 | Non-const set comparator | `include/graphics/GraphicsDrawer.h:57` (+ `.cpp`) | Mark `cGfxBufferCompare::operator()` `const` |
| 1.6 | MSVC: `<functional>` missing | PenumbraOverture `TriggerHandler.h`, `MainMenu.cpp` | Add `#include <functional>` (harmless on GCC) |
| 1.7 | MSVC: ambiguous `pow` | `sources/impl/SDLTexture.cpp:611` | `1 << mlSizeLevel` (same result, integer) |

- **Repo:** HPL1Engine (1.1–1.5, 1.7), PenumbraOverture (1.6).
- **Why:** these block compilation or will on Windows; they're purely
  mechanical.
- **Risk:** very low. Kristian-Popov's `fix_compilation` branch made the same
  fixes for 1.1, 1.3 and 1.4, so we can compare.
- **Verify:** each listed error disappears from the build log, and no new ones appear.

## Phase 2 — Build system foundations (all three repos) ✅ done 2026-11-26

Make CMake find dependencies the modern way instead of relying on the
`../dependencies` folder of prebuilt binaries.

**2.1 Raise CMake minimums and clean up the top-level file**
- **Repo/files:** `CMakeLists.txt` in all three repos.
- **Change:** `cmake_minimum_required(VERSION 3.16)`, set C++ standard
  explicitly, use `target_include_directories` / `target_link_libraries` on
  targets instead of global `include_directories` / `link_directories`.
- **Why:** CMake 4.x refuses minimums below 3.5. Target-based setup is also
  what lets Windows/macOS work later.
- **Also fix:** `add_subdirectory(../OALWrapper ../OALWrapper)` uses a
  *binary* dir of `../OALWrapper` relative to the build dir. With a build dir
  directly in the parent folder (e.g. `parent/build`), CMake writes object
  files, Makefiles and `libHPL.a` **into the sibling repos' source trees**.
  Use binary dirs inside the build dir instead (e.g. `${CMAKE_BINARY_DIR}/HPL1Engine`).
  Until then, build from a directory that isn't next to the repos.
- **Risk:** low. The build *layout* stays the same (repos side by side).
- **Verify:** configures with no deprecation warnings on CMake 3.28 (and 4.x
  if available); same set of errors as before.

**2.2 Find system libraries with `find_package`**
- **Repo/files:** HPL1Engine and PenumbraOverture `CMakeLists.txt`.
- **Change:** replace the hard-coded `GL GLU Cg CgGL ogg vorbis ... SDL_image`
  link list with `find_package(OpenGL)`, `find_package(SDL)` etc. Move the
  links onto the `HPL` target, where they belong. The game links only to
  `HPL`.
- **Why:** the engine target currently links nothing itself, so anything else
  that uses it (tools, tests) breaks.
- **Risk:** low. Cg has no standard CMake module, so we write a small `FindCg.cmake`.
- **Verify:** configure prints each library found; link errors are now only
  about Newton/AngelScript.

**2.3 Remove FLTK**
- **Repo/files:** HPL1Engine `sources/impl/LowLevelSystemSDL.cpp` (one
  `fl_alert` call), CMake.
- **Change:** use the platform's native message box instead
  (`MessageBoxW` on Windows; SDL2's `SDL_ShowSimpleMessageBox` once we're on
  SDL2; stderr on Linux until then).
- **Why:** a whole GUI toolkit dependency for one error popup.
- **Risk:** very low. A community fork already did this in 2020.
- **Verify:** builds without FLTK installed; trigger an error and see the message.

**2.4 Drop dead Windows-only bits**
- **Repo/files:** `LowLevelGraphicsSDL.cpp:22` (`#pragma comment(lib,
  "GLaux.lib")`), check where `TaskKeyHook.lib` comes from.
- **Why:** these libraries don't exist anymore and aren't in any bundle.
- **Verify:** handled properly when we reach Windows (Phase 7); for now just
  remove the dead references.

## Phase 3 — Replace GLee (HPL1Engine) ✅ done 2026-11-26

**Decision (made 2026-11-26): GLEW.**
- *GLEW*: closest drop-in for GLee. Same function names (`glBindBufferARB`,
  `glBeginFragmentShaderATI`, ...), extension flags map one-to-one
  (`GLEE_ARB_multisample` → `GLEW_ARB_multisample`), packaged on every
  Linux distro, vcpkg and Homebrew.
- *glad*: generated loader we'd vendor into the repo. No system dependency,
  but more edits, since it uses different naming for extension checks.
- *libepoxy*: nice API, but less common on Windows.

- **Files:** remove `include/GL/GLee.h`, `sources/impl/GLee.c`. Replace
  `#include <GL/GLee.h>` (about 10 files) with `<GL/glew.h>`. `GLeeInit()` →
  `glewInit()` in `LowLevelGraphicsSDL.cpp:226` (after the GL context is
  created). `GLEE_*` checks → `GLEW_*` (a handful, mostly in
  `LowLevelGraphicsSDL.cpp`).
- **Why:** GLee's header guard check clashes with modern Mesa/Khronos
  headers. That causes about 70 "not declared" errors and can't be fixed by
  include paths. GLee is also abandoned (its site is gone).
- **Risk:** low to medium. The calls are identical, but extension detection
  must be right or the renderer will pick the wrong shader path. ATI
  fragment shader fallbacks only matter on very old hardware.
- **Verify:** all GL errors gone from the build; at runtime the log's
  "supported features" list matches the original game's log on the same
  machine.

## Phase 4 — AngelScript from source (HPL1Engine) ✅ done 2026-11-26

**Decision (made 2026-11-26): stay on 2.7.1b for now**, built from source in
portability mode. The reason is to make only one big change at a time, and
Newton is the big one. A modern AngelScript port can come later as its own
project.
- Keeping 2.7.1b means the scripts behave exactly as they always have.
  Portability mode avoids the stack-alignment problem behind the AMD/Mesa
  vice crash, and HPL1 already supports it.
- Porting to 2.3x changes binding APIs (`asBEHAVE_*`, modules, string type)
  and can subtly change script behaviour across all the game's map scripts.
  ScummVM did this, so it's possible, but it's a bigger job.

**4.1 Vendor AngelScript 2.7.1b**
- **Repo/files:** new `HPL1Engine/external/angelscript/` (the SourceForge
  `branches/2.7.1` sources + licence, zlib), with its own small CMake target.
- **Change:** one-line GCC fix (`angelscript.h:784`, `return asUPtr();`).
  Always build with `AS_MAX_PORTABILITY`, and define it publicly so HPL1 and
  the game also use the generic calling convention.
- **Why:** removes the prebuilt 32-bit `libangelscript.a`.
- **Risk:** low for compiling, medium at runtime. 2.7.1b on 64-bit is not
  how it originally shipped, so script bindings must be checked. Community
  reports mention `SetJointControllerPropertyFloat` (the mine trapdoor)
  failing to bind.
- **Verify:** builds on x86_64 (already done in a scratch build); at
  startup, every script function registers without errors in the log; the
  trapdoor works in-game.

**4.2 Silence `__stdcall` warnings**
- **Files:** `include/system/Script.h`.
- **Change:** only define `__stdcall` as an attribute on 32-bit x86; empty
  elsewhere (removes 173 warnings).
- **Verify:** warnings gone.

## Phase 5 — Replace the prebuilt Newton 1.53 (HPL1Engine)

All Newton code lives in 11 files, `HPL1Engine/sources/impl/*Newton*` (about
2,700 lines). The game itself never calls Newton.

**Decision (made 2026-11-26): A, Julio Jerez's 1.53 compatibility layer on
Newton 4**, pinned to upstream commit `294f27f6cde10c3c269041c3c4944cefb0480dbf`
(master as of 2026-11-25). Upstream is still in progress, so we treat this as
an experiment and compare game feel carefully. **B (Newton 2.36, ScummVM
route) remains the documented fallback** if A turns out incomplete or changes
the game too much.

- **A. Julio Jerez's Newton 1.53 compatibility layer (recommended).** The
  Newton author is writing a backport of the *1.53 API* on top of Newton 4.
  It was started for an HPL1/Penumbra port (newton-dynamics discussion #435,
  2026). Path: `newton-4.00/applications/toolsAndWrapers/newton-1.53` in
  https://github.com/JulioJerez/newton-dynamics (master, still being worked on).
  Licence: zlib.
  - Our first check: its `Newton.h` declares **every one** of the original
    1.53 functions, including all 90 that HPL1 uses.
  - **Corrected by the Phase 5 walkthrough (compiling it and checking
    symbols):** 12 functions HPL1 calls are declared but have **no
    definition** (only inside `#if 0`), and 8 more are stubs that do
    nothing. The hinge/slider/corkscrew wrappers ignore HPL's limit
    callbacks, and the contact callbacks run in a different order than
    1.53. Details and the fix plan are below.
  - It was Windows-first: MSVC-only CMake and 3 small GCC errors (fixed).
  - Underneath, it is the Newton 4 solver, so physics feel needs in-game
    testing (true of any option except the 32-bit blob).
- **B. Port HPL1 to Newton 2.36** (ScummVM's approach). Proven to work, but
  means rewriting the contact callbacks and porting signatures across all 11
  files.
- **C. Keep the 32-bit blob.** Dead end for 64-bit.

### Plan (A, approved 2026-11-26)

**Priority: keep the physics feel as close to 1.53 as possible** (see
CLAUDE.md). Where Newton 4 defaults and 1.53 disagree, choose 1.53.

One commit per step, each logged in FIXES.md. Local changes to the layer are
listed in `HPL1Engine/external/newton-1.53/README.md` and never sent upstream.

1. ✅ Vendor the layer + its Newton 4 core, unmodified and pinned, into
   `HPL1Engine/external/newton-1.53/` (no `demos/`, no upstream CMake).
2. ✅ Our own CMake target `newton153`: static, C++17 for this target only,
   scalar vector math on GCC/Clang, optimised even in Debug, links Threads.
   Not linked into HPL yet (`EXCLUDE_FROM_ALL`).
3. ✅ The 3 GCC fixes. The library builds on GCC 13 x86_64.
4. ✅ Define the 12 missing functions HPL1 calls (`NewtonCollisionCollide`,
   joint force/angle/omega readouts, contact force, per-body freeze
   thresholds, solver/friction model, material continuous collision).
   Along the way: fixed a matrix-copy memory overrun in the core on GCC/Clang.
5. ✅ Fill the stubs: `NewtonAddBodyImpulse`, `NewtonWorldFreezeBody`, the
   four `Calculate Stop` helpers, body continuous-collision get/set. Also
   fixed the inverted `NewtonBodyGetSleepingState` and made
   `NewtonBallSetConeLimits` use the cone axis it's given.
6. ✅ Make the joint wrappers honour HPL's limit callbacks (return value,
   acceleration, min/max friction). **Highest risk for doors and levers.**
7. ✅ Contact callbacks: run begin/process/end back to back per pair (HPL keeps
   their state in statics), keep Newton single-threaded, 1 substep instead of
   3, match 1.53's restitution clamp. Check `NewtonMaterialGetContactForce`
   there. Along the way: contacts now use the bodies' materials (they never
   did, so no callbacks ran), the SIMD8 solver was swapped for the standard
   one, and stale contacts are dropped.
8. ✅ Link HPL to `newton153`, drop the IMPORTED `libNewton.a` and
   `HPL_DEPENDENCIES_DIR`. (The layer's `Newton.h` uses `size_t` without
   including `<stddef.h>`; add it if anything trips on it.)
9. ✅ A small headless physics test (falling box, hinge limit, impulse Δv,
   shape collision, callback order): `newton153_test`, 53 checks. See
   CLAUDE.md, "Physics test".

**Phase 5 is complete.** The game builds and links with the layer; nothing
is verified in-game yet.

After assets: in-game checks (Phase 6) and a side-by-side with the retail
Steam build. Feel knobs to watch: solver iterations for "exact" mode
(`HPL1_SOLVER_ITERATIONS_EXACT`, 16), and how quickly bodies freeze once
under their threshold (Newton 4 freezes them straight away; 1.53 may have
waited a few frames).

### Plan if we go with B

As before: vendor Newton 2.36, mechanical signature changes (`shapeID`,
`NewtonCreate`, callbacks gain `timestep`/`threadIndex`), restructure the
contact callbacks in `PhysicsMaterialNewton.cpp` following ScummVM, replace
the freeze/sleep API, check joints and the character body, remove the blob.

## Phase 6 — First run and in-game testing (needs the owner's assets)

**6.1 Running from a build dir**
- Add a CMake option or doc steps for pointing at an assets folder
  (`redist/` with `materials.cfg`, `resources.cfg`).
- **Verify:** game starts and reaches the main menu.

**6.1b Finding the game data automatically** (owner's idea, not started)
- The game currently expects its data next to it (the working folder: the
  `redist/` layout in the source release). The Steam Linux install keeps the
  same files (`graphics/`, `models/`, `materials.cfg`, `resources.cfg`, ...)
  directly in the game's root folder, not in `redist/`.
- Search order:
  1. Locally: the working folder or `./redist`, recognised by the data
     folders/config files being present.
  2. Common install paths: the Steam library (Linux first; also Windows),
     then GOG, and possibly other stores.
  3. If nothing is found: ask the player to pick the Penumbra Overture
     folder. That needs some kind of launcher/GUI; to be worked out later.
- Remember the folder that worked so the search only runs once.

**6.2 Test checklist** (owner plays, we compare against the original game):
- Menus, options, saving/loading.
- Rendering at each shader quality level (community reports: blue screen
  above "Very low", zoom issues).
- Physics: picking up, dragging and throwing objects; doors and levers;
  stacking; ladders; the character walking into things.
- Sound: impact and slide sounds from contacts, music, streams.
- Scripts: the mine trapdoor (`SetJointControllerPropertyFloat`), map
  transitions.
- AMD/Mesa: the scenes that crashed before (leaving the boat at the start).
- Log file: no script-binding or extension errors.

Every bug found becomes its own small fix plus a FIXES.md entry.

## Phase 7 — Windows

- Use CMake + current Visual Studio, with dependencies from vcpkg (SDL,
  GLEW, OpenAL Soft, ogg/vorbis). Cg from NVIDIA's last toolkit (x86/x64).
- Retire `Penumbra.sln` / `.vcproj` (VS2003) once CMake works, or keep them
  documented as historical.
- Fix MSVC-specific issues (the `<functional>` and `pow` fixes from Phase 1,
  `win32.cpp` paths, and the Debug/Release mismatch described in
  KNOWN_ISSUES.md).
- **Verify:** builds in Debug and Release; game runs.

## Later / optional

- **SDL 1.2 → SDL2 (or SDL3).** Better input, high-DPI, fullscreen handling,
  and controller support. OALWrapper already has `USE_SDL2`. Medium-sized:
  input, window, and text/font code.
- **Cg → GLSL.** Cg is discontinued and x86-only. Needed for ARM and Apple
  Silicon. The shaders can be translated (cgc can output GLSL as a starting
  point). Medium-large.
- **Modern AngelScript (2.3x).** Only if 2.7.1b causes problems.
- **macOS** via CMake (after Cg → GLSL).
- **Performance:** profile once it runs (renderer state changes, the
  occlusion queries, physics step).
- **CI:** a GitHub Actions workflow that builds on Linux and Windows for
  every push, so regressions show up right away.
