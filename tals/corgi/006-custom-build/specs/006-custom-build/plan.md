# Implementation Plan: Custom corgi builds from integrated branches and PRs

**Branch**: `006-custom-build` | **Date**: 2026-11-10 (first plan 2026-11-04) | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/006-custom-build/spec.md`, evolved 2026-11-07 and 2026-11-10

## Summary

Two deliverables, stacked. Deliverable 1 (#116, implemented) makes every development build read its identity from git: the repository of the branch's remote, compiled into `build_config`; a build from a fork shows it as an https link in `corgi --version --verbose` and the bug report; releases and builds of upstream are unchanged. Deliverable 2 (#117) adds `scripts/custom-build`, a Bun/TypeScript build-time tool that keeps a JSON patch list in `.custom-build/`, fetches upstream PRs, fork PRs and fork branches into a private ref namespace, rebuilds the custom branch from upstream `main` with rerere and merges titled `Merge <source>: <description>`, verifies with the repo's own checks (retrying a failed step once), writes `dist/corgi-custom`, calls a configured harness for conflicts and build failures with tool-side acceptance checks, attributes fixes in isolated worktrees, and writes run reports plus per-patch handoff prompts. Implemented except User Story 6.

The 2026-11-07 evolution adds User Story 6: the tool rebases the user's own patch branches onto upstream `main`, inside `rebuild` (never stopping it) and as a `rebase` command (stopping at unresolved conflicts), with the harness resolving rebase conflicts, checks on each rebased branch alone before it moves, backup refs, and a confirmed force-push with lease. Design decisions are in [research.md](research.md) (R25 to R32 for rebasing).

The 2026-11-10 evolution extends User Story 6 twice. `status` now also names own branches it leaves alone and why (a worktree checkout, a pending push, divergence), instead of silently omitting them. And `rebase`/`rebuild --in-worktree` rebases a branch checked out in a worktree of the user's inside that worktree, so the checkout moves with its branch: the rebase needs the worktree's tracked files committed, the checks run there, the backup ref is kept from the rebase's start (the rebase itself moves the branch), and any failure resets the branch and checkout back to the tip they started from.

## Technical Context

**Language/Version**: TypeScript on Bun (repo-pinned Bun; `bun-types` 1.4.2 in typecheck)

**Primary Dependencies**: none new. `git` 2.38+ (merge-tree), optional `gh` CLI (also needed to recognize own PRs), optional harness command

**Storage**: files: `.custom-build/config.json`, `.custom-build/run.json`, `.custom-build/reports/`, `.custom-build/work/`; refs under `refs/custom-build/` (fetched patches, rebase backups)

**Testing**: `bun test src` for Deliverable 1 units (`src/custom_build.test.ts`, bug report tests); `bun test tests/custom_build.test.ts` for the tool against temporary git repositories with stub step commands and a stub harness

**Target Platform**: macOS arm64 and Linux (where corgi builds today)

**Project Type**: CLI application (corgi) plus build-time script

**Performance Goals**: none beyond the spec. Cost is dominated by fetches and verification steps. Isolated runs happen only on failure paths, plus one install, typecheck and test run per own branch that a rebase moved.

**Constraints**: never touch upstream or branches the user doesn't own; own branches move only after their checks pass; no new dependencies; outputs of releases and upstream builds unchanged; tool works without any harness

**Scale/Scope**: tens of patches per config, a handful of them own branches; one config directory per custom build

## Global Constraints

Copied from the spec; every task inherits them.

- "git 2.38 or newer is available (needed to test-merge without touching the working tree)."
- "The tool is written in the repository's existing scripting runtime and adds no new dependencies."
- FR-007: "For releases and for development builds of upstream `main` with no merges on top, version output, bug report and crash events MUST be identical to today."
- FR-010: "The tool MUST be a build-time script in the repository's scripts area; it MUST NOT be a corgi subcommand."
- FR-028: "The tool MUST never amend patch branches, never rebase or push branches the user doesn't own (FR-040), and never push to upstream."
- FR-039: "The tool MUST work without any agent harness installed and without Claude Code."
- FR-042: "The rebase phase MUST NOT stop a rebuild."
- FR-044: "Each rebase MUST run in a throwaway worktree, never in a checkout the user works in, and MUST NOT touch a branch that is checked out in any worktree." Exception: with `--in-worktree`, a branch checked out in a worktree of the user's is rebased inside that worktree; its tracked files must be committed, its untracked files and the worktree itself are never removed, and a failure resets the branch back with its backup (FR-047) kept from the rebase's start.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is the unfilled template, so there are no project principles to gate on. Repository conventions from `AGENTS.md`/`HACKING.md` are used instead: build-time tooling lives in `scripts/` (TypeScript on Bun with a sh wrapper), unit tests next to sources, checks are `bun typecheck.ts`, `bun test src`, `bun build.ts`, tests wait on conditions rather than time (sleep budget). The design follows them. Post-design re-check (2026-11-07): pass.

## Project Structure

### Documentation (this feature)

```text
specs/006-custom-build/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── build-identity.md
│   ├── cli.md
│   ├── harness.md
│   └── handoff.md
└── tasks.md            # /speckit-tasks, extended by /speckit-converge
```

### Source Code (repository root)

```text
# Deliverable 1: build identity (#116, implemented)
build.ts                         # fromGit(root) for development builds → buildRepository
src/build_config.ts              # + buildRepository = ""
src/custom_build.ts              # Identity, repository(url), custom(), lines() (https link), versionText(), fromGit()
src/custom_build.test.ts
src/main.ts                      # --version --verbose
src/bug_report.ts                # the fork's link under the Corgi row
HACKING.md, docs/reference.md

