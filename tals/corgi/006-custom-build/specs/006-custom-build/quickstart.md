# Quickstart: validating custom builds

Prerequisites: Bun, git 2.38+, a corgi clone with an `upstream` remote for `tals/corgi` and `origin` pointing at your fork. `gh` optional (needed for PR titles and to recognize your own PRs).

## Deliverable 1: build identity

1. On a branch tracking upstream: `bun build.ts` then `./dist/corgi --version --verbose`: one line, unchanged from `main`.
2. On a branch pushed to your fork: `bun build.ts`, then `./dist/corgi --version` is unchanged and `--version --verbose` adds `https://github.com/<you>/corgi` (contract: [build-identity.md](contracts/build-identity.md)).
3. `corgi bugreport` shows the same link under the Corgi row.
4. Unit tests: `bun test src/custom_build.test.ts src/bug_report.test.ts`.

## Deliverable 2: the tool

1. `scripts/custom-build add 109` (an open upstream PR), `scripts/custom-build add owner/repo#2`, `scripts/custom-build add my-branch --description "..."`; `scripts/custom-build list`; `git status` is clean.
2. `scripts/custom-build rebuild`: exits 0, `dist/corgi-custom --version --verbose` names your fork, `git log --first-parent upstream/main..custom-build` lists one `Merge <source>: <description>` per patch, `.custom-build/reports/latest/report.md` exists.
3. Conflict without harness: add two patches that conflict, rebuild, expect exit 2 and the files listed; resolve, `git add`, `scripts/custom-build continue`, expect exit 0.
4. Conflict with a stub harness: set `harness.command` to a script that resolves by taking one side (`git checkout --theirs` on the listed files); rebuild completes; report lists the agent resolution.
5. Build failure with stub harness: a patch that breaks typecheck, stub harness applies a fix; report lists the fix commit and a handoff file. A step that fails once and passes on the second run is reported as possibly flaky, with no fix.
6. `scripts/custom-build rollback` after a stopped run: branch and binary match the state before the run, and dependencies are reinstalled; if that install fails, `scripts/custom-build continue` finishes the rollback.

## Rebasing own branches (User Story 6)

1. Own branches behind upstream: make `my-branch` (own) fall behind by pushing a commit to upstream `main`. `scripts/custom-build status` lists it as behind. `scripts/custom-build rebase` rebases it, runs install, typecheck and tests on it alone, moves `my-branch`, prints the backup ref and asks whether to force-push.
2. Mode `conflicting`: with one own branch behind but clean and one conflicting, `scripts/custom-build rebase --rebase=conflicting` rebases only the conflicting one.
3. Other people's patches: an upstream PR from someone else is never rebased; when it conflicts, the report has its `rebase` handoff.
4. Rebase conflict with a stub harness: the conflict is resolved and the branch moves. Without a harness, `rebase` stops with exit 2 and the worktree path; resolve there, `scripts/custom-build continue`; or `scripts/custom-build rebase --skip-branch`.
5. Inside `rebuild`: the same unresolved conflict does not stop the rebuild; the branch keeps its old tip, gets a `rebase` handoff, and the old version is merged.
6. Failing checks: a branch whose rebase breaks typecheck keeps its old tip and gets a `rebase-checks` handoff ([handoff.md](contracts/handoff.md)).
7. In a worktree: check `my-branch` out in a worktree. `status` names the worktree; `rebase` skips the branch; `rebase --in-worktree` rebases it inside that worktree (the checkout stays in step, untracked files survive, a failure resets it back), and a dirty worktree skips it with a reason.
8. Safety: with `origin` pointing at upstream itself, nothing is rebased or pushed; a branch whose remote copy moved after the fetch is refused at push, and the remote keeps its commits; `git branch -f my-branch refs/custom-build/rebase-backup/my-branch` undoes a rebase before it is pushed.
9. Tests: `bun test tests/custom_build.test.ts` (temporary git repositories, stub harness, stub step commands).
