# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) preset for the KevinDeBenedetti repositories.

## Usage

In each repository, `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>KevinDeBenedetti/renovate-config"]
}
```

`github>KevinDeBenedetti/renovate-config` loads [`default.json`](default.json) from this repository's default branch
([preset hosting](https://docs.renovatebot.com/config-presets/#github-hosted-presets)). Add a tag (`#v1.0.0`) to pin it.

Renovate runs from GitHub Actions through the reusable workflow
`KevinDeBenedetti/github-workflows/.github/workflows/renovate.yml`. This repository is public, so the App token that
workflow mints for the calling repository can read the preset.

## What `default.json` sets

| Setting | Why |
| --- | --- |
| `config:best-practices` | Renovate's own recommendation: `config:recommended` plus digest pinning (Docker, GitHub Actions), config migration PRs, pinned dev dependencies, abandoned-package detection, npm release age, weekly lock file maintenance. |
| `:separateMultipleMajorReleases` | One PR per major when several are behind. |
| `timezone` Europe/Paris, `schedule` `* 0-5 * * 1` | One window, Monday before 6am (cron syntax, which the docs recommend over the deprecated text syntax). The caller workflow must run inside it. |
| `minimumReleaseAge` 3 days for every package | A release pulled within days never reaches a PR. Update types without a release timestamp (lock file maintenance, pins, replacements…) are exempt, as in Renovate's `security:minimumReleaseAge*` presets. Security updates never wait. |
| `osvVulnerabilityAlerts` | Security PRs from the OSV database: the App token has no access to Dependabot alerts. |
| prek custom manager | Updates `rev = "…"` in any `prek.toml` under a `# renovate: datasource=… depName=…` comment (optional `packageName=`, `versioning=`, `extractVersion=`). |
| github-workflows pins disabled | Its `propagate-release` job opens those PRs. |
| Labels | `dependencies` on every PR; `github-actions`, `prek`, `major`, `security`, `data-migration` where they apply. GitHub Actions bumps are committed as `ci(deps)`. |
| Data-owning images | A major of postgres, redis, valkey, mariadb, mysql, mongo, qdrant, seaweedfs or minio waits for approval on the Dependency Dashboard and is never automerged. |

A repository's own `packageRules` come after the preset's, so they override it (for instance `"minimumReleaseAge": null`
for its own images, or a daily `schedule`).

## Migrating each repository

Remove from each `renovate.json` whatever the preset now sets, keep the rest.

| Repository | Today | Drop | Keep / add |
| --- | --- | --- | --- |
| dotfiles | Renovate | `config:recommended`, `schedule`, the prek.toml manager, the github-workflows rule | The `.prototools` and `setup-*.sh` managers, the groups, the SHA-256 notes, the kubectl rule. Its `schedule` was not in Europe/Paris before: the window now is. |
| github-workflows | Renovate | `extends` items, `timezone`, `schedule`, `configMigration`, `minimumReleaseAge`, `internalChecksFilter`, the prek manager (the preset's matches `examples/prek.toml` too), the self-reference rule, the GitHub Actions label rule | `enabledManagers` (it lists `custom.regex`, so the prek manager still runs), the other managers and rules. Update the header of `.github/workflows/renovate.yml`, which still points to `renovate/default.json`. |
| infra | Renovate | `extends` items, `timezone`, `configMigration`, the prek manager, the 3-day soak rule, the github-workflows rule, the GitHub Actions and major label rules, the data-migration rule | `"schedule": ["* 0-5 * * *"]` (daily), and everything specific to the cluster. `docker:pinDigests` is new for the images that have no digest yet. |
| kevindebenedetti.github.io | Renovate | `extends` items, `timezone`, `schedule`, `configMigration`, `minimumReleaseAge`, `internalChecksFilter`, the first prek manager, the github-workflows rule, the label rules | The oxlint manager and group, the bun `rangeStrategy: bump` rule (it overrides `:pinDevDependencies`), the TypeScript cap. |
| notes | Extends `github>KevinDeBenedetti/github-workflows` | The whole `extends` | `github>KevinDeBenedetti/renovate-config`. Today it reads github-workflows' `renovate.json` as a preset, a fallback Renovate has deprecated. |
| dataset-generator | Dependabot | `.github/dependabot.yml` | Caller workflow, then rules: `huggingface-hub` majors disabled; `python` and `node` Docker majors and minors disabled; `group:allNonMajor` for the single weekly PR. The compose images (postgres, redis, qdrant) are already gated by the preset. |
| k3s-lab | Dependabot | `.github/dependabot.yml` | Caller workflow, `"enabledManagers": ["github-actions", "custom.regex"]`, `"groupName": "actions"` for `matchManagers: ["github-actions"]`. |
| koaching | Dependabot | `.github/dependabot.yml` | Caller workflow, `group:allNonMajor`, `"prConcurrentLimit": 1`; disable `dockerfile` and `docker-compose` (`make pin-images` owns those digests). Add `# renovate:` comments in `prek.toml`. |
| portfolio-next | Dependabot | `.github/dependabot.yml` | Caller workflow, `"baseBranchPatterns": ["dev"]`, one group per manager; group the Bun image with `packageManager` (`matchPackageNames: ["oven/bun", "bun"]`). postgres and redis majors are gated by the preset. |
| portfolio-nuxt | Dependabot | `.github/dependabot.yml` | Caller workflow, `"reviewers": ["KevinDeBenedetti"]`, `group:allNonMajor`. `fast-xml-parser` was ignored only because Dependabot cannot update pnpm transitive dependencies; Renovate's lock file maintenance does. |
| web-check | Dependabot | `.github/dependabot.yml` | Caller workflow, `"rangeStrategy": "update-lockfile"` for `pep621` (Dependabot's `increase-if-necessary`), minor+patch groups per manager, major groups for Vite (`vite`, `vitest`, `@vitejs/*`) and React, `node` Docker majors disabled. |

Repositories with no dependency tool yet (agentic-hub, feedwise, finance, tools, van-conversion, reactive-resume…) need
only the caller workflow and the three-line `renovate.json` above. Their `prek.toml` pins need `# renovate:` comments
before Renovate sees them.

Check a migrated config before pushing:

```bash
npx --yes --package renovate -- renovate-config-validator --strict
```

## Developing this preset

```bash
npx --yes --package renovate -- renovate-config-validator --strict --no-global default.json
```

`--no-global` validates the file as a repository config, not as self-hosted global config.
