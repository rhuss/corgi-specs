# Brainstorm Overview

Last updated: 2026-11-02

## Sessions

| # | Date | Topic | Status | Spec | Issue |
|---|------|-------|--------|------|-------|
| 01 | 2026-10-26 | voice-dictation | active | - | - |
| 02 | 2026-10-26 | voice-glossary | parked | - | - |
| 03 | 2026-10-26 | voice-transcript-recording | active | - | - |
| 04 | 2026-10-26 | voice-speaker-separation | parked | - | - |
| 05 | 2026-10-28 | pane-click-selection | active | - | [#21](https://github.com/tals/corgi/issues/21) |
| 06 | 2026-10-28 | inline-sidebar-rename | active | - | [#42](https://github.com/tals/corgi/issues/42) |
| 07 | 2026-10-28 | agent-identity-color | active | - | [#44](https://github.com/tals/corgi/issues/44) |
| 08 | 2026-11-02 | workspace-section-tint | active | - | - |

## Open Threads
- How `corgi voice --ssh host` reaches the remote daemon (socket forwarding vs stdio bridge) (from #01)
- How the voice app identifies its target viewer, especially remotely (from #01)
- Capture tools and device enumeration on macOS and Linux (from #01)
- Corgi-owned model cache location (from #01)
- Mute shortcut per preset and indicator placement per the UI style guide (from #01)
- Whether "next" needs a new daemon/CLI command for Jump to attention (from #01)
- Minimum protocol change and behavior with older viewers (from #01)
- Per-project glossary file location and name (from #02)
- Target pane working directory for remote runtimes (from #02)
- Warning near Whisper's 224-token prompt limit (from #02)
- Default values and ranges for `recording.threshold` and `recording.max_chunk` (from #03)
- Existing-file handling and allowed characters in typed transcript names (from #03)
- Creating the transcript folder and reporting when it cannot be written (from #03)
- History mark for recorded-but-not-typed passages (from #03)
- `REC` in the `Mode` row or its own row (from #03)
- Timestamp format of the default transcript filename (from #03)
- Voice fingerprints vs separate audio sources, once pause-based recording has been used (from #04)
- Double-click window and same-cell tolerance (from #05)
- Whether plain-text (non-OSC 8) URLs also count as links (from #05)
- Persistent highlight vs output and scrolling (from #05)
- Whether mirror panes need the new selection ops (from #05)
- Trailing-trim character set (from #05)
- Double-click window and interaction with drag start and workspace switching (from #06)
- Inline field with long names and narrow sidebars (from #06)
- Whether to close the 256 KB reopen gap for chosen agent names (from #06)
- Whether the centered dialog stays for keyboard-only rename (from #06)
- Identity color variable name and value format (from #07)
- Dimmed vs focused color on both themes and light terminals (from #07)
- Focused colored row: full identity color or mix toward accent (from #07)
- Grouped rail with mixed agent colors in one workspace (from #07)
- Where the account color is set (settings UI, CLI) (from #07)
- Identity color for headless agents (from #07)
- Tint cap relative to the selected-row tint (from #08)
- Faking a light background in the sandbox (from #08)
- Tint in the framed sidebar and scope header row (from #08)
- Tint vs ghost row and glide animations (from #08)
- Tint setting name and storage format (from #08)

## Parked Ideas
- Voice glossary for domain vocabulary (#02)
  Reason: deferred until voice dictation (01) lands upstream
- Speaker separation by voice (#04)
  Reason: overkill for now; tinydiarize dropped (English only, one model), voice fingerprints or separate audio sources are the candidates if revisited
