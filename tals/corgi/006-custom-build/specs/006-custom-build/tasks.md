# Tasks: Custom corgi builds from integrated branches and PRs

**Input**: Design documents from `specs/006-custom-build/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Included. The spec gives an independent test per story and the repo requires `bun test src` to pass; tool tests follow `tests/release.test.ts` (import the script modules, temporary git repos).

**Delivery order**: Phase 3 (US1) is Deliverable 1 and is committed before any Deliverable 2 task, so PR 1 can be cut at the end of Phase 3 (research R20).

## Format: `[ID] [P?] [Story] Description`

## Phase 1: Setup

- [X] T001 Confirm the baseline is green before changes: run `bun typecheck.ts` and `bun test src` in the repo root and note any pre-existing failures in specs/006-custom-build/tasks.md under "Notes"

---

## Phase 2: Foundational (Deliverable 1 prerequisites)

- [X] T002 Create src/custom_build.ts exporting `type BuildPatch = { source: string; commit: string; description: string }`, `validate(value: unknown): BuildPatch[]` (array of objects with three non-empty strings, `description` single line; throws `Error("InvalidPatches: <reason>")` otherwise), `marker(patches)` returning `""` for an empty list, ` +custom (1 patch)` for one and ` +custom (N patches)` otherwise, and `verboseLines(patches)` returning `  <source> <commit.slice(0,7)> <description>` per patch in order (contracts/build-identity.md)
- [X] T003 Add `export const customPatches: import("./custom_build").BuildPatch[] = [];` to src/build_config.ts (type-only import so the generated replacement stays a plain module)

---

## Phase 3: User Story 1 - A patched binary never passes as vanilla (Priority: P1) 🎯 Deliverable 1

**Goal**: build metadata in, honest identity out (FR-001 to FR-009).

> **Superseded**: T002 to T013 describe the first design (`--patches`, `+custom`, crash tags, update message). #116 replaced it with the identity read from git, later reduced to the fork's https link (spec sessions 2026-11-05 and 2026-11-07). The checked boxes record history; FR-001 to FR-009 are met by the #116 commits.

**Independent Test**: build with and without `--patches`; compare `--version`, `--version --verbose`, bug report, crash event, `corgi update` (quickstart Deliverable 1).

### Tests for User Story 1

- [X] T004 [P] [US1] Create src/custom_build.test.ts covering `validate` (accepts valid list, rejects non-array, missing fields, empty strings, multi-line description), `marker` (0, 1, 5 patches), `verboseLines` (order, 7-char commit, fork PR `owner/repo#2`, fork branch `owner/repo:branch` display strings passed through)
- [X] T005 [P] [US1] Extend src/crash.test.ts: event for a custom build carries `tags.custom_build === "true"` and `tags.custom_patches === "2"` and no patch source/description strings anywhere in the serialized event; vanilla event has neither tag (test the pure `event()` builder extracted in T010, so no `init()` or native call is needed)
- [X] T006 [P] [US1] Extend src/bug_report.test.ts: `build()` with patches contains ` +custom (2 patches)` right after `(<commit>)`; the summary from `text()` (the default `corgi bugreport` box and the issue URL body) lists the patches as compact `<source> <commit[0:7]>` lines right after the build line; the full report contains a `## Custom build` section with the verbose lines after `## Report`; with an empty list both are absent and output equals today's (make the patch list injectable, defaulting to `config.customPatches`)

### Implementation for User Story 1

