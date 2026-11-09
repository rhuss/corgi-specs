# Brainstorm: aprofile Registry Support

**Date:** 2026-11-09
**Status:** active

Companion brainstorm on the aprofile side: `aprofile/brainstorm/003-corgi-integration.md`
(self-contained profile folders, registry `model` field). Related here: 07-agent-identity-color
(issue [#44](https://github.com/tals/corgi/issues/44)), 09-custom-build.

## Problem Framing

aprofile ("agent profile") manages side-by-side setups of coding harnesses. Each profile has its own
config folder (`CLAUDE_CONFIG_DIR` or `CODEX_HOME`), an exec-style wrapper script that sets the
endpoint, key, model and model aliases, and an entry in a machine-readable registry at
`~/.local/share/aprofile/registry.json` (`format_version` 1). Each entry carries `harness`, `name`,
`label`, `color`, `icon`, `auth_kind`, `primary`, `config_dir`, `wrapper_path`, `share`,
`history_group`, and (added by the aprofile side) `model`.

The author runs ten such profiles daily and wants Corgi's picker to list exactly those, each
launching correctly, showing its model and its color.

Today Corgi can list a profile folder (`corgi account add … --dir`), but launches it by setting only
the config-folder variable. For an API-key profile that misses the key and endpoint, unless the
folder is self-contained (which the aprofile side adds). Saved custom commands launch through a
shell but are capped at six.

Corgi already reads one external account list: Omarchy's
(`~/.local/state/omarchy/agents/accounts/<provider>.json`, `src/accounts.ts:177`), read only, never
written. aprofile's registry is the same kind of source.

Strategy: offer this as a patch to Tal. If it is not accepted, the author carries it in a custom
build (09-custom-build). The patch should therefore be small, follow the Omarchy precedent, and
touch as little else as possible.

Success: with the patch, the picker shows each aprofile profile under its label, launches it
through its wrapper, shows the profile's model, and its agents carry the profile's color.

Anti-goals:

- Corgi writes into aprofile's files.
- Duplicate rows: a registry profile also listed as a recorded or observed account.
- A patch so large or aprofile-specific that it cannot be maintained in a custom build.

## Approaches Considered

### A: Registry reader beside the Omarchy reader, wrapper launch, model, color via a Trick (chosen)
- Discovery: read `registry.json` (respecting `XDG_DATA_HOME`), add each profile of a supported
  harness as an account with `config_dir` as its folder and `label` as its name, like `omarchy()`.
  Missing or malformed file means no accounts; unknown `format_version` is ignored.
- Launch: an account that came from the registry starts through its `wrapper_path` instead of the
  bare binary with the config-folder variable. The wrapper is exec-style, so process detection
  still sees the real harness, and usage and plan still read from the folder.
- Model: shown from the registry's `model` when present, otherwise Corgi's current lookup.
- Color: a Trick on top of [#74](https://github.com/tals/corgi/pull/74) maps
  `agent.account.directory` to the registry's `config_dir` and draws `color` on the rail marker.
- Pros: small, mirrors existing code; color needs no new core UI.
- Cons: depends on #74 (a draft with conflicts) for color; the launch path gains a second way to
  start an account.

### B: Generic external account sources
- Corgi defines a neutral account-source format or plugin point; Omarchy and aprofile become two
  sources.
- Pros: more general, maybe easier to argue upstream.
- Cons: larger change, refactors the Omarchy reader, harder to carry in a custom build. aprofile
  does not want to write a Corgi-defined file (that was its Omarchy export, now removed).

### C: Color in core instead of a Trick
- Corgi reads `color` and draws it itself (as 07-agent-identity-color planned).
- Pros: no dependency on Tricks.
- Cons: overlaps with #74, which already implements #44; likely rejected upstream.

## Decision

Approach A: discovery, wrapper launch and model in the core patch, color through a Trick on #74.

## Key Requirements

- Read-only access to `registry.json`; never write it.
- Registry profiles appear under their `label`, deduplicated against accounts Corgi already knows
  by canonical folder (the same rule `known()` applies today).
- Launch through `wrapper_path`; resume of a conversation from such an account also goes through
  the wrapper.
- Model from the registry's `model`; fall back to the existing lookup.
- Color from the registry's `color`, drawn by a Trick through #74's account marker.
- `corgi account remove` (forget) works for registry accounts as it does for Omarchy's.
- The patch is self-contained enough to rebase in a custom build.

## Open Questions

- Does the launcher pass extra arguments (resume IDs, YOLO flags) through the wrapper unchanged?
  The aprofile wrapper forwards `"$@"`, so it should.
- Corgi's "Sign in" flow for a signed-out registry account: run the wrapper's login or Corgi's own?
- Should a registry account still set the config-folder variable itself (harmless, the wrapper sets
  it too) for code paths that read it before launch?
- Headless agents (`corgi headless start … --account DIR`): launch through the wrapper too?
- How to pitch it to Tal: aprofile-specific reader like Omarchy's, or framed as "external account
  registries" with aprofile as the first?
- Does the color Trick ship as an example in Corgi or in aprofile's repository?
