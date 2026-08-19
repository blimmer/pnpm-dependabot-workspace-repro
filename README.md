# pnpm-dependabot-workspace-repro

Minimal-as-possible reproduction of a Dependabot timeout on a large pnpm
workspace. Filed upstream at: _TODO: link to dependabot/dependabot-core issue_.

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

The Dependabot job times out before producing any pull requests.

_TODO: paste the relevant tail of the Dependabot job log (Insights → Dependency
graph → Dependabot → "Last checked" → View logs), including the final timeout
line and elapsed time._

## How to reproduce

1. Fork or push this repository to GitHub.
2. Enable Dependabot version updates (the included `.github/dependabot.yml`
   is picked up automatically).
3. Trigger a check manually from **Insights → Dependency graph → Dependabot**.
4. Watch the job log for the npm update job on directory `/`.

## Notes for maintainers

- The large workspace is load-bearing for the reproduction; shrinking it to a
  handful of packages makes the job succeed.
- `@aws-sdk/client-ssm` and `drizzle-orm` are intentionally pinned to exact
  versions in the catalog; that matches the source repo.
- `packageManager` in the root `package.json` pins pnpm `11.22.0`, which is
  the version the lockfile was generated with.
