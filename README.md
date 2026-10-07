# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) preset and reusable aube-lock workflow for johnsyweb repositories that use [aube](https://aube.jdx.dev/) + [mise](https://mise.jdx.dev/).

One place to pin Melbourne timezone, open schedules, dependency dashboards, Conventional Commits, and major automerge — plus lockfile regeneration that Mend Renovate cannot do alone — so every fleet repo inherits the same supply-chain posture.

## Getting started

In a consumer repo, extend this preset:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>johnsyweb/renovate-config"]
}
```

Save as `.github/renovate.json`, then finish the [Use in a repo](#use-in-a-repo) checklist (aube-lock workflow, Dependabot removal, cooling, README refresh) and list the repo in [johnsyweb/github-infra](https://github.com/johnsyweb/github-infra).

## Help

[GitHub Issues](https://github.com/johnsyweb/renovate-config/issues). Fleet membership questions belong in [johnsyweb/github-infra](https://github.com/johnsyweb/github-infra).

## Maintainers

[johnsyweb](https://github.com/johnsyweb) (Pete Johns).

## Development status

Maintained. Active preset for ambassy, progression, agent-skills, and further fleet migrations.

## Local development

This repository has no package install. Edit `default.json` for preset overrides and `.github/workflows/aube-lock.yml` for the reusable lock workflow. Consumer repos pin the workflow by full commit SHA.

Preset highlights (extends [jdx/renovate-config](https://github.com/jdx/renovate-config)):

| Override | Value |
| --- | --- |
| Timezone | `Australia/Melbourne` |
| Schedule | at any time (no branch-creation window) |
| Dependency dashboard | enabled |
| Commit messages | Conventional Commits `chore(deps): …` |
| Majors | automerged when CI is green |

Cooling (`minimumReleaseAge: 7 days`) and ecosystem grouping come from jdx’s preset.

### No-App aube-lock pattern

Renovate’s npm manager updates `package.json` but does not understand `aube-lock.yaml`. The reusable `aube-lock` workflow:

1. Regenerates every tracked `*aube-lock.yaml` on `renovate/**` pushes from renovate[bot]
2. Commits with `GITHUB_TOKEN` when needed
3. Runs the consumer’s verify command (default `aube ci`)
4. Publishes Check Runs on the **final** HEAD so automerge is not blocked by GitHub’s “token pushes do not retrigger workflows” rule

No GitHub App secrets are required.

### Use in a repo

1. Ensure the repo is listed in [johnsyweb/github-infra](https://github.com/johnsyweb/github-infra) `renovate/repos.yaml` (keep `repos:` sorted alphabetically; after the one-time Mend install documented there).
2. Add `.github/renovate.json` as in [Getting started](#getting-started).
3. Add `.github/workflows/aube-lock.yml` (pin this repo’s commit SHA):

```yaml
name: aube-lock

on:
  push:
    branches: ['renovate/**']

permissions:
  contents: write
  checks: write

jobs:
  aube-lock:
    uses: johnsyweb/renovate-config/.github/workflows/aube-lock.yml@<pin-sha>
    with:
      check_name: renovate-verify
      # verify_command defaults to `aube ci`. Pass a richer gate only when the
      # repo already has one (e.g. verify_command: ./scripts/cibuild).
```

4. Remove Dependabot npm/docker/github-actions config so you do not get duplicate PRs. Close open Dependabot PRs with a pointer to Renovate.
5. Align aube cooling with Renovate: set `minimumReleaseAge: 10080` (7 days, in minutes) in `aube-workspace.yaml`. Do not add an empty `allowBuilds: {}` — fail-closed is the default under `strictDepBuilds` / paranoid.
6. **Refresh the root README in the same change** — done when (a) no Dependabot claims remain, (b) install / update / CI commands match mise + aube, and (c) dependency ownership names Renovate (and the ADR) rather than Dependabot. If the README is substantially stale beyond those lines, run the `/readme` skill so the cutover does not leave a `#dependabot` anchor pointing at fiction.
7. **Lean consumer defaults** (skip unless the repo already has a richer pattern):
   - Verify with `aube ci` — no `script/ci-install` / `script/cibuild` wrappers that only call it.
   - One scripts directory (`scripts/`), not both `script/` and `scripts/`.
   - PR CI job named `build` that runs `aube ci` on `pull_request` only (not also on `push` to `main` when Release already installs).
   - Ruleset requiring `build` with strict up-to-date off, so Renovate platform automerge can arm (aube-lock mirrors that check name).

Local within-range bumps in consumers: `aube outdated` then `aube update`, or `mise run update-deps` where that task exists. Keep `mise run update` for “after git pull”.

### Fleet migration plan

**Phase 1 — aube-native (in progress).** Apply this preset + aube-lock to repositories that already use `aube-lock.yaml`.

| Order | Repository | Notes |
| --- | --- | --- |
| 1 | [ambassy](https://github.com/johnsyweb/ambassy) | Pilot on aube `1.40.0`; exit after a successful open → lock → automerge cycle |
| 2 | [progression](https://github.com/johnsyweb/progression) | Phase 2 hybrid → aube + Renovate (pnpm retired) |
| 3 | [agent-skills](https://github.com/johnsyweb/agent-skills) | aube-native; Renovate replaces Dependabot + Monday `aube-update` |
| 4 | Highest-touch aube-native next (e.g. eventuate, foretoken, parkrun-by-lga) | Stamp the ambassy pattern |
| 5 | Remaining aube-native | Including lower-traffic clones under `~/src` |

**Pilot exit criteria:** Renovate opens PRs → aube-lock regenerates `aube-lock.yaml` → verify Check Runs green → automerge lands at least one non-major; `mise run update-deps` run once locally.

**Phase 2 — hybrids and legacy lockfiles.** Repositories that still use `pnpm-lock.yaml` / `package-lock.json` are not on this pattern yet.

| Rule | Behaviour |
| --- | --- |
| Usage-first | Migrate repos you touch monthly+ before dormant ones |
| Opportunistic | Dormant repos wait until next real touch |
| Gate | If the touch changes dependencies or CI, migrate to aube + this Renovate pattern in the same effort |
| Content-only | One-line content fixes need not force migration |
| Docs | Same PR: ADR(s) + root README refreshed per step 6 above |

Record each migration with a short ADR in the target repo. Migration is incomplete while the README still documents Dependabot or the old package manager.

### Supply-chain posture

- Renovate and aube both wait **7 days** before taking a new release.
- `paranoid: true` and explicit `allowBuilds` stay fail-closed: a Renovate PR that needs a new lifecycle script stays red until a human blesses it.
- Automerge (including majors) only after green checks — Deliver!

## Contributing

Open a PR against this repository. Prefer Conventional Commits. Changes to `default.json` or `aube-lock.yml` affect every fleet consumer on the next Renovate run or the next SHA pin bump.

## Releasing

Merging to `main` publishes the preset (`github>johnsyweb/renovate-config`) and the reusable workflow at the new commit SHA. Consumers that pin aube-lock by SHA must bump the pin deliberately; the preset `extends` string tracks `main`.
