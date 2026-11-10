# Feature Specification: Custom corgi builds from integrated branches and PRs

**Feature Branch**: `006-custom-build`
**Created**: 2026-11-04
**Status**: Draft
**Input**: Brainstorm `corgi-specs/brainstorm/09-custom-build.md`: anyone can build a bespoke corgi binary from upstream `main` plus patches of their choice (local branches, upstream PRs, fork PRs, fork branches), rebuilt in one end-to-end run, with a configured agent harness resolving merge conflicts and build failures, handoff reports per affected branch, and a binary that identifies itself as a custom build. Delivered as two stacked upstream PRs.

## Delivery Scope

This feature is delivered as two stacked pull requests against `tals/corgi`:

- **Deliverable 1 (build identity)**: every development build reads from git which repository its commit lives in, and a build from a fork names that repository as an https link in `corgi --version --verbose` and the bug report. Useful on its own, for any fork or integration branch, independent of Deliverable 2. Covers User Story 1 and FR-001 to FR-009.
- **Deliverable 2 (custom build tool)**: a build-time tool that manages a patch list, rebuilds the custom branch, verifies and builds the binary, calls a configured agent harness on conflicts and failures, writes handoff reports, and keeps the user's own patch branches rebased onto upstream. Depends on Deliverable 1. Covers User Stories 2 to 6 and FR-010 onward.

Nothing of the tool's rebuild logic ships inside the corgi binary.

## Clarifications

### Session 2026-11-04

- Q: How does the tool hand context to the configured harness? → A: It writes a filled prompt file, substitutes its path for a `{prompt_file}` placeholder in the command template, also pipes the prompt on stdin, and runs the harness in the repository root.
- Q: Which patches are "implicated" in a build failure for attribution? → A: Patches whose changes touch a file modified by the agent fix or named in the failing step's output; if none match, all patches merged in the run. *(Superseded 2026-11-06.)*
- Q: What do crash events from custom builds carry? → A: Only the custom-build marker and the patch count, never patch names or sources. *(Superseded 2026-11-05.)*
- Q: How many backups are kept and how does rollback work? → A: One backup branch `<branch>-prev`, overwritten at each run start, restored by a `rollback` command.
- Q: How long are run reports kept? → A: Each run writes to `reports/<timestamp>/`, `reports/latest` points to the newest, and only the last 10 runs are kept.

### Session 2026-11-05 (build identity rework in #116)

- Q: Where does a build's identity come from? → A: From git at build time, for every development build: the repository of the branch's remote (origin when that remote is upstream), credentials stripped. The tool passes no metadata. *(The first version also listed the first-parent merges since `main`; dropped 2026-11-07.)*
- Q: Does the version line still get a `+custom (N patches)` marker? → A: No. The plain `--version` line is unchanged; `--version --verbose` and the bug report add the repository.
- Q: What do crash events carry? → A: Nothing new; crash events are unchanged.

### Session 2026-11-06 (findings from a real integration build)

- Q: What happens when a step fails once? → A: It runs once more before plain upstream or the harness is involved. Passing then lets the run go on, and the report and summary warn that the step may be flaky.
- Q: Which patches are implicated in a build failure? → A: Patches touching a file the agent fix changed or a file the failure names. The tool's own command lines and lines holding nothing but a path (a test runner's list of files) don't count. With no implicated patch, nothing runs and the report says the fix has no handoff.
- Q: Which steps rerun after a fix? → A: All of them, from the first, so a fix can't break a step that passed before it.

### Session 2026-11-07 (identity and rollback changes in #116 and #117)

- Q: Does the identity still list the merges? → A: No. A build from a fork shows only the fork's repository, as an https link (`https://github.com/you/corgi`). The tool still titles its merges `Merge <source>: <description>`, so the custom branch's log and the run report say what each patch is; the binary says where to look.
- Q: What does rollback do about dependencies? → A: After restoring an earlier tree, it reinstalls dependencies. The rollback is recorded before the checkout is restored, so if the install fails, `continue` finishes the rollback. A rollback refused because of uncommitted changes leaves the stopped run as it was.

### Session 2026-11-07 (bulk rebase of own branches)

