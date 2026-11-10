# Deep Review Findings

**Date:** 2026-11-08
**Branch:** 006-custom-build (PR #117, head `custom-build-tool`)
**Scope:** User Story 6 (rebasing own patch branches), commits 49334c80 and 96337081 plus three fix rounds
**Rounds:** 3
**Gate Outcome:** PASS (no Critical or Important finding left open; 2 Minor accepted)
**Invocation:** quality-gate (after_implement, via review-code)

## Summary

| Severity | Found | Fixed | Remaining |
|----------|-------|-------|-----------|
| Critical | 1 | 1 | 0 |
| Important | 26 | 26 | 0 |
| Minor | 24 | 19 | 5 |
| Notable | 6 | - | 6 |
| **Total** | **57** | **46** | **11** |

Counts are after deduplication across agents and rounds. Remaining Minor findings fail safe (they make the tool skip rebasing, never push or move something it shouldn't) or are cosmetic.

**Agents completed:** 6/6 internal (round 1), plus one combined re-review agent per later round; external: CodeRabbit (3 runs), Codex (3 runs). Copilot disabled in config.

## Findings (Critical and Important)

### FINDING-1 (Critical, fixed round 1)
- **File:** scripts/custom_build/rebase.ts `continueOrSkip`
- **Source:** correctness agent (also Codex P1); both reproduced
- **What was wrong:** when `git rebase --continue` failed without moving on (unstaged edits after `git add`, a rejecting hook), the tool ran `git rebase --skip`, dropping the user's resolved commit; the branch then passed its checks and was offered for force-push without it.
- **Resolution:** a commit is skipped only when index and worktree match HEAD (it became empty); otherwise the run stops with the git error as `problem:`. Test: the stop/continue test edits the file after `git add`, expects a second stop, then the edited content in the rebased branch.

### FINDING-2 (Important, fixed round 1): checkout re-check before the move
- **Source:** correctness, security, goal alignment agents, Codex. Branches checked out after selection were moved under the checkout (FR-044). Now `checkedOutIn` runs again just before the move. Test: a check step that checks the branch out in another worktree.

### FINDING-3 (Important, fixed rounds 1-3): an unpushed rebase reported as "diverged" forever
- **Source:** CodeRabbit, production agent, correctness agent, Codex (also a later rebuild merging the stale PR head). Each successful move now records three refs in one transaction: `rebase-backup/<b>` (tip before the rebase, for undo), `rebased/<b>` (new tip) and `rebase-lease/<b>` (fork tip then). A branch still containing its rebased tip, with the fork unchanged, is "rebased earlier, push pending": offered again, shown by `status`, rebased again from its tip, and merged by `rebuild`. Tests: declined push then later runs, new local commit, branch ahead of the fork, checked-out pending branch, rebuild of an own PR.

### FINDING-4 (Important, fixed rounds 1-2): upstream guard relied on exact URL spelling
- **Source:** security agent, re-review. `repositoryOf` normalizes case, `.git`, leading slashes, ports only for scheme URLs (numeric owners in scp form kept), `file://`; a second guard compares `config.upstream.repo`. Test: URL spellings.

### FINDING-5 (Important, fixed rounds 1 and 3): tips, leases and pushes from different URLs
- **Source:** Codex. All three use origin's push URL (first line when several), fetched into `refs/custom-build/fork/<b>`; `--push` is judged by destinations, not remote names. Tests: origin fetching upstream and pushing to the fork, with two push URLs and no upstream remote.

### FINDING-6 (Important, fixed round 1): a fetch failure or exception aborted `rebuild`
- **Source:** production and correctness agents. Per-branch try/catch; failures become `skipped`/`failed` results (FR-042).

### FINDING-7 (Important, fixed round 1): a failed initial rebase or a user-aborted one reported as rebased
- **Source:** Codex, production and correctness agents. The rebase exit code is checked, and `finish` requires upstream in the new tip. Tests: a `pre-rebase` hook refusing; the user aborting in the stopped worktree.

### FINDING-8 (Important, fixed round 1): refused pushes undiagnosable
- **Source:** production agent. Git's full output goes to `rebase/<b>-push.log`; the reason line and a `retry:` command are printed.

### FINDING-9 (Important, fixed round 1): `rollback` of a rebase run said nothing about moved branches
- **Source:** architecture agent. It lists them with their push commands.

### FINDING-10 (Important, fixed round 1): backup written after the move
- **Source:** goal alignment, correctness, production agents. Backup and move are one `update-ref --stdin` transaction with compare-and-swap; undo for a created branch is `git branch -D`.

### FINDING-11 (Important, fixed round 1): no flaky warning for rebase checks (FR-046/FR-025)
- Flaky steps are recorded and printed with their kept log.

### FINDING-12 (Important, fixed round 1): a deleted worktree blocked continue, skip and rollback
- **Source:** correctness agent. Tests for continue/skip and rollback with the worktree gone.

### FINDING-13 (Important, fixed rounds 1-3): rejected harness attempt replayed the wrong commits
- **Source:** correctness agent, re-review agents, Codex (reproduced). The replay is now `git rebase -i --onto <head> <upstream> <from>` with the commits before the conflict turned into `drop` lines, so git's original selection (cherry-picks excluded, merges dropped) and earlier resolutions stay. Test: X, A (rejected once), a merge of upstream U1, B; asserts two harness calls, `u1.txt` keeps upstream's newer content, no upstream commit in the branch.

### FINDING-14 (Important, fixed round 3): the user's git config changed rebases
- **Source:** round-2 re-review agent (reproduced). `rebase.autoSquash`, `abbreviateCommands`, `missingCommitsCheck`, `updateRefs` and `rebaseMerges` are pinned to defaults for every rebase; `updateRefs=true` had moved the user's branch before the checks and without backup. The replay test runs with all of them set.

### FINDING-15 (Important, fixed rounds 1-3): test gaps
- **Source:** test-quality agent and re-reviews. AC3 ownership (same-named branch on the fork), `rebuild --rebase`, harness rejection and retry, rerere replay, branch moved or checked out mid-rebase, AC6 handoff and merged commit, AC7 messages and exit 5, no-origin and leftover cleanup, `-prev` rollback reinstall, undo text, invalid `--rebase` message, and the claims of each fix round.

### FINDING-16 (Important, fixed round 1): `status` ignored the configured mode; `rebase` skipped the push and reserved-branch preflight
- **Source:** CodeRabbit, architecture agent. Shared `preflight`; `status` uses `config.rebase` and offers `--skip-branch` only for a stopped branch.

## Remaining Minor findings

- `sameRepository` compares only `owner/repo` for the upstream guard: a same-named repository on another host counts as upstream, so rebasing is skipped there (fails safe). (Codex round 2)
- Local paths are lowercased when comparing remotes, so two paths differing only in case on a case-sensitive file system count as one, and rebasing is skipped (fails safe). (Codex round 2)
- `removeWorktree`'s fallback compares `gitdir` literally; symlinked or relative gitdirs leave a stale entry that `worktree add --force` tolerates. (round-2 re-review)
- Harness defaults (`attempts ?? 3`, `timeoutSeconds ?? 900`) are repeated across modules. (architecture)
- `RunState` carries rebuild fields as dummies for rebase runs; a discriminated union would be cleaner. (architecture)

## Notable Observations

- **NOTABLE-1** (production): with mode `all` every own branch is rebased, checked (install, typecheck, tests) and offered for force-push on every rebuild once upstream moves; `conflicting` as a default, or a shared `node_modules`, would cut cost and PR noise.
- **NOTABLE-2** (production): rebuild's rebase phase runs before `run.json` exists, so `status` shows nothing and a second run is not refused while it runs.
- **NOTABLE-3** (architecture): `select` has network and ref side effects despite its name.
- **NOTABLE-4** (goal alignment): the PR #117 description still says patch branches are never touched; update it before pushing.
- **NOTABLE-5/6** (goal alignment): the verify-loop retry refactor and the handoff fallbacks also touch pre-existing paths; behavior unchanged.

## Goal Alignment

All of FR-040 to FR-050, the FR-036 tests and the `add` refusal are delivered after round 1; the three PARTIAL verdicts (FR-044, FR-046, FR-047) were fixed. Undeclared: the PR description (NOTABLE-4).

## Post-Fix Spec Coverage

All spec requirements verified after the fix loop: the fixes added code and tests; nothing implementing an FR was removed.

## Test Suite Results

| Round | Command | Result |
|-------|---------|--------|
| 1 | `bun test tests/custom_build.test.ts tests/sleep_budget.test.ts` | 85 passed |
| 2 | same | 89 passed |
| 3 | same | 91 tests; 84 passed in a run while the machine had a load average near 70, with 7 timeouts in unrelated tests; the 7 rerun: 6 passed, the four-world push test needed more than the 60 s default and got a 180 s limit |
