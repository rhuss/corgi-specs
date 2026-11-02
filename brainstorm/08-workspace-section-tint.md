# Brainstorm: Workspace Section Tint

**Date:** 2026-11-02
**Status:** active

## Problem Framing

The sidebar paints one background under both lists. The only thing that separates the workspace list from the agent list is the heavy split rule with its drag grip (`src/sidebar.ts`, `drawFrame`). The rule is clearly visible, but it is not enough to tell at a glance which part is workspaces and which is agents.

The user wants an optional tint on the workspace section's background. A slider in setup controls its strength, so each user can pick how much contrast they want, or none at all.

**Success:** with the slider turned up, the workspace section reads as its own area at a glance, on dark and light backgrounds alike. The active, selected and hovered workspace rows stay easy to spot.

**Non-goals:**
- Tinting the agent section.
- Letting the user choose a tint color or direction.
- Per-workspace colors (agent identity colors are a separate thread, see #07).

**Anti-goals (failure even if it works):**
- The tint swallows the selected or hover tint, so the active workspace gets harder to find.
- Users who never touch the slider see any change.
- On some themes the tint goes the "wrong" way (darker on dark, lighter on light).

## Approaches Considered

### A: Blend the background toward the theme's text color (chosen)
The slider sets how far the workspace section's background moves from the background color toward the text color, capped below the selected-row tint.
- Pros: the direction always comes out right (lighter on dark, darker on light), including for terminal themes Corgi has never seen. The cap ("never as strong as the selected row") is easy to state and test.
- Cons: the tint is a computed shade, not one a theme designer picked.

### B: Blend toward the palette's `surface` color
- Pros: reuses an existing palette color, so it feels native in Corgi's own themes.
- Cons: with a terminal's own theme, `surface` is not guaranteed to be lighter on dark or darker on light, which breaks the user's rule. The search box, which uses `surface`, would blend into the tinted section.

### Dropped
- **Fixed steps (low, medium, high):** the user asked for a slider.
- **Tinting the agent section instead:** the workspace list is the smaller, calmer area to mark.
- **User-chosen direction (lighter or darker):** the user wants the direction to be automatic.

## Decision

Approach A: blend the workspace section's background toward the text color. A single setup slider sets the strength, from Off up to a cap below the selected-row tint. The direction follows the background automatically.

## Key Requirements

Stated by the user:
- A slider in setup sets the strength of the workspace section tint.
- The slider goes in one direction only: more or less contrast.
- The tint is optional and can be switched off completely.
- The direction is automatic: lighter on dark backgrounds, darker on light ones.
- Testing support through a `./go.ts --sandbox` world.

Proposed by Claude and accepted in playback (the reasons are in parentheses):
- **Off state:** at 0 the slider reads "Off" and the sidebar looks exactly as it does today (this matches the corgi brightness slider).
- **Default:** Off (the user wants everything optional, and existing users should see no change until they turn it up).
- **Placement:** Visual flourishes page, under the corgi brightness slider, with the same keys, mouse dragging and percentage label (it is setup's only slider and already behaves this way).
- **Tinted area:** every row of the workspace list from the top down to the split rule. The rule and the agent list keep today's background, so the rule becomes the edge where the two colors meet.
- **Selected, active and hover rows:** they stay noticeably different from the tinted section at every slider position. The maximum stops short of the selected tint.
- **Workspaces drawer** (the list fills the whole sidebar): it uses the same tint, so the section looks the same everywhere.
- **Live preview:** the real sidebar updates while the slider is dragged, so no separate preview image is needed.
- **Sandbox world:** a new `tools/sandboxes/sidebar.ts`. It has enough workspaces to fill the section (one active, one with ports, readings row on) and several fake agents in different states below the split. Its intro says to open setup, go to Visual flourishes, drag the slider, hover a workspace, and compare the active row against the tint at the maximum. Screenshots for the PR come from `bun tools/ghostty_screenshot.ts --sandbox sidebar`.

## Open Questions
- Tint cap: the exact maximum relative to the selected-row tint, and whether the slider's 100% maps to that cap.
- Whether the sandbox can fake a light background to test "darker on light", or whether that test needs a light terminal profile.
- Whether the tint also covers the framed (boxed) sidebar layout and the scope header row.
- How the tint interacts with the dragged-workspace "ghost" row and the row glide or tint animations.
- Setting name and storage format in the viewer preferences.
