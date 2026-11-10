# Review Guide: Custom corgi builds from integrated branches and PRs

**Generated**: 2026-11-04 | **Spec**: [spec.md](spec.md)

## Why This Change

Contributors who run corgi with in-flight PRs merged in (their own branches, upstream PRs, fork PRs) keep that integration alive by hand, with ad-hoc scripts that break whenever upstream moves or two patches clash. The resulting binary also looks exactly like a vanilla build, so bug and crash reports from patched builds can mislead maintainers. Nothing today gives a patched binary an honest identity, and nothing rebuilds a patch stack reliably in one run.

## What Changes

Two parts. The first (build identity) stamps a development build with the repository its commit lives in: a build from a fork shows that repository as an https link in `corgi --version --verbose` and in the bug report, so a report from a patched binary says where to look. Plain `--version`, crash events and builds from upstream itself (or releases) are unchanged, and the existing managed-install check keeps `corgi update` from overwriting a development build. (An earlier design stamped the patch list itself, with a `+custom (N patches)` marker and crash tags; the 2026-11-07 clarification dropped it in favor of the repository alone.) The second part adds `scripts/custom-build`, a build-time tool (not a corgi subcommand) that manages a patch list outside tracked files, rebuilds a `custom-build` branch from upstream `main`, verifies it with the repo's own checks, writes `dist/corgi-custom`, and keeps a rollback point. When a merge conflicts or a check fails, it can hand the problem to a configured agent harness, accept the result only after its own checks, and write per-branch handoff prompts so fixes flow back to the PRs. It also rebases the user's own patch branches (own branches and PRs from their fork) onto upstream, inside every rebuild and with a `rebase` command, moving a branch only after its checks pass and pushing only with a lease after a yes or `--push`. No breaking changes.

## How It Works

**Deliverable 1** (`build.ts`, `src/`): a build without `--update-url` (any non-release build) derives its identity from git with `customBuild.fromGit()`: the repository of the branch's remote, or `origin` when that remote is upstream or missing, normalized to `host/owner/repo` without credentials. `build.ts` writes it as `buildRepository` into the generated `build_config` module; the checked-in `src/build_config.ts` exports an empty string. `src/custom_build.ts` holds `repository()`, `custom()`, `lines()` and `versionText()`, used by `main.ts` (`--version --verbose`) and `bug_report.ts` (one indented row under Corgi). A build whose repository is upstream or empty prints exactly what `main` prints.

**Deliverable 2** (`scripts/custom_build/`): a Bun/TypeScript tool split into `config.ts` (JSON config and `run.json` state in `.custom-build/`, overridable by `--dir` and `CORGI_CUSTOM_BUILD_DIR`), `sources.ts` (parsing four source kinds), `git.ts` (fetching PR heads and fork branches into `refs/custom-build/*` without adding remotes, rerere-enabled merges, `merge-tree` checks, worktrees), `github.ts` (optional `gh` lookups), `run.ts` (the `rebuild`/`rebase`/`continue`/`rollback` state machine, persisted before every action), `rebase.ts` (choosing own branches, rebasing them in throwaway worktrees, checks, atomic moves with backups, leased pushes), `steps.ts` (install, typecheck, test, build, plus isolated worktree runs for blame-first and attribution), `harness.ts` (prompt filling, invocation with timeout, acceptance checks) and `report.ts` (reports, handoffs, `latest` link, retention of 10 runs). Distinct exit codes 0 to 5 mark each stop reason. Tests in `tests/custom_build.test.ts` drive temporary git repositories with stub step commands and a stub harness.

## When It Applies

**Applies when**:
- A development binary is built from a fork (Deliverable 1 output changes only then)
- A contributor wants a personal corgi with extra patches rebuilt from current upstream
- A harness command is configured and the user wants unattended conflict and build-failure resolution

