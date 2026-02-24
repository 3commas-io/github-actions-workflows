# GitHub Actions Workflows

Reusable GitHub Actions workflows for 3commas projects. These workflows provide consistent CI/CD patterns across all repositories with automatic secret inheritance.

## Available Workflows

### 1. prepare-environment.yml

Extracts version information and determines the deployment environment (staging/production) based on git ref.

**Inputs:**
- `image_repository` (required): Base image repository path (e.g., `3commas/go-helloworld`)
- `build_dependencies` (optional, default: `false`): Whether to build a dependencies image
- `dependencies_dockerfile` (optional, default: `./Dockerfile.dependencies`): Path to dependencies Dockerfile

**Outputs:**
- `version`: Extracted version (e.g., `main-a1b2c3d` for branches, `v0.0.1` for tags)
- `registry`: Nexus registry URL (`nexus-docker-hosted.stg.3csb.com` or `nexus-docker-hosted.prd.3csb.com`)
- `proxy_url`: Proxy URL for Docker builds
- `docker_mirror`: Docker mirror URL
- `image_tag`: Full dependencies image tag (only if `build_dependencies=true`)

**Secrets Required:**
- `NEXUS_USERNAME`: Nexus registry username (only if `build_dependencies=true`)
- `NEXUS_PASSWORD`: Nexus registry password (only if `build_dependencies=true`)

**Example:**
```yaml
jobs:
  prepare:
    uses: 3commas-io/github-actions-workflows/.github/workflows/prepare-environment.yml@main
    with:
      image_repository: '3commas/go-helloworld'
    secrets: inherit
```

**Example with dependencies build:**
```yaml
jobs:
  prepare:
    uses: 3commas-io/github-actions-workflows/.github/workflows/prepare-environment.yml@main
    with:
      image_repository: '3commas/quantpilot-frontend'
      build_dependencies: true
      dependencies_dockerfile: './Dockerfile.dependencies'
    secrets: inherit
```

### 2. build-push-image.yml

Builds and pushes a Docker image to Nexus registry with consistent tagging.

**Inputs:**
- `registry` (required): Nexus registry URL
- `image_repository` (required): Base image repository path
- `service_name` (required): Service name (becomes part of image path, e.g., `main`, `api`, `web`)
- `version` (required): Version tag
- `proxy_url` (required): Proxy URL
- `docker_mirror` (required): Docker mirror URL
- `dockerfile` (required): Path to Dockerfile
- `runner` (optional, default: `sb-staging-arm-arc`): GitHub Actions runner to use
- `env_file` (optional): Path to env file for build args
- `additional_build_args` (optional): Extra build args as multiline string
- `deps_image_tag` (optional): Dependencies image tag to use as `IMAGE_TAG` build arg

**Secrets Required:**
- `NEXUS_USERNAME`: Nexus registry username
- `NEXUS_PASSWORD`: Nexus registry password

**Image Tagging:**
- Creates two tags: `<version>` and `latest`
- Images are pushed to: `<registry>/<image_repository>/<service_name>:<tag>`

**Example:**
```yaml
jobs:
  build:
    needs: prepare
    uses: 3commas-io/github-actions-workflows/.github/workflows/build-push-image.yml@main
    with:
      registry: ${{ needs.prepare.outputs.registry }}
      image_repository: '3commas/go-helloworld'
      service_name: 'main'
      version: ${{ needs.prepare.outputs.version }}
      proxy_url: ${{ needs.prepare.outputs.proxy_url }}
      docker_mirror: ${{ needs.prepare.outputs.docker_mirror }}
      dockerfile: './Dockerfile'
      runner: ${{ startsWith(github.ref, 'refs/tags/v') && 'sb-production-arm-arc' || 'sb-staging-arm-arc' }}
    secrets: inherit
```

**Example with matrix strategy:**
```yaml
jobs:
  build:
    needs: prepare
    strategy:
      matrix:
        include:
          - service_name: main
            dockerfile: ./Dockerfile
          - service_name: producer
            dockerfile: ./Dockerfile.producer
    uses: 3commas-io/github-actions-workflows/.github/workflows/build-push-image.yml@main
    with:
      registry: ${{ needs.prepare.outputs.registry }}
      image_repository: '3commas/go-helloworld'
      service_name: ${{ matrix.service_name }}
      version: ${{ needs.prepare.outputs.version }}
      proxy_url: ${{ needs.prepare.outputs.proxy_url }}
      docker_mirror: ${{ needs.prepare.outputs.docker_mirror }}
      dockerfile: ${{ matrix.dockerfile }}
      runner: ${{ startsWith(github.ref, 'refs/tags/v') && 'sb-production-arm-arc' || 'sb-staging-arm-arc' }}
    secrets: inherit
```

## Environment Detection

Both workflows automatically detect the environment based on the git ref:

- **Production**: Triggered by tags matching `v*` (e.g., `v0.0.1`, `v1.2.3`)
  - Uses production Nexus registry: `nexus-docker-hosted.prd.3csb.com`
  - Uses production proxy: `squid.prd.3csb.com:3128`
  - Version tag keeps the `v` prefix

- **Staging**: Triggered by branches (e.g., `main`, `develop`)
  - Uses staging Nexus registry: `nexus-docker-hosted.stg.3csb.com`
  - Uses staging proxy: `squid.stg.3csb.com:3128`
  - Version format: `<branch>-<short-sha>` (e.g., `main-a1b2c3d`)

## Complete Example

Here's a complete workflow using both reusable workflows:

```yaml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

jobs:
  prepare:
    uses: 3commas-io/github-actions-workflows/.github/workflows/prepare-environment.yml@main
    with:
      image_repository: '3commas/go-helloworld'
    secrets: inherit

  build:
    needs: prepare
    strategy:
      matrix:
        include:
          - service_name: main
            dockerfile: ./Dockerfile
    uses: 3commas-io/github-actions-workflows/.github/workflows/build-push-image.yml@main
    with:
      registry: ${{ needs.prepare.outputs.registry }}
      image_repository: '3commas/go-helloworld'
      service_name: ${{ matrix.service_name }}
      version: ${{ needs.prepare.outputs.version }}
      proxy_url: ${{ needs.prepare.outputs.proxy_url }}
      docker_mirror: ${{ needs.prepare.outputs.docker_mirror }}
      dockerfile: ${{ matrix.dockerfile }}
      runner: ${{ startsWith(github.ref, 'refs/tags/v') && 'sb-production-arm-arc' || 'sb-staging-arm-arc' }}
    secrets: inherit
```

## Benefits

- ✅ **Automatic secret inheritance** - no need to explicitly pass secrets
- ✅ **Centralized maintenance** - update once, all projects benefit
- ✅ **Consistent patterns** - all projects follow the same CI/CD approach
- ✅ **Easy to use** - simple workflow calls with clear inputs/outputs
- ✅ **Flexible** - supports multiple services, dependencies, env files, etc.

## Support

For issues or questions about these workflows, please open an issue in this repository.
