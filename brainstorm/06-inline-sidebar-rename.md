# Brainstorm: Inline Sidebar Rename

**Date:** 2026-10-28
**Status:** active
**Issue:** https://github.com/tals/corgi/issues/42

## Problem Framing

Renaming a workspace or agent in the sidebar takes a right-click, "Rename", and a centered dialog.

The use case is tagging sessions. With several agents running in separate worktrees, the branch-derived names are noisy and look alike. In cc-deck, renaming a session to tag what it is for was one of the most common actions. That only works if renaming is quick and the tag survives while the agent keeps working.

The second half (names staying put) turned out to be covered already; see "How it works today".

The goal is to file an upstream issue asking whether people want this, and only build it if there is interest.

Success: double-click a row, type a tag, press Return, and the tag is still there an hour later and after a restart.

Failure even if it works: a typed name silently replaced by an automatic one; single-click select or row drag becoming laggy or unreliable because of double-click detection; unnamed rows losing their automatic names.

### How it works today

- Sidebar presses are handled in `src/controller.ts` (around 5301-5383). A left click on an agent row selects it and starts an `AgentDrag`; on a workspace row it switches workspace and starts a `WorkspaceDrag`. Right-click opens the row menu. There is no double-click detection anywhere in the codebase.
- "Rename" in the agent and workspace menus opens the centered `name_dialog` (`src/name_dialog.ts`, `View.init`; drawn in `src/render.ts`). There is no in-row editing yet. The closest precedent is the agent search field, which is drawn inside the sidebar.
- Agent row text is `p.label`, then `p.agent.name`, then the provider name (`src/controller.ts:903`, `src/sidebar.ts:1294`). `agent.name` is written by provider transcript fallbacks (`src/providers/{claude,codex,pi}/transcript.ts`), child metadata, resume/fork hints and the auto-namer (`src/conversation_name.ts`, only while the name is empty).
- A user rename goes through `agent.rename` to `renamePane` (`src/agent_rename.ts`). It sends the name to the provider and calls `setName`; there is one name, shared with the session, and `p.label` is cleared.
- Verified: for Claude the rename becomes a `custom-title` transcript entry, which outranks later `ai-title` entries (`src/providers/claude/usage.ts:131-142`, test in `src/usage/worker.test.ts:58`). Claude does retitle sessions on its own (about 12% of local sessions have 2+ distinct `ai-title`s), but none of the 12 locally renamed sessions got an `ai-title` after the rename. Theoretical gap: on reopen Corgi reads only the last 256 KB (`src/usage/worker.ts:149`), so a `custom-title` further back is not seen and a later `ai-title` in the window would win.
- Without a rename channel, the rename is typed into the prompt as `/rename` and fails with `AgentBusy` while the prompt is not empty.
- `Workspace.name()` returns the custom name, else `basename(cwd)` (`src/workspace.ts:34`). `follow(cwd)` is driven by `Server.followWorkspaceDirectories` (`src/server.ts`). Typed workspace names are already stable.

## Approaches Considered

### A: Inline rename, and the typed name wins
Double-click or menu "Rename" edits the row in place. A typed agent name is marked as user-chosen, and automatic sources skip it. The row still shows one name.
- Pros: smallest change that delivers both goals; matches how renaming worked in cc-deck.
- Cons: the provider's own title is hidden while a typed name is set.

### B: Separate tag next to the automatic name
The typed text goes into its own tag field. The row shows the tag first, and the automatic name stays visible as a secondary line or tooltip.
- Pros: nothing is overwritten; both names stay visible.
- Cons: new concept, more row space in a dense sidebar, open question whether the tag goes to the provider.

### C: Inline editing only, no pinning
Same editing experience, but provider titles can still replace the name.
- Pros: cheapest.
- Cons: fails the main goal; a tag that silently disappears is worse than none.

## Decision

Approach A, narrowed after verification: the "typed name wins" part already holds, so the issue proposes only double-click and inline editing. B can follow later if people want to see both names.

## Key Requirements

- Double-click on a workspace row or an agent row starts an inline rename of that row.
- "Rename" in the workspace and agent menus starts the same inline rename instead of the centered dialog.
- The row text becomes an editable field with a cursor, prefilled with the current name. Return saves, Escape cancels.
- Single-click select and row drag keep working as they do now, with no added delay: the first click acts immediately, and only a second click on the same item (same workspace or agent id, not the same screen position) within the double-click interval and without movement starts the rename.
- Renaming a busy agent (`AgentBusy`) is shown clearly in the inline field rather than silently dropping the name.
- Proposed (my call): clicking elsewhere cancels rather than saving, so a half-typed name is never committed by accident.
- Keep sending a typed agent name to the provider, as today.

## Open Questions

- Double-click window and how it interacts with drag start and workspace switching on the first click.
- How the inline field fits long names and narrow sidebars (scrolling vs clipping).
- Whether to close the 256 KB reopen gap (remember a chosen name across reopen) as a separate fix.
- Whether the centered dialog stays for keyboard-only rename (`rename-tab` and similar actions).
