# component-ci

Shared GitHub Actions and reusable workflows for Keboola component CI/CD pipelines.

This repo holds two separate things. Never change one for the other:

| Part | Files | Used by |
|------|-------|---------|
| Per-repo pipeline | `.github/workflows/component-pipeline.yml`, `actions/` | One repository per component (sections below from Quick Start on) |
| Monorepo workflows | `.github/workflows/monorepo-*.yml` | `keboola/components` and monorepos that hold its kit in a subfolder |

## Monorepo workflows

The root CI of `keboola/components` (the kit) lives here once, so a CI change is one PR here instead of one copy per
monorepo. The tools these workflows run (`tools/release`, `tools/component`) stay in the kit; the workflows say only
which tool runs when. What the steps do: the kit's `docs/release.md`.

| Workflow | Caller (in the monorepo) | Does |
|----------|--------------------------|------|
| `monorepo-ci.yml` | `ci` on `pull_request` and `merge_group` | verify the changed folders, release rules |
| `monorepo-test-image.yml` | `test-image` on push to branches other than `main` | build and push test images, run test configs |
| `monorepo-publish.yml` | `publish` on push to `main` and `workflow_dispatch` | release raised versions; a dispatch is a dry run |

Inputs:

| Input | Workflows | Default | Description |
|-------|-----------|---------|-------------|
| `kit-path` | all | `.` | Where the kit is: `.` in the kit itself; a subfolder (e.g. `public`) gets `keboola/components` `main` checked out with the read-only deploy key in the caller's secret `PUBLIC_KIT_DEPLOY_KEY` |
| `folder` | publish | `""` | Folder to dry-run on `workflow_dispatch` |

A caller must:
- pass `secrets: inherit`: the jobs read the caller's repository secrets and variables, and the `ci-<folder>` and
  `release-<vendor>` environments of the caller repo;
- grant `contents: read` and `actions: read` (publish: `contents: read`); a called workflow cannot raise them;
- keep its own `concurrency` group; the workflows here set none, because a shared group cancels the caller;
- keep the `ci-ok` job itself (`needs: ci`, fails unless `needs.ci.result` is `success`), so the required check of
  its merge-queue ruleset keeps the name `ci-ok`. A job of a called workflow reports as `<caller job> / <job>`;
- keep the `PUBLISH_ENABLED` switch on its `publish` job: the repo decides whether a push to `main` releases.

```yaml
jobs:
  ci:
    uses: keboola/component-ci/.github/workflows/monorepo-ci.yml@master
    with:
      kit-path: public   # omit in keboola/components
    secrets: inherit
```

Callers use `@master`, as the kit follows its own `main`: a merge here reaches every monorepo at once, with no bump
PRs. So treat a PR here like a change to the release workflows themselves:
- **Test before merge:** point a caller branch at your branch (`@<branch>`), open it as a draft PR, and show `ci-ok`
  green, a test image built and a publish dry run. Close that PR; never merge a caller that points at a branch.
- **Kit first, then here:** the kit's tools move with the kit's `main`, the workflows with `master` here. A workflow
  change that needs a new or changed tool: merge the kit PR first (the tool must keep working with the workflows
  still on `master`), then the PR here. Removing a tool: first the PR here that stops calling it, then the kit PR.
- **Trust:** whoever can merge here changes what runs with the monorepos' release credentials. Release credentials
  still come only from environments that allow `main` of the caller repo.

## Quick Start (Standard Components)

Replace your entire `push.yml` with:

```yaml
name: CI
on:
  push:
    branches-ignore: [main]
    tags: ["*"]
concurrency: ci-${{ github.ref }}

jobs:
  ci:
    uses: keboola/component-ci/.github/workflows/component-pipeline.yml@master
    with:
      app_id: "kds-team.ex-my-component"
      vendor: "kds-team"
    secrets: inherit
```

### With optional parameters

```yaml
jobs:
  ci:
    uses: keboola/component-ci/.github/workflows/component-pipeline.yml@master
    with:
      app_id: "kds-team.ex-my-component"
      vendor: "kds-team"
      kbc_test_configs: "12345 67890"
      test_command: "python -m pytest tests/ --tb=short -q"
      dockerfile: "./Dockerfile"
      docker_build_target: "production"
    secrets: inherit
```

### Running tests via Docker Compose

Opt into compose-based testing when the test container needs auxiliary services
(e.g. an SFTP server, a database). The pipeline runs
`docker compose up <service> --exit-code-from <service> --build` against the
compose file in your repo. The production image pushed to ECR is still built
separately from your `Dockerfile`.

```yaml
jobs:
  ci:
    uses: keboola/component-ci/.github/workflows/component-pipeline.yml@master
    with:
      app_id: "kds-team.ex-my-component"
      vendor: "kds-team"
      use_docker_compose: true
      # Optional — defaults shown:
      # compose_file: "docker-compose.yml"
      # compose_service: "test"
    secrets: inherit
```

In compose mode the `test_command` and `test_build_target` inputs are ignored —
the test command and build live in the compose service definition.

