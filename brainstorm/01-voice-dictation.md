# Brainstorm: Voice Dictation

**Date:** 2026-10-26
**Status:** active

## Problem Framing

Bring voice dictation to Corgi as an upstream feature: speak, have a local Whisper model transcribe the speech, and have the text typed into the pane the user is working in. The feature set follows the voice relay that the author built for cc-deck, more or less 1:1, but the implementation is written fresh in TypeScript for Corgi and follows Corgi's architecture, conventions and review expectations (Corgi is maintained by Tal Shiri; this lands as a contribution).

This first iteration combines two slices of the cc-deck relay: the dictation core and its setup/controls. Glossary support, transcript recording with the reading view, and speaker separation are separate brainstorms (02, 03, 04).

Success: a user on macOS or Linux can run one setup command, start voice, and dictate into Claude Code, Codex or any other pane, whether Corgi runs locally or on a remote host via `ssh -t host corgi`.

Anti-goals:
- Any dependency on cc-deck (code, binaries, config files, model cache, pipe protocol). The implementation must be fully independent.
- Capturing audio anywhere other than the machine that has the microphone.
- A large viewer UI change that makes the upstream review hard.
- Phantom text (Whisper hallucinations such as "Thank you.") reaching agent prompts.

## Approaches Considered

### A: Standalone `corgi voice` app on the microphone machine (chosen)
The same `corgi` binary gains a voice mode that runs in a local terminal (or a local Corgi pane). It owns audio capture, voice activity detection, the managed whisper-server, and the voice UI (level meter, threshold, device picker, transcript history). It talks to the runtime directly when Corgi is local; for a remote runtime, `corgi voice --ssh host` opens a small bridge over SSH. The daemon gains a few voice commands (deliver text, jump to attention, mute state); the viewer gains a sidebar indicator and a mute shortcut.
- Pros: closest to the cc-deck experience; local and remote work the same way; small viewer footprint, easier upstream review.
- Cons: voice UI lives in a separate window instead of inside Corgi.

### C: Headless capture helper, UI inside Corgi
A helper on the microphone machine only captures and transcribes; meter, history and threshold controls render as a Corgi panel fed through the daemon.
- Pros: one place to look.
- Cons: remote still needs a separately started helper, so setup is not simpler; much more viewer UI code to review; audio levels stream through the daemon (and SSH) several times per second.
- Could follow later as a UI-only improvement on top of A's daemon commands.

### Dropped: Capture inside the viewer
Fails the remote requirement: with `ssh -t host corgi`, daemon and viewer both run on the remote host and cannot reach the local microphone.

### Dropped: Sidecar using only the existing CLI
Uses `corgi list`, `corgi pane send` and `corgi focus` with no Corgi change. No mute indicator or mute sync inside Corgi, and each utterance over SSH would pay a fresh SSH round-trip.

## Decision

Approach A: a standalone `corgi voice` mode on the microphone machine, with a small set of daemon voice commands and a sidebar indicator plus mute shortcut in the viewer.

## Key Requirements

Dictation core:
- Continuous listening with voice activity detection; an utterance ends after a configurable silence (default 2.5 s) and is capped at 30 s (Whisper's window).
- Local transcription through a managed `whisper-server` (started and stopped by `corgi voice`), default model `base.en`, other Whisper models selectable.
- Speech filter (Silero VAD model) on by default, so non-speech audio is skipped instead of transcribed; a reminder when it is not installed; can be turned off in settings.
- Filler words (um, uh, hmm, ah, er) are stripped.
- Text is typed into the focused pane of the target viewer.
- Command words fire only when the whole utterance (after filler stripping) equals the word: "send" submits (Enter), "next" jumps to the next agent needing attention (Corgi's existing Jump to attention). Trigger words per action are configurable; configured lists replace the defaults for that action.
- Mute: toggled from the voice UI and from inside Corgi (shortcut and clicking the sidebar indicator); state stays in sync both ways.
- Sidebar indicator inside Corgi shows voice connected/listening, connected/muted, or absent.
- Every delivery is bounded in time, so a stuck delivery cannot stall detection.
- Works when Corgi runs locally and when it runs remotely via SSH; audio is always captured on the machine with the microphone.

Setup and controls:
- A setup command checks for `whisper-server`, downloads the chosen model and the speech filter model, and reports readiness.
- Voice UI shows target, audio device, mode (listening/muted), a level meter with threshold marker, and a scrollable history with delivery status (delivered, failed, transcribing).
- Threshold adjustable with `+`/`-` and with the mouse wheel over the level meter.
- Audio device picker.
- Settings for threshold, silence, model, port, speech filter and command words; CLI flags override settings for one run.
- Optional verbose diagnostic log.

Constraints (user-owned):
- Upstream into Corgi; play nicely with its ecosystem (TypeScript/Bun, small native boundary, existing CLI and daemon command style, docs and tests).
- Fully independent of cc-deck.
- Feature parity with the cc-deck relay for this slice, more or less 1:1.

Proposals (my call, accepted in playback):
- whisper.cpp stays an external dependency (`brew install whisper-cpp` or distro package), not bundled; matches how Corgi already shells out to `afplay`/`paplay` and keeps the release small.
- Settings live in Corgi's own settings, not a separate config file.
- Platforms: the ones Corgi supports today (macOS Apple Silicon, Linux amd64).
- With several viewers attached, text goes to the focused pane of the viewer that last received input.
- Audio capture through an external tool subprocess (for example ffmpeg) rather than a new native audio library, to keep the native boundary small. To be confirmed in planning.

## Open Questions
- How does `corgi voice --ssh host` reach the remote daemon: OpenSSH Unix socket forwarding, or a `corgi` bridge command on the remote side speaking over the SSH session's stdio?
- How does the voice app identify its target viewer, especially remotely where the viewer runs on the remote host?
- Which capture tool(s) on macOS and Linux, and how is the device list obtained for the picker?
- Where are models cached (a Corgi-owned path under the user's cache directory)?
- Which Corgi shortcut toggles mute in each shortcut preset, and how does the indicator fit the sidebar style guide (`docs/ui_styleguide.md`)?
- Does "next" need a new daemon/CLI command for Jump to attention, or can it reuse the existing viewer action?
- What is the minimum Corgi wire/protocol version change, and how do older viewers behave when a voice app connects?