- [X] T007 [US1] In build.ts add option `--patches <file>`: read JSON, run it through `validate` from src/custom_build.ts (fail the build with the error message on invalid input), and add `customPatches` to the `config` object so it is emitted into the generated build_config module and included in `inputsSHA256`; absent option or empty array yields `customPatches=[]`
- [X] T008 [US1] In src/main.ts `--version`: print `corgi ${releaseVersion} (${releaseCommit})${marker(customPatches)}`; when `args` contains `--verbose` also print `verboseLines` one per line (FR-003, FR-004)
- [X] T009 [US1] In src/bug_report.ts: append `marker(...)` after `(${config.releaseCommit})` in `build()`, list the patches compactly (`<source> <commit[0:7]>`, one per line) right after the build line in `text()` so the default `corgi bugreport` box and the issue URL carry them, and add the `## Custom build` section (verbose lines with descriptions) right after the `## Report` block in the full report, only when patches exist (FR-005, FR-007)
- [X] T010 [US1] In src/crash.ts: extract the event object built inline in `notePanic` into an exported `event(exception: Error, id: string, role: string, release: string, patches: BuildPatch[] = config.customPatches)` that `notePanic` calls; add `custom_build: "true"` and `custom_patches: String(n)` to `tags` only when patches exist; never include names, sources or descriptions (FR-006)
- [X] T011 [US1] In src/updates.ts `command()`: before `managed()`, if `config.customPatches.length > 0` print `This is a custom build (N patches). Rebuild it with scripts/custom-build rebuild instead of updating.` to stderr and `throw Error("CustomBuild")`; like `UnmanagedInstallation` today it needs no entry in `failure()` in src/main.ts (the default case prints `corgi: CustomBuild`, exit 1) (FR-009)
- [X] T012 [US1] Document `corgi --version --verbose` and the `+custom` marker in docs/reference.md next to the existing `--version` entry
- [X] T013 [US1] Verify Deliverable 1: `bun typecheck.ts`, `bun test src`, `bun build.ts`, then write a two-entry patches JSON to the scratch dir, run `bun build.ts --outfile dist/corgi-custom --patches <file>` and check `./dist/corgi-custom --version`, `--version --verbose`, `./dist/corgi-custom update` and that `./dist/corgi --version` is unchanged; commit as `build: stamp custom patch metadata into the binary`

**Checkpoint**: Deliverable 1 complete and committed (PR 1 cut point).

---

## Phase 4: Foundational for Deliverable 2

- [X] T014 Create scripts/custom-build (POSIX sh, executable, same shape as scripts/release) that execs `bun run "$root/scripts/custom_build/main.ts" "$@"`, and add `/.custom-build/` to .gitignore; the wrapper prefers `<config dir>/work/tool/main.ts` when `<config dir>/run.json` exists (a run in progress), so `continue`, `rollback` and `status` work even when the reset tree has no tool
- [X] T015 [P] Create scripts/custom_build/sources.ts: `PatchSource` union `{kind:"branch",name} | {kind:"pr",repo,number} | {kind:"fork-branch",repo?,url?,branch}`; `parse(arg, upstreamRepo)` per research R8 (`#123`, `123`, upstream pull URL → upstream PR; `owner/repo#12` and pull URLs → pr; `owner/repo:branch`, `https://github.com/owner/repo/tree/branch`, `<git url>#branch=<name>` → fork-branch; else branch); `display(source)` per data-model.md ("`<repo>#<number>`", "`<repo>:<branch>`", "`<name>`"); `slug(display)` lowercased with non-alphanumerics replaced by `-`; `fetchUrl(source)` (`https://github.com/<repo>.git` or the given url); `privateRef(source)` under `refs/custom-build/<owner>/<repo>/pull/<N>` or `.../heads/<branch>`
- [X] T016 [P] Create scripts/custom_build/config.ts: `resolveDir({dir, env, repoRoot})` (`--dir` > `CORGI_CUSTOM_BUILD_DIR` > `<repoRoot>/.custom-build`); `load(dir)` returning defaults when absent (`version: 1`, `upstream: {repo: "tals/corgi", branch: "main"}`, `branch: "custom-build"`, `patches: []`, `retired: []`), refusing `version` > 1, validating `harness.command` "non-empty array of strings", `afterBuild` array of strings, optional undocumented `steps` override; `save(dir, config)` with 2-space JSON and trailing newline, creating the directory; `loadRun/saveRun/clearRun` for run.json with the RunState fields from data-model.md
- [X] T017 [P] Create scripts/custom_build/git.ts: `git(args, opts)` spawn helper returning `{code, stdout, stderr}` with `-c rerere.enabled=true -c rerere.autoUpdate=true` prepended for merges and for the commit that concludes a merge (rerere records the resolution at commit time, so FR-020 replay depends on it); helpers `repoRoot()`, `requireVersion("2.38")`, `isClean()` (tracked files only), `currentBranch()`, `revParse(ref)`, `isAncestor(a,b)`, `fetch(url, refspec)`, `fetchRemote(remote, branch)`, `resolveUpstreamRemote(config)` (`upstream` if present else `origin`), `mergeNoFF(ref, message)`, `unmergedFiles()`, `mergeTreeCheck(base, ref)` → `{clean, files}` via `git merge-tree --write-tree --name-only --no-messages`, `filesTouchedByMerge(sha)` via `git diff --name-only <sha>^1 <sha>`, `addWorktree(path, commit)`/`removeWorktree(path)`
- [X] T018 [P] Create scripts/custom_build/github.ts: `prInfo(repo, number)` using `gh pr view <N> --repo <repo> --json state,title,mergeCommit` when `gh` is on PATH, returning `{state, title, mergeCommit} | null`; `stripArea(title)` removing a leading `area: ` prefix
- [X] T019 Create tests/custom_build.test.ts scaffolding: helper that builds a temporary "upstream" bare repo plus a clone with `upstream` remote, commits a base file, and a helper to create patch branches; a helper that puts a stub `gh` script first on `PATH` returning canned `gh pr view` JSON (used by T020 and T029); unit tests for sources.ts parse/display/slug (all forms in R8) and config.ts dir resolution, defaults, version refusal, save round-trip

