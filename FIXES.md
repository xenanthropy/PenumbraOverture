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

_No entries yet._

## Dependencies

_No entries yet._

## Gameplay and engine bugs

_No entries yet._

## Performance

_No entries yet._

## Documentation

### Add CLAUDE.md and FIXES.md
- **Repo:** PenumbraOverture
- **Files:** CLAUDE.md, FIXES.md
- **Change:** Added collaboration notes, an analysis of the three repos, and
  this change log.
