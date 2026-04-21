# component-ci

Shared GitHub Actions and reusable workflows for Keboola component CI/CD pipelines.

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
| `deploy_ready` | No | `false` | Force deploy-ready behavior: push image as `latest` and run the `deploy` job regardless of the computed `is_deploy_ready` |

## Secrets

All secrets should be set as **organization-level secrets** in the Keboola GitHub org. With `secrets: inherit`, each calling repo automatically passes all org secrets to the reusable workflow.

| Secret | Required | Description |
|--------|----------|-------------|
| `DOCKERHUB_USER` | Yes | Docker Hub username for ECR push |
| `DOCKERHUB_TOKEN` | Yes | Docker Hub token for ECR push |
| `KBC_DEVELOPERPORTAL_PASSWORD` | Yes | Developer Portal password |
| `KBC_STORAGE_TOKEN` | If using KBC tests | KBC Storage API token |

The `KBC_DEVELOPERPORTAL_USERNAME` should be an **org-level variable** (`vars.KBC_DEVELOPERPORTAL_USERNAME`).

## Individual Actions

For repos that need custom pipelines (multi-component, multi-identity, etc.), use the actions directly:

### `keboola/component-ci/actions/push-event-info`

Determines image tag, branch info, and deploy-readiness. Follows the Hyperion standard: semantic tags are `1.2.3` (no `v` prefix), branch builds get `-run_number` suffix, deploy requires default branch + semantic tag.

```yaml
- uses: actions/checkout@v4
- uses: keboola/component-ci/actions/push-event-info@master
  id: info
# Outputs: app_image_tag, is_semantic_tag, is_default_branch, is_deploy_ready
```

### `keboola/component-ci/actions/docker-build`

Builds Docker image and uploads as artifact.

```yaml
- uses: actions/checkout@v4
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
- uses: actions/checkout@v4
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
      - uses: actions/checkout@v4
      - uses: keboola/component-ci/actions/push-event-info@master
        id: info

  build:
    needs: info
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
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
