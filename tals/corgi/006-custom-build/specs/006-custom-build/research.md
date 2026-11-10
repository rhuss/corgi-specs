# Research: Custom corgi builds

All Technical Context unknowns are resolved below. Each entry: Decision, Rationale, Alternatives considered.

## R1. How the identity reaches the binary (revised 2026-11-05 and 2026-11-07, #116)

- **Decision**: `build.ts` calls `fromGit(root)` from `src/custom_build.ts` for every development build (no `--update-url`); releases get no identity. The result becomes `buildRepository` in the generated `src/build_config.ts`; the checked-in module exports an empty value, so `bun src/main.ts` is vanilla. `fromGit` takes the repository of the branch's remote (origin when that remote is upstream or missing) via `repository(url)`, which keeps only `host/owner/repo`.
- **Rationale**: Any fork gets an honest identity without the tool, and the tool needs no build option. The merge list of the first #116 version was dropped on 2026-11-07: the fork link is enough to find the branch, whose `Merge <source>: <description>` log says what it carries, and it keeps the version output and bug report short.
- **Alternatives**: `--patches` metadata from the tool (first design; only the tool's builds benefited, and the metadata could disagree with git); the merge list from git (first #116 version; long output, duplicated what the fork shows).

## R2. Version output (revised 2026-11-07)

- **Decision**: `corgi --version` is unchanged. `--version --verbose` adds `  https://<host/owner/repo>` when `custom()` holds (the repository is known and is not `github.com/tals/corgi`).
- **Rationale**: First-line parsers keep working, and no marker can fragment anything that groups by version.

## R3. Bug report (revised 2026-11-07)

- **Decision**: `bug_report.rows()` adds the same link under the Corgi row, depth 1. Nothing is added for vanilla builds.

## R4. Crash reports (revised 2026-11-05)

- **Decision**: Unchanged. Crash events carry no identity.
- **Rationale**: Crashes are sent without review, so no fork or patch names; the release string already distinguishes development builds.

## R5. Self-update guard (revised 2026-11-05)

- **Decision**: Unchanged code. A custom binary is a development build without update URL or public key, so `corgi update` refuses it like any development build, and the tool never writes into a managed `releases/` directory.

## R6. Tool runtime and layout

- **Decision**: TypeScript on Bun, like `scripts/release.ts`. Entry `scripts/custom-build` (POSIX sh wrapper like `scripts/release`) execs `bun run scripts/custom_build/main.ts`. Modules under `scripts/custom_build/`. No new dependencies. Git is driven via `Bun.spawnSync`/`Bun.spawn` of the `git` binary.
- **Rationale**: Matches repo conventions; contributors already have Bun.
- **Alternatives**: POSIX sh like our `rebuild.sh` (fragile JSON handling, harder to test); a Claude skill (rejected: must work without any harness, FR-039).

## R7. Config format and location

- **Decision**: `.custom-build/config.json`, JSON with 2-space indentation, rewritten by the tool. Directory resolution: `--dir` option > `CORGI_CUSTOM_BUILD_DIR` > `<repo root>/.custom-build`. `/.custom-build/` is added to `.gitignore`. Schema in [data-model.md](data-model.md).
- **Rationale**: Bun parses and writes JSON natively; TOML can be parsed by Bun but not written. Hand-editable enough with descriptions as fields.
- **Alternatives**: Markdown table (our current `INTEGRATION.md`; fragile to rewrite), YAML (needs a dependency).

## R8. Patch source syntax and fetching

- **Decision**: The `add` argument accepts:
  - `#123` or `123` or `https://github.com/<upstream>/pull/123` → upstream PR
  - `owner/repo#12` or `https://github.com/owner/repo/pull/12` → PR on another repo
  - `owner/repo:branch` or `https://github.com/owner/repo/tree/branch` → branch on another repo; any git URL as `<url>#branch=<name>`
  - anything else → branch of the user's own repository
- Fetching never adds remotes. PRs fetch `refs/pull/<N>/head` from the repo's HTTPS URL into a private namespace `refs/custom-build/<owner>/<repo>/pull/<N>`; fork branches fetch into `refs/custom-build/<owner>/<repo>/heads/<branch>`. Own branches use the local branch, or `origin/<branch>` when the local one is missing or strictly behind (same rule as today's `rebuild.sh`), with a note when diverged.
- **Rationale**: GitHub publishes every PR head (fork PRs included) on the base repo (FR-015). A private ref namespace keeps `git branch` clean and survives between runs.

