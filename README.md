# actions

Reusable GitHub Actions workflows shared by every repository in Project Graphite.

Project repositories do not define their own CI. They call the workflows here, so the pipeline is
defined in one place and changes to it land once.

## The service convention

**A top-level folder containing a `Dockerfile` is a deployable service.**

```
my-project/
├── frontend/Dockerfile    -> service "frontend"
├── server/Dockerfile      -> service "server"
├── worker/Dockerfile      -> service "worker"
└── docs/                  -> not a service
```

Nothing declares this list. `reusable-detect-services` derives it, and the build, check and deploy
workflows fan out over it. Adding a service means adding a folder with a `Dockerfile`; no CI
change is needed.

A single-service project still puts its code in a folder (`app/`, `api/`), not at the repository
root. That keeps image names uniform: `ghcr.io/project-graphite/<repo>/<service>`.

## Per-service checks

| Folder contains | Runs |
| :--- | :--- |
| `package.json` | `npm ci`, then `lint`, `test` and `build` scripts if present |
| `Makefile` | `make install`, `make lint`, `make test` — all three are required |
| neither | nothing; the image build is the only gate |

A `Makefile` service owns its own environment setup. CI does not guess at pip, poetry or uv.

## Workflows

| Workflow | Purpose |
| :--- | :--- |
| `reusable-detect-services.yml` | Outputs the services changed in this event, as a JSON array |
| `reusable-checks.yml` | Lint, test and build each changed service |
| `reusable-docker-build-push.yml` | Build each changed service and push to GHCR |
| `reusable-deploy.yml` | Record the new image tags in the `platform` repository |
| `reusable-pr-checks.yml` | Conventional PR title, diff-size label, path labels |
| `reusable-gitleaks.yml` | Scan full history for committed secrets |
| `reusable-sync-registry.yml` | Ask the `platform` registry to resync this repository's manifest |

### reusable-detect-services

| | |
| :--- | :--- |
| Inputs | `all` (boolean, default `false`) — select every service, not just changed ones |
| Outputs | `services` — JSON array, e.g. `["frontend","server"]` |

On a pull request it compares against the base commit. On a push it compares against the previous
commit. When that commit is unreachable — a new branch, a force push — it selects every service.

### reusable-checks

| | |
| :--- | :--- |
| Inputs | `services` (required), `node-version` (default `22`), `python-version` (default `3.12`) |

### reusable-docker-build-push

| | |
| :--- | :--- |
| Inputs | `services` (required), `push` (boolean, default `true`) |
| Outputs | `tag` — `sha-<short>`, the tag applied to every image in the run |

Pushes `ghcr.io/<owner>/<repo>/<service>` tagged `sha-<short>` and `latest`. Layer cache is scoped
per service, so one service rebuilding does not invalidate another.

### reusable-deploy

| | |
| :--- | :--- |
| Inputs | `services` (required), `tag` (required) |
| Secrets | `PLATFORM_APP_ID`, `PLATFORM_APP_PRIVATE_KEY` |

Merges the new tags into `tags/<slug>.json` in the `platform` repository and pushes. This records
the deployable release for review; it does not contact Coolify or mutate the VPS.

It fails if `projects/<slug>.yml` does not exist in `platform` — a repository cannot deploy until
it has a registry entry.

### reusable-pr-checks

| | |
| :--- | :--- |
| Inputs | `xs-max` (10), `s-max` (50), `m-max` (200), `l-max` (600) |

The title check requires `type(scope): summary`, the types being `feat fix chore refactor docs test
ci perf revert style build`. Squash merges put the PR title into `main`'s history, which is why the
title is checked rather than the branch name.

Size counts changed lines excluding lockfiles, so a dependency bump is not labelled `size/xl`.

Path labels apply only if the calling repository has a `.github/labeler.yml`.

Size and path labels are skipped on pull requests from forks, whose `GITHUB_TOKEN` cannot write
labels.

### reusable-gitleaks

| | |
| :--- | :--- |
| Inputs | `version` (default `8.30.1`) |

Runs the pinned release binary against full history. The `gitleaks-action` marketplace action needs
a paid licence key for organisation repositories; the binary does not.

### reusable-sync-registry

| | |
| :--- | :--- |
| Secrets | `PLATFORM_APP_CLIENT_ID`, `PLATFORM_APP_PRIVATE_KEY` |

Sends a `sync-registry` repository dispatch to `platform`, which rereads every `.graphite.yml` in
the organisation. Call it on pushes to `main` that touch `.graphite.yml`; the registry also runs a
daily reconcile for repositories that never call it and for repositories that were renamed,
archived or removed.

It dispatches rather than calling the registry workflow directly because the deploy app holds
`contents: write` but not `actions: write`.

## Versioning

Call these workflows at `@main`:

```yaml
uses: project-graphite/actions/.github/workflows/reusable-checks.yml@main
```

There is no version tag. A moving `v1` had to be force-pushed after every change here, which is a
step that gets forgotten, and a stale tag is harder to diagnose than a bad commit. Changes to these
workflows go through review on this repository and reach every caller at once.

A repository that needs to hold back can pin a commit SHA at the call site.

## Calling them

See [project-template](https://github.com/project-graphite/project-template) for the two files
every project repository needs.

## Permissions

The organisation leaves `GITHUB_TOKEN` read-only by default. A reusable workflow cannot request
more than its caller grants, so jobs that call a workflow needing write access must grant it at
the call site:

```yaml
  build:
    permissions:
      contents: read
      packages: write
    uses: project-graphite/actions/.github/workflows/reusable-docker-build-push.yml@main
```

| Calling a workflow that | Grant |
| :--- | :--- |
| pushes images (`reusable-docker-build-push`) | `contents: read`, `packages: write` |
| labels pull requests (`reusable-pr-checks`) | `contents: read`, `issues: write`, `pull-requests: write` |
| anything else | nothing; the read-only default is enough |

Without the grant the run fails at startup with no job log, which is an unhelpful error for a
misleading cause. Widening the organisation default to read-write would also fix it, and is the
wrong trade: every workflow in every repository would get write access it does not need.