- Q: Which patch branches may the tool rebase? → A: Only the user's own: own-branch patches, and PRs whose head branch is on the user's fork (the PR's head repository is `origin`). Other people's PRs and branches on other forks are never rebased and keep getting `rebase` handoffs.
- Q: Which own branches get rebased? → A: By default every own branch that is behind upstream `main`, conflicting or not. A `rebase` mode in the config (`all`, `conflicting`, `off`; default `all`) limits it to branches that no longer merge cleanly or switches it off; `rebuild --rebase=<mode>` overrides it for one run.
- Q: When does it run? → A: Inside `rebuild`, after fetching and before merging, so the build merges freshly rebased branches; and as a separate `rebase` command that only rebases (no build, no reset of the custom branch).
- Q: What happens when a rebase can't be completed inside `rebuild`? → A: The rebase never stops a rebuild. The branch keeps its old tip, gets a `rebase` handoff, and the build merges the old version through the existing merge path.
- Q: And in the `rebase` command? → A: An unresolved conflict stops at that branch, like `git rebase`; the user resolves it and runs `continue`, or skips the branch.
- Q: Must a rebased branch pass checks before it moves? → A: Yes: typecheck and tests on the rebased branch alone, in its throwaway worktree, with the flaky retry. The harness resolves rebase conflicts but does not fix failing checks on the user's branch; that goes into the handoff.
- Q: What if `origin` is upstream itself (no fork)? → A: Then no patch is own: the tool never rebases or pushes patch branches, because pushing to `origin` would push to upstream. The report says why.
- Q: How are rebased branches pushed? → A: At the end of a successful run, on a terminal, the command lists the rebased branches and asks whether to force-push them; `--push` pushes without asking; otherwise the push command is printed. Every push uses a lease against the fetched tip.

### Session 2026-11-08 (reconciled with the implementation at finish)

- Q: Does the push question break "one command, no further input" (SC-001)? → A: Only when own branches were rebased; with patches that need no rebase, or with `--push`, `--rebase=off` or no terminal, a rebuild asks nothing. SC-001 is scoped accordingly.
- Q: Where does the harness run for a rebase conflict? → A: In the throwaway worktree of the branch being rebased, where the rebase is in progress; every other call runs in the repository root.
- Q: Does the `rebase` command test-merge every patch onto upstream too? → A: No. The test-merge belongs to `rebuild` reports (stopped or not); a `rebase` run reports only the branches it handled.
- Q: Who does a lone implicated patch clash with? → A: A clash needs two sides, so a single implicated patch that passes alone names every other merged patch.
- Q: Which exit status does a stopped `rebase` use? → A: The same codes as `rebuild`, by kind of stop: 2 for a conflict it can't resolve, 5 when the harness ran out of attempts.
- Q: What happens to the backup of a branch that was rebased but not pushed, when it is rebased again? → A: It is replaced with the tip that rebase started from, so the backup always undoes the latest move.
- Q: What else does the implementation do that the spec left implicit? → A: The unit-test step builds the native layer first (`bun build.ts --native-only`); a run stops with exit 5 once the harness made `attempts` × 4 fixes and a step still fails, so fixes that break each other can't loop; a run whose push was refused exits 1; and logs of rollbacks made without a stopped run are pruned to the last 10 like run reports.

### Session 2026-11-10 (rebasing inside the branch's own worktree)

- Q: A branch is checked out in a worktree of the user's; can it be rebased? → A: With `--in-worktree` on `rebuild` or `rebase`, yes: the rebase runs inside that worktree, so the checkout moves with its branch instead of being left behind. Without the flag the branch is still skipped, and the skip reason names the worktree and the flag.
- Q: What does an in-worktree rebase require? → A: The worktree's tracked files must be committed; otherwise the branch is skipped. Its untracked files are never removed, and neither is the worktree.
- Q: What happens when an in-worktree rebase or its checks fail? → A: The branch is reset back to the tip the rebase started from, which a clean tree makes safe. The backup of that tip exists from the moment the rebase starts, so an interruption during the checks stays undoable; a branch whose push from an earlier rebase is still due keeps that earlier backup until the new rebase succeeds.
- Q: Where does the harness run for an in-worktree conflict? → A: In the branch's own worktree, where the rebase is in progress; a stop leaves the conflict there, and `continue` resumes it.
- Q: What does `status` say about a branch skipped for being checked out? → A: The skip reason names the worktree and the `--in-worktree` flag that would rebase it there.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A patched binary never passes as vanilla (Priority: P1)

A person runs a corgi built from a fork, for example a custom build with extra patches. Whenever they (or a maintainer reading their bug report) look at the verbose version, they see which repository the build's commit lives in, and can look up there what it carries on top of upstream `main`.

**Why this priority**: It is the part upstream benefits from directly (bug reports from patched builds point at the fork instead of passing as upstream), and every binary Deliverable 2 produces carries it.

**Independent Test**: Build from a branch pushed to a fork and once from upstream `main`. The first shows the fork as an https link in `--version --verbose` and the bug report; the second's output is what it is today.

**Acceptance Scenarios**:

1. **Given** a development build from a branch on a fork, **When** the user runs `corgi --version`, **Then** the line is unchanged from today.
2. **Given** the same binary, **When** the user runs `corgi --version --verbose`, **Then** it adds the fork's repository as an https link (`https://host/owner/repo`, no credentials).
3. **Given** the same binary, **When** the user generates a bug report, **Then** the report shows the same link under the Corgi row.
4. **Given** a release, or a development build whose repository is upstream, **When** the user runs the version command or the bug report, **Then** the output is unchanged from today.
5. **Given** a custom binary, **When** the user runs the self-update command, **Then** it refuses like for any development build and does not install a stable release over it.

---

### User Story 2 - Rebuild a custom binary in one command (Priority: P1)

A contributor lists the patches they want (their own branches, PRs against upstream, PRs or branches on other people's forks) and runs one command. It fetches everything, rebuilds the custom branch from the latest upstream `main`, merges the patches in order, verifies the result, and produces a separately named binary next to the normal one.

**Why this priority**: It is the core value of Deliverable 2; everything else builds on it.

**Independent Test**: In a clone of corgi, configure three patches of different source kinds that merge cleanly, run the rebuild, and confirm the custom binary exists, runs, and names the fork's repository, and that the custom branch's log and the run report list the three patches.

**Acceptance Scenarios**:

1. **Given** a patch list with a local branch, an upstream PR number, a fork PR and a fork branch, **When** the user runs the rebuild, **Then** all four are fetched and merged in list order onto the latest upstream `main`, each as its own merge commit naming the patch.
2. **Given** a successful rebuild, **Then** typecheck, unit tests and the build all ran, and the custom binary was written under its own name without touching the regular build output or an installed stable corgi.
3. **Given** a patch whose PR has been merged upstream, **When** the user rebuilds, **Then** the patch is skipped and the report tells the user it can be removed from the list.
4. **Given** an earlier custom branch exists, **When** a rebuild starts, **Then** the last successful state stays available as the backup branch `<branch>-prev` plus the binary `corgi-custom.prev`, and a `rollback` command returns to the last successful build (the start of an aborted run, otherwise these backups).
5. **Given** the user has uncommitted changes to tracked files in the checkout, **When** they start a rebuild, **Then** the tool refuses and changes nothing.
6. **Given** no harness is configured and a merge conflicts, **Then** the tool stops with a distinct exit status, names the patch and the conflicting files, leaves the merge in progress, and a documented `continue` resumes after the user resolves it.

---

### User Story 3 - Manage the patch list without editing files by hand (Priority: P2)

A contributor adds, removes and lists patches with simple commands, and keeps the list outside the repository's tracked files, optionally in a directory of their choice such as a dotfiles repository.

**Why this priority**: Makes the tool usable day to day; the list can also be edited by hand, so it is not blocking.

**Independent Test**: Add each source kind, list them, remove one with a reason, and confirm the config reflects it and `git status` stays clean.

**Acceptance Scenarios**:

1. **Given** no config exists, **When** the user adds a first patch, **Then** a `.custom-build/` directory is created in the checkout, ignored by git, holding the config.
2. **Given** an upstream PR number, **When** the user adds it without a description, **Then** the description is taken from the PR title.
3. **Given** the user sets the config directory through an environment variable or option, **Then** all reads and writes use that directory instead of `.custom-build/`.
4. **Given** a patch is removed with a reason, **Then** it leaves the active list and is recorded as retired with the date and reason.
5. **Given** the list command, **Then** it shows each active patch with source, description and the date it was added.

---

### User Story 4 - Unattended conflict and build-failure resolution (Priority: P2)

A contributor configures an agent harness command once. From then on the rebuild runs end to end: when a merge conflicts, or typecheck, tests or the build fail, the tool hands the problem to the harness, checks the result itself, and continues. The user reads afterwards which merges and fixes an agent made.

**Why this priority**: Turns the rebuild into a single unattended run, which is what makes keeping many patches practical.

**Independent Test**: Configure a stub harness command that resolves a known conflict deterministically, run a rebuild with two conflicting patches, and confirm the run completes and the report lists the agent resolution.

**Acceptance Scenarios**:

1. **Given** a configured harness and a conflicting merge that rerere cannot replay, **When** the rebuild reaches it, **Then** the harness is called with the patch identity, its PR link and description, and the conflicting files, using the shipped resolution instructions.
2. **Given** the harness returns, **Then** the tool accepts the resolution only if no conflict markers remain and nothing outside the conflicted files changed; it then commits the merge and continues.
3. **Given** typecheck, tests or the build fail after all merges and fail again when run once more, **When** the same step on plain upstream `main` passes, **Then** the harness is called with the failing step, its output and the patches merged in this run; its fix becomes a separate fix commit on top of the merges, and every step reruns from the first.
4. **Given** the same step also fails on plain upstream `main`, **Then** the harness is not called and the report says upstream itself is broken.
5. **Given** the harness fails to produce an accepted result within the attempt limit, **Then** the tool stops with the state left in place, a distinct exit status, and a pointer to the rollback command.
6. **Given** the run completes, **Then** the report lists every agent-resolved merge and every agent fix commit separately from clean merges.
7. **Given** no harness is configured, **Then** behavior is exactly as in User Story 2 (stop and wait for the user).
8. **Given** a step fails once and passes when run again, **Then** neither upstream nor the harness is involved, and the report warns that the step may be flaky.

---

### User Story 5 - Handoff reports that push problems back to the branches (Priority: P3)

After a rebuild, a contributor learns which integrated branches need work upstream (rebase because upstream moved on, a build break caused by the branch, or a clash between two in-flight patches) and gets, for each, a ready-to-run prompt to fix it in that branch's own checkout.

**Why this priority**: Keeps the custom build healthy over time; fixes made during the rebuild are not carried forward, so this is how they reach the PRs.

**Independent Test**: Rebuild with one patch that conflicts with upstream, one that only clashes with another patch, and one that breaks the build against upstream; check the three handoff files and the note.

**Acceptance Scenarios**:

1. **Given** a rebuild, **Then** every active patch is test-merged alone against plain upstream `main`, without touching the working tree, and the report lists which ones need a rebase.
2. **Given** a patch needs a rebase, **Then** a handoff file for it contains the problem, the cause (upstream drift), the conflicting files and a prompt to run in that branch's checkout.
3. **Given** an agent fix was needed for a build failure, **Then** the tool attributes it by running the failing step for each implicated patch merged alone onto plain upstream in an isolated checkout: every patch that fails alone gets a handoff naming upstream drift; if none fails alone, each implicated patch gets a handoff naming the other implicated patches it clashes with, leaving which side adapts to the person.
4. **Given** two patches conflict with each other but each is clean against upstream, **Then** the report notes the clash and no prompt is generated.
5. **Given** any handoff prompt, **Then** it states that changes go into the PR branch, never into the custom build branch, and that the custom build branch is regenerated.
6. **Given** a later rebuild, **Then** earlier agent fix commits are not reapplied; the problem reappears and is fixed and reported again until the PRs are updated.
7. **Given** an agent fix touches no file a merged patch changed, and the failure names none (a test runner's list of files doesn't count), **Then** no isolated runs happen and the report says the fix has no handoff.

---

### User Story 6 - Keep own branches rebased onto upstream (Priority: P2)

When upstream `main` moves, a contributor's own patch branches fall behind and some stop merging cleanly. Instead of rebasing each one by hand, they let the tool rebase all of them, inside every rebuild or on demand, with the harness resolving rebase conflicts, and then push the rebased branches with one confirmation.

**Why this priority**: Rebasing several branches by hand every time main moves is the main chore left after Stories 2 to 5; handoffs only describe it.

**Independent Test**: Own two branches, one behind upstream but clean and one conflicting, plus another person's PR. Run `rebuild` with a stub harness that resolves the conflict. Both own branches end up on top of upstream `main` with their checks passed, the other person's PR is untouched and gets its `rebase` handoff, and the push is offered at the end.

**Acceptance Scenarios**:

1. **Given** the default mode `all` and two own branches behind upstream `main`, one of them conflicting, **When** the user runs `rebuild`, **Then** both are rebased onto upstream before any patch is merged, and the build merges the rebased versions.
2. **Given** the mode `conflicting`, **Then** only the own branch that no longer merges cleanly onto upstream is rebased; with `off`, none is.
3. **Given** a patch that is another person's PR or a branch on another person's fork, **Then** it is never rebased and keeps getting a `rebase` handoff when it conflicts.
4. **Given** a rebase stops at a conflict and a harness is configured, **Then** the harness resolves it under the same acceptance checks as a merge conflict, and the rebase goes on.
5. **Given** a rebased branch, **Then** typecheck and tests run on it alone before it gets its new tip; if they fail (after the flaky retry), the branch keeps its old tip and gets a handoff naming what broke.
6. **Given** a rebase inside `rebuild` that cannot be completed, **Then** the rebuild does not stop: the branch keeps its old tip, gets a `rebase` handoff, and the old version is merged.
7. **Given** the `rebase` command and a conflict the harness cannot resolve (or no harness), **Then** the command stops at that branch with a distinct exit status; the user resolves the conflict and runs `continue`, or skips the branch.
8. **Given** branches were rebased and the command runs on a terminal, **Then** it lists them at the end and asks whether to force-push them; with `--push` it pushes without asking; without a terminal and without `--push` it prints the push command.
9. **Given** a rebased branch whose remote copy changed since it was fetched, **When** the push runs, **Then** that push is refused and reported, and the remote copy is kept.
10. **Given** any rebase, **Then** the branch's previous tip stays available under `refs/custom-build/` until it is pushed, so the rebase can be undone.
11. **Given** `--in-worktree` and an own branch checked out in a worktree of the user's, **When** the run rebases it, **Then** the rebase happens inside that worktree, the checkout stays in step with the branch, and a failure puts the branch back where it was.

---

### Edge Cases

- A patch's branch or PR no longer exists or cannot be fetched: the rebuild stops before resetting anything, naming the patch.
- A PR was closed without merging: it is still merged, with a warning in the report.
- A fork PR's source branch was force-pushed: the latest head is used, and its merge commit in the custom branch reflects it.
- The same commits arrive through two patches (stacked PRs): already-merged patches are reported as already contained and skipped without error.
- The local branch is behind or diverged from its pushed copy: the report says which one was used and why.
- The harness command is missing or not executable: treated as no harness configured, with a warning.
- The harness hangs: a time limit per call applies, after which the attempt counts as failed.
- The harness edits files outside the conflicted set during conflict resolution: the attempt is rejected and retried; companion edits needed for the build belong to the build-failure stage.
- Upstream `main` itself fails a step: reported as upstream breakage; no harness call, no patch blamed.
- Interrupted run (Ctrl-C, closed terminal): the backup stays intact and the next invocation refuses a fresh run, explaining how to continue or roll back.
- Every patch is skipped (all merged upstream): the custom branch equals upstream `main` and the binary still names the fork it was built from; there is no patch count anywhere to show `0`.
- After a successful or stopped run, the custom branch is checked out in the working tree.
- The config directory is a path outside the repository that does not exist yet: it is created on first write.
- A custom build runs as the live corgi runtime: the rebuild never restarts it unless the user configured an after-build command.
- An own branch is checked out in any worktree: it is not rebased, because moving it would leave that checkout's files out of step with its branch; the report names the worktree. With `--in-worktree` it is rebased inside that worktree instead, which keeps the checkout in step.
- An own branch's local copy and its pushed copy have diverged: it is not rebased (a push would drop remote commits), and the report says so.
- An own PR's head branch exists only on the fork: the rebased tip goes to a local branch of the same name, created for it, and is pushed from there.
- A rebase leaves a branch with no changes left (everything is already upstream): the branch is not moved, and the report suggests removing the patch.
- The GitHub CLI is unavailable: PRs can't be matched to the fork, so only own-branch patches are rebased.
- `origin` is upstream itself (no fork): nothing is rebased or pushed; the report says why.
- There is no `origin` remote at all: the same; the report says there is nowhere to push rebased branches.
- A run is interrupted during a rebase: the branch still has its old tip (it moves only after its checks pass); the throwaway worktree is removed on the next run. A branch rebased in its own worktree has its backup from the rebase's start, so an interruption during its checks undoes like any other rebase.

## Requirements *(mandatory)*

### Functional Requirements

**Deliverable 1: build identity**

- **FR-001**: Every development build MUST read its identity from git at build time: the repository of the branch's remote, or origin when that remote is upstream or missing. Releases carry no identity.
- **FR-002**: The identity MUST be compiled into the binary alongside the existing release version and commit.
- **FR-003**: The plain `corgi --version` line MUST stay unchanged for every build.
- **FR-004**: For a build whose repository is not `tals/corgi`, `corgi --version --verbose` MUST add the repository as an https link, `https://host/owner/repo`.
- **FR-005**: The bug report MUST show the same link under the Corgi row.
- **FR-006**: The repository MUST never carry credentials, user names or ports from the remote URL.
- **FR-007**: For releases and for development builds whose repository is upstream (or unknown), version output, bug report and crash events MUST be identical to today.
- **FR-008**: The build MUST support writing the binary under the name `corgi-custom` in the regular output directory, so it never replaces the regular build output.
- **FR-009**: The self-update command run from a custom build MUST NOT install a stable release over it; it refuses as for any development build. A custom build MUST never be installed into the managed releases directory.

**Deliverable 2: custom build tool**

- **FR-010**: The tool MUST be a build-time script in the repository's scripts area; it MUST NOT be a corgi subcommand.
- **FR-011**: Configuration and reports MUST live in `.custom-build/` at the repository root by default, which the repository ignores.
- **FR-012**: The config directory MUST be overridable by the `CORGI_CUSTOM_BUILD_DIR` environment variable and a `--dir` option (option wins).
- **FR-013**: The config MUST be a structured, human-editable file listing active patches in merge order, each with source, description and date added, plus retired patches with retirement date and reason.
- **FR-014**: The tool MUST support these patch sources: a local or pushed branch of the user's own repository; an upstream PR by number; a PR on another repository; a branch on another repository given as repository plus branch.
- **FR-015**: Upstream PRs, including PRs opened from forks, MUST be fetchable without adding the fork as a remote.
- **FR-016**: The tool MUST provide `add`, `remove` and `list` commands; `add` for a PR without a description takes it from the PR title; `remove` records a reason (asked for or given).
- **FR-017**: The rebuild MUST: refuse on uncommitted tracked changes; fetch upstream and all patch sources before changing anything; refuse to start while an earlier run is stopped or in progress (offering `continue` or `rollback`); reset the custom branch to the latest upstream `main`; merge each active patch in order as a non-fast-forward merge commit titled `Merge <source>: <description>`; then run `bun install` (only when nothing is installed yet or `bun.lock` or `package.json` changed), typecheck, unit tests (after building the native layer they need) and the build, writing `corgi-custom`, which names the user's fork (FR-001); the merge titles in the branch's log and the run report say what each patch is. A patch that names the custom branch or its `-prev` backup is refused. Only after a fully successful run are the previous successful branch and binary saved as `<branch>-prev` and `corgi-custom.prev`, replacing older backups; a failed or stopped run never overwrites them.
- **FR-018**: The custom branch name MUST default to `custom-build` and be configurable.
- **FR-019**: The rebuild MUST skip patches whose PR is merged into upstream, and patches already fully contained in the branch, reporting both.
- **FR-020**: Recorded conflict resolutions MUST be replayed automatically (rerere) before any harness is called.
- **FR-021**: Without a harness, an unresolved conflict or failed step MUST stop the run with a distinct exit status per case, naming the patch or step and files, leaving the state in place; a `continue` command MUST resume from that point.
- **FR-022**: The config MAY name a harness as a command template. For each call the tool writes a filled prompt file, substitutes its path for the `{prompt_file}` placeholder, pipes the same prompt on stdin, and runs the command in the repository root (for a conflict while rebasing an own branch, in the worktree the rebase runs in, where it is in progress). The configured command is the sole owner of the harness's permissions.
- **FR-023**: The repository MUST ship harness-neutral instruction files for conflict resolution and build-failure fixing, filled with the run's context.
- **FR-024**: After a harness conflict resolution, the tool MUST verify that no conflict markers remain and that only the conflicted files changed before committing the merge.
- **FR-025**: On a failed step, the tool MUST first run that step once more; if it passes then, the run goes on and the report and summary warn that the step may be flaky. If it fails again, the tool MUST run it on plain upstream `main` in an isolated location; only if it passes there is the harness called.
- **FR-026**: A harness build fix MUST be committed as a separate fix commit on top of the merges, never inside a merge commit or a patch branch. After any fix, the harness's or the user's, every step reruns from the first.
- **FR-027**: Harness calls MUST be bounded by an attempt limit (default 3, configurable) and a per-call time limit; exhausting them stops the run with the state in place and the rollback command printed. A run MUST also stop once the harness has made `attempts` × 4 build fixes (one round per step) and a step still fails, so fixes that break each other can't loop.
- **FR-028**: The tool MUST never amend patch branches, never rebase or push branches the user doesn't own (FR-040), and never push to upstream. Pushing the custom branch to the user's own remote happens only on explicit request (`--push`), and is refused when `origin` pushes to the same repository the upstream remote fetches from.
- **FR-029**: At the end of every `rebuild` run, stopped or not, the tool MUST test-merge each active patch alone against plain upstream `main` without touching the working tree and report which need a rebase.
- **FR-030**: A patch is implicated in a build failure when its changes touch a file modified by the agent fix or named in the failing step's output. The tool's own command lines and lines holding nothing but a path (such as a test runner's list of files) MUST NOT count as naming a file. If no patch is implicated, attribution runs nothing, and the report says the fix has no handoff. Each isolated attribution run MUST print which patch it runs and how many are left.
- **FR-031**: Attribution MUST run the failing step for each implicated patch merged alone onto plain upstream in an isolated checkout. Patches that fail alone are attributed to upstream drift; if none fails alone, all implicated patches are attributed to a clash with each other (a single implicated patch clashes with every other merged patch, since a clash needs two sides).
- **FR-032**: For each patch that needs a rebase, causes a build failure against upstream, or is attributed a build failure clash with another in-flight patch, the tool MUST write a handoff file in the config directory's `reports/` area with the problem, the cause, the fix applied in the custom build (if any) and a ready-to-run prompt for that branch's checkout.
- **FR-033**: Merge conflicts only between two patches that are each clean against upstream MUST be noted in the report without a handoff prompt, whether the user or the harness resolved them. Handoff prompts are written only for rebase-needed patches and for build failures.
- **FR-034**: Agent fix commits MUST NOT be carried into later rebuilds.
- **FR-035**: Each run MUST write its report and handoffs to `reports/<timestamp>/` in the config directory, point `reports/latest` at it, and keep only the 10 most recent runs. Logs of a rollback made without a stopped run go to `reports/rollback-<timestamp>/`, of which also only the 10 most recent are kept.
- **FR-036**: A `rollback` command MUST return to the last successful custom build: with a run in progress, it aborts the run and restores the branch tip and binary the run started from; with no run in progress, it restores the branch from `<branch>-prev` and the binary from `corgi-custom.prev`. With a run in progress, it MUST refuse while another branch is checked out with uncommitted changes, since restoring would discard them, and leave the stopped run as it was. After restoring an earlier tree, rollback MUST reinstall dependencies; the rollback is recorded before the checkout changes, so when the install fails, `continue` finishes the rollback.
- **FR-037**: The final report MUST list: upstream commits pulled in, patches merged, skipped or already contained, agent-resolved merges, agent fix commits, rebase-needed patches, handoff files written, and the result of each verification step.
- **FR-038**: The config MAY name an after-build command (for example, hot-reloading a running corgi from the new binary), run only after a fully successful build.
- **FR-039**: The tool MUST work without any agent harness installed and without Claude Code.

**Rebasing own branches (User Story 6)**

- **FR-040**: A patch is the user's own when it is an own-branch patch, or a PR whose head repository is the repository `origin` pushes to. Only own patches MAY be rebased or pushed by the tool; PR ownership needs the GitHub CLI, and without it only own-branch patches count. When `origin` pushes to the repository the upstream remote fetches from, no patch is own, and the report says rebasing is off because there is no fork to push to.
- **FR-041**: The config MUST accept a `rebase` mode: `all` (default) rebases every own patch whose branch is behind upstream `main`; `conflicting` only those that no longer merge cleanly onto upstream `main`; `off` none. `rebuild --rebase=<mode>` and the `rebase` command's `--rebase=<mode>` MUST override it for one run.
- **FR-042**: `rebuild` MUST run the rebase phase after fetching and before resetting the custom branch, so the merges use the rebased tips. The rebase phase MUST NOT stop a rebuild: an own patch whose rebase or checks fail keeps its old tip, gets a `rebase` handoff, and is merged as before.
- **FR-043**: A `rebase` command MUST run only the rebase phase: no reset of the custom branch, no merges, no build. It MUST refuse to start while a rebuild or rebase run is stopped, like `rebuild`.
- **FR-044**: Each rebase MUST run in a throwaway worktree, never in a checkout the user works in, and MUST NOT touch a branch that is checked out in any worktree. Exception: with `--in-worktree`, a branch checked out in a worktree of the user's is rebased inside that worktree, so the checkout moves with its branch; its tracked files MUST be committed, its untracked files and the worktree itself MUST NOT be removed, and a rebase or check that fails MUST reset the branch back to the tip the rebase started from, with the backup of FR-047 kept from the rebase's start.
- **FR-045**: A rebase conflict that rerere cannot replay MUST go to the configured harness, which is accepted under the same checks as a merge conflict (FR-024) and the same attempt and time limits (FR-027). Inside `rebuild`, a conflict that stays unresolved abandons that branch's rebase (FR-042). In the `rebase` command, it MUST stop with the exit status of its kind of stop (2 for an unresolved conflict, 5 when the harness ran out of attempts, as in `rebuild`), naming the branch, the commit being replayed and the files, with the rebase in progress in its worktree; `continue` resumes after the user resolved it there, and `rebase --skip-branch` abandons that branch and goes on with the next.
- **FR-046**: After a branch rebased, the tool MUST run install, typecheck and tests on the rebased tip alone, with the flaky retry (FR-025). Only when they pass does the branch move to the new tip; otherwise it keeps its old tip and gets a handoff naming the failing step, with its output and a prompt for the branch's checkout (a branch rebased in its own worktree is moved by the rebase itself and reset back when its checks fail). The harness MUST NOT fix failing checks on a patch branch.
- **FR-047**: Before moving a branch, the tool MUST keep its previous tip under `refs/custom-build/rebase-backup/<branch>` until the branch is pushed (a branch rebased again before it was pushed replaces the backup with the tip that rebase started from), and the report MUST say how to restore it.
- **FR-048**: An own branch whose local copy and pushed copy have diverged MUST NOT be rebased; a rebase that leaves no changes on top of upstream MUST NOT move the branch, and the report suggests removing the patch. For an own PR whose head branch exists only on the fork, the rebased tip goes to a new local branch of that name.
- **FR-049**: At the end of a successful `rebuild` or `rebase` run (a stopped run offers nothing until it completes), the tool MUST list the branches it moved. On a terminal it MUST ask whether to force-push them to `origin`; `--push` MUST push without asking; without a terminal and without `--push` it MUST print the push command. Every push MUST use `--force-with-lease` against the tip the tool fetched; a refused push is reported, leaves the remote copy untouched, and makes the run exit 1.
- **FR-050**: The run report MUST list each own patch's rebase result: rebased (cleanly or with the harness), already current, skipped and why, or failed and why; and each push result.

### Key Entities

- **Patch**: one entry of the list. Source (kind plus reference), description, date added; retired patches add date and reason.
- **Custom build config**: the patch list, custom branch name, optional harness command template, attempt and time limits, optional after-build command, rebase mode.
- **Build identity**: what the binary carries: the repository its commit lives in.
- **Rebase result**: per own patch and run: old tip, new tip, how it was rebased (cleanly, harness, rerere), check results, push result.
- **Run report**: summary of one rebuild run, including agent actions and verification results.
- **Handoff**: per-patch file with problem, cause, applied fix and prompt.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: With 5 cleanly merging patches, a full rebuild needs exactly one command and no further input; the only question it may ask is whether to push own branches rebased and not pushed yet (FR-049), which `--push`, `--rebase=off` or a run without a terminal skip.
- **SC-002**: With a harness configured, a rebuild that hits a new merge conflict and a build failure still completes in one command in 100% of cases where the harness's result passes verification.
- **SC-003**: 100% of development builds from a fork name the fork in `--version --verbose` and the bug report; 0 changes in those outputs for releases and for builds whose repository is upstream.
- **SC-004**: Every patch that needs a rebase on upstream appears in the run report (no false "clean").
- **SC-005**: After any failed or interrupted run, the user returns to the previous custom build with the single `rollback` command.
- **SC-006**: Adding a patch of any of the four source kinds takes one command.
- **SC-007**: A run never modifies upstream or branches the user doesn't own: their commits are identical before and after. Own branches change only by a rebase whose checks passed, and stay restorable from their backup ref; with `--in-worktree`, a branch moves with its rebase, is put back when its checks fail, and is restorable from its backup ref from the rebase's start.
- **SC-008**: After upstream `main` moves, bringing all own patch branches up to date and pushed takes one command and one confirmation when no rebase conflict is left unresolved.

## Assumptions

- Upstream, fork PRs and fork branches are hosted on GitHub; fork branches may also be given as any git URL. The GitHub CLI is used when available for PR state and titles; without it, merged detection falls back to commit ancestry and descriptions must be given by hand.
- The existing verification commands (typecheck, unit tests, build) are the verification steps; integration suites are not part of the rebuild by default.
- The tool is written in the repository's existing scripting runtime and adds no new dependencies (proposal from brainstorm).
- The config format is chosen in planning; it must be easy to edit by hand and to rewrite by the tool.
- git 2.38 or newer is available (needed to test-merge without touching the working tree).
- Users with several custom builds use separate config directories and branch names.
- `origin` is the user's own fork, the only place the tool pushes patch branches to; own PRs are recognized by their head repository matching it.