---

## Phase 5: User Story 2 - Rebuild a custom binary in one command (Priority: P1)

**Goal**: deterministic `rebuild`, `continue`, `rollback`, `status` without any harness (FR-010, FR-014, FR-015, FR-017 to FR-021, FR-028, FR-036, FR-038, FR-039).

**Independent Test**: temp repo with patches of all four kinds that merge cleanly; `rebuild` exits 0, stub build writes `dist/corgi-custom`, patches.json lists them (quickstart Deliverable 2 steps 1 to 3, 6).

### Tests for User Story 2

- [X] T020 [P] [US2] In tests/custom_build.test.ts add rebuild tests with stub `steps`: clean merge of a local branch, an upstream PR (create `refs/pull/1/head` in the bare upstream), a fork PR and a fork branch (second bare repo addressed by file URL); merge commits named `custom-build: merge <display>` in order; patches.json content; refusal on dirty tracked files (exit 1, nothing changed); refusal when run.json exists (exit 1); unfetchable patch fails with exit 1 before the branch is reset; patch already contained is skipped as `already-contained`; upstream PR reported merged by the stub `gh` is skipped as `skipped-merged` and the report says it can be removed; closed-unmerged PR is merged with a `closed-warning`; patch branch tips and the upstream ref are identical before and after the run (SC-007); `afterBuild` stub runs only after a successful run, not after a stopped one (FR-038)
- [X] T021 [P] [US2] Add conflict-without-harness tests: two conflicting patches → exit 2, files printed, merge left in progress, run.json kept; user resolves and stages, `continue` → exit 0; rerere replays the same resolution on the next rebuild without stopping
- [X] T022 [P] [US2] Add backup and rollback tests: success sets `<branch>-prev` to the start tip and moves the start binary copy to `dist/corgi-custom.prev`; a stopped run leaves both untouched; `rollback` during a stopped run restores start tip and start binary and removes run.json; `rollback` with no run restores `-prev` and `.prev`; first-ever run `rollback` reports nothing to roll back; after a successful run and after a stopped run the custom branch is checked out; a local own-branch strictly behind `origin/<branch>` is merged from origin with a note, and a diverged one uses local with a warning
- [X] T023 [P] [US2] Add verification-failure-without-harness test: stub typecheck step fails on the custom branch but passes on upstream → exit 3; failing on upstream too → exit 4 and report says upstream is broken

### Implementation for User Story 2

