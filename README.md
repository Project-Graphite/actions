# actions

Reusable GitHub Actions workflows called by Project Graphite repositories.

Project repositories call the workflows here for service detection, checks, image builds, deploy
records, releases, PR conventions and secret scanning, so each step is defined in one place and
changes to it land once.

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

A service that builds from paths outside its folder lists them in `<service>/.build-contexts`, one
`name=path` per line, relative to the repository root:

```
emulator=./gb-emulator
```

`reusable-detect-services` counts a change under a listed path as a change to the service, and
`reusable-docker-build-push` passes each line as a named build context, which the `Dockerfile`
reads with `COPY --from=emulator`.

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
| `reusable-deploy.yml` | Write the new image tag into `platform`, which deploys it |
| `reusable-pr-checks.yml` | Diff-size label, path labels |
| `reusable-gitleaks.yml` | Scan full history for committed secrets |
| `reusable-package-release.yml` | Check an npm package repository, and on a version tag publish it as a GitHub release |
| `reusable-changelog.yml` | Tag the next version from conventional commits and publish it as a GitHub Release |
| `reusable-pipeline.yml` | Optional: the whole pull request and `main` pipeline above, with a boolean switch per stage |

There is no registry workflow any more. The Hub receives the organisation webhook, and a push that
changes a repository's `.graphite.yml` makes it dispatch `platform`'s **Sync registry**; a
`registry.yml` in a project repository is dead and can be deleted.

### reusable-pipeline

| | |
| :--- | :--- |
| Inputs | `all` (`false`), `checks`, `deploy`, `changelog`, `secrets-scan`, `conventions` (all `true`), `node-version` (`24`), `python-version` (`3.14`) |
| Secrets | `PLATFORM_APP_CLIENT_ID`, `PLATFORM_APP_PRIVATE_KEY` (optional; deploy and changelog need them) |
| Caller permissions | `contents: read`, `packages: write`, `issues: write`, `pull-requests: write` |

Call it with `secrets: inherit`, as `project-template` does, or pass the two secrets by name.

On a pull request it runs checks, an image build without push, gitleaks and conventions. On a push
to `main` it runs checks, pushes the images, deploys, publishes the release and runs gitleaks.

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
| Inputs | `services` (required), `node-version` (default `24`), `python-version` (default `3.14`) |

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
| Inputs | `services` (required, e.g. `["app"]`, or `["site"]` for a static site), `tag` (default `sha-<short>` of the pushed commit) |
| Secrets | `PLATFORM_APP_CLIENT_ID`, `PLATFORM_APP_PRIVATE_KEY` |

The slug is the calling repository's name in lower case, so `Graphite-Tracker` reads
`projects/graphite-tracker.yml`. When that file in `platform` declares `deploy.kind: compose` or
`pages` and `deploy/<slug>/` exists there, it writes `IMAGE_TAG=<tag>` as the first line of
`deploy/<slug>/values`, commits `chore(<slug>): deploy <tag>` and pushes to `main`, retrying a
race with another deploy three times. That push runs the platform **Deploy** workflow. Any other
kind, or no deploy folder, records nothing.

It fails if `projects/<slug>.yml` does not exist in `platform` — a repository cannot deploy until
it has a registry entry.

### reusable-pr-checks

| | |
| :--- | :--- |
| Inputs | `xs-max` (10), `s-max` (50), `m-max` (200), `l-max` (600) |

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

### reusable-package-release

| | |
| :--- | :--- |
| Inputs | `node-version` (default `24`), `release` (boolean, default `false`) |

For repositories that ship an npm package from their root instead of services, such as
`graphite-ui`. They have no `Dockerfile`, so `reusable-detect-services` finds nothing; call this
workflow instead.

Every run does `npm ci`, then `lint` and `test` if present, then `build`. Without `release`, it lists
the files `npm pack` would ship.

With `release: true`, call it from a workflow triggered by a `v*` tag. It checks that the tag matches
the `package.json` version, packs the package, attests the tarball's build provenance, and creates
the GitHub release with the tarball attached. Apps install that release asset by URL, so installs
need no registry account or token.

### reusable-changelog

| | |
| :--- | :--- |
| Secrets | `PLATFORM_APP_CLIENT_ID`, `PLATFORM_APP_PRIVATE_KEY` |

Checks out `graphite-hub` and runs its `npm run sync:releases -- --cut` for the calling
repository: it reads the listed projects from `platform/projects/`, tags the next version from the
conventional commits since the last tag (`v1.2.0`, or `<slug>/v1.2.0` for a project inside an
epic), and publishes a GitHub Release with generated notes for each new tag. The Hub records the
release from the organisation webhook. The platform GitHub App's token needs `contents: write` on
`graphite-hub`, `platform` and the calling repository, which receives the tag and the release.

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

See [project-template](https://github.com/project-graphite/project-template) for the one workflow
file every project repository needs: `pipeline.yml`.

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
| publishes a package release (`reusable-package-release` with `release: true`) | `contents: write`, `id-token: write`, `attestations: write` |
| anything else | nothing; the read-only default is enough |

Without the grant, `reusable-pr-checks` fails at startup with no job log and
`reusable-docker-build-push` fails when it pushes. Widening the organisation default to read-write
would also fix it, and is the wrong trade: every workflow in every repository would get write access
it does not need.
