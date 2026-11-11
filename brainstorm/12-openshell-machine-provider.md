# Brainstorm: OpenShell Machines

**Date:** 2026-11-11
**Status:** active
**Proposed issue title:** feat: support OpenShell sandboxes as Corgi machines

## Problem Framing

This document is both the brainstorm record and the draft body for an upstream issue.

**Maintainer question:** @tals, before I spend time on an implementation, would you be interested in this as a contribution?

I would be happy to take it on if the direction fits Corgi. I am raising it as an issue first because it changes the remote-machine transport boundary, the saved machine catalog, and the relationship between Corgi and an external execution-control plane. If you would rather keep remote machines SSH-only for now, I will leave this as a design note.

OpenShell is a policy-enforced runtime for autonomous agents. It provides sandbox lifecycle, isolation policy, credential injection, provider management, and a gateway for local or remote sandboxes. Corgi already has remote machines over SSH: a daemon owns each machine's workspaces, panes, agents, terminal state, and reconnects, while one local viewer brings those machines together.

The fit is to keep that division of responsibility:

- OpenShell owns sandbox provisioning, policy, credentials, and lifecycle.
- Corgi owns the persistent terminal viewer, workspaces, panes, agent UX, and reconnecting sessions.
- Each OpenShell sandbox runs one Corgi daemon instead of Corgi driving individual agent commands through one-shot sandbox calls.

The chosen direction is a full OpenShell-native integration, not the generated-SSH-alias shortcut. A generated alias could prove the daemon and bridge work, but it would not produce the right trust, lifecycle, or credential model.

**Success:** a user can keep one local Corgi viewer while working with local machines, ordinary SSH machines, and OpenShell sandboxes in the same sidebar. A sandbox can be selected, attached to, disconnected from, restarted, and diagnosed without Corgi treating it as an ordinary SSH host or weakening the trust rules for SSH.

**Anti-goals:**

- Running every keypress or terminal surface through a separate `openshell sandbox exec` call.
- Exposing a Corgi daemon on an unauthenticated public TCP port.
- Weakening host-key checks for ordinary SSH machines.
- Automatically upgrading Corgi inside an OpenShell sandbox.
- Replacing OpenShell's policy, provider, or inference controls with Corgi settings.
- Automatically deleting an OpenShell sandbox when its Corgi profile is removed.

## Approaches Considered

### A: OpenShell-native machine transport (chosen)

Add an OpenShell machine kind with its own transport adapter. Use OpenShell's interactive exec stream to run Corgi's native bridge against the daemon socket inside the sandbox. OpenShell owns gateway authentication and sandbox lifecycle; Corgi owns the existing daemon protocol and viewer behavior.

- Pros: preserves Corgi's PTY, hooks, workspace state, and reconnect model; no SSH alias or host-key exception; OpenShell errors and lifecycle can map cleanly to Corgi machine states; a typed CLI wrapper can be replaced by a direct gateway client later without changing the user model.
- Cons: requires a shared machine transport boundary instead of the current SSH-only implementation; adds sandbox image and protocol-pinning work; introduces OpenShell as a runtime dependency for this machine kind.

### B: Generated OpenShell SSH alias

Use `openshell sandbox ssh-config` to create an SSH host alias, then attach Corgi to it as an ordinary SSH machine.

- Pros: smallest prototype; reuses the existing SSH bridge and setup path almost unchanged; useful for checking terminal and agent behavior quickly.
- Cons: OpenShell's generated config disables host-key verification, while Corgi enforces strict host-key checking; relaxing that rule for all SSH machines would weaken the existing trust model; `corgi machine add` would also try to manage the Corgi installation inside an immutable sandbox. This is a prototype path, not the target.

### C: One gateway daemon with sandboxes as workspaces

Run one Corgi daemon outside the sandboxes and make each OpenShell sandbox a workspace within that daemon.

- Pros: fewer daemons; one machine group could contain many sandboxes; lifecycle might be centralized in one place.
- Cons: the daemon would need to own PTYs and provider processes in separate sandbox contexts, or every interaction would become a remote command. That loses the native terminal and hook model Corgi already relies on and makes the change much larger than a transport adapter.

### Dropped

- A separate `sandbox exec` per pane or agent command: loses daemon-owned terminal state and turns OpenShell into the terminal transport.
- Forwarding the Corgi daemon over public TCP: creates a new network listener and authorization boundary that Corgi does not need.