## R9. Upstream identity and PR metadata

- **Decision**: Config `upstream` holds `{ repo: "tals/corgi", remote?: string, branch: "main" }`. Default remote is `upstream` if it exists, else `origin`; the tool fetches `<remote> <branch>`. PR state, title and merge commit come from `gh pr view <N> --repo <repo> --json state,title,mergeCommit` when `gh` is on PATH and authenticated. Without `gh`: merged detection by ancestry (patch tip contained in upstream), and `add` requires `--description` for PRs.

## R10. Run state, exit codes and `continue`

- **Decision**: An in-progress run is recorded in `.custom-build/run.json` (start tip, start binary copy path, patch cursor, current phase, attempt counters, report directory, merged patch metadata). Its presence makes `rebuild` refuse (FR-017) and enables `continue` and `rollback`. Exit codes: `0` success; `1` precondition or usage error; `2` stopped at an unresolved merge conflict; `3` stopped at a failed verification step; `4` upstream itself fails a step; `5` harness attempts exhausted. Codes are in [contracts/cli.md](contracts/cli.md).
- **Rationale**: Distinct codes per stop reason (FR-021) let wrappers and agents react without parsing text.

## R11. Backups and rollback

- **Decision**: At run start the tool records the current branch tip and copies `dist/corgi-custom` (if present) to `.custom-build/work/corgi-custom.start`. On full success: `git branch -f <branch>-prev <start tip>` and move the start copy to `dist/corgi-custom.prev`. `rollback` with `run.json` present aborts any merge, resets the branch to the start tip and restores the start copy; without it, resets to `<branch>-prev` and restores `corgi-custom.prev`. A first-ever run has nothing to roll back to and says so.
- **Rationale**: Satisfies FR-017 (backups only overwritten by success) and FR-036 (rollback returns to last success in both situations). This refined FR-036 during planning: the literal "restore from -prev" during a failed run would have skipped the last good build.

## R12. Verification steps

- **Decision**: Fixed sequence: `bun install --frozen-lockfile` (only when nothing is installed yet or `bun.lock` or `package.json` differ from the start tip), `bun typecheck.ts`, `bun build.ts --native-only` plus `bun test src`, `bun build.ts --outfile dist/corgi-custom`. A failed step runs once more before anything else; passing then is recorded as flaky (revised 2026-11-06). Step outputs are captured to `<report dir>/steps/<step>.log` and tee'd to the terminal.
- **Rationale**: These are the repo's documented checks (AGENTS.md, HACKING.md); integration suites need `--test` builds and are too slow for every rebuild (spec assumption).

## R13. Isolated checkouts for blame-first and attribution

- **Decision**: `git worktree add --detach .custom-build/work/<name> <commit>` per check, `bun install --frozen-lockfile` inside it, run the failing step, then `git worktree remove --force`. Blame-first (FR-025) uses plain upstream. Attribution (FR-031) uses one worktree per implicated patch: upstream with that patch merged via `git merge --no-edit`; a patch that does not merge cleanly alone is attributed to upstream drift without running the step. The build step in isolation writes to a scratch `--outfile` inside the worktree.
- **Rationale**: Never disturbs the user's checkout; Bun's install cache keeps installs fast. Native Zig build cost is accepted; it only happens on failure paths.

## R14. Harness invocation

- **Decision**: Config `harness: { command: string[], timeoutSeconds?: number (default 900), attempts?: number (default 3) }`. The tool fills a prompt template, writes it to `<report dir>/prompts/<n>-<kind>.md`, replaces the literal element `{prompt_file}` (and `{prompt_file}` inside elements) with its absolute path, pipes the prompt on stdin, runs with `cwd` = repo root and env `CORGI_CUSTOM_BUILD_KIND=conflict|build`. Exit code is advisory; acceptance is decided by the tool's own checks. A missing executable is treated as no harness with a warning. Example command for Claude Code in docs: `["claude", "-p", "--allowedTools", "Read,Edit,Write,Grep,Glob,Bash(git diff:*),Bash(git status:*),Bash(bun typecheck.ts),Bash(bun test:*)"]` (prompt from stdin).
- **Rationale**: Array form avoids shell quoting; stdin plus file covers CLIs that read either (spec clarification).

