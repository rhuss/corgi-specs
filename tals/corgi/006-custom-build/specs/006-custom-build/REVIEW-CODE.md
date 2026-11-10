# Code Review: Custom corgi builds from integrated branches and PRs

**Spec:** specs/006-custom-build/spec.md
**Date:** 2026-11-04
**Reviewer:** Claude (speckit-spex-gates-review-code, ship pipeline, oversight smart)
**Diff:** `main...HEAD`, commits `794cb670` (Deliverable 1) and `0941e5de` (Deliverable 2)

## Compliance Summary

**Overall Score: 97% before fixes, 100% of FRs implemented after the fix round**

- Functional Requirements: 39/39 implemented (FR-009 second half was implicit only, via manifest hash checks; now an explicit `CustomBuild` guard in `install()`)
- Acceptance scenarios US1 to US5: all implemented; test coverage for US1 AS1/AS2/AS6, US2 AS2, US4 AS1/AS3 and FR-020/FR-028 was added in the fix round
- Edge cases: all implemented; resume after a run interrupted during a harness call now checks the leftovers (Deep Review FINDING-4)
- Spec-declared constants: none (no `## Constants` section)

## Detailed Review (deviations only)

| Requirement | Status before | Resolution |
|-------------|---------------|------------|
| FR-009 never install a custom build into the releases dir | Implicit (manifest hash) | Explicit guard added |
| FR-009 / Delivery Scope: D1 stands alone | D1 message named a D2 script | D1 names `bun build.ts --patches`; D2 adds `scripts/custom-build rebuild` |
| FR-028 / SC-007 never modify patch branches | `continue` could merge onto a patch branch after a failed reset | Reset failure clears the run; `continue` checks the branch |
| FR-027 per-call time limit | Grandchildren kept running | Process group killed |
| FR-031 clash attribution | Lone implicated patch clashed with nobody | Names the other merged patches |
| FR-032 / FR-035 one handoff set per run | Duplicates after stop and continue | Handoffs rewritten per report |

### Extra features (not in spec)

- `status` command, `--push`, `upstream.remote` config, the undocumented `steps` override and `CORGI_CUSTOM_BUILD_GITHUB` (test hooks). Assessment: helpful or test-only; `status` and `--push` are covered by contracts/cli.md and FR-028.

## Deep Review Report

**Gate:** PASS after fix round 1 (re-review found no new Critical or Important)

| Agent | Found | Fixed | Remaining | Status |
|-------|-------|-------|-----------|--------|
| Correctness | 7 | 6 | 1 | completed |
| Architecture & Idioms | 13 | 5 | 8 | completed |
| Security | 3 | 2 | 1 | completed |
| Production Readiness | 11 | 7 | 4 | completed |
| Test Quality | 14 | 7 | 7 | completed |
| Goal Alignment | 0 | 0 | 0 | skipped (no PR) |
| CodeRabbit (external) | 2 | 0 | 0 | completed (both on files outside this diff, discarded) |
| Copilot (external) | 0 | 0 | 0 | skipped (disabled in config) |
| Codex (external) | 0 | 0 | 0 | completed |
| Re-review (fix diff) | 4 | 3 | 1 | completed |

Counts per agent are before dedup; the deduplicated totals are below.

| Severity | Found | Fixed | Remaining |
|----------|-------|-------|-----------|
| Critical | 0 | 0 | 0 |
| Important | 15 | 15 | 0 |
| Minor | 32 | 11 | 21 |
| Notable | 3 | - | 3 |

MVP: Test Quality (14 findings)

No Critical or Important findings remain. The one ambiguous Important finding (resume after an interrupted harness call) was decided by the user ("Check leftovers") and implemented: `continue` runs the harness acceptance checks on the leftovers, commits them as the agent's when accepted, and stops with exit 2 or 3 otherwise. Details and all other findings: [review-findings.md](review-findings.md).

## Checks

- `bun typecheck.ts`: pass (also a separate tsc pass over scripts/custom_build and tests/custom_build.test.ts, which the repo tsconfig does not cover)
- `bun test src`: 2624 pass, 1 fail (known baseline: src/discovery.test.ts "an install too old for Corgi")
- `bun test tests/custom_build.test.ts`: 50/50
- `bun build.ts`: pass
- Deliverable 1 commit alone: typecheck pass, its unit tests 46/46