- [X] T024 [US2] Create scripts/custom_build/steps.ts: ordered steps `install` (only when bun.lock or package.json differ from the start tip; `bun install --frozen-lockfile`), `typecheck` (`bun typecheck.ts`), `test` (`bun test src`), `build` (`bun build.ts --outfile dist/corgi-custom --patches <reportDir>/patches.json`), each overridable by `config.steps`; `runStep(step, cwd, logPath)` tees output to the terminal and `<reportDir>/steps/<step>.log` and returns pass/fail; `runIsolated(step, commit, merges[], workDir)` creates a detached worktree under `<dir>/work/`, merges the given refs, runs `bun install --frozen-lockfile` then the step (build writes a scratch outfile inside the worktree), always removes the worktree (research R13)
- [X] T025 [US2] Create scripts/custom_build/run.ts `rebuild()`: preflight (git ≥ 2.38, clean tracked tree, no run.json, branch name valid); fetch upstream and every patch into private refs (own branches: local or `origin/<branch>` when local is missing or strictly behind, note when diverged; PR state via github.ts: merged into upstream → skip `skipped-merged`, closed → `closed-warning`); record start tip, copy `dist/corgi-custom` to `<dir>/work/corgi-custom.start`, create report dir `reports/<UTC YYYYMMDD-HHMMSS>/`, save run.json; check out or create the custom branch and `reset --hard` to upstream; merge loop by cursor with `--no-ff` and message `custom-build: merge <display>`; on rerere-resolved merges commit and record `replayed`; on conflict save run.json and exit 2; persist run.json before every action (plan Design Notes). Before the reset, copy `scripts/custom_build/` (code and prompts) to `<dir>/work/tool/` and load prompts from there for the whole run; if `git check-ignore` says the config dir is not ignored and it lies inside the repo, append it to `.git/info/exclude` (research R23)
- [X] T026 [US2] In scripts/custom_build/run.ts implement the verify loop: run steps from run.json `step`; on failure run blame-first `runIsolated(step, upstream)`; upstream fails → record `upstream-broken`, exit 4; else without harness exit 3 (FR-021, FR-028); on full success write report (T040 provides the writer; until then a minimal summary), finalize backups (`git branch -f <branch>-prev <startTip>` when startTip exists, move start copy to `dist/corgi-custom.prev`), clear run.json, run `afterBuild` if configured (failure reported, run still succeeds), `--push` does `git push --force-with-lease origin <branch>` and is refused with exit 1 before the run starts when the resolved upstream remote is `origin` (FR-028: never push to upstream)
- [X] T027 [US2] In scripts/custom_build/run.ts implement `resume()` for the `continue` command (`continue` is a reserved word and cannot name a function) (requires run.json; in merge phase: refuse while unmerged files remain, commit a staged resolution as `user-resolved`, then resume the loop; in verify phase: commit any working-tree changes as `custom-build: fix <step> (user)` and rerun from the failed step) and `rollback()` per research R11 (abort merge if in progress, reset branch, restore binary, clear run.json)
- [X] T028 [US2] Create scripts/custom_build/main.ts: parse `[--dir <path>] <command> [args]`, dispatch `rebuild [--push]`, `continue`, `rollback`, `status` (run state plus rebase check), map results to exit codes 0 to 5 from contracts/cli.md, progress on stderr prefixed `custom-build: `, final summary on stdout; usage errors exit 1

**Checkpoint**: US2 works end to end without a harness.

---

## Phase 6: User Story 3 - Manage the patch list (Priority: P2)

**Goal**: `add`, `remove`, `list` (FR-011 to FR-013, FR-016).

**Independent Test**: add each source kind, list, remove with reason; config reflects it and `git status` is clean.

- [X] T029 [P] [US3] In tests/custom_build.test.ts add tests: `add` creates `.custom-build/config.json` (ignored by git), appends with today's date, rejects duplicates by display string, takes PR description from a stubbed `prInfo` title with `area:` stripped, requires `--description` when no title is available; `remove` moves to `retired` with date and reason, exits 1 without `--reason` on non-TTY; `list` prints display, description, added; `--retired`; `CORGI_CUSTOM_BUILD_DIR` and `--dir` redirect all reads and writes (option wins)
- [X] T030 [US3] Implement `add`, `remove`, `list [--retired]` in scripts/custom_build/main.ts using sources.ts, config.ts and github.ts (TTY prompt for a missing reason via `prompt()`)

---

## Phase 7: User Story 4 - Unattended conflict and build-failure resolution (Priority: P2)

**Goal**: configured harness resolves conflicts and fixes failing steps with tool-side acceptance (FR-022 to FR-027).

**Independent Test**: stub harness scripts resolve a known conflict and fix a typecheck failure; rebuild completes in one command; report lists both.

### Tests for User Story 4

- [X] T031 [P] [US4] In tests/custom_build.test.ts add harness conflict tests with stub harness shell scripts: (a) resolver takes `--theirs` on the files from the prompt → merge committed with `Resolved-By: harness` trailer, event `agent-resolved`; (b) resolver leaves a marker → attempt rejected, conflict state restored, retried up to `attempts`, then exit 5; (c) resolver edits an extra file → rejected; (d) resolver commits → rejected; (e) resolver sleeps past `timeoutSeconds` → counted as failed attempt; (f) prompt file path substituted for `{prompt_file}` and same text on stdin; `CORGI_CUSTOM_BUILD_KIND=conflict` set; missing executable → warning and behaves as no harness (exit 2)
- [X] T032 [P] [US4] Add harness build-fix tests: stub step fails until a file exists; stub harness creates it → commit `custom-build: fix typecheck (agent)` with `Fixed-By: harness`, failed and later steps rerun, exit 0; harness making no change → counts as failed attempt; upstream-broken → harness not called (exit 4); next rebuild does not contain the earlier fix commit

