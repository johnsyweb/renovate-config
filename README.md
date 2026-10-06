# johnsyweb/renovate-config

Shared [Renovate](https://docs.renovatebot.com/) preset and aube lockfile workflow for johnsyweb repositories that use [aube](https://aube.sh/) + [mise](https://mise.jdx.dev/).

Extends [jdx/renovate-config](https://github.com/jdx/renovate-config) with:

| Override | Value |
| --- | --- |
| Timezone | `Australia/Melbourne` |
| Schedule | at any time (no branch-creation window) |
| Dependency dashboard | enabled |
| Commit messages | Conventional Commits `chore(deps): …` |
| Majors | automerged when CI is green (non-majors stay grouped per jdx) |

Cooling (`minimumReleaseAge: 7 days`) and ecosystem grouping come from jdx’s preset.

## No-App aube-lock pattern

Renovate’s npm manager updates `package.json` but does not understand `aube-lock.yaml`. This repo’s reusable `aube-lock` workflow:

1. Regenerates every tracked `*aube-lock.yaml` on `renovate/**` pushes from renovate[bot]
2. Commits with `GITHUB_TOKEN` when needed
3. Runs the consumer’s verify command (default `./script/cibuild`)
4. Publishes Check Runs on the **final** HEAD (including mirrored CI job names) so automerge is not blocked by GitHub’s “token pushes do not retrigger workflows” rule

No GitHub App secrets are required.

Fleet membership for the Mend Renovate app is managed in [johnsyweb/github-infra](https://github.com/johnsyweb/github-infra).

## Use in a repo

1. Ensure the repo is listed in [johnsyweb/github-infra](https://github.com/johnsyweb/github-infra) `renovate/repos.yaml` (after the one-time Mend install documented there).
2. Add `.github/renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>johnsyweb/renovate-config"]
}
```

3. Add `.github/workflows/aube-lock.yml` (pin the commit SHA of this repo — ambassy’s `verify-github-action-pins` requires full SHAs):

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
      verify_command: ./script/cibuild
      check_name: renovate-verify
```

4. Remove Dependabot npm/docker/github-actions config so you do not get duplicate PRs.
5. Align aube cooling with Renovate: set `minimumReleaseAge: 10080` (7 days, in minutes) in `aube-workspace.yaml`.

### Local dependency bumps

Keep `mise run update` for “after git pull”. Add a separate task that runs within-range bumps:

```bash
aube outdated
aube update
```

Or `mise run update-deps` where that task exists.

## Fleet migration plan

### Phase 1 — aube-native (in progress)

Apply this preset + aube-lock workflow to repositories that already use `aube-lock.yaml` as the source of truth.

| Order | Repository | Notes |
| --- | --- | --- |
| 1 | [ambassy](https://github.com/johnsyweb/ambassy) | Pilot on aube `1.40.0`; exit after a successful open → lock → automerge cycle |
| 2 | [progression](https://github.com/johnsyweb/progression) | Phase 2 hybrid → aube + Renovate (pnpm retired) |
| 3 | Highest-touch aube-native next (e.g. agent-skills, eventuate, foretoken, parkrun-by-lga) | Stamp the ambassy pattern |
| 4 | Remaining aube-native | Including lower-traffic clones under `~/src` |

**Pilot exit criteria:** Renovate opens PRs → aube-lock regenerates `aube-lock.yaml` → verify Check Runs green → automerge lands at least one non-major; `mise run update-deps` run once locally.

### Phase 2 — hybrids and legacy lockfiles

Repositories that still use `pnpm-lock.yaml` / `package-lock.json` (even if mise lists aube) are **not** on this pattern yet.

| Rule | Behaviour |
| --- | --- |
| Usage-first | Migrate repos you touch monthly+ before dormant ones |
| Opportunistic | Dormant repos wait until next real touch |
| Gate | If the touch changes dependencies or CI, migrate to aube + this Renovate pattern in the same effort |
| Content-only | One-line content fixes need not force migration |

Record each migration with a short ADR in the target repo.

### Supply-chain posture (shared)

- Renovate and aube both wait **7 days** before taking a new release.
- `paranoid: true` and explicit `allowBuilds` stay fail-closed: a Renovate PR that needs a new lifecycle script stays red until a human blesses it.
- Automerge (including majors) only after green checks — Deliver!
