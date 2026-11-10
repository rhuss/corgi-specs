---
runbook_version: 1
feature: 006-custom-build
status: passed
mode: light
steps_total: 14
step_results: passed,passed,passed,passed,passed,passed,passed,passed,passed,passed,passed,passed,passed,passed
drafted_at: 0941e5de
drafted_fingerprint: e9a4651412e948d007af6026b64eae6d70110565f25bf20b78e4c36db8e525bb
tested_at: 6bf60e85
tested_fingerprint: 4919dd7b0060963948bb2ff5aef78bd3a4f37198a7104d71f3df28c52c28289a
updated: 2026-11-04
---

# Try it: Custom corgi builds from integrated branches and PRs

<!-- runbook-status -->
> **Status:** passed · 14 of 14 steps (14 passed) · Mode: light · Tested at 6bf60e85 · 2026-11-04
<!-- /runbook-status -->















## What you're testing

You build a corgi of your own from upstream `main` plus a list of patches, and check that the binary says what it carries. Everything happens in a throwaway clone under `/tmp`, so neither your corgi checkouts nor a running corgi are touched. Every patch is a local branch, so no code from other authors runs.

The journey:

1. Set up a throwaway clone with the feature branch.
2. Build identity (Deliverable 1): a plain build is unchanged, a build given a patch list shows `+custom`, and `corgi update` refuses.
3. The tool (Deliverable 2): add patches, rebuild, look at the binary and the report.
4. A conflict between two patches stops the run; you resolve it and `continue`.
5. A stub harness resolves the next conflict unattended, while rerere replays your earlier resolution.
6. Roll back to the previous build, retire a patch, clean up.

The clone uses its `origin` (your local corgi repository) as upstream, because it has no `upstream` remote. Patch `006-custom-build` brings the tool and build stamping on top of plain `main`.

Each rebuild runs the real checks (typecheck, about 2600 tests, build), so a rebuild takes a few minutes.

## Test environment

- **Throwaway clone** at `/tmp/corgi-custom-build-try`, made in Step 1 by `git clone` from `/Users/rhuss/Development/context-engineering/corgi`, plus `bun install`. Started by whoever runs Step 1. Teardown: Step 14 (`rm -rf /tmp/corgi-custom-build-try /tmp/corgi-custom-build-patches.json /tmp/corgi-custom-build-harness.sh`).

No other fixtures.

**Workaround:** rebuild and continue commands set `CORGI_TEST_SYSTEM_ROOT=/nonexistent`. Without it, `src/discovery.test.ts` finds a Homebrew `pi` install on the host and fails, on plain upstream too, so every rebuild stops with exit 4 (an upstream test bug, found in Step 7).

## Steps

### Step 1 of 14: Make a throwaway clone (prerequisite)

**Why this step:** Everything that follows rewrites a branch, writes `dist/` binaries and a `.custom-build/` directory. A clone keeps all of that away from your real checkouts.

**Needs:** nothing

**Run:**

```
rm -rf /tmp/corgi-custom-build-try && git clone -q --branch 006-custom-build /Users/rhuss/Development/context-engineering/corgi /tmp/corgi-custom-build-try && cd /tmp/corgi-custom-build-try && bun install --frozen-lockfile >/dev/null && git log --oneline -3 && git remote
```

**What you should see:** the three newest commits of `006-custom-build`: `scripts: add custom-build tool for patched corgi builds`, `build: stamp custom patch metadata into the binary`, and the upstream commit below them. `git remote` prints only `origin`.

**If you see something else:** a clone error means the source path moved. A `bun install` lockfile error means `bun.lock` and `package.json` disagree on the branch.

**Result:** passed (agent). Log showed `10d56cd7`, `794cb670`, `c51a3f57`; only `origin` remote.

### Step 2 of 14: A plain build is unchanged (US1)

**Why this step:** Without a patch list, nothing about the binary may change (FR-007).

**Needs:** Step 1

**Run:**

```
cd /tmp/corgi-custom-build-try && bun build.ts >/dev/null && ./dist/corgi --version && ./dist/corgi --version --verbose
```