### Implementation for User Story 4

- [X] T033 [P] [US4] Create scripts/custom_build/prompts/conflict.md and scripts/custom_build/prompts/build.md with `{{name}}` placeholders and the rules from contracts/harness.md ("edit only the listed files", "do not commit", "do not touch `.custom-build/` or patch branches")
- [X] T034 [US4] Create scripts/custom_build/harness.ts: `fill(template, vars)` (lists as `- item` lines), `invoke(config.harness, kind, vars, reportDir)` writing `prompts/<nn>-<kind>.md`, substituting `{prompt_file}`, piping stdin, setting `CORGI_CUSTOM_BUILD_KIND` and `CORGI_CUSTOM_BUILD_PROMPT`, cwd repo root, killing after `timeoutSeconds` (default 900), treating a missing executable as unavailable; `acceptConflict(files, headBefore, statusBefore)` and `acceptFix(headBefore, statusBefore)` per research R15 and R16 (restore with `git checkout -m -- <files>` on rejection; soft-reset if the harness committed). `statusBefore` is `git status --porcelain` taken right before the call: only paths that are new or changed relative to it count as harness changes, and paths under the config directory never do, so pre-existing untracked files in the checkout neither reject a resolution nor end up in a fix commit
- [X] T035 [US4] Wire harness.ts into scripts/custom_build/run.ts: on a merge conflict with harness configured loop up to `attempts` (default 3) calling `invoke("conflict")` with patch display, description, PR link, files, upstream, merged-so-far; on acceptance commit with trailer and continue; exhausted → exit 5. In the verify loop after blame-first passes, loop `invoke("build")` with step, command, last 200 log lines, merged patches and attempt number, commit accepted fixes, rerun from the failed step; exhausted → exit 5. Record `agent-resolved` and `agent-fix <sha>` events

**Checkpoint**: one-command runs with a harness.

---

## Phase 8: User Story 5 - Handoff reports (Priority: P3)

**Goal**: rebase check, attribution, handoffs and reports (FR-029 to FR-035, FR-037).

**Independent Test**: one patch conflicting with upstream, one clashing only with another patch, one breaking the build against upstream; three outcomes as in US5 scenarios.

### Tests for User Story 5

- [X] T036 [P] [US5] In tests/custom_build.test.ts add report tests: rebase check lists the patch that conflicts with upstream (handoff kind `rebase` with files); patch-vs-patch conflict clean against upstream → clash note, no handoff, whether resolved by user or harness; agent build fix with one implicated patch failing alone → `build-drift` handoff; two implicated patches neither failing alone → `build-clash` handoffs naming each other; every prompt contains "never into the custom build branch"; `reports/latest` points at the newest run; only 10 run directories kept
- [X] T037 [P] [US5] Create scripts/custom_build/prompts/handoff.md following contracts/handoff.md

### Implementation for User Story 5

- [X] T038 [US5] In scripts/custom_build/steps.ts implement `attribute(step, upstream, merged, fixFiles, logPath): { kind: "build-drift" | "build-clash"; patches: BuildPatch[] }[]`, called from run.ts after an accepted agent fix: attribution (research R17, FR-030, FR-031): implicated patches = those whose `filesTouchedByMerge` intersect files changed by agent fix commits or repo-relative paths found in the step log, else all merged; for each, `runIsolated(step, upstream, [patchRef])`, a patch that does not merge cleanly alone counts as failing alone; classify `build-drift` vs `build-clash`
- [X] T039 [US5] Implement the end-of-run rebase check in scripts/custom_build/run.ts: `mergeTreeCheck(upstream, ref)` for every active patch, recording rebase-needed patches with files, and clash notes for run conflicts on patches that are clean in this check (FR-029, FR-033); also run it for `status`
- [X] T040 [US5] Create scripts/custom_build/report.ts: `writeReport(run, results)` producing `report.md` with the sections from data-model.md RunReport (upstream commits pulled with notable subjects, merged/skipped/already-contained, agent-resolved merges, agent fix commits, step results, rebase-needed, clash notes, handoffs written, rollback hint), `writeHandoff(kind, patch, details)` into `handoffs/<slug>.md` from the handoff template, `linkLatest(reportsDir, runDir)` (symlink, replaced), `prune(reportsDir, keep = 10)` deleting older timestamped directories; print the summary to stdout; call from run.ts on success and on every stop (exit 2 to 5)