## R15. Accepting a conflict resolution

- **Decision**: After the harness returns: (1) no path remains unmerged (`git diff --name-only --diff-filter=U` empty after staging the conflicted set); (2) none of the conflicted files contains a line starting with `<<<<<<< `, `>>>>>>> ` or equal to `=======`; (3) compared with `git status --porcelain` taken before the call, the only new or changed paths are from the conflicted set (paths under the config directory are ignored); (4) HEAD is unchanged. On success the tool stages the conflicted files and commits the merge with its standard message plus a `Resolved-By: harness` trailer. On failure it resets the conflicted files to the conflict state (`git checkout -m -- <files>`) and retries until attempts are exhausted.

## R16. Accepting a build fix

- **Decision**: The harness edits the working tree only; HEAD must be unchanged (if the harness committed, the tool soft-resets to the pre-call HEAD and keeps the changes). Any change relative to the pre-call `git status --porcelain` snapshot (excluding the config directory) is committed as `custom-build: fix <step> (agent)` with trailer `Fixed-By: harness`; then every step reruns from the first (revised 2026-11-06). No change, or the step still failing after `attempts` fix rounds, stops with exit 5. Fix commits are never carried forward: every run resets to upstream (FR-034).

## R17. Implicated patches

- **Decision** (revised 2026-11-06): Files touched by a patch = `git diff --name-only <merge>^1 <merge>` for its merge commit. Implicated = patches whose touched files are among the files changed by the agent fix, or appear in the failure log after dropping the tool's `$ <command>` lines and lines holding nothing but a path (bun prints one such header per test file). If none is implicated, no isolated run happens and the report says the fix has no handoff.
- **Rationale**: A real 8-patch build implicated every patch through bun's file headers and spent an hour of full test runs on a flaky test. The old fallback to all patches had the same cost with no signal.

## R18. Rebase check and clash notes

- **Decision**: For every active patch: `git merge-tree --write-tree --name-only --no-messages <upstream> <patch ref>`; exit 1 means "needs rebase" with the listed files. Merge conflicts during the run on a patch that is clean in this check are recorded as clash notes (FR-033). Requires git 2.38+; the tool checks `git --version` at start and fails with exit 1 if older.

## R19. Reports and handoffs

- **Decision**: `<dir>/reports/<UTC timestamp YYYYMMDD-HHMMSS>/report.md`, `handoffs/<patch-slug>.md`, `prompts/`, `steps/`. `reports/latest` is a symlink to the newest directory. After writing, keep the 10 newest timestamped directories and delete older ones. Handoff template in [contracts/handoff.md](contracts/handoff.md). The report summary is also printed to stdout.

## R20. Delivering two stacked PRs from one branch

- **Decision**: Implement Deliverable 1 first and commit it as its own commits on `006-custom-build`; Deliverable 2 follows in later commits. At submit time, PR 1 is a branch at the last Deliverable 1 commit; PR 2 is the full branch based on PR 1.
- **Rationale**: Keeps one spec and one worktree while preserving the agreed PR split.

## R21. Rerere

- **Decision**: All tool merges run with `git -c rerere.enabled=true -c rerere.autoUpdate=true`, so recorded resolutions replay and are staged; if no unmerged paths remain after the merge attempt, the tool commits (FR-020) and notes "replayed".

## R22. After-build command

- **Decision**: Config `afterBuild?: string[]`, executed with `cwd` = repo root after a fully successful run; its failure is reported but does not fail the run (the binary is already built). Example in docs: `["./dist/corgi-custom", "dev"]`.

## R23. Self-hosting before the tool is upstream

