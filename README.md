# renovate-config

[![CI/CD](https://img.shields.io/github/actions/workflow/status/KevinDeBenedetti/renovate-config/ci.yml?style=for-the-badge&label=CI%2FCD)](https://github.com/KevinDeBenedetti/renovate-config/actions/workflows/ci.yml) [![License](https://img.shields.io/github/license/KevinDeBenedetti/renovate-config?style=for-the-badge)](LICENSE)

> A shared [Renovate](https://docs.renovatebot.com/) preset that any repository can extend to get Renovate's best practices, one weekly update window and a 3-day soak on every release.

## Features

- [`config:best-practices`](https://docs.renovatebot.com/presets-config/#configbest-practices) as the base: digests pinned for Docker images and GitHub Actions, config migration, abandoned-package detection, weekly lock file maintenance
- One update window a week, Monday before 6am (Europe/Paris), with no hourly cap on pull requests
- A 3-day [minimum release age](https://docs.renovatebot.com/key-concepts/minimum-release-age/) on every update, whatever the ecosystem, with security fixes (GitHub and OSV advisories) exempt from it and from the schedule
- prek hook revs kept current, from a `# renovate:` comment above each `rev` in `prek.toml`
- Majors kept apart: one pull request per major, and majors of data-owning images (`postgres`, `redis`, `valkey`, `mariadb`, `mysql`, `mongo`, `qdrant`, `seaweedfs`, `minio`) held for approval on the Dependency Dashboard
- Labels by kind (`dependencies`, `security`, `major`, `github-actions`, `prek`, `data-migration`) and the `ci` commit type for GitHub Actions bumps

## Prerequisites

- Renovate on the repository: the [Mend Renovate app](https://github.com/apps/renovate) or a [self-hosted run](https://docs.renovatebot.com/getting-started/running/)

## Usage

`renovate.json` at the root of the repository:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>KevinDeBenedetti/renovate-config"]
}
```

Renovate reads [`default.json`](default.json) from this repository's default branch: a change here reaches every repository that extends it at its next run. Each rule there carries a `description` of its reason.

The repository's own [settings](https://docs.renovatebot.com/configuration-options/) come after the preset's, so they win — another timezone, a daily window, release dates for images outside Docker Hub:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>KevinDeBenedetti/renovate-config"],
  "timezone": "America/New_York",
  "schedule": ["* 0-5 * * *"],
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "matchPackageNames": ["ghcr.io/**"],
      "minimumReleaseAgeBehaviour": "timestamp-optional"
    }
  ]
}
```

Check the config before a push:

```sh
npx --yes --package renovate -- renovate-config-validator --strict
```

## Caveats

- **Schedule.** Pull requests open only inside the window, security fixes aside: a self-hosted run has to start there, or it opens no update until the next Monday. The [Mend Renovate app](https://docs.renovatebot.com/mend-hosted/job-scheduling/) picks its own times: every 4 hours at best, daily until a Renovate pull request is merged, so a 6-hour window can pass with no run. With the app, widen the window to the whole day, lock file maintenance included: `"schedule": ["* * * * 1"]` and `"lockFileMaintenance": { "schedule": ["* * * * 1"] }`.
- **Cargo.** `config:best-practices` pins dev-dependencies, Cargo's too: `serde_json = "1"` would become `=1.0.x` while every other crate keeps its range. A Rust repository adds a rule with `"matchManagers": ["cargo"]` and `"rangeStrategy": "update-lockfile"` to move `Cargo.lock` only.
- **Release dates.** The soak needs a release timestamp, and Docker images get one from Docker Hub only: an image from another registry never passes it unless a rule sets `minimumReleaseAgeBehaviour` to `timestamp-optional`, as above.
- **prek.** A `rev` is updated only with the comment on the line above it: `# renovate: datasource=github-tags depName=<owner>/<repo>`, any datasource, with optional `packageName=`, `versioning=` and `extractVersion=`. The `rev` is a tag (`rev = "v1.2.3"`) or a commit frozen to its tag (`rev = "<sha>"  # frozen: v1.2.3`, as `prek update --freeze` writes it), whose commit and comment move together.
- **Examples.** `config:best-practices` ignores `**/examples/**`: set your own `ignorePaths` to update the pins there.

## Git hooks

The [prek](https://prek.j178.dev) hooks of `prek.toml` check file hygiene and the workflow, and validate `default.json` with `renovate-config-validator --strict --no-global`. CI runs them on every pull request and every push to `main`.

```sh
prek install           # once after cloning
prek run --all-files
```

## License

MIT — see [LICENSE](LICENSE).
