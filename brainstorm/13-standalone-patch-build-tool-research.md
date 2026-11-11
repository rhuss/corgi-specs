# Research: tools for carrying patches and building patched upstream

Date: 2026-11-11. Repo metadata (stars, last push, archived flag) was read from the GitHub REST API on this date. Everything else comes from the cited READMEs, docs and posts. Feature numbers 1-7 refer to the feature list in the brief: (1) patch list config, (2) rebuild with rerere and check steps, (3) stop/continue/rollback plus agent-based resolution with verification and token counting, (4) flaky rerun and blame-upstream fallback, (5) per-patch handoff prompts, (6) rebasing own patch branches, (7) status test-merge and live view.

## Summary

- **Closest single match for feature 1 is `patchy`** (nik-rev/patchy, Rust, about 185 stars). It reads a TOML list of upstream PR numbers, fork branches and local `.patch` files, resets a throwaway branch to upstream and merges them all. It has no check steps, no rerere (you capture a conflict resolution as a hand-made patch), no AI and no blame logic. The last release was v1.3.0 on 2025-01-29. Since then there have been toolchain-bump commits only.
- **The closest established practice is the Linux DRM `dim rebuild-tip` / drm-tip workflow**: a config file (`nightly.conf`) lists the branches to merge, and the merges are rebuilt with a shared rerere cache plus fixup patches. It is a maintainer script tied to the DRM trees, with no checks or AI built in.
- **OpenShift `rebasebot` handles the opposite direction**: it replays a fork's `UPSTREAM: <carry>` commits onto a new upstream and opens a PR, with five lifecycle hooks. Conflicts are left for a human. It is actively developed (last push 2026-11-11). The OADP team layered Claude Code plugins on top of it for AI conflict resolution (migtools/oadp-rebase-ai-helpers). The closest "carry + agent" pairing we found is these two used together, and it is org-specific.
- **AI fork-sync tools are emerging, all young**: mergai (Percona, Claude/Gemini CLI as agents), Backport Agent (sctg, TypeScript, host-side confidence and validation gates), Code Merge System (multi-agent with a judge, can drive Claude Code/Codex), and Cohere's vllm-skills (agent skills that rebase, then run a baseline-vs-new test). All of them sync *upstream into a long-lived fork*. None of them builds from a *list of PRs and fork branches*.
- **Merge queues (Zuul, Mergify, GitHub merge queue, Prow Tide) already do speculative merging of several changes and bisect batch failures to find the culprit**, which is a close match to feature 4. They gate merges into the project itself, though, not a private downstream build.
- **Stacked-PR tools (ghstack, spr, git-machete, git-town, git-branchless, Graphite, Sapling, jj) cover feature 6** (restacking your own branches onto main), and git-branchless `git test` covers running checks per commit in worktrees with caching. None of them consumes other people's PRs as inputs.
- **Answer to C: no tool found combines (1) a PR/fork-branch list rebuild, (2) checks with blame-on-upstream, and (3) verified agent-based resolution.** Features 4 (in a downstream build context), 5 (handoff prompts) and token accounting did not appear in any tool we read.
- **Names to avoid**: patchy, rebasebot, patchwork, git-pile, merge-mate, mergai, mergefix, fork-sync, carry (weak collision), git-assembler, git-upstream.

## Landscape table

