# Container Image Publishing

This document explains how container images are built and published for the kaniko project.

## Published Images

Container images are automatically published to GitHub Container Registry (GHCR) at:

- **Executor images:**
  - `ghcr.io/nostoslabs/kaniko-executor:latest` - Standard executor image
  - `ghcr.io/nostoslabs/kaniko-executor:debug` - Debug variant with shell and busybox tools
  - `ghcr.io/nostoslabs/kaniko-executor:slim` - Slim variant without credential helpers
  - `ghcr.io/nostoslabs/kaniko-executor:<commit-sha>` - Commit-specific tags
  - `ghcr.io/nostoslabs/kaniko-executor:<commit-sha>-debug` - Commit-specific debug variant
  - `ghcr.io/nostoslabs/kaniko-executor:<commit-sha>-slim` - Commit-specific slim variant

- **Warmer image:**
  - `ghcr.io/nostoslabs/kaniko-warmer:latest` - Cache warming utility
  - `ghcr.io/nostoslabs/kaniko-warmer:<commit-sha>` - Commit-specific tags

## Image Variants

### Executor (Standard)
The standard executor image includes:
- Kaniko executor binary
- Docker credential helpers (GCR, ECR, ACR)
- Root CA certificates

### Executor Debug
The debug image adds:
- Busybox with a shell (`/busybox/sh`)
- Debugging utilities
- Same credentials and certificates as standard

### Executor Slim
The slim image is minimal:
- Only the kaniko executor binary
- Root CA certificates
- No credential helpers

### Warmer
The warmer image:
- Kaniko warmer binary for cache pre-warming
- Docker credential helpers (GCR, ECR, ACR)
- Root CA certificates

## Platform Support

Images are built for multiple architectures:
- `linux/amd64` - All variants
- `linux/arm64` - All variants
- `linux/s390x` - Executor (standard and slim), Executor Debug, and Warmer
- `linux/ppc64le` - Executor (standard and slim) and Warmer only

## Build Process

### Pull Request Builds

When a pull request is opened or updated:
- The `.github/workflows/images.yaml` workflow runs
- Builds all 4 image variants for `linux/amd64` only (for faster validation)
- Images are **not pushed** to any registry
- This validates that the images can be built successfully

### Publishing Builds

The `.github/workflows/release-images.yaml` workflow publishes images:

**Triggers:**
- Push to `main` branch
- Published releases
- Manual workflow dispatch

**Process:**
1. Builds all 4 image variants for all supported platforms
2. Authenticates to GHCR using `GITHUB_TOKEN`
3. Pushes images with multiple tags:
   - Commit-specific tag (e.g., `abc123` or `abc123-debug`)
   - Stable tag (e.g., `latest`, `debug`, `slim`)

## Dockerfile Details

The Dockerfile (`deploy/Dockerfile`) uses multi-stage builds:

1. **Builder stage** (`golang:1.25`):
   - Compiles Go binaries (executor, warmer, credential helpers)
   - Uses build cache for faster builds

2. **Certs stage** (`debian:bookworm-slim`):
   - Installs and updates root CA certificates
   - This stage is always rebuilt (`no-cache-filters: certs`) to ensure fresh certificates

3. **Busybox stage** (`busybox:musl`):
   - Provides shell and utilities for debug image

4. **Base stages** (`scratch`):
   - Creates minimal runtime environments
   - Copies certificates, binaries, and configuration

5. **Final stages**:
   - Each variant (executor, debug, slim, warmer) built from appropriate base

## Cache Strategy

The workflows use GitHub Actions cache to speed up builds:
- `cache-from: type=gha` - Reuses cache from previous builds
- `cache-to: type=gha,mode=max` - Saves all layers to cache
- `no-cache-filters: certs` - Always rebuilds certificate stage for security

## Security

### Root Certificates
Images include up-to-date root CA certificates from Debian to ensure secure HTTPS connections. The certificate stage is rebuilt on every image build to ensure the latest security updates are included.

## Manual Building

To build images locally:

```bash
# Build all images
make images

# Push to your own registry
REGISTRY=your-registry.io/your-project make images push
```

See [DEVELOPMENT.md](DEVELOPMENT.md) for more details on local development.

## Troubleshooting

### Pull Image Authentication Errors
If you can't pull images from GHCR, ensure:
1. The repository visibility is public, or
2. You're authenticated: `echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USERNAME --password-stdin`

### Build Failures
Check the [Actions tab](https://github.com/nostoslabs/kaniko/actions) for workflow run logs.

### Using Images in Kubernetes
See the main [README.md](README.md) for examples of running kaniko in Kubernetes.
