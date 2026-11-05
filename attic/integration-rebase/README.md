# Retired: integration-rebase

The manual integration workflow for corgi-integration, retired on 2026-11-05.

- `SKILL.md` and `rebuild.sh`: the `integration-rebase` skill that rebuilt the `integration` branch on upstream/main.
- `INTEGRATION.md`: what the branch carried, why, and its upstream PRs and issues.

Replaced by `scripts/custom-build` ([tals/corgi#117](https://github.com/tals/corgi/pull/117)), configured in
`corgi-integration/.custom-build/config.json`. The last manually built state is commit `bf8b6a06`.
