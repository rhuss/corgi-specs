# Brainstorm: Pane Click Selection

**Date:** 2026-10-28
**Status:** active
**Issue:** https://github.com/tals/corgi/issues/21

## Problem Framing

In a Corgi terminal pane, double-clicking a word does nothing. Ghostty, most other terminals, Claude Code and Codex select the word on double-click and the line on triple-click. That is the quickest way to grab a file path, commit hash, URL or identifier without dragging exactly or entering copy mode (`prefix s`, `v`, `y`).

The maintainer (tals) wants the same behavior and asked for it to be run as a daily-driver fork first, paying attention to wrapped lines and comparing against Ghostty, before opening a PR ([comment](https://github.com/tals/corgi/issues/21#issuecomment-6419079180)).

Success: double- and triple-click feel like Ghostty in everyday use, while URLs and `path:line` references come out whole.

Failure even if it works: link clicks become unreliable, mouse-aware apps (vim, htop) lose clicks they expect, or wrapped lines select differently from Ghostty.

### How it works today

- Corgi receives raw SGR press/release events from the host terminal, so it has to count clicks itself (`src/controller.ts`, pane mouse handling around the `app_mouse` check).
- Drag selection calls the native rectangle `select` op, copies on release via OSC 52 (`copyText`, which also shows the "copied" toast), then clears the highlight.
- Shift already bypasses app mouse reporting for drag selection (`(button & 4)`).
- Plain clicks on http(s) OSC 8 links open on release (`port_click`).
- The bundled Ghostty library (`native/build.zig.zon`) has `Screen.selectWord`, `selectWordBetween` and `selectLine`, which follow soft wraps, handle wide cells and stop at semantic prompt boundaries. The native bridge (`native/app_bridge.zig`, `native/zig/pane/terminal.zig`) only exposes rectangle `select` today.
- Ghostty's default word boundaries are `` \t '"│`|:;,()[]{}<>$ ``; the `:` splits URLs and `path:line`.

## Approaches Considered

### A: Ghostty selection through the native bridge
New native ops for "word at cell", "line at cell" and their drag-extending variants, backed by Ghostty's `selectWord` / `selectWordBetween` / `selectLine` with Corgi's boundary set and trailing-punctuation trim. The controller does click counting, deferred link open, and gesture state.
- Pros: wrapping, wide cells and prompt boundaries already handled and tested upstream; behavior stays close to Ghostty, which keeps the fork comparison meaningful.
- Cons: Zig changes and native rebuild for both SDKs; tests need a real terminal.

### B: Word boundaries computed in TypeScript
Read row text through existing text ops, compute boundaries in TS, then call the rectangle `select`.
- Pros: no native change; easy unit tests.
- Cons: re-derives wrapped-row, wide-cell and column mapping; no prompt boundary for triple-click; drifts from Ghostty exactly where the trial compares.

### Dropped: let the host terminal handle multi-click
Corgi must own the mouse for panes and drag selection, so the host never sees the clicks.

## Decision

Approach A: expose Ghostty's word and line selection through the native bridge, with click counting and link deferral in the controller.

## Key Requirements

- **Double-click** selects the word under the pointer and copies it to the clipboard, with the existing "copied" toast.
- **Triple-click** selects the whole logical line, following soft wraps, stopping at shell prompt boundaries (as Ghostty does), and copies it.
- **Word boundaries:** Ghostty's default set without `:`, so `https://x.com/a`, `terminal.ts:42` and `localhost:8080` are selected whole (user requirement). *(Proposed: a trailing `.` or `:` is trimmed, so "see `src/a.ts`." drops the period.)*
- **Links:** a single click on a link opens it after the double-click window (about 300 ms); a second click within the window cancels the open and selects the URL. The user accepted the added latency on single link clicks.
- **Mouse-reporting apps** keep receiving plain clicks. Shift+double-click and Shift+triple-click select anyway, consistent with Shift+drag today.
- *(Proposed)* Dragging after a double- or triple-click extends the selection by words or lines, as in Ghostty.
- *(Proposed)* The double/triple-click highlight stays until the next click or keypress. Drag selection keeps its current clear-on-release behavior.
- **Constraint:** build on the bundled Ghostty library's selection functions rather than reimplementing boundary logic.
- **Out of scope** *(proposed)*: the sign-in overlay's own selection, copy mode, configurable boundary characters.
- **Trial:** run as a fork for a few days before a PR; check wrapped lines, long URLs and `path:line` against Ghostty.

## Open Questions

- Exact double-click window, and whether the click count must also require the same cell (or a small tolerance).
- Whether URL detection also covers plain-text URLs that are not OSC 8 hyperlinks (today only OSC 8 http(s) links open on click).
- How the persistent highlight interacts with output arriving or the viewport scrolling (clear on scroll, or keep).
- Whether mirror panes (`src/mirror.ts`) need the new ops as well.
- Trailing-trim set: only `.` and `:`, or also `!`, `?`, `'`.
