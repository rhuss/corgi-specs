# Contract: harness invocation

## Configuration

```json
"harness": { "command": ["<exe>", "<arg>", "{prompt_file}"], "timeoutSeconds": 900, "attempts": 3 }
```

## Call

- The tool fills a prompt from `scripts/custom_build/prompts/<kind>.md` (`conflict` or `build`), writes it to `<report dir>/prompts/<nn>-<kind>.md`.
- `{prompt_file}` in any command element is replaced by that file's absolute path.
- The same prompt is written to the process's stdin, then stdin is closed.
- Rebase conflicts of own patches (FR-045) use the `conflict` kind with their own template, `prompts/rebase-conflict.md` (variables `patch`, `description`, `branch`, `commit` being replayed, `upstream`, `files`), because the merge template refers to `MERGE_HEAD` and a merge commit; the call runs in the rebase worktree and its acceptance check runs there too.
- Working directory: repository root (the rebase worktree for rebase conflicts). Environment adds `CORGI_CUSTOM_BUILD_KIND=<kind>` and `CORGI_CUSTOM_BUILD_PROMPT=<path>`.
- Killed after `timeoutSeconds`; a timeout counts as a failed attempt.
- The harness's exit code is logged but does not decide acceptance.

## Prompt variables

| kind | variables |
|------|-----------|
| conflict | `patch` (display), `description`, `prLink`, `files` (list), `upstream` (short SHA + subject), `mergedSoFar` (list) |
| build | `step`, `command`, `logExcerpt` (last 200 lines), `patches` (merged list with descriptions), `attempt` |

Placeholders use `{{name}}`; lists render as `- item` lines.

## Rules stated in every prompt

- conflict: edit only the listed files; remove all conflict markers keeping the intent of both sides; do not commit, do not run git commands that change refs; do not touch other files.
- build: change only what is needed to make the step pass; do not commit; do not edit `.custom-build/` or patch branches.

## Acceptance (tool side)

- conflict: research R15.
- build: research R16.