**What you should see:** both commands print the same single line, `corgi 0.1.0-dev (<commit>)`, with no `+custom` and no patch lines.

**If you see something else:** a `+custom` marker or extra lines means the vanilla path leaks patch metadata.

**Result:** passed (agent). Both commands printed `corgi 0.1.0-dev (10d56cd7...)`, no marker, no patch lines.

### Step 3 of 14: Stamp a patch list into a build by hand (US1)

**Why this step:** This is the whole of Deliverable 1: `build.ts --patches` takes a JSON list and the binary reports it.

**Needs:** Step 1

**Run:**

```
cd /tmp/corgi-custom-build-try && cat > /tmp/corgi-custom-build-patches.json <<'EOF'
[
  { "source": "tals/corgi#109", "commit": "1111111aaaaaaa", "description": "Jump to blocked agents" },
  { "source": "my-branch", "commit": "2222222bbbbbbb", "description": "Section tint" }
]
EOF
bun build.ts --outfile dist/corgi-custom --patches /tmp/corgi-custom-build-patches.json >/dev/null && ./dist/corgi-custom --version && ./dist/corgi-custom --version --verbose
```

**What you should see:** `--version` prints `corgi 0.1.0-dev (<commit>) +custom (2 patches)`. `--verbose` adds two indented lines: `  tals/corgi#109 1111111 Jump to blocked agents` and `  my-branch 2222222 Section tint`.

**If you see something else:** `InvalidPatches: ...` means the JSON shape was rejected. A missing marker means the patch list did not reach the generated build config.

**Result:** passed (agent). `corgi 0.1.0-dev (10d56cd7...-dirty) +custom (2 patches)`, verbose lines `  tals/corgi#109 1111111 Jump to blocked agents` and `  my-branch 2222222 Section tint`. The `-dirty` comes from untracked `.bun-build` temp files that `bun build.ts` leaves behind (pre-existing upstream behavior, not this feature).

### Step 4 of 14: `corgi update` refuses a custom build (US1)

**Why this step:** An update would silently replace your patched binary with a plain release.

**Needs:** Step 3

**Run:**

```
cd /tmp/corgi-custom-build-try && ./dist/corgi-custom update; echo "exit=$?"
```

**What you should see:** `This is a custom build (2 patches). Rebuild it with ...` on stderr, then a non-zero exit code. Nothing gets downloaded.

**If you see something else:** an "unmanaged build" message means the custom check runs too late. A download attempt is a real bug.

**Result:** passed (agent). `This is a custom build (2 patches). Rebuild it with ... instead of updating.`, `corgi: CustomBuild`, `exit=1`.

### Step 5 of 14: Create demo patch branches (prerequisite)

**Why this step:** You need small local patches: one that adds two files, and two that each clash with one of those files. Each of them is clean against upstream on its own.

**Needs:** Step 1

**Run:**

```
cd /tmp/corgi-custom-build-try && git switch -q -c demo-note origin/main && printf 'note from demo-note\n' > docs/demo-note.md && printf 'other from demo-note\n' > docs/demo-other.md && git add docs && git commit -qm "demo: add notes" && git switch -q -c demo-clash origin/main && printf 'note from demo-clash\n' > docs/demo-note.md && git add docs && git commit -qm "demo: clash on note" && git switch -q -c demo-clash2 origin/main && printf 'other from demo-clash2\n' > docs/demo-other.md && git add docs && git commit -qm "demo: clash on other" && git switch -q 006-custom-build && git branch --list 'demo-*'
```

**What you should see:** `demo-clash`, `demo-clash2` and `demo-note` are listed, and you are back on `006-custom-build`.

**If you see something else:** a commit error usually means git has no user identity configured in this environment.

**Result:** passed (agent) after a runbook correction. First run failed with `fatal: invalid reference: main`: a clone made with `--branch` has no local `main`. Command changed to `origin/main` (FR-026), rerun listed all three demo branches.

### Step 6 of 14: Add patches and list them (US3)

**Why this step:** The patch list is plain config managed by the tool, kept outside git.

**Needs:** Step 5

**Run:**

