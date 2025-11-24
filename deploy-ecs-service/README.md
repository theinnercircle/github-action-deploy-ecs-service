# Deploy ECS Service - Composite Action

A flexible, reusable GitHub composite action for deploying ECS services across all Inner Circle projects.

## Features

- ✅ **Single container deployments** (e.g., node/sockets, cron workers)
- ✅ **Multi-container deployments** (e.g., webserver + app)
- ✅ **Pre-deployment migrations** (e.g., database migrations, cache warming)
- ✅ **Smart desired count management** (auto-scale from 0)
- ✅ **Service stability waiting**
- ✅ **Flexible naming conventions**

## Usage

### Pattern 1: Single Container (Worker-only)

**Example: node/sockets service**

```yaml
- name: "Deploy Sockets Service"
  uses: theinnercircle/cicd/github-actions/deploy-ecs-service@main
  with:
    environment: test1
    service: sockets
    aws_region: eu-central-1
    aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
    containers: |
      [
        {
          "name": "ic-sockets",
          "image": "ic-sockets",
          "image_tag": "${{ needs.build.outputs.DOCKER_IMAGE_SHA }}"
        }
      ]
    wait_for_stability: false
```

### Pattern 2: Multi-Container (Webserver + App)

**Example: theic/backend service**

```yaml
- name: "Deploy Backend Service"
  uses: theinnercircle/cicd/github-actions/deploy-ecs-service@main
  with:
    environment: test1
    service: backend
    aws_region: eu-central-1
    aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
    containers: |
      [
        {
          "name": "ic-backend-webserver",
          "image": "ic-app-webserver",
          "image_tag": "${{ needs.build.outputs.DOCKER_IMAGE_SHA }}"
        },
        {
          "name": "ic-backend",
          "image": "ic-app",
          "image_tag": "${{ needs.build.outputs.DOCKER_IMAGE_SHA }}"
        }
      ]
    manage_desired_count: true
    wait_for_stability: true
```

**Example: imagick/imgresize service**

```yaml
- name: "Deploy Imgresize Service"
  uses: theinnercircle/cicd/github-actions/deploy-ecs-service@main
  with:
    environment: test1
    service: imgresize
    aws_region: eu-central-1
    aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
    containers: |
      [
        {
          "name": "ic-imgresize-webserver",
          "image": "ic-imgresize-webserver",
          "image_tag": "${{ needs.build.outputs.DOCKER_IMAGE_SHA }}"
        },
        {
          "name": "ic-imgresize",
          "image": "ic-imgresize",
          "image_tag": "${{ needs.build.outputs.DOCKER_IMAGE_SHA }}"
        }
      ]
    wait_for_stability: false
```

### Pattern 3: Multi-Container with Migrations

**Example: admin-panel/adminpanel service**

```yaml
- name: "Deploy Admin Panel Service"
  uses: theinnercircle/cicd/github-actions/deploy-ecs-service@main
  with:
    environment: test1
    service: adminpanel
    aws_region: eu-central-1
    aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
    containers: |
      [
        {
          "name": "ic-adminpanel-webserver",
          "image": "ic-adminpanel-webserver",
          "image_tag": "${{ needs.build.outputs.DOCKER_IMAGE_SHA }}"
        },
        {
          "name": "ic-adminpanel",
          "image": "ic-adminpanel",
          "image_tag": "${{ needs.build.outputs.DOCKER_IMAGE_SHA }}"
        }
      ]
    run_migrations: true
    migration_image: ic-adminpanel
    migration_image_tag: ${{ needs.build.outputs.DOCKER_IMAGE_SHA }}
    migration_container_name: ic-task
    migration_command: php artisan migrate --force
    manage_desired_count: true
    wait_for_stability: true
    propagate_tags: SERVICE
```

### Pattern 4: Worker with Desired Count Management

**Example: theic/cron service**

```yaml
- name: "Deploy Cron Service"
  uses: theinnercircle/cicd/github-actions/deploy-ecs-service@main
  with:
    environment: test1
    service: cron
    aws_region: eu-central-1
    aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
    containers: |
      [
        {
          "name": "ic-cron",
          "image": "ic-cli",
          "image_tag": "${{ needs.build.outputs.DOCKER_IMAGE_SHA }}"
        }
      ]
    manage_desired_count: true
    wait_for_stability: true
```

## Inputs

### Core Configuration

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `environment` | ✅ | - | Target environment (e.g., `test1`, `acceptance1`) |
| `service` | ✅ | - | Service name (e.g., `backend`, `sockets`) |
| `cluster_name` | ❌ | `theic-{environment}` | ECS cluster name |
| `service_name` | ❌ | `theic-{environment}-{service}` | ECS service name |

