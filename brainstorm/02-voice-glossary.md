# Brainstorm: Voice Glossary

**Date:** 2026-10-26
**Status:** parked

## Problem Framing

Whisper often mangles technical terms (library names, function names, project jargon). A glossary of global and per-project terms, passed to Whisper as its initial prompt, biases recognition toward that vocabulary. This was split out of the voice dictation brainstorm (01) to keep the first iteration small.

Success: project-specific names are transcribed correctly without the user correcting them by hand. Anti-goal: a glossary so long that it exceeds Whisper's prompt limit and silently drops the terms that matter.

## Approaches Considered

Not explored yet. Starting point: global terms in Corgi settings plus a per-project term file resolved from the target pane's working directory, merged with project terms last so truncation keeps them.

## Decision

Parked: deferred until voice dictation (01) has landed upstream.

## Key Requirements
- Builds on the voice dictation feature (01).
- Fully independent of cc-deck.

## Open Questions
- Where does the per-project term file live and what is it called?
- How is the target pane's working directory obtained, especially for remote runtimes?
- How to warn when the merged glossary approaches Whisper's 224-token prompt limit?
