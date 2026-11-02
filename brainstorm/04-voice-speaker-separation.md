# Brainstorm: Voice Speaker Separation

**Date:** 2026-10-26
**Status:** parked

## Problem Framing

When recording a conversation (brainstorm 03), start a new block whenever the speaker changes, even without a pause. The cc-deck relay does this with the tinydiarize model (`small.en-tdrz`) through `whisper-cli`, English only, falling back to pause-based separation when the model fails or is too slow. Split out of the voice dictation brainstorm (01).

Success: a two-person call produces one block per contribution in the reading view and the transcript file. Anti-goal: blocking or degrading dictation while the speaker model downloads or runs.

## Approaches Considered

Not explored yet.

## Decision

Parked: depends on transcript recording (03), which depends on voice dictation (01).

## Key Requirements
- Builds on transcript recording (03).
- Fully independent of cc-deck.
- Model download on demand with progress, without interrupting dictation.

## Open Questions
- Is tinydiarize still the best option, or is there a multilingual alternative by the time this is picked up?
- Per-passage time budget before falling back to pause-based separation.

---

## Revisit: 2026-11-02

### Updated Problem Framing
Revisited while planning the transcript port (03). Recording separates contributions by pause only (a gap longer than `pause_break`, or a leading dash from Whisper). Separating by voice is left out of that work.

### New Approaches Considered
- tinydiarize (`small.en-tdrz` via `whisper-cli`): dropped. English only and tied to one model, so it caps transcription quality and language; the user will not use it.
- Voice fingerprints (for example sherpa-onnx speaker embeddings): compare each passage's voice with earlier ones. Multilingual, but a new native dependency plus tuning of when two voices count as different.
- Separate audio sources: capture the microphone and the call audio on separate streams, so contributions split into "you" and "them". Very reliable for calls, but macOS needs a loopback device (such as BlackHole) for call audio.

### Updated Decision
Still parked: overkill for now. If revisited, the two ideas above replace tinydiarize as the candidates.

### Open Threads
- Whether fingerprints or separate audio sources fit better once pause-based recording (03) has been used for a while.