```
cd /tmp/corgi-custom-build-try && scripts/custom-build add 006-custom-build --description "Custom build tool" && scripts/custom-build add demo-note --description "Demo notes" && scripts/custom-build add demo-note --description "again"; echo "duplicate exit=$?"; scripts/custom-build list && git status --short
```

**What you should see:** both adds succeed. The second `demo-note` add is refused with a duplicate message and `duplicate exit=1`. `list` shows `006-custom-build` then `demo-note` with descriptions and today's date. `git status --short` prints nothing, because `.custom-build/` is ignored.

**If you see something else:** output from `git status` means the config directory is not ignored.

**Result:** passed (agent), rerun from a fresh config after the Step 5 fix. Adds succeeded, duplicate refused (`demo-note is already in the list`, `duplicate exit=1`), list in order, `.custom-build/` absent from `git status` (only the `.bun-build` temp files show). Observation: on the first attempt `add demo-note` accepted a branch that did not exist yet; allowed by the spec (rebuild stops at fetch), a warning would be friendlier.

### Step 7 of 14: Rebuild (US2)

**Why this step:** This is the main event: reset to upstream, merge the patches, run the checks, write `dist/corgi-custom`.

**Needs:** Step 6

**Run:**

```
cd /tmp/corgi-custom-build-try && CORGI_TEST_SYSTEM_ROOT=/nonexistent scripts/custom-build rebuild; echo "exit=$?"
```

**What you should see:** `custom-build:` progress lines on stderr (fetch, merge of each patch, typecheck, test, build). Then a summary on stdout naming the report directory, and `exit=0`. It takes a few minutes.

**If you see something else:** exit 1 points to a precondition (dirty tree, git older than 2.38). Exit 3 or 4 names the failing step and its log under `.custom-build/reports/latest/steps/`.

**Result:** passed (agent) on the second attempt. First attempt: `exit=4`, `test fails on plain upstream c51a3f5 too: upstream is broken`, caused by `src/discovery.test.ts` finding Homebrew `/opt/homebrew/bin/pi` 1.3.0 (an upstream test bug; CI has no `pi`). The tool behaved per spec. `rollback` after exit 4 restored `006-custom-build` (`exit=0`). Rerun with `CORGI_TEST_SYSTEM_ROOT=/nonexistent`: `exit=0`, `built dist/corgi-custom on upstream c51a3f5 with 2 patches`. Log: `.walkthrough/step-7.log`.

### Step 8 of 14: The binary and the report say what was merged (US2, US5)

**Why this step:** You should be able to tell from the binary alone, and from the report, which patches it carries.

**Needs:** Step 7

**Run:**

```
cd /tmp/corgi-custom-build-try && ./dist/corgi-custom --version --verbose && git log --oneline --first-parent -3 && sed -n 1,60p .custom-build/reports/latest/report.md
```

**What you should see:**
- `--version` ends with `+custom (2 patches)` and lists `006-custom-build` and `demo-note`, each with a 7-character commit that matches the branch tip (`git rev-parse --short demo-note`).
- The first-parent log shows `custom-build: merge demo-note` and `custom-build: merge 006-custom-build`.
- The report lists the merged patches, each step as passed, no patch that needs a rebase, and how to roll back.

**If you see something else:** commits in `--verbose` that differ from the branch tips mean the stamped list is not the merged one.

**Result:** passed (agent). `+custom (2 patches)` with `006-custom-build 10d56cd` and `demo-note fbffbae`, matching the branch tips. First-parent log: `custom-build: merge demo-note`, `custom-build: merge 006-custom-build`. Report: all steps passed, rebase none, handoffs none, rollback hint present.

### Step 9 of 14: A conflict between patches stops the run (US2)

**Why this step:** Without a harness, a conflict must stop cleanly, with the files and the next commands named.

**Needs:** Step 8

**Run:**

```
cd /tmp/corgi-custom-build-try && scripts/custom-build add demo-clash --description "Clashing note" && CORGI_TEST_SYSTEM_ROOT=/nonexistent scripts/custom-build rebuild; echo "exit=$?"; git status --short
```

