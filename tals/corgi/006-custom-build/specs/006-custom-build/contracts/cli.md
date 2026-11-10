# Contract: `scripts/custom-build` CLI

```
scripts/custom-build [--dir <path>] <command> [args]
```

Config directory: `--dir` > `CORGI_CUSTOM_BUILD_DIR` > `<repo root>/.custom-build`. Created on first write.

## Commands

| command | effect |
|---------|--------|
| `add <source> [--description <text>]` | Append a patch (syntax in research R8). PR description defaults to its title. Duplicate sources, and the custom branch or its `-prev` backup, are rejected. |
| `remove <source> [--reason <text>]` | Move to `retired` with today's date. Without `--reason`: asks on a TTY, otherwise exit 1. |
| `list` | Active patches in merge order: display source, description, added date. `--retired` lists retired ones. |
| `rebuild [--rebase=<mode>] [--push] [--in-worktree]` | Full run (FR-017), with the rebase phase for own patches first (FR-042). `--push` force-pushes the custom branch and the rebased branches to `origin` with lease after success; without it, a terminal is asked about the rebased branches. |
| `rebase [--rebase=<mode>] [--push] [--in-worktree]` | Only the rebase phase (FR-043). Stops at an unresolved conflict. `--push` as for `rebuild`, for the rebased branches. `--in-worktree` rebases a branch checked out in a worktree of the user's inside that worktree (FR-044's exception), instead of skipping it. |
| `rebase --skip-branch` | In a stopped rebase run: abandon the current branch's rebase and go on with the next. |
| `continue` | Resume a stopped run. Rebuild: after the user resolved the conflict (committed or staged) or fixed the step (committed or left in the tree, which the tool then commits as a user fix). Rebase: after the user resolved the conflict in the printed worktree. |
| `rollback` | Rebuild run or none: return to the last successful build (FR-036, research R11). Rebase run: abort the rebase in progress; branches already moved keep their new tips. |
| `status` | Show run state if any, the rebase check (FR-029), which own patches are behind or conflicting, and which own branches are left alone with the reason (the worktree it is checked out in, a pending push, or divergence), without changing anything. |

`<mode>` is `all`, `conflicting` or `off` and overrides the config's `rebase` for one run.

## Exit codes

| code | meaning |
|------|---------|
| 0 | success |
| 1 | usage or precondition error (dirty tree, run in progress, git < 2.38, unknown source, fetch failure, `origin` pushes to upstream with `--push`) |
| 2 | stopped at a merge or rebase conflict that needs resolution |
| 3 | stopped at a failed verification step |
| 4 | upstream itself fails a verification step |
| 5 | harness attempts exhausted (conflict or step) |

On codes 2 to 5, the last stdout lines name the patch, branch or step, the files, the worktree for a rebase stop, the report directory, and the next commands (`continue`, `rebase --skip-branch`, `rollback`).

## Output

Progress lines on stderr prefixed `custom-build: `. The final report summary on stdout, including each rebased branch, its undo command and push result.

## Guarantees

- Never amends patch branches; rebases and pushes only own patches (FR-040), each moved only after its checks passed, never one checked out in a worktree unless `--in-worktree` rebases it there; never pushes to upstream (FR-028).
- Never adds git remotes; fetched refs and rebase backups live under `refs/custom-build/`.
- Only touches the configured custom branch, `<branch>-prev`, own patch branches as above, `refs/custom-build/*`, `dist/corgi-custom*`, and the config directory, plus the remote-tracking refs it fetches and the rerere cache.
