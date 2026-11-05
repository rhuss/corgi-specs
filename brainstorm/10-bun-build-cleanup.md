# Brainstorm: Clean up the temp files bun compile leaves behind

**Date:** 2026-11-05
**Status:** idea

## Problem Framing

Each `bun build.ts` run leaves a `.<hash>-00000000.bun-build` file in the repository root. `Bun.build({ compile })` (build.ts, the native binary step) writes it as a temp file and does not always remove it. One appeared during the first `scripts/custom-build rebuild` in corgi-integration; 15 had piled up there over time.

Two effects:

- `git status` fills with untracked noise.
- `build.ts` stamps the build as `<commit>-dirty` whenever `git status --porcelain` prints anything, untracked files included. A leftover from one build makes every following build report `-dirty` in `--version`, bug reports and Sentry, even from a clean tree. That undermines the build identity that #116 relies on.

Success: a build from a clean tree reports a clean commit, and the repository root stays clean after any build, successful or failed, whichever path started it (`bun build.ts`, `scripts/custom-build`, release builds).

Anti-goals: hiding the files instead of removing them; deleting temp files of another build running in the same checkout at the same time; growing the custom-build PR (#117) for a problem that predates it.

## Approaches Considered

### A: Remove them in the custom-build `afterBuild` command
- Pros: local, no code change, works today.
- Cons: only after a fully successful custom-build run; plain `bun build.ts` still leaks; every user has to know to do it.

### B: The custom-build tool deletes the temp files its build step created
- Pros: cleans up instead of hiding; covers failed custom builds; benefits every tool user.
- Cons: cleans up at the wrong layer: `bun build.ts` for `dist/corgi` and release builds still leak. Grows #117's scope and adds a path to its "what it never touches" list.

### C: Ignore `.*.bun-build` in `.gitignore`
- Pros: one line; the files no longer count as dirty and vanish from `git status` for every build.
- Cons: hides the leak instead of fixing it; the files keep piling up on disk.

### D: `build.ts` removes the temp files its own compile created
- Pros: fixes the leak where it happens, for every build path; the `-dirty` stamp is right again without special cases; the tool and #117 stay unchanged.
- Cons: needs care to delete only the files this run created (snapshot before `Bun.build`, delete new ones in a `finally`), so a concurrent build in the same checkout keeps its temp file.

## Decision

Not decided yet. The user leaned toward B as cleaner than A and C; the recommendation (Claude's call) is D, B's idea applied one level down in `build.ts`, because the leak comes from the build itself and B would leave `bun build.ts` leaking. Delivered as a separate small upstream PR on tals/corgi, then added to the custom build with `scripts/custom-build add <branch>` until it lands.

## Key Requirements

- After any `bun build.ts` run, successful or failed, no new `.*.bun-build` file remains in the repository root.
- Files that existed before the run are left alone (Claude's call: another build may be running in the same checkout).
- A build from a clean tree is stamped with the plain commit, not `-dirty`.
- Independent of #116 and #117; a branch from `upstream/main`.

## Open Questions

- Why bun leaves the file behind: every compile, only on some outcomes, or a known Bun bug with a fix in a newer release (which would make a version bump the real fix).
- Whether the `-dirty` check should ignore untracked files altogether, as a separate question from the leak.
- How to test it: a build test that checks the root for new `.*.bun-build` files, or a unit test around the cleanup helper.
- Whether to also add `.*.bun-build` to `.gitignore` as a belt-and-braces measure for crashes that skip the `finally`.
