# Data Model: Custom corgi builds

## PatchSource (tagged union)

| kind | fields | display (FR-004) |
|------|--------|------------------|
| `branch` | `name` | `<name>` |
| `pr` | `repo` (owner/repo), `number` | `<repo>#<number>` (upstream PRs show `tals/corgi#N` too) |
| `fork-branch` | `repo` (owner/repo) or `url`, `branch` | `<repo>:<branch>` (URL form: `<url>:<branch>`) |

Upstream PRs are `pr` with `repo` equal to `upstream.repo`. Identity of a patch is its display string; two entries with the same display string are rejected by `add`.

## Patch (config entry)

| field | type | rule |
|-------|------|------|
| `source` | PatchSource | required |
| `description` | string | required, single line; from PR title (without `area:` prefix) when omitted for a PR and `gh` is available |
| `added` | `YYYY-MM-DD` | set by `add` |

## RetiredPatch

Patch plus `retired` (`YYYY-MM-DD`) and `reason` (non-empty string).

## CustomBuildConfig (`config.json`)

```json
{
  "version": 1,
  "upstream": { "repo": "tals/corgi", "remote": "upstream", "branch": "main" },
  "branch": "custom-build",
  "patches": [Patch],
  "retired": [RetiredPatch],
  "harness": { "command": ["claude", "-p"], "timeoutSeconds": 900, "attempts": 3 },
  "afterBuild": ["./dist/corgi-custom", "dev"],
  "rebase": "all"
}
```

- `version`: schema version, `1`. Unknown higher versions are refused.
- `upstream.remote` optional (default: `upstream` if present, else `origin`); `upstream.branch` default `main`.
- `branch` default `custom-build`; must not be `upstream.branch` or the current non-custom branch of another worktree.
- `harness` and `afterBuild` optional. `harness.command` non-empty array of strings.
- `rebase` optional: `all` (default), `conflicting` or `off` (FR-041).
- A patch whose display source is `branch` or `<branch>-prev` is refused (FR-017).
- Order of `patches` is the merge order.

## Identity (compiled into the binary, Deliverable 1)

| field | type |
|-------|------|
| `repository` | string, `host/owner/repo` or empty |

`buildRepository` in `build_config`, filled by `build.ts` from git for development builds; empty for releases. Shown as `https://<repository>` when it is not `github.com/tals/corgi` (FR-004); otherwise nothing extra (FR-007). The tool's merge subjects `Merge <display source>: <description>` stay in the custom branch's log.

## MergedPatch (run state)

| field | type |
|-------|------|
| `source` | string (display form) |
| `commit` | string (full SHA of the merged patch tip) |
| `description` | string |

## RunState (`run.json`, present only while a run is stopped or in progress)

| field | type | meaning |
|-------|------|---------|
| `startedAt` | ISO time | run start |
| `startTip` | SHA or null | custom branch tip at start (null on first run) |
| `startBinary` | path or null | copy of `dist/corgi-custom` taken at start |
| `upstream` | SHA | upstream commit the run reset to |
| `reportDir` | path | this run's report directory |
| `phase` | `rebase` \| `merge` \| `verify` \| `report` \| `rollback` | where the run is (`rebase` only for `kind: "rebase"`; `rollback` while a rollback restores the tree and dependencies) |
| `cursor` | number | index of the next patch to merge |
| `step` | `install` \| `typecheck` \| `test` \| `build` \| null | current verification step |
| `attempts` | number | harness attempts used for the current problem |
| `kind` | `rebuild` \| `rebase` | which command the run belongs to (absent means `rebuild`) |
| `merged` | MergedPatch[] | patches merged so far |
| `fixes` | Fix[] | fix commits (step, commit, by `agent` or `user`, files, failure log, attribution) |
| `rebased` | RebaseResult[] | own patches handled by the rebase phase or command |
| `rebaseTargets` | RebaseTarget[] | `rebase` runs: the own branches selected for rebasing (patch, branch, old tip, lease, local tip, conflicts, worktree path with `--in-worktree`) |
| `rebaseCursor` | number | `rebase` runs: index of the own patch being rebased |
| `rebaseStop` | object or null | `rebase` runs: where the run stopped (target, worktree, commit being replayed, files) |
| `events` | Event[] | per patch: `merged`, `replayed`, `agent-resolved`, `user-resolved`, `skipped-merged`, `already-contained`, `closed-warning`, `note`; per step: `passed`, `flaky`, `agent-fix <sha>`, `user-fix`, `upstream-broken` |

State transitions for `kind: "rebuild"`: none → `merge` → `verify` → `report` → none (success). `merge` or `verify` → stopped (exit 2/3/4/5, file kept) → `continue` resumes the same phase. Any stopped or in-progress state → `rollback` → none; a rollback whose dependency install fails stays in phase `rollback`, and `continue` finishes it. The rebuild's rebase phase runs before `run.json` exists and only leaves its results in `rebased`.

State transitions for `kind: "rebase"`: none → `rebase` → none (all own patches handled). Stopped at a branch (exit 2/5) → `continue` or `rebase --skip-branch` resumes; `rollback` aborts the rebase in progress → none.

## RebaseResult

| field | type | meaning |
|-------|------|---------|
| `patch` | string | display source |
| `branch` | string | local branch moved (or that would be) |
| `from` | SHA | old tip |
| `to` | SHA or null | new tip when moved |
| `lease` | SHA or `""` | remote tip fetched before the rebase; `""` when the branch was never pushed |
| `outcome` | `rebased` \| `current` \| `skipped` \| `failed` | `current`: not behind (or not conflicting in mode `conflicting`) |
| `how` | `clean` \| `rerere` \| `harness` \| `user` | for `rebased` |
| `reason` | string | for `skipped` (checked out in a worktree, uncommitted changes in its worktree, diverged, no gh, nothing left, branch moved meanwhile) and `failed` (unresolved conflict, failing step) |
| `step`, `log` | StepName, path | for a failed check: the failing step and its log, for the `rebase-checks` handoff |
| `push` | `pushed` \| `refused` \| null | push result |

The backup of a moved branch is `refs/custom-build/rebase-backup/<branch>` until a push succeeds; a branch rebased in its own worktree (`--in-worktree`) has its backup from the rebase's start, because the rebase itself moves the branch.

## RunReport (`report.md`)

Sections: upstream (old..new commit count, notable subjects), rebased branches (each RebaseResult with undo command), merged patches (with events), skipped, agent-resolved merges, agent fix commits, verification results per step (with flaky warnings), rebase-needed patches, clash notes, handoffs written, rollback hint.

## Handoff (`handoffs/<slug>.md`)

| field | content |
|-------|---------|
| kind | `rebase` \| `rebase-checks` \| `build-drift` \| `build-clash` |
| patch | display source, description, PR link if any |
| problem | failing step and log excerpt, or conflicting files |
| cause | upstream drift, or clash with named patches |
| applied fix | agent fix commit SHA and diff stat in the custom build, if any |
| prompt | ready-to-run prompt for the patch's own checkout |

Slug: display source with non-alphanumerics replaced by `-`, lowercased.
