# Contract: handoff file

`<report dir>/handoffs/<slug>.md`

```markdown
# Handoff: <display source>

- Kind: rebase | rebase-checks | build-drift | build-clash
- Patch: <description> (<PR link or branch>)
- Upstream: <short sha> <subject>

## Problem
<conflicting files, or failing step with a log excerpt>

## Cause
<"upstream main moved on" | "clashes with: <other patches>">

## Fix applied in the custom build
<agent fix commit sha and diff stat, or "none">

## Prompt
Run this in the checkout of <branch or PR head>:

> <self-contained prompt: what to rebase or fix, the error, the other patches involved,
> and the rule that changes go into this PR branch, never into the custom build branch,
> which is regenerated on every rebuild>
```

Kinds:
- `rebase`: patch conflicts with plain upstream (FR-029, FR-032), or an own patch's rebase could not be completed inside `rebuild` (FR-042); the problem then names the commit and files the rebase stopped at.
- `rebase-checks`: an own patch rebased onto upstream, but typecheck or tests fail on it alone (FR-046); the branch kept its old tip. The problem has the failing step and a log excerpt, the fix section says "none", and the prompt asks to rebase and fix the step in the branch's checkout.
- `build-drift`: implicated patch fails the step merged alone onto upstream (FR-031).
- `build-clash`: no implicated patch fails alone; the prompt names the others and leaves which side adapts, or waiting for one to merge, to the person.
