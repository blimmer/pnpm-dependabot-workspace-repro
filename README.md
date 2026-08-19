# pnpm-dependabot-workspace-repro

Reproduction for [dependabot/dependabot-core#15959](https://github.com/dependabot/dependabot-core/issues/15959):
repeated root-level `VersionResolver` pnpm checks exhaust the 55-minute
Dependabot job limit on a large pnpm workspace.

## What this repo is

A **sanitized copy of the dependency shape of a real private monorepo**. All
source code, scripts, and identifying metadata have been stripped; every
workspace package has been renamed to `@fixture/package-NNN`. The only things
preserved are the parts that affect how Dependabot resolves the workspace:

- `package.json` for the root and 30 packages (27 workspace members under
  `packages/`, plus 3 non-importer manifests under `fixtures/` that pnpm does
  not treat as workspace projects)
- `pnpm-workspace.yaml` with a 43-entry `catalog:`, `overrides`,
  `minimumReleaseAgeExclude`, and `allowBuilds`
- `pnpm-lock.yaml` (lockfile v9, 28 importers, ~1.9k package records)
- `.github/dependabot.yml` with a single npm job using `groups`, `ignore`,
  and `cooldown`

Version specifiers, catalog usage, `workspace:*` edges, and the public
dependency graph are identical to the source repo. No private registry or
`.npmrc` is involved; every dependency resolves from the public npm registry.

## Expected behaviour

Dependabot's npm job for `/` completes and opens PRs for outdated
dependencies. As a sanity check, the repo installs cleanly and quickly
offline and online:

```sh
corepack enable
pnpm install --frozen-lockfile
```

At least one direct dependency has a newer published version than the locked
one, so Dependabot has work to do.

## Observed behaviour

The Dependabot job is cancelled at the fixed 55-minute limit before version
discovery finishes. In the
[reference run](https://github.com/blimmer/pnpm-dependabot-workspace-repro/actions/runs/32261586214/job/96095884095)
(55.5 minutes), only 32 of the 130 unique external direct dependencies had been
checked. Almost all of the time went to exact-version pnpm checks issued by
`VersionResolver` from the workspace root:

| Activity | Completed | Median | Total |
|---|---:|---:|---:|
| Exact-version pnpm checks (`VersionResolver`) | 19 | 158.5s | 47.9m |
| Registry version checks | 32 | 0.3s | — |
| `pnpm --filter . peers check` | 19 | 0.7s | 14.1s |
| `pnpm install --lockfile-only` | 7 | 6.7s | 46.7s |

Extrapolating the observed throughput, a full pass over all 130 dependencies
would take roughly 3h45m. Later runs can be found under
[Actions → Dependabot Updates](https://github.com/blimmer/pnpm-dependabot-workspace-repro/actions/workflows/dependabot/dependabot-updates).

## How to reproduce

1. Fork or push this repository to GitHub.
2. Enable Dependabot version updates (the included `.github/dependabot.yml`
   is picked up automatically).
3. Trigger a check manually from **Insights → Dependency graph → Dependabot**.
4. Watch the job log for the npm update job on directory `/`.