---

## Phase 9: Polish & Cross-Cutting

- [X] T041 [P] Write docs/custom-builds.md: purpose, prerequisites (git 2.38+, optional gh), patch source syntax, commands and exit codes (contracts/cli.md), config reference with an example including a Claude Code harness using a narrow `--allowedTools` list (research R14) and `afterBuild: ["./dist/corgi-custom", "dev"]`, reports and handoffs, rollback, and the rules (never touches patch branches or upstream); link it from HACKING.md and docs/reference.md
- [X] T042 Run the full check: `bun typecheck.ts`, `bun test src`, `bun test tests/custom_build.test.ts`, `bun build.ts`; then run quickstart.md Deliverable 2 steps 1 and 2 against this repository with one real open upstream PR, and commit Deliverable 2 as `scripts: add custom-build tool for patched corgi builds`

## Dependencies & Execution Order

- Phase 1 → Phase 2 → Phase 3 (US1, Deliverable 1, committed) → Phase 4 → Phase 5 (US2) → Phases 6, 7 (US3, US4, parallel after US2) → Phase 8 (US5, needs US4 for agent-fix attribution) → Phase 9.
- Within US1: T004 to T006 parallel; T007 to T011 touch different files and may run in parallel after T002/T003; T013 last.
- Within Phase 4: T015 to T018 parallel; T019 after T015/T016.
- T040 provides the report writer used by T026 and T035; T026 uses a minimal summary until T040 lands.
- Phase 10 (US6): T043 to T046 first (config and CLI, GitHub, git helpers, retry helper; parallel), then T047 → T048 → T049 in rebase.ts, then T050 to T054 in run.ts and report.ts, T056 and T057 anytime. Write the T055 test for each acceptance scenario before the task that makes it pass.
- Phase 11 (evolution 2026-11-10): T059, T060 and T064 parallel; T061 and T062 after T060 (same file); T063 written first per scenario (RED), then the task that makes it pass.

## Parallel Example: User Story 1

```text
T004 src/custom_build.test.ts | T005 src/crash.test.ts | T006 src/bug_report.test.ts
then T008 src/main.ts | T009 src/bug_report.ts | T010 src/crash.ts | T011 src/updates.ts
```

## Implementation Strategy

1. MVP = Deliverable 1 (Phases 1 to 3): small, independently useful, cut as PR 1.
2. Deliverable 2 increments: US2 (deterministic rebuild) → US3 (list management) → US4 (harness) → US5 (handoffs), each testable on its own with stub steps.

## Notes

- T001 baseline: `bun typecheck.ts` passes. `bun test src` needs the native layer (`bun build.ts --native-only`, once per worktree); after that 2615 pass, 6 skip, 1 pre-existing failure in src/discovery.test.ts ("an install too old for Corgi isn't installed once a probe turns it down"), reproducible when run alone.
- Checkpoint 1 (after T014): 3 findings, 3 fixed (native crash events now get the custom tags via a new `app_crash_tag`; benchmark build config gets `customPatches`; the wrapper reads `--dir` only before the command). Folded into the Deliverable 1 commit.
- Checkpoint 2 (after T028): 5 findings, 5 fixed (fork PRs merged in their own repo are not skipped; exclude entry written regardless of the current .gitignore; `continue` after an interrupt commits tracked changes only; preflight refuses a custom branch checked out in another worktree; backups always replaced as a pair).
- Within US2 to US5, implementation came before the matching test tasks so each per-task checkpoint stayed green; all tests were written and pass.
- Added beyond tasks: the real `test` step runs `bun build.ts --native-only` first (isolated worktrees have no native layer, which would have misreported every test failure as upstream breakage); a rejected harness resolution is also forgotten by rerere so it never replays.
- T042 quickstart, in a throwaway clone under the session scratchpad: step 1 passed (`add 109` took the PR title, `add <branch> --description`, `list`, `git status` clean). Step 2: `rebuild` stopped with exit 2 because PR #109 really conflicts with upstream main in src/controller.ts; it wrote the report and a `rebase` handoff, and `rollback` through the copied tool restored the start. A full real rebuild was not run: the auto-mode classifier refused executing an outside PR's code (#114), and then a rebuild with only the local branch. The stub-step suite (35 tests) covers the successful path end to end.

