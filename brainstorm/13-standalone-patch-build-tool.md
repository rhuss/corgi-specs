# Brainstorm: A standalone tool for patched builds

**Date:** 2026-11-11
**Status:** active
**Research:** [13-standalone-patch-build-tool-research.md](13-standalone-patch-build-tool-research.md)
**Related:** [09-custom-build.md](09-custom-build.md) (this revisits its dropped approach C, "keep the tool outside")

## Problem Framing

corgi's `scripts/custom-build` (tals/corgi#117) turns a list of patches (upstream PRs, PRs and branches on forks, own branches) into a build of upstream plus those patches, every time upstream moves. It merges with rerere, runs the project's checks, tells a broken upstream apart from a broken patch, lets an AI agent resolve conflicts and fix failing steps under strict checks, writes handoff prompts for patches that need work, keeps your own branches rebased, and shows all of it in a live terminal view.

Almost none of that is about corgi. The idea: pull it out into its own tool that works for any git project on GitHub, and have corgi use that tool instead of carrying the script.

Why now:
- **It's general already.** About 6,900 lines of code (4,100 of them the core run logic, the rest the live view and tables) plus about 5,000 lines of tests, no dependencies, and the corgi-specific parts are a short list (see Key Requirements).
- **The same need exists elsewhere.** Anyone running a project with unmerged PRs, their own or other people's: power users of fast-moving tools, contributors who live on their own PR branches, and teams with downstream builds (the OpenShift "carry patch" world, distro packaging, internal forks).
- **Nothing covers this combination** (see Landscape). The closest tools each cover one slice.
- **It keeps corgi small.** An upstream project shouldn't have to carry ~12,000 lines of build tooling and tests that serve people running patched builds. A standalone tool plus a small config file is a much easier ask for tals/corgi.

Success: one installable tool that rebuilds any GitHub project from upstream plus a patch list with the same behavior corgi's script has today; corgi uses it with a small config file and keeps only its build stamping.

Goals:
- Works for any git repo with an upstream on GitHub; the project describes its checks and artifact in a small config file.
- Same core behavior as today: ordered merges with rerere, checks, blame on upstream, stops with continue/rollback, verified agent repair, handoffs, own-branch rebasing with leases, status, live view.
- One binary, no runtime dependencies beyond git (and `gh` for PR metadata).
- Usable headless (CI, cron): machine-readable result, exit codes, report.

Non-goals (first version):
- A hosted service, GitHub App or web UI.
- Gating merges into upstream (merge queues do that).
- Replacing stacked-PR tools for authoring branches.

Anti-goals:
- Silent auto-fixing: every agent change stays visible, separate and revertable.
- Rewriting or pushing anything that isn't yours.
- Locking a project into one agent vendor.

## Landscape

From the research: no tool found combines (1) rebuilding from a list of upstream PRs, fork branches and own branches, (2) checks with blame on upstream, and (3) agent-based repair whose result the tool verifies. Per-patch handoff prompts, the flaky-then-upstream rerun, token accounting, and a live view with estimates didn't show up anywhere.

Closest neighbors, and what to take from each:

| Tool | Overlap | Worth learning from |
|---|---|---|
| patchy (nik-rev, Rust, ~185 stars, feature work paused since 2025-01) | builds a branch from a TOML list of PRs, fork branches and `.patch` files, SHA pinning | pinning (`"145 @ <sha>"`), `.patch` files as a source kind, the install story (crates, Homebrew, Nix) |
| DRM `dim rebuild-tip` | integration branch from `nightly.conf`, shared rerere cache + fixup patches on a `drm-rerere` branch | team-shared rerere and fixups as a first-class way to keep resolutions |
| git-assembler (Python, 2023) | declarative merge/rebase/stage recipes | the recipe model (stages that rebuild when inputs change) |
| openshift-eng/rebasebot (+ OADP AI helpers) | the opposite direction: replays a fork's carry commits onto upstream; 5 lifecycle hooks; runs as Prow jobs | hooks at fixed points, dry-run, PR creation, a "pause" label; the carry-commit direction as a possible second mode |
| Backport Agent (TS) | host-side gates on an LLM's resolution (confidence, validation must pass) | confirms the "verify the agent" stance; ideas for more gates |
| mergai (Percona), Code Merge System | agent-driven upstream-into-fork merges, git notes, judge agents | git notes for resolution metadata; a judge or second-opinion step |
| Cohere vllm-skills | agent skills: rebase, then test old branch (baseline) vs new | baseline testing is the upstream-blame idea in another form |
| Zuul, Mergify, GitHub merge queue | speculative merging and culprit bisection, server side | bisection when several patches together break a step |
| git-branchless `git test` | per-commit checks in worktrees with result caching | caching check results by tree hash |
| jj "megamerge" | one octopus merge over all working branches, auto-rebased | an alternative backend later; conflicts as first-class data |

