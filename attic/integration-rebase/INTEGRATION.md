# Integration branch

Personal daily-driver build of corgi: `upstream/main` plus the branches below, each merged with `--no-ff`.
It is rebuilt by the `integration-rebase` skill (`corgi-dev-rebase`) and never proposed upstream.

## Active

Merged in this order. `rebuild.sh` reads the branch and PR columns of this table.
Keep `|` out of the cells.

| Branch | PR | Issues | Description | Added |
|---|---|---|---|---|
| `001-voice-dictation` | [#36](https://github.com/tals/corgi/pull/36) | | Dictate into the focused pane with a local Whisper model, locally or over SSH (`corgi voice --ssh HOST`) | 2026-10-28 |
| `005-voice-transcripts` | [#2](https://github.com/rhuss/corgi/pull/2) | | Record voice mode speech to a transcript file, with a reading view (fork PR, stacked on `001-voice-dictation`) | 2026-11-02 |
| `003-inline-sidebar-rename` | [#54](https://github.com/tals/corgi/pull/54) | [#42](https://github.com/tals/corgi/issues/42) | Rename workspace and agent rows in place | 2026-10-30 |
| `004-workspace-section-tint` | [#93](https://github.com/tals/corgi/pull/93) | | Add an optional tint for the workspace section | 2026-11-02 |
| `fix-jump-to-blocked` | [#109](https://github.com/tals/corgi/pull/109) | | Keep blocked agents as jump-to-attention targets until answered, oldest first, focused pane last | 2026-11-03 |

## Retired

Branches that left the integration branch, because their PR was merged upstream or they were dropped.

| Branch | PR | Issues | Description | Added | Retired | Reason |
|---|---|---|---|---|---|---|
| `002-pane-click-selection` | [#43](https://github.com/tals/corgi/pull/43) | [#21](https://github.com/tals/corgi/issues/21) | Double-click copies a word, triple-click a line | 2026-10-28 | 2026-11-02 | merged upstream as a0adb2e |
| `fix/held-enter-new-workspace` | [#51](https://github.com/tals/corgi/pull/51) | | Keep the held ↵ out of the new shell when creating a workspace | 2026-10-30 | 2026-11-02 | obsolete: upstream dropped hold-to-create in [#82](https://github.com/tals/corgi/pull/82) (adecbb1), PR closed |