## Phase 10: Convergence

- [X] T043 Add the `rebase` mode (`all` | `conflicting` | `off`, default `all`) to the config type, validation and defaults in scripts/custom_build/config.ts, plus `kind`, `rebased` and `rebaseCursor` in RunState (data-model.md); parse `rebuild --rebase=<mode>`, the new `rebase [--rebase=<mode>] [--push]` command and `rebase --skip-branch` in scripts/custom_build/main.ts, with usage text per FR-041, FR-043, contracts/cli.md (missing)
- [X] T044 Extend the PR query in scripts/custom_build/github.ts with `headRepositoryOwner,headRepository,headRefName` and add an own-PR check that compares `github.com/<owner>/<name>` with `git.remoteTarget(repo, "origin", "push")`; without `gh` no PR is own, and when `origin` pushes to the upstream repository no patch is own at all per FR-040, research R25 (missing)
- [X] T045 Add git helpers in scripts/custom_build/git.ts: start, continue and abort a rebase in a worktree with rerere and `GIT_EDITOR=true`, report the commit being replayed, compare-and-swap `update-ref`, find the worktree a branch is checked out in (any worktree, including the current one), and push with `--force-with-lease=refs/heads/<b>:<lease>` per research R28, R30, R32 (missing)
- [X] T046 Factor the flaky retry out of `verifyLoop` in scripts/custom_build/run.ts into a `steps.runWithRetry` helper in scripts/custom_build/steps.ts that keeps the first failure's log, and use it from both places per FR-025, FR-046 (partial)
- [X] T047 Create scripts/custom_build/rebase.ts with the selection of own patches: own-branch patches and own PRs; old tip and lease from the local branch and `origin/<branch>` (R26); behind or conflicting against upstream (R27); skipped with a reason when checked out in a worktree, diverged, or a PR without `gh` per FR-040, FR-041, FR-044, FR-048 (missing)
- [X] T048 In scripts/custom_build/rebase.ts rebase one branch in a throwaway worktree under the config's work directory: replay with rerere, send unresolved stops to the harness with the conflict template (branch, files, commit being replayed; `cwd` and `acceptConflict` in the worktree), respect attempts and the time limit, discard rejected attempts, and remove the worktree, including leftovers of an interrupted run, per FR-045, research R28 (missing)
- [X] T049 In scripts/custom_build/rebase.ts run install, typecheck and tests on the rebased tip in its worktree with `runWithRetry`; on success write `refs/custom-build/rebase-backup/<branch>` and move the branch by compare-and-swap (creating the local branch for an own PR when missing); leave the branch untouched when no commits remain or the swap fails; record a RebaseResult per FR-046, FR-047, FR-048, research R29, R30 (missing)
- [X] T050 Add the rebase phase to `rebuild` in scripts/custom_build/run.ts after fetching and before the reset: use the rebased tips as the plan's commits; turn any unresolved conflict or failed check into "keep the old tip" plus a `rebase` or `rebase-checks` handoff, never a stop; keep the results in `run.rebased` per FR-042, US6/AC1, US6/AC6 (missing)
- [X] T051 Implement the `rebase` command run in scripts/custom_build/run.ts: `run.json` with `kind: "rebase"` and a cursor; stop with exit 2 (no harness) or 5 (attempts exhausted), printing the worktree, the replayed commit and the files; `continue` checks the worktree for unmerged paths and continues the rebase, then the checks; `rebase --skip-branch` abandons the branch and goes on; `rollback` aborts the rebase in progress; `rebuild` refuses while a rebase run is stopped per FR-043, FR-045, US6/AC7, research R31 (missing)
- [X] T052 Offer the push at the end of a successful `rebuild` and of `rebase`: list the moved branches; with `--push`, or a yes from `ask` on a terminal, push each with its lease and delete its backup on success; otherwise print one push command per branch; record `pushed` or `refused` per FR-049, US6/AC8, US6/AC9, research R32 (missing)
- [X] T053 Report rebasing in scripts/custom_build/report.ts and the stdout summary: a "Rebased branches" section with outcome, how, reason (including "no commits left, remove the patch" and "no fork to push to"), push result and the `git branch -f <branch> refs/custom-build/rebase-backup/<branch>` undo command; `rebase` handoffs for failed own rebases and `rebase-checks` handoffs (step, log excerpt, prompt) using scripts/custom_build/prompts/handoff.md per FR-050, FR-046, contracts/handoff.md (missing)
- [X] T054 Make `status` list the own patches that are behind or conflicting with upstream, and a stopped rebase run with its worktree, without changing anything, per contracts/cli.md (missing)
- [X] T055 Add tests in tests/custom_build.test.ts for US6/AC1 to AC10 and the US6 edge cases: modes `all`, `conflicting` and `off`; other people's PRs untouched; harness resolution; failing checks keep the old tip; `rebuild` never stops on a rebase; a `rebase` stop with `continue` and `--skip-branch`; push asked, forced and printed; lease refused when the remote moved; backup ref; checked-out, diverged, nothing-left and no-`gh` skips per SC-007, SC-008 (missing)
- [X] T056 Update docs/custom-builds.md: a "Rebasing your own branches" section, the `rebase` command and `--rebase` / `--skip-branch` in the commands table, the `rebase` config key, `rebase-checks` handoffs, and "What it never touches" rewritten to rebase and push only own branches as described, per FR-028, FR-040 to FR-050 (contradicts)
- [X] T057 Refuse the custom branch and its `-prev` backup already in `add` in scripts/custom_build/run.ts or main.ts, not only at `rebuild`, per contracts/cli.md `add` (partial)
- [X] T058 Add tests in tests/custom_build.test.ts for the rollback behavior of FR-036: dependencies are reinstalled after a rollback restores an earlier tree; a failing reinstall leaves phase `rollback` and `continue` finishes it; a rollback refused for uncommitted changes leaves the stopped run so `continue` still resumes it (partial: implemented in 4f4ef3cf, 51930fd1, 0dbcdfac without tests)

