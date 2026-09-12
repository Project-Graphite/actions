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

### reusable-detect-services

| | |
| :--- | :--- |
| Inputs | `all` (boolean, default `false`) — select every service, not just changed ones |
| Outputs | `services` — JSON array, e.g. `["frontend","server"]`<br>`any` — `"true"` or `"false"` |

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

Merges the new tags into `tags/<slug>.json` in the `platform` repository and pushes. The reconcile
agent on the host picks the commit up within a minute.

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

### reusable-gitleaks

| | |
| :--- | :--- |
| Inputs | `version` (default `8.30.1`) |

Runs the pinned release binary against full history. The `gitleaks-action` marketplace action needs
a paid licence key for organisation repositories; the binary does not.

## Versioning

Call these workflows at `@v1`, never `@main`:

```yaml
uses: project-graphite/actions/.github/workflows/reusable-checks.yml@v1
```

`v1` moves forward with backwards-compatible changes. A breaking change gets a `v2` tag, and
repositories move to it deliberately. Pinning to `@main` means one bad commit here breaks CI in
every repository at once.

## Calling them

See [project-template](https://github.com/project-graphite/project-template) for the two files
every project repository needs.