**What you should see:** `exit=2`. The last stdout lines name `demo-clash`, the file `docs/demo-note.md`, the report directory, and the `continue` and `rollback` commands. `git status` shows `docs/demo-note.md` as conflicted (`AA`).

**If you see something else:** exit 0 would mean the conflict was resolved silently. Any other code means a different stop than the one expected here.

**Result:** passed (agent). `exit=2`, `merge conflict in demo-clash`, `files: docs/demo-note.md`, continue and rollback commands printed, `AA docs/demo-note.md`. Log: `.walkthrough/step-9.log`.

### Step 10 of 14: Resolve by hand and continue (US2)

**Why this step:** Your resolution becomes part of the build and is recorded by rerere for the next rebuild.

**Needs:** Step 9

**Run:**

```
cd /tmp/corgi-custom-build-try && git checkout -q --theirs -- docs/demo-note.md && git add docs/demo-note.md && CORGI_TEST_SYSTEM_ROOT=/nonexistent scripts/custom-build continue; echo "exit=$?"; ./dist/corgi-custom --version
```

**What you should see:** the run carries on through the checks and ends with `exit=0`. `--version` now ends with `+custom (3 patches)`.

**If you see something else:** exit 1 saying no stopped run means the state from Step 9 was lost.

**Result:** passed (agent). `continue` ran the checks, `exit=0`, `+custom (3 patches)`. Log: `.walkthrough/step-10.log`.

### Step 11 of 14: A stub harness resolves a conflict unattended (US4)

**Why this step:** With a harness configured, a conflict no longer stops the run. The tool checks the harness's result itself. Your Step 10 resolution should replay through rerere without the harness being called for it.

**Needs:** Step 10

**Run:**

```
cd /tmp/corgi-custom-build-try && printf '#!/bin/sh\necho call >> /tmp/corgi-custom-build-harness.calls\nfor f in $(git diff --name-only --diff-filter=U); do git checkout -q --theirs -- "$f"; done\n' > /tmp/corgi-custom-build-harness.sh && chmod +x /tmp/corgi-custom-build-harness.sh && rm -f /tmp/corgi-custom-build-harness.calls && jq '.harness = {command: ["/tmp/corgi-custom-build-harness.sh", "{prompt_file}"], timeoutSeconds: 120, attempts: 2}' .custom-build/config.json > /tmp/corgi-custom-build-config.json && mv /tmp/corgi-custom-build-config.json .custom-build/config.json && scripts/custom-build add demo-clash2 --description "Clashing other" && CORGI_TEST_SYSTEM_ROOT=/nonexistent scripts/custom-build rebuild; echo "exit=$?"; wc -l < /tmp/corgi-custom-build-harness.calls; git log --first-parent -1 --format=%B custom-build | tail -3; ./dist/corgi-custom --version
```

**What you should see:**
- `exit=0`.
- The harness was called once, for `demo-clash2` only.
- The newest merge commit message ends with a `Resolved-By: harness` trailer.
- `--version` ends with `+custom (4 patches)`.
- `.custom-build/reports/latest/report.md` lists the agent-resolved merge and notes the clash between patches.

**If you see something else:** two calls mean rerere did not replay the Step 10 resolution. Exit 5 means every harness attempt was rejected: see `prompts/` in the latest report directory.

**Result:** passed (agent). `exit=0`, harness calls: 1 (only `demo-clash2`; `demo-clash` replayed by rerere), trailer `Resolved-By: harness`, `+custom (4 patches)`. Report: agent-resolved `demo-clash2: docs/demo-other.md`; clashes list both, with `resolved by rerere` and `resolved by harness`. Log: `.walkthrough/step-11.log`.

### Step 12 of 14: Roll back to the previous build (US2)

**Why this step:** When a new build turns out bad, you can go back to the last good one.

**Needs:** Step 11

**Run:**

```
cd /tmp/corgi-custom-build-try && scripts/custom-build rollback; echo "exit=$?"; ./dist/corgi-custom --version && git log --first-parent -1 --format=%s custom-build
```

**What you should see:** `exit=0`. `--version` is back to `+custom (3 patches)`, and the newest merge on `custom-build` is `custom-build: merge demo-clash`.

**If you see something else:** `4 patches` still showing means the binary backup was not restored.