## Reusable Workflow Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app_id` | Yes | - | Component ID (e.g. `kds-team.ex-airtable`) |
| `vendor` | Yes | - | Developer Portal vendor (e.g. `kds-team`) |
| `kbc_test_configs` | No | `""` | Space-separated KBC test config IDs |
| `test_command` | No | `python -m pytest tests/ --tb=short -q` | Test command to run in container |
| `dockerfile` | No | `./Dockerfile` | Path to Dockerfile |
| `docker_build_target` | No | `""` | Docker build target for multi-stage |
| `context` | No | `.` | Docker build context directory |
| `build_args` | No | `""` | Docker build args (multiline) |
| `update_properties_script` | No | `scripts/developer_portal/update_properties.sh` | Properties update script path |
| `kbc_developerportal_username` | No | `vars.KBC_DEVELOPERPORTAL_USERNAME` | Developer Portal username override |
| `tag` | No | `""` | Override the computed image tag (e.g. `0.0.1`) for one-off runs |
| `update_properties` | No | `false` | Force the `update_properties` job regardless of deploy-readiness |
| `use_docker_compose` | No | `false` | Run tests via `docker compose up` instead of `docker run` (enables test service dependencies like SFTP) |
| `compose_file` | No | `docker-compose.yml` | Path to compose file (used only when `use_docker_compose` is true) |
| `compose_service` | No | `test` | Name of the test service in the compose file (used only when `use_docker_compose` is true) |

## Secrets

All secrets should be set as **organization-level secrets** in the Keboola GitHub org. With `secrets: inherit`, each calling repo automatically passes all org secrets to the reusable workflow.

| Secret | Required | Description |
|--------|----------|-------------|
| `DOCKERHUB_USER` | Yes | Docker Hub username for ECR push |
| `DOCKERHUB_TOKEN` | Yes | Docker Hub token for ECR push |
| `KBC_DEVELOPERPORTAL_PASSWORD` | Yes | Developer Portal password |
| `KBC_STORAGE_TOKEN` | If using KBC tests | KBC Storage API token |

The `KBC_DEVELOPERPORTAL_USERNAME` should be an **org-level variable** (`vars.KBC_DEVELOPERPORTAL_USERNAME`).

### External vendors (repositories outside the Keboola GitHub org)

`secrets: inherit` only passes secrets when the caller workflow lives in the
**same GitHub organization** as the reusable workflow. Repositories in other
organizations must pass the secrets **explicitly** and provide the Developer
Portal username via the `kbc_developerportal_username` input:

```yaml
jobs:
  ci:
    uses: keboola/component-ci/.github/workflows/component-pipeline.yml@master
    with:
      app_id: "my-vendor.ex-my-component"
      vendor: "my-vendor"
      kbc_developerportal_username: "my-vendor+github_actions"
    secrets:
      DOCKERHUB_USER: ${{ secrets.DOCKERHUB_USER }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
      KBC_DEVELOPERPORTAL_PASSWORD: ${{ secrets.KBC_DEVELOPERPORTAL_PASSWORD }}
      KBC_STORAGE_TOKEN: ${{ secrets.KBC_STORAGE_TOKEN }}
```

