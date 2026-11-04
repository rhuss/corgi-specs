# Brainstorm: Custom corgi builds from integrated branches and PRs

**Date:** 2026-11-04
**Status:** active

## Problem Framing

The personal `integration` branch (upstream/main plus in-flight PR branches, rebuilt by the `integration-rebase` skill and `rebuild.sh`) works, but only for us: it hardcodes `tals/corgi` and the `integration` branch, merges only branches present locally or on `origin`, carries its own tooling as cherry-picked commits, and needs Claude Code for conflicts.

Goal: an upstream contribution so that anyone can build a bespoke corgi binary from `upstream/main` plus patches of their choice, rebuilt in a single end-to-end run, including merge conflicts and build failures.
Patches can be local branches, PRs against upstream, PRs on other forks, and branches on other forks.

Success: one command rebuilds, verifies and produces a separately named binary; conflicts and build breaks are handled by a configured agent harness; the report says which integrated branches need work upstream and hands over ready-to-run prompts for that.

Non-goals and anti-goals:
- Not a runtime feature: nothing of the rebuild logic ships in the corgi binary (no `corgi` subcommand). It is a build-time concern.
- A patched binary must never pass as vanilla upstream in `--version`, bug reports or crash reports.
- The tool never rebases, amends or pushes PR branches, and never pushes to upstream.
- Fixes for build breaks are not silently carried forward across rebuilds; the problem must be pushed back to the PR branches.

## Approaches Considered

### A: One upstream PR with everything
Script, prompts, config, build stamping, `--version`, `bugreport`, Sentry tag and `.gitignore` in one PR.
- Pros: one review; everyone gets the full feature at once.
- Cons: large; the AI harness orchestration is the most contestable part and would block the uncontroversial parts.

### B: Two stacked upstream PRs (chosen)
- PR 1: `build.ts` accepts patch metadata and stamps it into the binary (`--version` marker, `bugreport` section, Sentry tag, `corgi-custom` output name).
- PR 2, on top: `scripts/custom-build` with config, rebuild, harness hooks, handoff reports and prompts.
- Pros: PR 1 is small and useful to upstream on its own (honest reports from patched builds); PR 2 can be debated or iterated without blocking it.
- Cons: two PRs to keep in sync.

### C: Upstream only the stamping hook, keep the tool outside
- Pros: zero maintenance burden for upstream; own release cadence.
- Cons: discoverability; misses "everyone can directly use it"; couples to `build.ts` internals from the outside.

### Dropped
- Keep it fork-only (status quo): reaches nobody else.
- Ship as a `corgi` subcommand: wrong layer, it is build-time only.

## Decision

Approach B: two stacked PRs against `tals/corgi`.
Both are dogfooded first by listing them in our own integration build.

## Key Requirements

### Naming (agreed)
- `.custom-build/`: config and `reports/`, gitignored (added to upstream `.gitignore` in the PR).
- Location overridable via `CORGI_CUSTOM_BUILD_DIR` or `--dir` (e.g. a dotfiles repo).
- `scripts/custom-build`: the script, next to `scripts/release`.
- `dist/corgi-custom`: the binary, never colliding with `dist/corgi` or a stable install, so `corgi update` does not overwrite it.
- `+custom` as the marker word in the binary.

### Patch sources (user requirement)
- Local branch.
- PR against upstream by number (fetchable via `refs/pull/N/head`, which also covers fork PRs against upstream).
- PR on another fork (`owner/repo#N`).
- Branch on another fork (repo plus branch).
- PRs merged upstream are skipped and reported for removal.

### Script behavior (agreed)
- Deterministic, no harness required for the main path: add, remove, list, rebuild.
- Rebuild: fetch, back up the current state for rollback, reset onto `upstream/main`, merge each source in order (`--no-ff`), typecheck, test, build, write `dist/corgi-custom`, report.
- rerere replays known conflict resolutions.
- Without a configured harness, it stops on conflict or build failure with a clear exit code, names branch and files, and leaves the state in place; `continue` resumes.

### Harness integration (user requirement: single end-to-end run)
- A resolver is configured in the config as a command template (e.g. `claude -p`, `codex exec`).
- The repo ships harness-neutral prompt files (the current skill, generalized), filled with branch, conflicting files, PR link and description.
- **Merge conflicts:** the agent only resolves and stages; the script then verifies (no conflict markers, no changes outside the conflicted files), commits the merge and continues. Bounded retries, then stop.
- **Build failures (typecheck, tests, build), also on clean merges:** the agent fixes them in a separate `integration: fix <step>` commit on top of the merges, never in a PR branch or merge commit. The failed step and all later ones are rerun, up to about three attempts (proposal).
- **Blame first (proposal):** before calling the harness, rerun the failing step on `upstream/main` alone; if upstream itself is broken, do not call the harness.
- **Trust (proposal):** AI resolutions and fixes are accepted when final typecheck, tests and build pass, and are listed separately in the report.
- **Permissions (proposal):** the resolver config passes a narrow tool allowlist, not blanket permissions.

### Report and handoffs (agreed)
- Each patch is test-merged alone against vanilla `upstream/main` to list which branches need a rebase.
- AI fix commits are not carried forward to later rebuilds. Instead, each affected branch gets a handoff file in `.custom-build/reports/` with: the problem (failing step, error output, the fix applied in the integration build), the cause (upstream drift or a clash with patch X), and a ready-to-run prompt for that branch's worktree, reminding that changes go into the PR branch.
- Same handoff format for three cases: needs rebase on upstream, build failure caused by the branch against upstream, and a clash between two in-flight patches. For clashes, the prompt names the other patch and leaves which side adapts (or waiting for the other to merge) to the person.
- Conflicts that only clash between two integrated patches but are clean against upstream get a note, no prompt.

### Build stamping, PR 1 (agreed; specific formats are proposals)
- Patch metadata compiled into the binary like `config.releaseVersion` / `config.releaseCommit`, recording per patch: source, merged commit SHA, description.
- `corgi --version`: first line keeps its format plus a short marker, e.g. `corgi 0.9.2 (abc1234) +custom (5 patches)`, so first-line parsers keep working.
- Verbose version output lists one line per patch, e.g. `tals/corgi#109 3f2a1c9 Keep blocked agents as jump targets`.
- `corgi bugreport` always includes the full patch list.
- Sentry: a `custom-build` tag for release builds that report crashes.

### Proposals (my call, with reason)
- Config outside the repo by default, structured format instead of a Markdown table: a script that adds and removes entries is fragile on tables, and the integration branch becomes fully disposable (pure upstream plus merges, no carried tooling commits).
- Script in TypeScript with Bun, matching `scripts/release.ts` and `scripts/tooling.ts`, no new dependencies.

## Open Questions
- Config file format and schema (how sources, order and descriptions are written).
- Exact `--version` marker and verbose output format; whether `hello`/RPC also exposes the patch list.
- Whether upstream wants the Sentry tag and how custom builds interact with update channels.
- Resolver command template contract: how context is passed (prompt file, env, stdin) and how success is detected.
- Retry limits and what "nothing changed outside the conflicted files" means when a resolution legitimately needs a companion edit.
- How the branch-name and `integration-prev` style backup are configured for users with several custom builds.
- Migration of our own `INTEGRATION.md` and the `integration-rebase` skill onto the new script.
- How handoff prompts reference worktrees for fork branches the user does not own.