| Tool | Category | Covers (features 1-7) | Main gaps vs ours | Status (as of 2026-11-11) | Link |
|---|---|---|---|---|---|
| patchy | Build fork from PR list | 1 (PRs, fork branches, `.patch` files, SHA pinning), part of 2 (reset branch + merge) | No check steps, no rerere, no AI, no blame, no status, no own-branch rebase. Conflicting PRs are skipped and left as branches | Rust, ~185 stars, last release v1.3.0 2025-01-29, last commit 2026-09-14 (toolchain bump), not archived | https://github.com/nik-rev/patchy |
| git-assembler | Assemble branches from recipe | 1 (declarative `merge`/`rebase`/`base`/`stage` rules), 2 (rebuild stage branches, recommends rerere), 6 (rebase rules) | No fetching (user runs `git fetch`), no PR resolution, no checks, no AI, stops plainly on conflict | Python 3, v1.5 2023-06-12 | https://www.thregr.org/~wavexx/software/git-assembler/ |
| dim rebuild-tip (DRM maintainer-tools) | Integration branch from config | 1 (`nightly.conf`), 2 (rebuild drm-tip, shared rerere + fixups in `drm-rerere` branch) | Specific to DRM kernel trees, no checks in the tool, no AI | Active kernel workflow | https://drm.pages.freedesktop.org/maintainer-tools/repositories/drm-tip.html |
| openshift-eng/rebasebot | Downstream carry rebase bot | Part of 2 (replay carries onto upstream), hooks for checks/vendoring, dry-run, PR creation, Slack | No PR list input (carries are commits on the fork), conflicts left to humans, no AI, no blame | Python, 2 stars / 21 forks, last push 2026-11-11 | https://github.com/openshift-eng/rebasebot |
| OADP rebase AI helpers | Claude Code plugin on top of rebasebot | 3 (Claude-driven conflict resolution and downstream commit verification) for carry rebases | OADP/Velero specific, plugin skills rather than a standalone tool | Last push 2026-06-09, 0 stars | https://github.com/migtools/oadp-rebase-ai-helpers |
| Cohere vllm-skills | Agent skills for fork rebase | 2 (rebase onto upstream tag), 3 (agent resolves conflicts), test baseline on old branch vs new branch (close to 4) | Markdown skills for agents, not a tool. Own commits only, no PR list, no explicit verification of agent edits | Published 2026-06; repo last push 2026-06-24 | https://cohere.com/blog/automating-fork-maintenance-with-ai-agents , https://github.com/cohere-ai/vllm-skills |
| mergai (Percona) | AI fork sync CLI | 3 (Claude CLI or Gemini CLI resolves conflicts, retries), merge gate and pick strategies, git notes, PRs | Merges upstream into fork, no PR list, no blame, "use with caution" | Python, last push 2026-10-13, 1 star | https://github.com/Percona-Lab/mergai |
| Backport Agent (sctg) | AI fork sync agent | 3 with host-side gates (confidence, validation must pass, report reconciliation), validation commands, exit codes for cron | Cherry-picks upstream commits into a fork, no PR list, uses its own LLM SDK loop | TypeScript (npm), last push 2026-08-07, 6 stars | https://github.com/sctg-development/backport-agent |
| Code Merge System | AI fork sync, multi-agent + judge | 3 (plan, resolve, judge, resumable; can drive Claude Code/Codex via ACP) | Fork-wide upstream merge with browser UI, no PR list, no blame | Python (PyPI), last push 2026-08-04, 2 stars | https://github.com/GOSICK-Angel/code-merge-system |
| Zuul | Gating CI with speculative merging | Close to 2 + 4: merges dependent changes (Depends-On) speculatively, across repos, before running jobs | CI server for gating, not a local downstream build | Active (OpenDev) | https://zuul-ci.org/docs/zuul/latest/gating.html |
| Mergify merge queue | Merge queue (SaaS) | 4-like: batches, speculative checks, bisects failed batches to find the culprit PR | Merges into main, SaaS, no fork build | Active, commercial | https://docs.mergify.com/merge-queue/ |
| GitHub merge queue | Merge queue | Tests PR + base + PRs ahead in temp branches | Same as above | Active | https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue |
| Prow Tide | Merge automation | Batch merge testing | Same, Kubernetes-centric | kubernetes-sigs/prow, last push 2026-11-07 | https://docs.prow.k8s.io/docs/components/core/tide/ |
| bors-ng | Merge bot | Batch testing with bisection | Archived | Archived, last push 2024-04-04 | https://github.com/bors-ng/bors-ng |
| stgit | Patch stack | 6 (`stg rebase` the stack onto upstream) | Own patches only, no PR list, no checks, no AI | Rust, 679 stars, last push 2026-10-12 | https://github.com/stacked-git/stgit |
| quilt / guilt | Patch series files | 1-ish (series file of local patches) | No git-native merge, no PRs; guilt inactive since 2020 | quilt: Savannah; guilt last push 2020-12-18 | https://savannah.nongnu.org/projects/quilt , https://github.com/jeffpc/guilt |
| topgit | Topic branch dependencies | 6 (`tg update` propagates base changes) | Own branches only | Shell, last push 2025-02-07 | https://github.com/mackyle/topgit |
| git-series | Versioned patch series | Tracks series over time | No build/check/PR input | Rust, last push 2024-07-14 | https://github.com/git-series/git-series |
| git-pile (git-pile org) | Patch pile on a branch | Patches as files, regenerated on rebase | Own patches only | Python, last push 2025-06-02 | https://github.com/git-pile/git-pile |
| git-upstream (OpenStack) | Carry patches on upstream | Rebasing locally carried patches onto upstream | Own patches only | OpenStack x/ namespace | https://git.openstack.org/x/git-upstream |
| gbp pq (Debian) | Distro patch queue | Imports/exports `debian/patches` as a branch for rebasing | Debian packaging only | Active | https://manpages.debian.org/testing/git-buildpackage/gbp-pq.1.en.html |
| rebase-helper / packit (Fedora) | Distro rebase | rebase-helper rebases spec + patches to a new upstream version; packit syncs upstream and downstream | RPM-specific | rebase-helper last push 2026-06-08; packit 2026-11-09 | https://github.com/rebase-helper/rebase-helper , https://github.com/packit/packit |
| git-imerge | Incremental merge | Pairwise conflict isolation for hard merges | One merge at a time, no list, no checks | Python, 2850 stars, last push 2024-07-17 | https://github.com/mhagger/git-imerge |
| ghstack / spr (ejoffe, spacedentist) | Stacked PRs | 6 (own commits to PRs) | No foreign PRs, no checks | ghstack 2026-09; ejoffe/spr 2026-04; spacedentist/spr 2026-10 | https://github.com/ezyang/ghstack , https://github.com/ejoffe/spr , https://github.com/spacedentist/spr |
| git-machete | Branch layout + rebase automation | 6 (traverse/rebase/push own branches), 7-ish (status tree) | Own branches only | Python, 1149 stars, last push 2026-11-08 | https://github.com/VirtusLab/git-machete |
| git-town | Branch workflow | 6 (`sync` own branches) | Own branches only | 3395 stars, last push 2026-11-11 | https://github.com/git-town/git-town |
| git-branchless | Stacked workflow | 6 (restack/sync), `git test run` executes a command per commit in worktrees with result caching, `git test fix` | Own commits only, no PR inputs | Rust, 4142 stars, last push 2026-10-24 | https://github.com/arxanas/git-branchless/wiki/Command:-git-test |
| Graphite | Stacked PRs (SaaS) | 6, merge queue | Commercial, acquired by Cursor 2025-12-19 | Active | https://techcrunch.com/2025/12/19/cursor-continues-acquisition-spree-with-graphite-deal |
| Sapling / jj megamerge | VCS | jj "megamerge": one octopus merge commit over all working branches, auto-rebased by jj; conflicts are first-class | Workflow pattern, no checks/AI/PR fetching | jj 32k stars, sapling 7k | https://github.com/jj-vcs/jj , https://v5.chriskrycho.com/journal/jujutsu-megamerges-and-jj-absorb/ |

