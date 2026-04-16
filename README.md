# Shared-Workflows

A collection of reusable GitHub Actions workflows for common CI/CD tasks.

---

## Vue.js – Reusable Workflows

Two composable reusable workflows are provided for Vue.js projects that run as
Docker containers and are deployed to one or more servers via SSH.

### Workflow overview

```
vue-build.yml          vue-deploy.yml
──────────────         ─────────────────────────────────────
1. Checkout code       1. SSH into each target host
2. docker buildx       2. docker login (Harbor)
3. Push image ──────►  3. docker pull new image
   Harbor registry     4. docker stop/rm old container
                       5. docker run new container
                       6. docker image prune
```

### Files

| Path | Description |
|------|-------------|
| `.github/workflows/vue-build.yml` | Build Vue Docker image and push to Harbor |
| `.github/workflows/vue-deploy.yml` | Deploy image to multiple servers via SSH |
| `docker/vue/Dockerfile` | Multi-stage Dockerfile (Node build → nginx serve) |
| `docker/vue/nginx.conf` | nginx configuration for Vue SPA |

---

## `vue-build.yml` – Build & Push to Harbor

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `harbor_url` | ✅ | – | Harbor registry URL (e.g. `harbor.example.com`) |
| `harbor_project` | ✅ | – | Harbor project / namespace (e.g. `my-org`) |
| `image_name` | ✅ | – | Docker image name (e.g. `my-vue-app`) |
| `image_tag` | | short SHA | Docker image tag |
| `dockerfile_path` | | `Dockerfile` | Path to the Dockerfile |
| `build_context` | | `.` | Docker build context |
| `node_version` | | `20` | Node.js version used during the build |
| `build_args` | | – | Extra `docker build --build-arg` values (one per line) |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `harbor_username` | ✅ | Harbor username |
| `harbor_password` | ✅ | Harbor password / robot-account token |

### Outputs

| Output | Description |
|--------|-------------|
| `image_full_name` | Fully-qualified image reference that was pushed |
| `image_tag` | The tag that was applied to the image |

### Usage example

```yaml
# .github/workflows/ci.yml  (in your Vue project)
name: CI

on:
  push:
    branches: [main]

jobs:
  build:
    uses: mentrd/Shared-Workflows/.github/workflows/vue-build.yml@main
    with:
      harbor_url:     harbor.example.com
      harbor_project: my-org
      image_name:     my-vue-app
      node_version:   "20"
      # Optional multi-line build args
      build_args: |
        VITE_API_URL=https://api.example.com
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}
```

---

## `vue-deploy.yml` – Deploy via SSH

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `harbor_url` | ✅ | – | Harbor registry URL |
| `harbor_project` | ✅ | – | Harbor project / namespace |
| `image_name` | ✅ | – | Docker image name |
| `image_tag` | | `latest` | Docker image tag to deploy |
| `container_name` | ✅ | – | Name for the running container |
| `host_port` | | `80` | Host port exposed on the server |
| `container_port` | | `80` | Port inside the container |
| `env_vars` | | – | Extra env vars for the container (one `KEY=VALUE` per line) |
| `docker_run_extra_args` | | – | Extra flags appended to `docker run` |
| `deploy_hosts` | ✅ | – | Comma-separated list of target hosts (e.g. `10.0.0.1,10.0.0.2`) |
| `ssh_user` | | `deploy` | SSH username |
| `ssh_port` | | `22` | SSH port |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `harbor_username` | ✅ | Harbor username (for `docker pull` on the server) |
| `harbor_password` | ✅ | Harbor password / robot-account token |
| `ssh_private_key` | ✅ | PEM-encoded SSH private key for target servers |

### Usage example

```yaml
# .github/workflows/cd.yml  (in your Vue project)
name: CD

on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: mentrd/Shared-Workflows/.github/workflows/vue-deploy.yml@main
    with:
      harbor_url:     harbor.example.com
      harbor_project: my-org
      image_name:     my-vue-app
      image_tag:      ${{ needs.build.outputs.image_tag }}
      container_name: my-vue-app
      host_port:      "8080"
      container_port: "80"
      deploy_hosts:   "10.0.0.1,10.0.0.2,10.0.0.3"
      ssh_user:       deploy
      env_vars: |
        APP_ENV=production
        API_URL=https://api.example.com
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}
      ssh_private_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

---

## End-to-end example (build then deploy)

You can chain both workflows in a single pipeline file:

```yaml
# .github/workflows/pipeline.yml
name: Build & Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    uses: mentrd/Shared-Workflows/.github/workflows/vue-build.yml@main
    with:
      harbor_url:     harbor.example.com
      harbor_project: my-org
      image_name:     my-vue-app
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}

  deploy:
    needs: build
    uses: mentrd/Shared-Workflows/.github/workflows/vue-deploy.yml@main
    with:
      harbor_url:     harbor.example.com
      harbor_project: my-org
      image_name:     my-vue-app
      image_tag:      ${{ needs.build.outputs.image_tag }}
      container_name: my-vue-app
      deploy_hosts:   "10.0.0.1,10.0.0.2"
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}
      ssh_private_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

---

## Dockerfile usage

Copy `docker/vue/Dockerfile` and `docker/vue/nginx.conf` into the root of your
Vue project.  The Dockerfile accepts a `NODE_VERSION` build argument (default
`20`) so you can pin the exact Node.js version without editing the file:

```bash
docker build --build-arg NODE_VERSION=20 -t my-vue-app .
```

The multi-stage build keeps the final image small – only nginx and the compiled
`dist/` assets are included in the production image.