# Deliverable 2: custom build tool (#117)
scripts/custom-build             # sh wrapper → bun run scripts/custom_build/main.ts (or the run's tool copy)
scripts/custom_build/
├── main.ts                      # argument parsing, command dispatch, exit codes; + rebase command, --rebase, --skip-branch, --in-worktree
├── config.ts                    # dir resolution, config.json and run.json; + rebase mode, RebaseResult, run kind
├── sources.ts                   # add argument → PatchSource, display(), slug()
├── git.ts                       # spawn helpers, fetch, merge, merge-tree, worktrees; + rebase, update-ref CAS, push with lease
├── github.ts                    # gh-backed PR state/title/merge commit; + PR head repository and branch
├── run.ts                       # rebuild/continue/rollback state machine; + rebase phase, rebase run, push offer, status skip reasons
├── rebase.ts                    # NEW: own-patch selection, rebase in a worktree, conflict loop, checks, apply, push; + in-worktree rebase and backup timing
├── steps.ts                     # verification steps, flaky retry, isolated runs, attribution
├── harness.ts                   # prompt filling, invocation (cwd parameter), timeouts, acceptance checks
├── report.ts                    # report.md, handoffs, latest link, retention; + rebase section, rebase-checks handoff
└── prompts/
    ├── conflict.md              # also used for rebase conflicts (commit being replayed instead of patch merge)
    ├── build.md
    └── handoff.md
tests/custom_build.test.ts       # temp repos, stub harness, stub steps; + rebase scenarios
docs/custom-builds.md            # user guide; + rebasing own branches, "What it never touches" rewritten
```

**Structure Decision**: Single project. Deliverable 1 touches `src/` and `build.ts` only; Deliverable 2 is self-contained under `scripts/custom_build/`; every binary it builds carries Deliverable 1's identity without any option. Rebasing gets its own module `rebase.ts` because it has its own loop, state and failure rules; `run.ts` calls it as a phase and as a command.

## Design Notes

- **Testability of the tool**: config may carry an undocumented `steps` override used by tests to replace the real checks with fast stub commands; the build stub writes the merge log as a fake binary. `main` takes an injectable clock and `ask` function, so the push confirmation is testable without a terminal.
- **State machine**: `rebuild` = preflight → fetch all → rebase phase (own patches, never stops) → start (record start tip, copy binary, reset branch to upstream) → merge loop (cursor) → verify loop (step, flaky retry, restart from the first step after a fix) → report (rebase check, attribution, handoffs) → finalize backups → afterBuild → push offer. `rollback` records phase `rollback` before restoring the tree and reinstalls dependencies afterwards, so `continue` can finish a rollback whose install failed. `rebase` = preflight → fetch all → rebase loop (cursor over own patches, can stop) → report → push offer. Each persists `run.json` (with `kind: "rebuild" | "rebase"`) before acting, so `continue` after Ctrl-C resumes safely; `rebuild` refuses while either kind is stopped.
- **Error boundaries**: all fetches happen before any branch moves or the reset, so an unfetchable patch fails with exit 1 and nothing changes. An own branch moves only by compare-and-swap after its checks passed, so an interrupted rebase leaves it at its old tip. A branch rebased in its own worktree (`--in-worktree`) is the exception the spec names: the rebase itself moves it, so its backup ref is written before the rebase starts and every failure path (failed rebase, failed checks, no commits left, rejected resolution) resets it to that tip.
- **Harness kinds** share one invocation path (`harness.ts`), differing in prompt template, working directory and acceptance check. Rebase conflicts reuse the conflict template and acceptance check, run in the rebase worktree.

## Complexity Tracking

No constitution violations to justify.