Set these as repository (or your own organization's) secrets. Environment-scoped
secrets are not picked up here — `on.workflow_call` does not support the
`environment` keyword.

## Individual Actions

For repos that need custom pipelines (multi-component, multi-identity, etc.), use the actions directly:

### `keboola/component-ci/actions/push-event-info`

Determines image tag, branch info, and deploy-readiness. Follows the Hyperion standard: semantic tags are `1.2.3` (no `v` prefix), branch builds get `-run_number` suffix, deploy requires default branch + semantic tag.

```yaml
- uses: actions/checkout@v7
- uses: keboola/component-ci/actions/push-event-info@master
  id: info
# Outputs: app_image_tag, is_semantic_tag, is_default_branch, is_deploy_ready
```

### `keboola/component-ci/actions/docker-build`

Builds Docker image and uploads as artifact.

```yaml
- uses: actions/checkout@v7
- uses: keboola/component-ci/actions/docker-build@master
  with:
    app_id: "kds-team.ex-my-component"
    # Optional: dockerfile, context, build_target, build_args
```

### `keboola/component-ci/actions/docker-load`

Downloads and loads the Docker image artifact.

```yaml
- uses: keboola/component-ci/actions/docker-load@master
  with:
    app_id: "kds-team.ex-my-component"
```

### `keboola/component-ci/actions/run-tests`

Runs tests inside the Docker container. Defaults to pytest.

```yaml
- uses: keboola/component-ci/actions/run-tests@master
  with:
    app_id: "kds-team.ex-my-component"
    test_command: "python -m pytest tests/ --tb=short -q"  # default
```

### `keboola/component-ci/actions/kbc-tests`

Runs KBC platform integration tests (skipped if configs or token are empty).

```yaml
- uses: keboola/component-ci/actions/kbc-tests@master
  with:
    app_id: "kds-team.ex-my-component"
    image_tag: ${{ needs.info.outputs.app_image_tag }}
    kbc_test_configs: "12345 67890"
    kbc_storage_token: ${{ secrets.KBC_STORAGE_TOKEN }}
```

### `keboola/component-ci/actions/push-to-ecr`

Pushes Docker image to Keboola ECR.

```yaml
- uses: keboola/component-ci/actions/push-to-ecr@master
  with:
    app_id: "kds-team.ex-my-component"
    vendor: "kds-team"
    image_tag: ${{ needs.info.outputs.app_image_tag }}
    is_deploy_ready: ${{ needs.info.outputs.is_deploy_ready }}
    dockerhub_user: ${{ secrets.DOCKERHUB_USER }}
    dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}
    kbc_developerportal_username: ${{ vars.KBC_DEVELOPERPORTAL_USERNAME }}
    kbc_developerportal_password: ${{ secrets.KBC_DEVELOPERPORTAL_PASSWORD }}
```

### `keboola/component-ci/actions/deploy`

Sets release tag in Developer Portal.

```yaml
- uses: keboola/component-ci/actions/deploy@master
  with:
    app_id: "kds-team.ex-my-component"
    vendor: "kds-team"
    image_tag: ${{ needs.info.outputs.app_image_tag }}
    kbc_developerportal_username: ${{ vars.KBC_DEVELOPERPORTAL_USERNAME }}
    kbc_developerportal_password: ${{ secrets.KBC_DEVELOPERPORTAL_PASSWORD }}
```

### `keboola/component-ci/actions/update-properties`

Updates component properties in Developer Portal.

```yaml
- uses: actions/checkout@v7
- uses: keboola/component-ci/actions/update-properties@master
  with:
    app_id: "kds-team.ex-my-component"
    vendor: "kds-team"
    kbc_developerportal_username: ${{ vars.KBC_DEVELOPERPORTAL_USERNAME }}
    kbc_developerportal_password: ${{ secrets.KBC_DEVELOPERPORTAL_PASSWORD }}
```

## Examples

### Multi-component monorepo (like motherduck/kafka)

```yaml
name: ex-motherduck
on:
  push:
    paths: ['components/ex-motherduck/**', '.github/workflows/**']
    branches: ['*']
    tags: ['*']
concurrency: ci-ex-motherduck-${{ github.ref }}

jobs:
  ci:
    uses: keboola/component-ci/.github/workflows/component-pipeline.yml@master
    with:
      app_id: "keboola.ex-motherduck"
      vendor: "keboola"
      dockerfile: "./components/ex-motherduck/Dockerfile"
      context: "."
    secrets: inherit
```

### Multi-identity component (like meta: 1 image, 3 app IDs)

Use individual actions to build once and push/deploy to multiple component IDs:

```yaml
jobs:
  info:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.info.outputs.app_image_tag }}
      deploy: ${{ steps.info.outputs.is_deploy_ready }}
    steps:
      - uses: actions/checkout@v7
      - uses: keboola/component-ci/actions/push-event-info@master
        id: info

  build:
    needs: info
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: keboola/component-ci/actions/docker-build@master
        with:
          app_id: "keboola.ex-facebook-pages"

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: keboola/component-ci/actions/docker-load@master
        with:
          app_id: "keboola.ex-facebook-pages"
      - uses: keboola/component-ci/actions/run-tests@master
        with:
          app_id: "keboola.ex-facebook-pages"

  push:
    needs: [info, test]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        app_id:
          - keboola.ex-facebook-pages
          - keboola.ex-facebook-ads-v2
          - keboola.ex-instagram-v2
    steps:
      - uses: keboola/component-ci/actions/docker-load@master
        with:
          app_id: "keboola.ex-facebook-pages"
      - uses: keboola/component-ci/actions/push-to-ecr@master
        with:
          app_id: ${{ matrix.app_id }}
          vendor: "keboola"
          image_tag: ${{ needs.info.outputs.tag }}
          is_deploy_ready: ${{ needs.info.outputs.deploy }}
          dockerhub_user: ${{ secrets.DOCKERHUB_USER }}
          dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}
          kbc_developerportal_username: ${{ vars.KBC_DEVELOPERPORTAL_USERNAME }}
          kbc_developerportal_password: ${{ secrets.KBC_DEVELOPERPORTAL_PASSWORD }}

  deploy:
    needs: [info, push]
    if: needs.info.outputs.deploy == 'true'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        app_id:
          - keboola.ex-facebook-pages
          - keboola.ex-facebook-ads-v2
          - keboola.ex-instagram-v2
    steps:
      - uses: keboola/component-ci/actions/deploy@master
        with:
          app_id: ${{ matrix.app_id }}
          vendor: "keboola"
          image_tag: ${{ needs.info.outputs.tag }}
          kbc_developerportal_username: ${{ vars.KBC_DEVELOPERPORTAL_USERNAME }}
          kbc_developerportal_password: ${{ secrets.KBC_DEVELOPERPORTAL_PASSWORD }}
```