## Phase 11: Evolution 2026-11-10 (status skip reasons, rebasing in the branch's own worktree)

**Goal**: `status` names own branches it leaves alone and why (spec session 2026-11-10, research R33); `rebase`/`rebuild --in-worktree` rebases a checked-out own branch inside its worktree with the backup kept from the rebase's start (FR-044's exception, FR-047, research R34).

**Independent Test**: check an own branch out in a worktree, let upstream move; `status` names the worktree, `rebase` skips, `rebase --in-worktree` moves branch and checkout together and survives a dirty worktree, a rejected resolution, failed checks and an interruption (quickstart "Rebasing own branches" step 7).

- [X] T059 List the own branches `rebase` leaves alone in `status` in scripts/custom_build/run.ts, each with its reason from the RebaseResult (the worktree it is checked out in, with the `--in-worktree` hint, a pending push, divergence), instead of omitting them per research R33, contracts/cli.md
- [X] T060 Parse `--in-worktree` for `rebuild` and `rebase` in scripts/custom_build/main.ts and thread it through scripts/custom_build/run.ts into `select()`/`assess()` in scripts/custom_build/rebase.ts: a checked-out own branch with a local tip and no pending push becomes a target carrying its worktree path instead of a skip per FR-044's exception
- [X] T061 Rebase such a target inside its own worktree in scripts/custom_build/rebase.ts (`inOwnWorktree`): skip with a reason on uncommitted tracked changes, run the rebase, the harness conflict loop and the checks there, never remove the worktree, never `git clean` it (untracked files survive a rejected resolution), and reset the branch and checkout back to the starting tip on every failure path (`restore`) per FR-044's exception, FR-045, FR-046, research R34
- [X] T062 Write the backup ref `refs/custom-build/rebase-backup/<branch>` before the rebase starts in `inOwnWorktree` in scripts/custom_build/rebase.ts (kept when a push is still pending), because the rebase itself moves the branch; make `finish()` not move the branch again for such a target per FR-047
- [X] T063 Add tests in tests/custom_build.test.ts: `status` names the worktree of a checked-out own branch; `--in-worktree` rebases it there (checkout in step, clean, harness resolves there) or skips a dirty one with a reason; failed checks reset branch and checkout with the backup naming the restored tip; a rejected resolution keeps untracked files; the backup exists with the old tip while the checks run (spec session 2026-11-10, US6/AC11)
- [X] T064 Update docs/custom-builds.md (status row, `--in-worktree` in the command table and the per-branch list, the in-worktree paragraph with the backup timing) and the spec artifacts (research.md R33/R34, contracts/cli.md, data-model.md, quickstart.md) per FR-044's exception, FR-047