**Does not apply when**:
- Building or running vanilla corgi: output stays byte-identical
- Release builds and the managed update channel: custom builds never enter `releases/`
- Integration test suites: not part of the rebuild's verification by default (too slow)
- Fixing patch branches themselves: the tool never amends them, and never rebases or pushes other people's branches; handoff prompts leave fixes to the person. Only the user's own branches are rebased, and pushed only on a yes or `--push`

## Key Decisions

1. **Identity compiled into `build_config`**, not a sidecar file or build-time env var. A sidecar is lost on copy and spoofable.
2. **The identity is the repository, not the patch list.** The custom branch's merge titles and the run report already say what each patch is; the binary only says where to look. Crash events stay unchanged, so no branch names reach Sentry.
3. **The tool is a script, not a corgi subcommand or a Claude skill.** Rebuild logic stays out of the shipped binary and the tool must work without any harness (FR-039).
4. **JSON config** over TOML (Bun cannot write it), YAML (new dependency) or a Markdown table (fragile to rewrite).
5. **PR heads fetched into a private ref namespace** (`refs/custom-build/...`) from the base repo, so fork PRs need no extra remotes and `git branch` stays clean.
6. **Harness invoked as a command template** with the prompt as a file and on stdin; the command owns its own permissions, and the tool decides acceptance with its own checks (no markers, only conflicted files touched, HEAD unchanged).
7. **Blame-first before calling the harness:** a failing step reruns on plain upstream in an isolated worktree; if upstream fails too, no agent runs and no patch is blamed.
8. **Agent fixes are never carried forward.** Every run resets to upstream; handoff prompts are the only way fixes reach the PRs.
9. **Backups only overwritten on success**, and `rollback` restores the run's start state during a stopped run (a planning refinement of FR-036, see research R11).
10. **Own branches only are rebased**, in throwaway worktrees, moved atomically with a backup and an old-value guard after their checks pass, and pushed with `--force-with-lease`; the harness resolves rebase conflicts but never fixes checks on the user's branch.

## Areas Needing Attention

- **Self-hosting during the transition.** The rebuild resets the working tree to upstream `main`. Until both PRs are merged upstream, that tree lacks `scripts/custom-build`, the prompt templates and the `/.custom-build/` ignore entry, so a stop before the tool's own patch is merged leaves `continue`/`rollback` without a wrapper in the tree. Check whether the plan should load prompts at startup and document running the tool from a separate checkout.
- **Identity from git.** `fromGit()` picks the tracked remote, falling back to `origin` when that is upstream. A checkout whose remotes don't reflect where the commit lives (for example a local path as `origin`) stamps nothing, which reads as vanilla.
- **Harness acceptance against a dirty-but-clean checkout.** Acceptance and fix commits now compare against a pre-call `git status` snapshot so untracked files in the user's checkout do not reject resolutions or leak into fix commits. Worth a close look in code review.
- **Attribution heuristics** (files touched by the fix or named in the log) can over- or under-implicate patches; the fallback implicates all merged patches.
- **rerere replay** applies past resolutions, including agent-made ones, without re-asking. That is intended (FR-020), but a bad recorded resolution keeps replaying until `rr-cache` is cleared.
- **Isolated worktrees run `bun install` and a native build** on failure paths, which can be slow.

## Open Questions

- How should the tool behave while upstream does not yet contain it (see self-hosting above)?
- Is the plan's "under 5 s tool overhead for 10 patches" goal meant to be checked, or is it informational?

## Review Checklist

- [ ] Key decisions are justified
- [ ] Breaking changes are documented with migration guidance
- [ ] Scope matches the stated boundaries
- [ ] Success criteria are achievable
- [ ] No unstated assumptions
- [ ] Vanilla `--version`, bug report and crash events are byte-identical to `main`
- [ ] Crash events are unchanged for every build
- [ ] The tool never amends patch branches, never rebases or pushes other people's branches, and never pushes to upstream (including `--push` when `origin` is upstream)

---

<!-- Code phase sections are appended below this line by the phase-manager command -->
