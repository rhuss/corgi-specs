# Idea Inbox

Ideas captured from code reviews for future brainstorming.

### workspace-hover-under-overlays

- **Source**: deep-review
- **Date**: 2026-11-02
- **Reference**: 004-workspace-section-tint
- **Summary**: Workspace row hover shares the agent-hover gate, so it is off while setup is open, while the sidebar has keyboard focus and in the workspaces drawer.

> FR-018 says "at every slider value"; the drawer/focus cases follow existing agent behavior but are unstated, and the sidebar sandbox intro does not say to close setup before hovering.

### droppable-setup-lines

- **Source**: deep-review
- **Date**: 2026-11-02
- **Reference**: 004-workspace-section-tint
- **Summary**: Visual flourishes' short-frame fitting is list surgery tied to exact line order; a droppable marker on preview lines would make "the preview gives way first" generic.

> Reordering the page or adding a slider silently changes what is dropped; a duplication bug was already found here.

### golden-hash-baseline-drift

- **Source**: deep-review
- **Date**: 2026-11-02
- **Reference**: 004-workspace-section-tint
- **Summary**: SC-001 golden SHA-256 hashes match main today but lose their baseline meaning once re-recorded after an unrelated sidebar change.

> A self-contained "tint off equals no-tint path" assertion, or documented regeneration from main, would keep the check honest.