## Decision

Approach A.

The user explicitly chose the full OpenShell-native solution. One sandbox is one Corgi machine, with one daemon inside it. Corgi continues to render and coordinate the workspace; OpenShell remains the execution, policy, credential, and lifecycle boundary.

The first implementation should use the interactive `openshell sandbox exec` stream to run the existing Corgi bridge, conceptually:

```sh
openshell sandbox exec \
  --name "$SANDBOX" \
  --no-tty \
  --timeout 0 \
  -- /usr/local/bin/corgi __remote-bridge "$CORGI_SOCKET"
```

That stream carries stdin and stdout unchanged and lets the current link, input receipt, stream, retry, and reconnect behavior continue to work. A typed wrapper around the `openshell` CLI is the proposed first adapter because it leaves gateway authentication and token refresh in OpenShell's existing client. A direct gateway client can replace or supplement it later.

The exact CLI spelling and direct-gateway choice remain open for maintainer feedback.

## Key Requirements

**User decisions:**

- Integrate through the full OpenShell-native path rather than the generated SSH alias.
- Keep this document as both the brainstorm and the issue draft.

**Chosen behavior:**

- Add an OpenShell machine kind and profile identifying the gateway, workspace, sandbox, Corgi executable, socket, home, and bridge.
- Run one Corgi daemon per sandbox and keep the existing runtime and viewer protocols unchanged.
- Reach the daemon through an OpenShell exec bridge with a duplex raw byte stream and no pseudo-terminal.
- Do not expose the daemon through an unauthenticated network listener.
- Keep ordinary SSH machines on their current strict host-key policy.
- Require a Corgi-prepared sandbox image or template with Corgi, `corgi-remote-bridge`, the supported agent CLIs, shell tools, hooks, and a persistent work directory.
- Pin the Corgi build in the sandbox image. A protocol mismatch is an explicit incompatible-build state, not an automatic in-place upgrade.
- Show provisioning, ready, stopped, gateway-unavailable, policy-denied, authentication, and incompatible-build states distinctly.
- Keep local Corgi responsive while an OpenShell gateway or sandbox is unavailable. Start a stopped sandbox only after an explicit user action.
- Upload pasted images through the OpenShell machine's own transport and return the sandbox path.
- Make sandbox deletion explicit; removing a Corgi profile must not delete the sandbox.
- Add a safe machine-catalog migration or skip-unknown-kind rule so an older Corgi build never interprets an OpenShell profile as an SSH target.

**My calls, subject to maintainer review:**

- One sandbox, not one gateway, is the Corgi machine boundary.
- The first version uses a typed `openshell` CLI wrapper instead of an embedded gRPC client.
- Corgi-led account login is hidden for OpenShell machines at first; OpenShell providers or managed inference supply credentials.
- Discovery imports selected sandboxes rather than mirroring every sandbox automatically.
- The first version supports manual machine registration and attach before adding provisioning and full lifecycle controls.

**First user-facing slice:**

```sh
corgi machine add corgi-dev --openshell --sandbox corgi-dev
corgi --machine corgi-dev
```

The command spelling is illustrative. The required outcome is attaching to a running, version-compatible sandbox through a typed OpenShell profile without using a generated SSH config.

## Open Questions

- Is an OpenShell machine provider a direction Corgi wants, and is a provider-neutral machine transport acceptable?
- Is a CLI-backed first implementation acceptable, or should the Corgi transport speak directly to OpenShell's gRPC API?
- Is a sandbox the right Corgi machine boundary, or should one OpenShell gateway be the machine and its sandboxes become workspaces?
- Should the first version only attach to existing sandboxes, or also create and provision them from Corgi?
- Where should the Corgi sandbox image live: the Corgi repository, Corgi release artifacts, or the OpenShell Community sandbox catalog?
- Which paths survive `openshell sandbox stop` and `sandbox start`, and should Corgi state live under `/workspace` to survive sandbox recreation?
- Should Corgi's agent account management be hidden inside OpenShell sandboxes, or mapped to OpenShell provider attachment and status commands?
- Should a stopped sandbox start automatically when a user opens one of its workspaces, or always require an explicit start action?
- How should Corgi distinguish OpenShell policy denials from gateway, authentication, and network failures in the machine header?