Positioning (draft): **a downstream build of upstream plus your patch list, kept green by checks, blame and an agent you can trust.**

Who it is for:
- **Power user running a patched tool** (the corgi case): upstream plus a few of your PRs and a few of other people's, rebuilt whenever main moves.
- **Contributor with many open PRs**: keep them all rebased and checked, get a handoff prompt when one needs work, see at a glance which still merge.
- **Team downstream build**: an internal fork that must stay on top of upstream with a known set of carried patches; nightly in CI, report and notifications on failure.
- **Maintainer preview**: "what does main look like with these 5 PRs merged together?" (the drm-tip use case) before merging any of them.
- **Release/backport branches**: a release branch plus a list of fixes to carry (later; overlaps with backport tools).

## Approaches Considered

### A: Extract into a standalone tool (chosen direction)
Move `scripts/custom_build/` into its own repo, cut the corgi specifics, ship one binary; corgi keeps its build stamping and a config file.
- Pros: reaches every project with the same need; corgi stays small; the tool gets its own release cadence.
- Cons: a second project to maintain; while #117 lives on in corgi, fixes have to flow into both copies.

### B: Keep it inside corgi only (status quo of #117)
- Pros: no extra project; corgi users get it with the repo.
- Cons: upstream carries ~12,000 lines that serve patched builds; nobody outside corgi benefits.

### C: Build on an existing tool instead (patchy, rebasebot, git-assembler)
- Pros: less code of our own.
- Cons: each covers only the assembly or the carry-rebase slice; the checks, blame, verified agent repair, handoffs and live view would all have to be added on top, in a foreign codebase (Rust, Python) with different goals. patchy's feature work is paused.

### Dropped
- A hosted GitHub App or merge-queue style service: wrong scope for a downstream build.

## Decision

**Direction A, worked on in parallel (Roland, 2026-11-11).** #117 keeps going in corgi to iron things out (review rounds, fixes, real use), and the standalone tool is developed alongside it. corgi switches over to the standalone tool later.

Consequence: fixes made in #117 must flow into the standalone tool too, so the extraction should keep the code close to #117's layout (same modules, same tests) until the switch, to keep porting changes cheap.

## Key Requirements