**Result:** passed (agent). `rolled back to custom-build-prev (23e5e96) and dist/corgi-custom.prev`, `exit=0`, `+custom (3 patches)`, tip `custom-build: merge demo-clash`.

### Step 13 of 14: Retire a patch and check the rebase status (US3, US5)

**Why this step:** Removing a patch keeps a record of why. `status` tells you without a rebuild whether any patch needs a rebase onto upstream.

**Needs:** Step 12

**Run:**

```
cd /tmp/corgi-custom-build-try && scripts/custom-build remove demo-clash2 --reason "walkthrough done" && scripts/custom-build list && scripts/custom-build list --retired && scripts/custom-build status; echo "exit=$?"
```

**What you should see:** `list` no longer shows `demo-clash2`. `list --retired` shows it with today's date and the reason. `status` reports no stopped run and no patch that needs a rebase (each demo branch merges cleanly onto upstream alone), then `exit=0`.

**If you see something else:** `remove` asking for a reason means `--reason` was not picked up.

**Result:** passed (agent). `retired demo-clash2: walkthrough done`; list without it; retired list shows date and reason; `status`: `no run in progress`, all three patches `merges cleanly onto upstream`, `exit=0`.

### Step 14 of 14: Clean up

**Why this step:** Removes the throwaway clone and the temporary files.

**Needs:** nothing

**Run:**

```
rm -rf /tmp/corgi-custom-build-try /tmp/corgi-custom-build-patches.json /tmp/corgi-custom-build-harness.sh /tmp/corgi-custom-build-harness.calls && ls /tmp | grep -c corgi-custom-build
```

**What you should see:** `0`.

**If you see something else:** a leftover file name is printed. Delete it by hand.

**Result:** passed (agent). Printed `0`.

## Not covered here

- **Real upstream PRs and fork PRs or branches.** These run code from other authors as you, so this journey uses local branches only. Source parsing and fetching are covered by `tests/custom_build.test.ts`, and an earlier real run against [#109](https://github.com/tals/corgi/pull/109) stopped correctly at a conflict.
- **Build-failure fixes by the harness, and the `build-drift` / `build-clash` / `rebase` handoffs.** These need a patch that breaks a check, or one that conflicts with upstream on its own, plus an agent that can fix code. They are covered by the stub tests.
- **Crash tags and the bug report's custom section.** They only show up in a crash event or a filed bug report. Unit tests cover them (`src/crash.test.ts`, `src/bug_report.test.ts`).
- **`rebuild --push`** pushes to a remote. It is covered by a test against a local bare repository.

## Coverage

- US1 (build identity): Steps 2, 3, 4.
- US2 (rebuild, conflicts, continue, rollback): Steps 7, 8, 9, 10, 12.
- US3 (manage the patch list): Steps 6, 13.
- US4 (harness): Step 11, conflicts only. Build fixes are not covered (see above).
- US5 (reports and handoffs): Steps 8, 13, report and rebase status only. Handoff files are not covered (see above).

## Results

14 steps: 14 passed, 0 failed, 0 skipped (run 2026-11-04 by the agent).

What a reader now knows: a plain build is unchanged; `--patches` stamps the list into `--version`, `--version --verbose` and the update refusal; the tool adds and retires patches outside git, rebuilds from upstream plus local branches with the real checks, stops cleanly at a conflict with `continue`, replays earlier resolutions through rerere, lets a harness resolve the next conflict with tool-side checks and a trailer, and rolls back to the previous build.

Findings:
- Upstream test bug: `src/discovery.test.ts` ("an install too old for Corgi ...") searches real system install locations because it does not set `CORGI_TEST_SYSTEM_ROOT`; with Homebrew `pi` installed it fails, so every rebuild stops with exit 4 on such a machine. Workaround here: `CORGI_TEST_SYSTEM_ROOT=/nonexistent`.
- Runbook correction in Step 5 (`origin/main`).
- Minor: `add` accepts an own branch that does not exist yet (rebuild catches it at fetch).
- Pre-existing: `bun build.ts` leaves `.bun-build` temp files, which mark builds `-dirty`.
