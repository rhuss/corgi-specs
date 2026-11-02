# Brainstorm: Voice Transcript Recording

**Date:** 2026-10-26
**Status:** active

## Problem Framing

Record everything said during a voice session to a plain text file, for notes, meetings or commentary, and read it live in a reading view that groups the conversation into blocks per contribution. Recording continues while dictation is muted, so the user can capture speech that should not reach an agent. Split out of the voice dictation brainstorm (01).

Success: start, pause, resume and stop a recording from the voice UI; the file contains one line per utterance with blank lines between contributions; the reading view follows new speech. Anti-goal: recording that delivers text to agents while muted.

## Approaches Considered

Not explored yet. The cc-deck relay offers: `r` start/pause/resume, `R` stop, `v` reading view, a filename prompt with a timestamped default, recording-specific VAD settings (shorter silence, pause break between contributions), and separation of contributions by pause.

## Decision

Parked: deferred until voice dictation (01) has landed upstream.

## Key Requirements
- Builds on the voice dictation feature (01).
- Fully independent of cc-deck (own storage location under Corgi's data directory).

## Open Questions
- Default location for transcript files in Corgi's layout.
- Which recording-specific VAD settings are worth exposing.
- Whether the reading view belongs in `corgi voice` only or could later show inside Corgi.

---

## Revisit: 2026-11-02

### Updated Problem Framing

Port the rest of cc-deck's transcription feature to Corgi's `corgi voice`, staying close to cc-deck: recording to a text file, the reading view, recording-specific timing, and separating contributions by pause. The work goes on its own branch and worktree, stacked on `001-voice-dictation` ([#36](https://github.com/tals/corgi/pull/36)), so it can follow that PR upstream.

Two things differ from cc-deck. The transcript folder is a setting, and the UI goes where it fits best in Corgi, which turned out to be the `corgi voice` window, as in cc-deck.

Success: during a call or while taking notes, the user records with `r`, watches the conversation in the reading view, keeps dictating into agents (or mutes dictation and keeps recording), and ends up with a plain text file on the microphone machine that the meeting-notes workflow can read.

Anti-goals:
- Text reaching an agent pane while dictation is muted.
- Transcripts written on the remote runtime host.
- Viewer or daemon changes that grow the stacked PR's review surface.
- Any dependency on cc-deck (code, config, model cache, paths).

### New Approaches Considered

#### A: One audio pipeline whose timing changes while recording (chosen)
Recording becomes a second destination for the passages voice mode already transcribes: every passage goes to the file, and to the pane unless muted. While recording, the segmenter uses the `recording` timing.
- Pros: cc-deck's behaviour 1:1; one Whisper pass per passage; the new code is mostly the file writer, the turn-break logic and the reading view, so the diff on top of #36 stays small.
- Cons: dictation during a recording arrives in shorter pieces. Corgi already joins consecutive pieces in the same pane with a space, so this mostly shows up as text appearing sooner.

#### B: Two segmenters on the same microphone stream
Dictation keeps its 2.5 s timing, and recording gets its own 1 s timing.
- Pros: dictation behaves exactly the same whether or not a recording runs.
- Cons: Whisper transcribes most speech twice, a real CPU cost with `small.en` or larger that slows dictation during calls; two history streams to keep in step.

#### C: A separate record-only mode (`corgi voice record`)
- Pros: simplest code; the dictation path stays untouched.
- Cons: no dictating and recording in one session; recording while muted loses its point; drifts from cc-deck.

#### Dropped: Speaker separation by voice (tinydiarize via `whisper-cli`)
English only and tied to one model (`small.en-tdrz`), so it caps the transcription model and language. The user will not use it. Separating by pause covers the needed cases.

#### Dropped: Transcript folder in Corgi setup
Corgi setup saves to the runtime host's `settings.json`/`viewer.json`. With `corgi voice --ssh host`, the folder would be stored on the remote host while transcripts are written on the microphone machine, which reads only its local `voice.json`.

### Updated Decision

Approach A, with the reading view inside the `corgi voice` window, transcripts always written on the microphone machine, and speakers separated by pause only.

### Key Requirements

Recording (user-stated, from cc-deck):
- `r` starts a recording, pauses it, and resumes it; `R` stops it and closes the file. Quitting also closes the file cleanly.
- `r` opens a filename prompt with a timestamped default (`transcript-<timestamp>.txt`); Enter accepts, Esc cancels.
- A red `REC` shows while recording and a pause symbol while paused.
- While paused, passages still appear in the history but are not written.
- While dictation is muted and a recording is active, speech is transcribed and written to the file but never typed into a pane. While muted with no recording, audio is discarded as today.
- File format: one line per passage, no timestamps or metadata; a single blank line before each passage that starts a new contribution; never a blank first line.

Transcript location (user-stated):
- Always on the machine running `corgi voice`, never on the runtime host.
- Folder set by `transcripts` in `voice.json` and the `--transcripts DIR` flag (flag overrides the file for one run). Relative filenames go into the folder; absolute paths are used as given.
- Default `$XDG_DATA_HOME/corgi/transcripts/`, falling back to `~/.local/share/corgi/transcripts/` (my call: transcripts are user data, not state, and cc-deck used the same XDG data location).

Recording timing (from cc-deck):
- A `recording` group in `voice.json` with `threshold`, `silence` (default 1 s), `max_chunk`, and `pause_break` (default 2 s), each validated per field like the existing settings.
- These values apply to the whole pipeline while a recording runs, so dictation also uses them then.
- The pause break is measured from the last loud audio of one passage to the first loud audio of the next, so `pre_roll` does not affect it.

Contribution separation (user-stated: by pause only):
- A new contribution starts after a gap longer than `pause_break`, or where Whisper marks a speaker change with a leading dash.
- No `s` key and no speaker-mode choice.

Reading view (user-stated: in the `corgi voice` window, like cc-deck):
- `v` opens it while recording or paused; `v` or Esc closes it.
- One block per contribution: start time, then the text; the gutter alternates between two theme colors.
- The level meter and `T:` stay visible, and `+`/`-` still adjust the threshold.
- The view follows new speech; scrolling up pauses following and the footer counts new blocks below; jumping to the newest resumes following.

My proposals (accepted in playback):
- Reading-view scroll keys match the voice window's history keys (`j`/`k`, `↓`/`↑`, `Ctrl+D`/`Ctrl+U`, `PgDn`/`PgUp`, `g`/`G`) plus `End`, instead of cc-deck's set.
- Passages that were recorded but not typed (muted) get their own history mark.
- No viewer or daemon changes; the Corgi sidebar indicator stays as in #36.
- Docs: extend `docs/voice.md` and the settings tables in `docs/reference.md`.

Constraints:
- Stacked on `001-voice-dictation` (#36), in its own branch and worktree.
- Fully independent of cc-deck.
- Follows #36's patterns: per-field `voice.json` validation with notices, CLI flags overriding settings, theme colors, footer key hints.

### Open Threads
- Default values and valid ranges for `recording.threshold` and `recording.max_chunk`.
- Filename handling: what happens when the file already exists (append, refuse, or suffix), and which characters a typed name may contain.
- Whether the folder is created on first recording, and what the UI shows when it cannot be written.
- How the history mark for recorded-but-not-typed passages looks, per `docs/ui_styleguide.md`.
- Whether the `Mode` row is the right place for `REC`, or whether it needs its own row in the voice table.
- Whether the timestamp in the default filename uses local time (cc-deck) and which format.