### Starting point: what corgi's tool does today
1. **Patch list** in `.custom-build/config.json` (gitignored): upstream PRs (`109`, `owner/repo#12`, PR URLs), fork branches (`owner/repo:branch`, tree URLs, `<git url>#branch=<name>`), own branches; `add`, `remove` (retire with a reason), `list`.
2. **`rebuild`**: fetch upstream and every patch into `refs/custom-build/*`, reset an integration branch to upstream, merge in order with rerere (`Merge <source>: <description>` titles), run install, typecheck, test, build, keep the previous build as `-prev`, write a per-run report.
3. **Stops and recovery**: a conflict or failing step stops with `continue` and `rollback`; the tool copies itself so `continue` works even when the reset tree lacks it.
4. **Agent harness** (`claude -p`, `codex exec`, or any command): resolves conflicts and fixes failing steps; the tool verifies every result (marker-free, only conflicted files touched, fixes as separate `(agent)` commits, attempt limits, timeouts in a process group), parses Claude/Codex stream output for readable logs and token counts.
5. **Blame**: a failing step reruns once (flaky); if it fails again it runs on plain upstream, so a broken upstream is reported as such; fixes are never carried forward.
6. **Handoffs**: per affected patch, a ready-to-run prompt for the patch's own branch (needs a rebase, or a fix that touches its files).
7. **Own-branch rebasing**: own branches and PRs from the fork, rebased in throwaway worktrees (or in place with `--in-worktree`), checked alone, backed up, pushed with a lease on request; modes all/conflicting/off.
8. **`status`**: test-merges each patch onto the last fetched upstream and says what `rebase` would do, without changing anything.
9. **Live view**: timeline vs. the previous run, estimates from recorded timings, checklist, harness tokens, retries, tab progress, notifications, OSC 8 links, a final card; `--verbose` pins the view over the full log; plain output when piped.
10. **Build stamping** (from tals/corgi#116, inside corgi's `src/` and `build.ts`): a development build names the fork its commit lives in.

Module map (`scripts/custom_build/`): `main.ts` (CLI), `config.ts`, `sources.ts` (patch sources, URLs), `git.ts`, `github.ts` (`gh` for PR info), `steps.ts`, `run.ts` (rebuild/continue/rollback/status), `rebase.ts`, `harness.ts` + `harness_stream.ts`, `report.ts`, `timings.ts`, `live_events.ts`, `live_state.ts`, `live_render.ts`, `screen.ts`, `live_worker.ts`, `live.ts`, `term.ts`, `list_view.ts`, `status_view.ts`, `prompts/*.md`; tests in `tests/custom_build.test.ts` (129, CLI level against temporary repos) and beside the modules (116).

### The cut list: what is corgi-specific

| Where | What | Generic replacement |
|---|---|---|
| `steps.ts` defaults | `bun install --frozen-lockfile`, `bun typecheck.ts`, `bun build.ts --native-only` before `bun test src`, `bun build.ts --outfile {outfile}` | no defaults, or presets per ecosystem; steps come from config |
| `run.ts`, `report.ts`, `main.ts` | `dist/corgi-custom`, `corgi-custom.start/.prev` | a configured artifact path (or none: some projects only need the branch) |
| `rebase.ts`, `steps.ts` | `dist/corgi-rebase`, `dist/corgi-isolated` outfiles | derived from the artifact setting |
| `config.ts` | default upstream `tals/corgi` | required in config, or derived from the `upstream` remote |
| env vars | `CORGI_CUSTOM_BUILD_DIR`, `_REPO`, `_KIND`, `_PROMPT`, `_GITHUB` | the new tool's prefix |
| `prompts/conflict.md`, `build.md` | "a custom corgi build" | project name from config |
| `scripts/custom-build` (shell) | resolves the repo, runs the copy of the tool during a stopped run | the tool's own binary; the copy mechanism goes away |
| `src/custom_build.ts`, `build.ts` | build stamping (#116) | stays in corgi; the tool exposes the patch list to the build (env var or manifest) so any project can stamp |
| `docs/custom-builds.md`, `HACKING.md` | corgi docs | the tool's README; corgi keeps a short "patched builds" note |

### Rough phases (to refine when picked up)
1. **Spike (a day):** compile the current tool with `bun build --compile`; check the worker, binary size, startup; run it against a non-corgi repo with explicit steps.
2. **Extraction:** new repo, move `scripts/custom_build/` and its tests, apply the cut list, rename env vars and paths, keep behavior identical. CI with the existing 245 tests.
3. **Generalize:** config format decision, presets (or none), artifact setting, hooks, GitHub-only sources plus `.patch` files and pinning.
4. **corgi migration:** after #117 has settled in corgi; corgi keeps stamping and a config, and `scripts/custom-build` is replaced by the standalone tool.
5. **Publish:** name, README with the GIFs from #117, Homebrew tap, release binaries, an Actions example.
6. **Later:** shared rerere, bisection, check caching, carry-commit mode, GitLab.

### Risks
- **Maintenance load** of a second tool, and support questions from projects with unusual setups.
- **Scope creep** toward a CI system or merge queue. The non-goals guard this.
- **Agent trust**: a general tool will meet riskier repos; the verification gates and "never carry fixes forward" rule matter even more.
- **Binary size** of a compiled Bun tool, if it bothers users.
- **#117 timing**: landing it in corgi and later replacing it costs upstream review effort twice (accepted: parallel work, switch later).
- **Drift between the two copies** while #117 and the standalone tool both evolve; keep the module layout identical until the switch.

## Open Questions

Answer first when picking this up:
1. How do fixes flow between #117 and the standalone repo (manual port, or a sync script over the shared module layout)?
2. Name shortlist and collision check.
3. Bun-compiled binary acceptable, or rewrite?
4. Config: format, and private vs. shareable patch list.
5. Which of shared rerere, bisection, check caching, carry mode go into v1.

### Identity
- **Name.** Avoid: patchy, rebasebot, merge-bot, patchwork, git-pile, git-assembler, git-upstream, mergai, mergefix, Merge Mate, Evergreen, fork-sync, carry (reads as OpenShift jargon), graft, stitch, splice, forklift. Unchecked ideas: *tributary*, *patchbay*, *drydock*, *overlay*, *keelson*, *tip* (as in drm-tip), *rebuildr*. Needs a collision check on GitHub, crates/npm/Homebrew, and `git <name>`.
- **CLI shape.** `git <name>` subcommand (needs a `git-<name>` binary on PATH) or a standalone command? Keep the verbs (`add`, `remove`, `list`, `rebuild`, `rebase`, `continue`, `rollback`, `status`).
- **License and home.** Personal GitHub, an org, or a Red Hat umbrella? Affects naming and contribution model.

### Technology
- **Language and distribution.** Keep TypeScript on Bun and ship with `bun build --compile` (reuses all ~6,900 lines and the tests; binaries are large, ~60-100 MB) versus a rewrite in Go or Rust (small binaries, familiar to the kernel/OpenShift audience, but a full rewrite). First instinct: keep Bun, compile, revisit only if size or startup hurts.
- **The live view's worker** relies on a Bun Worker writing to stdout; check it survives `bun build --compile`.
- **Running from a copy during a stopped run** goes away with an installed binary, which is a nice simplification.

### Configuration
- **Format and location.** JSON in a gitignored `.custom-build/` today. TOML (patchy uses it, friendlier to hand-edit), YAML, or keep JSON? A committed project file (checks, artifact, prompts) plus a private patch list? Or a patch list a team can share (committed on a separate branch, like drm-tip's `nightly.conf`)?
- **Project presets.** `bun`, `npm`/`pnpm`, `go`, `cargo`, `make`, `gradle`: sensible default steps so a first run needs only the patch list. Or always explicit steps, simpler and no wrong guesses.
- **Step model.** Fixed four steps or any named list? Per-step "only when files changed" rules (today install runs only when lockfiles changed)? Timeouts per step?
- **Artifact.** Optional path plus a post-build command (`afterBuild` today: `./dist/corgi-custom reload`); keep `-prev` copies for rollback.
- **More patch sources.** Local `.patch` files and SHA pinning (from patchy); GitLab merge requests; plain git URLs (exists); maybe Gerrit.

### Behavior
- **Shared rerere.** Store the rerere cache (and fixup patches) on a branch so a team or several machines replay the same resolutions (DRM's model); also reuses an agent's accepted resolution instead of asking again (saves tokens).
- **Bisection.** When the combined build fails a step, rebuild subsets to find the patch or pair that breaks it (Mergify/Zuul style), before any agent call.
- **Check caching.** Cache step results by tree hash (git-branchless): an unchanged combined tree doesn't run the tests again.
- **Carry-commit mode.** Support the opposite direction too (a fork's own commits replayed onto a new upstream, like rebasebot), or stay focused on patch lists?
- **Hooks.** Fixed hook points (before fetch, after merge, before push, on stop, on finish), like rebasebot's five, for vendoring, codegen or notifications without code changes.
- **Agent harness.** Keep `kind: claude | codex` plus "any command". Add gemini/opencode kinds? A token or cost cap per run? A second-opinion (judge) call before accepting a resolution for risky files? Where do prompt templates live, and how does a project add its own invariants (mergai's `invariants.md`)?
- **Headless and CI.** A `--json` result, documented exit codes (they exist), a GitHub Actions example for a nightly downstream build, notifications via hooks.
- **Build stamping.** Offer the data instead of reaching into the build: an env var during the build step (`<TOOL>_PATCHES`, the fork URL, the integration commit) and a manifest file, so corgi keeps #116's code, fed by the tool.

### Live view and UX
- Keep the live view as is, with the parked items from #117: the 20-row `--verbose` threshold, the Ghostty visual check, SIGWINCH in the worker.
- A `doctor` command: git version, `gh` auth, upstream remote, fork push target, harness availability.
- An `init` command that writes a starter config (detects the ecosystem).

### corgi migration
- corgi's steps (notably `bun build.ts --native-only` before unit tests) become corgi's config, not tool code.
- Migration of `INTEGRATION.md` and the `integration-rebase` skill (open since #09) onto the standalone tool instead of the corgi script.