### AWS Configuration

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `aws_region` | ✅ | - | AWS region |
| `aws_account_id` | ✅ | - | AWS account ID |

### Container Configuration

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `containers` | ✅ | - | JSON array of containers (see examples) |

### Migration Configuration

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `run_migrations` | ❌ | `false` | Whether to run migrations |
| `migration_task_definition` | ❌ | `{service}-task` | Migration task definition name |
| `migration_container_name` | ❌ | `ic-task` | Container name in migration task |
| `migration_command` | ❌ | `php artisan migrate --force` | Command to run |
| `migration_image` | ❌ | - | Image for migrations (without tag) |
| `migration_image_tag` | ❌ | - | Image tag for migrations |

### Deployment Options

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `manage_desired_count` | ❌ | `false` | Auto-scale from 0 to 1 |
| `wait_for_stability` | ❌ | `true` | Wait for service to stabilize |
| `propagate_tags` | ❌ | `NONE` | Propagate tags (`TASK_DEFINITION`, `SERVICE`, or `NONE`) |

## Outputs

| Output | Description |
|--------|-------------|
| `task-definition-arn` | ARN of the deployed task definition |
| `desired-count` | Desired count used (if `manage_desired_count: true`) |

## Complete Workflow Example

```yaml
name: "Deploy to Test Environment"

on:
  pull_request:
    types: [labeled]

jobs:
  build:
    # ... build and push docker images

  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [backend, admin, sockets]
        include:
          - service: backend
            containers: |
              [
                {"name": "ic-backend-webserver", "image": "ic-app-webserver", "image_tag": "sha-abc123"},
                {"name": "ic-backend", "image": "ic-app", "image_tag": "sha-abc123"}
              ]
            manage_desired_count: true
          - service: admin
            containers: |
              [
                {"name": "ic-admin-webserver", "image": "ic-app-webserver", "image_tag": "sha-abc123"},
                {"name": "ic-admin", "image": "ic-app", "image_tag": "sha-abc123"}
              ]
            manage_desired_count: true
          - service: sockets
            containers: |
              [
                {"name": "ic-sockets", "image": "ic-sockets", "image_tag": "sha-abc123"}
              ]
            manage_desired_count: false

    steps:
      - name: "Checkout"
        uses: actions/checkout@v4
        with:
          repository: theinnercircle/cicd
          path: .cicd

      - name: "Configure AWS Credentials"
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/theic-test1-${{ matrix.service }}-ci-cd
          aws-region: eu-central-1

      - name: "Deploy Service"
        uses: ./.cicd/github-actions/deploy-ecs-service
        with:
          environment: test1
          service: ${{ matrix.service }}
          aws_region: eu-central-1
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
          containers: ${{ matrix.containers }}
          manage_desired_count: ${{ matrix.manage_desired_count }}
```

## Migration from Existing Workflows

### Before (theic service)
```yaml
- name: "Render webserver container"
  id: render-web
  if: ${{ matrix.has_webserver }}
  uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition-arn: ...
    container-name: ic-backend-webserver
    image: ...

- name: "Render app container"
  id: render-app
  uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition: ${{ steps.render-web.outputs.task-definition }}
    container-name: ic-backend
    image: ...

- name: "Get desired count"
  # ... 15 lines of bash

- name: "Deploy"
  uses: aws-actions/amazon-ecs-deploy-task-definition@v2
  # ...
```

### After
```yaml
- name: "Deploy Service"
  uses: theinnercircle/cicd/github-actions/deploy-ecs-service@main
  with:
    environment: test1
    service: backend
    aws_region: eu-central-1
    aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
    containers: |
      [
        {"name": "ic-backend-webserver", "image": "ic-app-webserver", "image_tag": "sha-abc123"},
        {"name": "ic-backend", "image": "ic-app", "image_tag": "sha-abc123"}
      ]
    manage_desired_count: true
```

**Result: ~60 lines → ~10 lines per service** 🎉

## Development

### Testing

Test the action locally using [act](https://github.com/nektos/act) or in a test environment before deploying to production.

### Versioning

Tag releases for stability:
```bash
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0
```

Then reference in workflows:
```yaml
uses: theinnercircle/cicd/github-actions/deploy-ecs-service@v1.0.0
```

## Troubleshooting

### Container not found in task definition

Ensure the `name` in the containers JSON matches exactly the container name in your ECS task definition.

### Migration task fails

Check CloudWatch logs for the migration task. Ensure the migration command is correct and the task has necessary permissions.

### Service doesn't scale from 0

Ensure `manage_desired_count: true` is set. Without this, services scaled to 0 will remain at 0.

## Support

For issues or questions, contact the DevOps team or open an issue in the `cicd` repository.