## Closest matches in detail

### 1. patchy (nik-rev/patchy)
- **What it does:** `.patchy/config.toml` holds `repo`, `remote-branch`, `local-branch` (erased on each run), `pull-requests` (numbers, optionally pinned `"145 @ <sha>"`), `branches` (`owner/repo/branch`, optionally pinned) and `patches` (local `.patch` files from `patchy gen-patch <sha>`). `patchy run` syncs upstream and merges everything. Source: README, https://github.com/nik-rev/patchy
- **Conflicts:** PRs that do not merge cleanly are skipped and left as branches for you to merge by hand. To persist a resolution you `git merge --squash`, fix it, commit, turn the commit into a patch, and swap the PR entry for that patch. Reordering PRs can bring conflicts back. Same README.
- **Doesn't:** run checks, use rerere, use AI, detect flaky runs or upstream breakage, rebase your own branches, or give a status view.
- **Status:** Rust, distributed through crates.io (`patchy-bin`), Homebrew, Nix and a shell installer. About 185 stars. Releases: v1.3.0 on 2025-01-29. The latest commits (2025-10 to 2026-09) only bump the toolchain, so feature development looks paused. One aggregator lists it as archived (https://awesome.ecosyste.ms/projects/github.com%2Fnik-rev%2Fpatchy), but the GitHub API reported `archived: false` on 2026-11-11.
- **Key gap:** It only handles assembly. Everything from feature 2's check steps onward is missing.

### 2. git-assembler
- **What it does:** Declarative `.git/assembly` rules (`merge target branch...`, `rebase branch base`, `base`, `stage` that deletes and recreates a branch from its base when inputs change). Any ref works as an input, including remote fork branches. It recommends rerere for repeated conflicts, and supports worktrees and wildcards. Source: https://www.thregr.org/~wavexx/software/git-assembler/
- **Doesn't:** fetch (you run `git fetch --all` yourself), resolve GitHub PRs, run checks, or offer any conflict help beyond stopping.
- **Status:** Python 3, v1.5 released 2023-06-12, GitLab hosted (https://gitlab.com/wavexx/git-assembler).
- **Key gap:** It is a generic recipe engine with no build, test or agent layer.

### 3. DRM `dim rebuild-tip`
- **What it does:** `nightly.conf` on the `drm-rerere` branch lists the repos and branches to merge into drm-tip. `dim rebuild-tip` rebuilds the integration branch and pushes the shared rerere resolutions. Recurring conflicts are stored as fixup patches (`git diff | dim cat-to-fixup`). Sources: https://drm.pages.freedesktop.org/maintainer-tools/repositories/drm-tip.html , conflict doc https://maintainer-tools-rgallaispoustm-68a7c03928f3408583bf11b078a25b0.pages.freedesktop.org/committer/conflict-resolution.html
- **Relevance:** This is the most mature precedent for "rebuild an integration branch from a list of branches with team-shared rerere". Checks run in separate CI (Intel GFX CI), not in the tool. Kernel-specific shell script.

### 4. openshift-eng/rebasebot (+ OADP AI helpers)
- **What it does:** `--source` (upstream), `--dest` (downstream), `--rebase` (intermediate repo). Replays downstream carries onto upstream, with `--tag-policy` over `UPSTREAM:` tags, `--exclude-commits`, an optional `--sync-mode merge`, and Go vendor updates. Five lifecycle hooks (pre-rebase, pre-carry-commit, post-rebase, pre-push, pre-create-pr) can come from local files, git refs or remote repos. It also offers dry-run, Slack notifications and a `rebase/manual` label to pause. In merge mode, conflicts produce a PR with manual instructions. Source: https://github.com/openshift-eng/rebasebot
- **AI:** None in rebasebot itself (its CLAUDE.md is developer guidance only). The OADP team ships Claude Code plugins that wrap rebasebot for about 20 repos in 5 dependency waves, "including automated conflict resolution, downstream commit verification". Source: https://github.com/migtools/oadp-rebase-ai-helpers
- **Status:** Very active (commits through 2026-11-11), runs as Prow periodic jobs. Python.
- **Key gap:** The input is a fork's carry commits, not a PR/branch list. It doesn't separate upstream breakage from patch breakage, and the AI layer is org-specific prompt plugins.

### 5. Cohere vllm-skills
- **What it does:** Agent skills (`/auto-rebase`, `rebase-assistant`, `local-test-runner`, `install-vllm`) that replay fork commits onto a new upstream tag, resolve conflicts, build, and run tests first on the old branch (baseline) and then on the new one. Earlier pipeline: GitHub Actions + a shared rerere cache. The post does not name a specific agent harness. Source: https://cohere.com/blog/automating-fork-maintenance-with-ai-agents (2026-06-25), skills at https://github.com/cohere-ai/vllm-skills
- **Relevance:** The baseline-on-old-branch idea is close to our "run on plain upstream to blame upstream" (feature 4). The worked example traces a regression to an upstream transformers bump.
- **Key gap:** Prompt documents rather than a tool. No PR list, and no host-side verification of what the agent changed.

### 6. mergai (Percona-Lab)
- **What it does:** Long-running fork sync. Picks which upstream commit to merge next (by conflict, huge-commit, branching-point or important-files strategies, plus a merge gate), captures conflict context and hands it to Claude CLI or Gemini CLI (`resolve.agent`, `max_attempts`), stores metadata and AI solutions as git notes, and opens PRs for clean merges and for resolutions. `.mergai/invariants.md` is appended to prompts. Source: https://github.com/Percona-Lab/mergai
- **Status:** Python, "under active development; use with caution", last push 2026-10-13.
- **Key gap:** Upstream-into-fork direction only. No PR list and no check-step blame.

### 7. Backport Agent (sctg-development)
- **What it does:** Unattended fork sync with an LLM agent loop. Enforces safety rules in host code: a confidence gate (won't write low-confidence resolutions; marker, syntax and consensus guards), a validation gate (won't merge if validation failed or never ran), and report reconciliation. Exit codes 0/2/1 for cron. Customization tests declared in `customizations.yaml`. Source: https://github.com/sctg-development/backport-agent
- **Status:** TypeScript, npm `@sctg/backport-agent`, last push 2026-08-07.
- **Relevance:** The closest in spirit to our "tool verifies the agent's result" (feature 3). It still syncs a fork with upstream commits rather than assembling PRs.

### 8. Code Merge System
- Multi-agent pipeline with planning, risk categories, intent analysis, a judge verdict, resumable runs, a browser UI and a `--ci` JSON mode. Can drive local Claude Code/Codex/Gemini CLIs via ACP instead of API keys. Source: https://github.com/GOSICK-Angel/code-merge-system , https://pypi.org/project/code-merge-system/0.3.0/
- **Key gap:** Built for one big upstream merge into a fork, not a recurring rebuild from a patch list.

## Answer to C

No. We did not find any tool that combines (1) rebuilding a downstream build from a list of upstream PRs, fork PRs/branches and own branches, (2) automated check steps with attribution to upstream vs patch, and (3) agent-based conflict and step repair whose result the tool verifies.

What exists are partial combinations:
- **(1) only:** patchy, git-assembler, dim rebuild-tip.
- **Carry-rebase + hooks, then AI on top as a separate plugin:** rebasebot + OADP AI helpers (org-specific).
- **Rebase + tests + agent, as prompts:** Cohere vllm-skills. Its baseline test on the old branch is the nearest thing to our upstream blame.
- **(3) with host-side verification:** Backport Agent (confidence/validation gates), mergai, Code Merge System. All of them sync upstream into a fork.
- **Culprit isolation (the idea behind feature 4):** Mergify batch bisection (https://docs.mergify.com/merge-queue/), Zuul speculative merging (https://zuul-ci.org/docs/zuul/latest/gating.html), bors-ng. These are server-side merge gates for the project's own main branch.

None of the docs we read describe per-patch handoff prompts (feature 5), flaky-step rerun followed by an upstream-only rerun (feature 4 as specified), token accounting for the agent, or a live view with time estimates from earlier runs (feature 7). git-branchless `git test` (per-commit runs in worktrees with result caching) is the nearest analog to our check-step runner.

## AI conflict-resolution tools (2025-2026)

| Tool | Form | Notes | Link |
|---|---|---|---|
| GitHub Copilot "Fix with Copilot" / Agent Merge | Hosted | Resolves PR conflicts in its own environment, runs CI, pushes (2026-04). Agent Merge watches PRs and fixes conflicts and failing workflows (2026-10-28) | https://github.blog/changelog/2026-04-13-fix-merge-conflicts-in-three-clicks-with-copilot-cloud-agent/ , https://github.blog/changelog/2026-10-28-agent-merge-watches-and-fixes-pull-requests-in-the-cloud |
| GitHub Next Evergreen | Agentic workflow | Scheduled, keeps PRs green and mergeable (CI failures, conflicts, behind base), 2026-07-22 | https://githubnext.com/posts/evergreen/ |
| CodeRabbit resolve merge conflicts | Hosted | Triggered from a PR comment; GitHub/GitLab/Azure DevOps | https://docs.coderabbit.ai/finishing-touches/resolve-merge-conflict |
| GitKraken Merge Mate | GitHub Action + App | Syncs PRs with target via rebase/merge, AI conflict resolution | https://github.com/gitkraken/merge-mate-action |
| GitLab conflict resolver agent | Planned | Epic and issues in progress | https://gitlab.com/groups/gitlab-org/-/epics/20688 |
| claude-mergetool | git/jj mergetool | Runs `claude` with `acceptEdits` on base/left/right; v1.2.0 2026-02-27 | https://github.com/9999years/claude-mergetool |
| mergefix | CLI (Claude/OpenAI/Ollama) | Repo returned 404 on fetch, so status unverified | https://github.com/faw21/mergefix |
| erk (Dagster) | Claude Code command for stack-rebase conflicts | Asks the user on ambiguous semantic conflicts | https://dagster-io.github.io/erk/howto/conflict-resolution |
| mergai, Backport Agent, Code Merge System, OADP helpers, vllm-skills | Fork sync | See detail above | see above |
| Weave | Tree-sitter semantic merge driver (not an LLM) | Mentioned in news coverage only, not verified | https://feedbagel.com/post/weave-semantic-merge-driver-for-git-uses-tree-sitter-to-resolve-code-conflicts |

Research:
- ICST 2026, LLMs on 83 real build conflicts: mini diffs help localization, while resolution rules help less and sometimes mislead. https://people.cs.vt.edu/nm8247/publications/sheikh_icst_2026-3.pdf
- Rover, context-aware LLM conflict resolution (2026). https://www.alphaxiv.org/abs/2605.17279.md
- PPATHF, LLM patch porting for hard forks (Vim to Neovim, 2024). https://arxiv.org/pdf/2404.17964

Many Claude Code "rebase"/"resolve conflicts" skills also exist (for example openshift/lightspeed-console `rebase`, which rebases, resolves, then checks lint/i18n/build: https://mcpservers.org/agent-skills/openshift/rebase). These are prompts without host-side verification.

## Name collisions

Taken in this space, so avoid:
- **patchy**: nik-rev/patchy, same problem space. Also adamchainz/patchy and SethRobinson/Patchy.
- **rebasebot**: openshift-eng/rebasebot, chrisledet/rebasebot, oadp-rebasebot org.
- **merge-bot**: shiftstack/merge-bot.
- **patchwork**: kernel patch tracker (getpatchwork), plus patched-codes/patchwork (an agentic AI framework, 1.5k stars).
- **git-pile**: two separate projects, git-pile/git-pile and keith/git-pile.
- **git-assembler / `git as`**, **git-upstream**, **git-series**, **git-imerge**, **stgit/`stg`**, **topgit/`tg`**, **guilt**, **quilt**.
- **mergai**, **mergefix**, **Merge Mate**, **MergeForge** (Product Hunt), **Evergreen** (GitHub Next), **Agent Merge** (GitHub).
- **fork-sync** (several GitHub Actions), **Fork Manager** (agent skill).
- **spr**, **ghstack**, **git-town**, **git-machete**, **Graphite**, **Tide**, **Zuul**, **bors**.
- **carry**: weak. There are unrelated repos (metametadata/carry ClojureScript, toaco/carry ETL), and "carry patch" is OpenShift jargon (`UPSTREAM: <carry>`), so a tool called "carry" would read as OpenShift-flavored.
- **graft**, **stitch**, **splice**, **forklift**: generic words already used by well-known unrelated projects (orbitinghail/graft, kubev2v/forklift, Google Stitch).

## Notes on staleness and confidence
- Stars and last-push dates come from the GitHub API on 2026-11-11.
- The mergefix repo returned 404, so its status is unverified.
- The Weave and ConvergeAI mentions come from aggregator pages only.
- linux-next (daily merge of many trees with build tests) is a known precedent for feature 2 but was not researched here.
- Chromium/Android downstream tooling was not researched beyond this. Nothing surfaced that is meant for general use outside those projects.

## Sources
- https://github.com/nik-rev/patchy
- https://awesome.ecosyste.ms/projects/github.com%2Fnik-rev%2Fpatchy
- https://www.thregr.org/~wavexx/software/git-assembler/
- https://gitlab.com/wavexx/git-assembler
- https://drm.pages.freedesktop.org/maintainer-tools/repositories/drm-tip.html
- https://maintainer-tools-rgallaispoustm-68a7c03928f3408583bf11b078a25b0.pages.freedesktop.org/committer/conflict-resolution.html
- https://github.com/openshift-eng/rebasebot
- https://github.com/shiftstack/merge-bot
- https://github.com/migtools/oadp-rebase-ai-helpers
- https://github.com/atiratree/kubernetes/blob/master/README.openshift.md
- https://issues.redhat.com/browse/OAPE-150
- https://cohere.com/blog/automating-fork-maintenance-with-ai-agents
- https://github.com/cohere-ai/vllm-skills
- https://github.com/Percona-Lab/mergai
- https://github.com/sctg-development/backport-agent
- https://github.com/GOSICK-Angel/code-merge-system
- https://pypi.org/project/code-merge-system/0.3.0/
- https://zuul-ci.org/docs/zuul/latest/gating.html
- https://opendev.org/zuul/zuul/src/commit/ded241e59865d98b1fe3e55798f26ff25f9ccf4c/doc/source/user/gating.rst
- https://docs.mergify.com/merge-queue/
- https://blog.mergify.com/speculative-check-and-batch-under-the-hood
- https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue
- https://docs.prow.k8s.io/docs/components/core/tide/
- https://github.com/bors-ng/bors-ng
- https://github.com/stacked-git/stgit
- https://github.com/mhagger/git-imerge
- https://github.com/mackyle/topgit
- https://github.com/git-series/git-series
- https://github.com/jeffpc/guilt
- https://savannah.nongnu.org/projects/quilt
- https://github.com/git-pile/git-pile
- https://github.com/keith/git-pile
- https://git.openstack.org/x/git-upstream
- https://manpages.debian.org/testing/git-buildpackage/gbp-pq.1.en.html
- https://github.com/rebase-helper/rebase-helper
- https://github.com/packit/packit
- https://github.com/ezyang/ghstack
- https://github.com/ejoffe/spr
- https://github.com/spacedentist/spr
- https://github.com/VirtusLab/git-machete
- https://github.com/git-town/git-town
- https://github.com/arxanas/git-branchless/wiki/Command:-git-test
- https://techcrunch.com/2025/12/19/cursor-continues-acquisition-spree-with-graphite-deal
- https://github.com/jj-vcs/jj
- https://v5.chriskrycho.com/journal/jujutsu-megamerges-and-jj-absorb/
- https://github.com/9999years/claude-mergetool
- https://docs.rs/crate/claude-mergetool/latest
- https://github.com/faw21/mergefix
- https://dagster-io.github.io/erk/howto/conflict-resolution
- https://github.blog/changelog/2026-04-13-fix-merge-conflicts-in-three-clicks-with-copilot-cloud-agent/
- https://github.blog/changelog/2026-10-28-agent-merge-watches-and-fixes-pull-requests-in-the-cloud
- https://githubnext.com/posts/evergreen/
- https://docs.coderabbit.ai/finishing-touches/resolve-merge-conflict
- https://github.com/gitkraken/merge-mate-action
- https://gitlab.com/groups/gitlab-org/-/epics/20688
- https://people.cs.vt.edu/nm8247/publications/sheikh_icst_2026-3.pdf
- https://www.alphaxiv.org/abs/2605.17279.md
- https://arxiv.org/pdf/2404.17964
- https://mcpservers.org/agent-skills/openshift/rebase
