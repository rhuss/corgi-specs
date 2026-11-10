# Contract: build identity (Deliverable 1, revised 2026-11-07 for #116)

## Build input

None beyond git. `bun build.ts` reads the repository from the checkout for every development build (no `--update-url`); a release build carries none. See research R1.

## Outputs

| surface | release, or build whose repository is upstream | build from a fork |
|---------|-----------------------------------------------|-------------------|
| `corgi --version` | `corgi 0.9.2 (abc1234…)` | unchanged |
| `corgi --version --verbose` | first line only | first line, then `  https://<host/owner/repo>` |
| bug report | unchanged | the same link under the Corgi row, depth 1 |
| crash event | unchanged | unchanged |
| `corgi update ...` | unchanged | refuses as for any development build |

Example for a custom build made by the tool:

```
corgi 0.1.0-dev (6170222f76c06e8813d60a029e24f96676290dd9)
  https://github.com/you/corgi
```

What the build carries is in the custom branch's log on that fork: one `Merge <source>: <description>` commit per patch.
