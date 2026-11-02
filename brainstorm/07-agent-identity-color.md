# Brainstorm: Agent Identity Color

**Date:** 2026-10-28
**Status:** active
**Issue:** https://github.com/tals/corgi/issues/44

## Problem Framing

The green bar on the left of an agent row in the sidebar is Corgi's `accent` color (`src/palette.ts:13`) and marks the focused agent. It is drawn only in the flat list (`src/sidebar.ts:1399`); in the grouped list the workspace rail takes that role, turning heavy and accented (`┣┃`) beside the focused agent. While a row is dragged it becomes a "ghost" bar in `strong`. So the bar means focus, not identity.

The user runs the same harness in several setups, for example Claude Code via a private subscription and via a company Vertex account. Each setup has its own config folder and so its own billing and models. Telling these agents apart in the sidebar takes reading. cc-deck colors its rows per setup, and the user wants the same here: a permanent, configurable color per setup, visible in both the flat and the grouped list.

The user is also building a tool that creates aliases and start scripts for such setups. Corgi should honor an environment variable such a script sets, so the tool needs no Corgi-specific configuration beyond that variable.

Success: a glance at the sidebar shows which setup every agent runs on, in both views, and focus stays clearly visible.

Failure even if it works technically:
- Focus becomes hard to spot once rows carry colors.
- The color works in only one of the two views.
- The color is lost when an agent is restarted or resumed.
- Setting the color requires per-launch manual steps instead of being part of an alias or start script.

## Approaches Considered

### A: Left edge, identity color dimmed, focus as brightness (chosen)
- Flat list: every colored row shows a dimmed `▌` in column 1 in its identity color. The focused row shows the same color at full brightness. Uncolored rows behave as today (green bar on focus only).
- Grouped list: the agent's stretch of the workspace rail (`├│` beside its two lines) takes its identity color, dimmed. On focus the rail goes heavy (`┣┃`) in the full color, the same shape that marks focus today in green.
- Pros: one location in both views, no text columns lost, matches what the user knows from cc-deck.
- Cons: a colored agent loses the green focus cue; focus is carried by brightness, the heavy rail, and the selected row background. In grouped view the rail is no longer one solid color per workspace.

### B: Right border segment
- The sidebar border beside an agent's two lines takes its color.
- Pros: identical in both views; leaves focus and rail untouched.
- Cons: the border already carries the scrollbar thumb and the focused list's outline tint; colored segments would get lost or clash.

### C: Tinted agent name
- Pros: no layout change.
- Cons: name styling already signals unread (bold, `strong`); colored text reads poorly on some themes; short names give little color.

### Dropped early
- Tinting the status icon (`○ ▶ ●`): it already uses color for agent state, so it would mix two meanings.

## Decision

Approach A. The bar stays where the user already looks, and one rule (dim = identity, bright/heavy = focus) holds in both views.

## Key Requirements

- **Opt-in:** a setting turns the feature on; off by default, so nothing changes for users who don't enable it. Requested by the user.
- **Identity unit:** the account (provider plus config folder, as in `src/accounts.ts`), which Corgi already detects from `CLAUDE_CONFIG_DIR` and similar variables in each agent's environment. Confirmed by the user: their setups use separate config folders.
- **Color sources, in priority order:**
  1. An environment variable on the agent process (name to be decided, placeholder `CORGI_AGENT_COLOR`), set by an alias or start script. Required by the user.
  2. A color saved on the account in Corgi's account settings (`accounts.json`). Proposed by Claude, not objected to: configure once per setup instead of per launch.
  3. None: no identity indicator, current behavior.
- **Start scripts must work:** Corgi already reads the environment of agents started in a pane's shell (`Launch.observed`, `src/runtime.ts:164`). The variable must be picked up whether the script `exec`s the agent or runs it as a child.
- **Survives restart and resume:** the variable joins the set Corgi keeps from an observed launch, so a restarted agent keeps its color. Proposed by Claude, with the reason that otherwise the color disappears after a restart.
- **Both views:** flat list uses a dimmed `▌` in column 1; grouped list tints the agent's rail segment. Focus shows as full brightness (flat) or heavy rail in full color (grouped).
- **Uncolored agents:** unchanged from today.
- **Dragging:** the ghost bar keeps its current look (Claude's call, keeps drag feedback consistent).

## Open Questions

- Variable name and accepted value format (hex `#rrggbb`, named colors, palette tokens, or a mix).
- How "dimmed" is computed so it stays visible but distinct from the focused state on both the Corgi and Animal Style themes and on light terminals.
- Whether focus on a colored row in the flat list should stay the identity color at full brightness or briefly mix toward `accent`.
- Grouped view: how the rail reads when neighboring agents in one workspace have different colors, and whether the workspace header keeps a neutral rail.
- Where the account color is set in the UI (account settings row, CLI, or both).
- Whether headless agents (no pane) should take the color from their launch environment the same way.