- **Decision**: At run start, before resetting the tree, the tool copies `scripts/custom_build/` (code and prompts) to `<config dir>/work/tool/` and uses that copy for prompts during the run. While `run.json` exists, the `scripts/custom-build` wrapper runs the copy, so `continue`, `rollback` and `status` work even if the reset tree lacks the tool (upstream without PR 2, or a stop before the tool's own patch merged). Users who invoke the tool from outside the tree can call `bun <config dir>/work/tool/main.ts` directly; the stop message prints that command. If the config directory is inside the repository and `git check-ignore` reports it as not ignored, the tool appends it to `.git/info/exclude`.
- **Rationale**: The tool must keep working while its own PR is only one of the integrated patches, and even if PR 2 is never accepted upstream.
- **Alternatives**: requiring a separate checkout for the tool (more setup for every user); refusing to run unless the tool is in upstream (defeats dogfooding).

## R24. Bug report scope (revised 2026-11-07)

- **Decision**: The default summary and the full report show the same fork link (R3); what the build carries is read from the fork's branch, not from the report.

## R25. Which patches are the user's own (2026-11-07)

- **Decision**: Own-branch patches (`kind: "branch"`) are own. A `pr` patch is own when `gh pr view <N> --repo <repo> --json headRepositoryOwner,headRepository,headRefName` names `github.com/<owner>/<name>` equal to `git.remoteTarget(repo, "origin", "push")`; the PR query is extended with these fields so one `gh` call per PR covers state, title and ownership. Without `gh`, PRs never count as own. Fork branches are never own, even on the user's own fork (left for a later change).
When `git.remoteTarget(repo, "origin", "push")` equals `git.remoteTarget(repo, <upstream remote>, "fetch")`, no patch is own (the same comparison guards `--push` for the custom branch), and the report says rebasing is off because there is no fork.
- **Rationale**: `origin` is where the user can push; the head repository is the only reliable owner signal for a PR. Comparing normalized host/path with `remoteTarget` already handles SSH and HTTPS spellings.
- **Alternatives**: comparing the PR author with the `gh` user (wrong for PRs opened from organization forks); a per-patch `own: true` flag in the config (manual, easy to forget).

## R26. Which branch a rebase moves (2026-11-07)

- **Decision**: For an own branch, the local branch `refs/heads/<name>`. For an own PR, the local branch named after `headRefName`, created when missing. Before rebasing, the tool fetches `origin <name>` into `refs/remotes/origin/<name>` (it already does for own branches) and compares: local missing → start from the remote tip; local behind remote → start from the remote tip (the local branch is fast-forwarded as part of the move); remote missing or local ahead → start from local; diverged → skip (FR-048). The tip the rebase starts from is the old tip; the remote tip is the lease for the push (empty when the branch was never pushed, meaning "must not exist").
- **Rationale**: Matches how own-branch patches already pick local versus `origin/<branch>` (R8), and never drops remote commits.

## R27. Which own branches need a rebase (2026-11-07)

- **Decision**: Behind = upstream `main` is not an ancestor of the old tip (`git merge-base --is-ancestor`). Conflicting = `git.mergeTreeCheck(upstream, oldTip)` is not clean. Mode `all` rebases behind branches; `conflicting` only conflicting ones; `off` none. Branches checked out in any worktree (`git worktree list --porcelain`, including the current one) are skipped with the worktree path named.

## R28. Rebasing in a throwaway worktree (2026-11-07)

- **Decision**: `git worktree add --detach <work>/rebase-<slug> <oldTip>`, then `git -c rerere.enabled=true -c rerere.autoUpdate=true rebase <upstream>` with `GIT_EDITOR=true`. While the rebase stops with unmerged paths: rerere-staged paths need nothing; for the rest the harness is called with `cwd` = the worktree and the `rebase-conflict` template (branch, `files`, the commit being replayed); a rejected attempt restarts the rebase from the old tip, since accepted resolutions replay through rerere, and accepted by `harness.acceptConflict` against the worktree; then `git add` the files and `git rebase --continue`. Attempts and time limit apply per stop. An empty result (no commits left on top of upstream) is not applied (FR-048). The worktree is removed at the end, and leftovers from an interrupted run are removed before the next one.
- **Rationale**: Never touches a checkout the user works in (FR-044) and reuses the merge-conflict machinery unchanged except for its working directory, which `harness.invoke` and `acceptConflict` already take as a parameter.
- **Alternatives**: `git rebase` in the user's checkout (would clobber their work tree); replaying with `merge-tree` (no conflict stops for the harness); `--rebase-merges` (keeps merge commits of main into the branch, which a rebase onto main should drop).

## R29. Checks on a rebased branch (2026-11-07)

- **Decision**: In the same worktree, after the rebase completes: install (always, since the worktree is fresh; FR-046), typecheck and test via `steps.commands`, each with the flaky retry from `verifyLoop` factored out into `steps.runWithRetry`. The build step is not run: it adds minutes and checks nothing the tests don't for a branch on its own. Any failure leaves the branch at its old tip and writes a `rebase-checks` handoff with the step, a log excerpt and a prompt. The harness never fixes these (spec clarification).
- **Rationale**: The user doesn't review rebased branches before pushing, so the checks are the safety net; only branches that moved pay for them.

## R30. Moving a branch and keeping a backup (2026-11-07)

- **Decision**: `git update-ref refs/custom-build/rebase-backup/<branch> <oldTip>`, then `git update-ref refs/heads/<branch> <newTip> <localTipOrZero>` (compare-and-swap; a branch the user moved meanwhile fails the swap and is skipped with a note). A successful push deletes the backup ref; the report prints `git branch -f <branch> refs/custom-build/rebase-backup/<branch>` for undoing. Inside `rebuild`, the plan's `commit` for that patch becomes the new tip before the reset, so the merges use it.

## R31. The `rebase` run and its stops (2026-11-07)

- **Decision**: The `rebase` command records `run.json` with `kind: "rebase"`, `phase: "rebase"`, a cursor over the selected own patches and the results so far. A conflict the harness can't resolve (or no harness) stops with exit 2 (no harness) or 5 (attempts exhausted), printing the worktree path, the commit being replayed and the files. `continue` checks that no unmerged path remains in that worktree and runs `git rebase --continue` there; the branch then goes through the checks (R29). `rebase --skip-branch` aborts that branch's rebase, removes its worktree, records it as skipped and goes on. `rollback` aborts the rebase in progress and removes the run; branches already moved keep their new tips (their checks passed; backups remain). `rebuild` refuses while a rebase run is stopped. Inside `rebuild`, the same loop runs with stops turned into "failed, keep old tip, `rebase` handoff" (FR-042), so rebuild's `run.json` only needs the results, not a rebase cursor: the phase runs before the run starts recording and an interruption leaves at worst some branches already moved, each with its backup.

## R32. Pushing rebased branches (2026-11-07)

- **Decision**: At the end of `rebuild` (after a successful run) and of `rebase`, the moved branches are listed. With `--push`, or when `ask("Force-push N rebased branches to origin? [y/N]")` answers yes on a terminal, each is pushed with `git push --force-with-lease=refs/heads/<branch>:<lease> origin <newTip>:refs/heads/<branch>`; otherwise the summary prints that command per branch. A refused push is reported and keeps the backup ref. In `rebuild`, `--push` keeps pushing the custom branch as before (with its origin-is-upstream guard), plus the rebased branches; a stopped rebuild offers no push until it completes.
- **Rationale**: One confirmation for all branches (SC-008); the lease is the fetched remote tip, so anything pushed meanwhile is never overwritten.

## R33. Reporting why an own branch is left alone (2026-11-10)

- **Decision**: `status` lists own branches that need a rebase as before, and additionally lists those it would leave alone with the reason: the worktree a branch is checked out in (with the `--in-worktree` hint), a push still pending, or local and pushed tips diverged. Selection itself is unchanged.
- **Rationale**: `status` is the tool's answer to "what would happen if I ran `rebase` now"; omitting the branches it skips made the answer silently incomplete, which is what prompted the evolution.

## R34. Rebasing inside the branch's own worktree (2026-11-10)

- **Decision**: `rebase`/`rebuild --in-worktree` selects a checked-out own branch (local tip, no pending push) as a target carrying its worktree path instead of skipping it. The rebase runs in that worktree: skipped with a reason when its tracked files aren't committed, the backup ref written before `git rebase` starts (the rebase itself moves the branch, so the backup must predate it; a pending push keeps the earlier backup), conflicts driven by the same harness loop, and the checks run there. Every failure path (failed rebase, rejected resolution after abort, failed checks, no commits left) resets the branch with `git rebase --abort` when in progress and `git reset --hard` to the starting tip; the worktree is never removed, and `git clean` never runs there, so untracked files survive.
- **Alternatives**: moving the branch from outside and letting the checkout go stale (the very problem `--in-worktree` exists to avoid); removing and re-adding the worktree (loses untracked files); recording the backup only after the checks pass like a throwaway-worktree rebase (leaves a window where the branch has moved with no undo ref).
