# PCOM-CICD

Shared, reusable GitHub Actions workflows for the NNP Platform Components (PCOM)
container images. Image repositories call these workflows instead of copying
build logic, so the whole set of images shares one audited build/scan pipeline.

## Workflows

| Workflow | Purpose |
|----------|---------|
| `container-pipeline.yml` | End-to-end entry point: build & publish, then scan. Image repos call this. |
| `container-build.yml` | Build with Buildx, publish multi-arch to GHCR, sign keyless (cosign / OIDC), attach SBOM + provenance. |
| `container-scan.yml` | Scan with Trivy and Grype, publish SARIF to the Security tab. |

Publishing happens on every event except pull requests, which build and scan
without pushing (the image is scanned from an exported tarball).

## Usage

Add a thin workflow to an image repository:

```yaml
name: build-and-publish

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 6 * * 1"
  workflow_dispatch:

permissions:
  contents: read
  packages: write
  id-token: write        # cosign keyless signing
  security-events: write # SARIF upload

jobs:
  pipeline:
    uses: NNP-Platform-Components-PCOM/PCOM-CICD/.github/workflows/container-pipeline.yml@main
    with:
      image-name: pcom-brc-ubuntu
      image-version: "22.04-v1"
      build-args: |
        os_ver=jammy
```

### Inputs

| Input | Default | Notes |
|-------|---------|-------|
| `image-name` | — (required) | Published as `ghcr.io/<org>/<image-name>`. |
| `image-version` | `""` | Primary human-readable tag. `latest` and `sha-<short>` are always added. |
| `context` | `.` | Docker build context. |
| `dockerfile` | `./Dockerfile` | Path to the Dockerfile. |
| `platforms` | `linux/amd64,linux/arm64` | Published architectures. |
| `build-args` | `""` | Multiline `KEY=VALUE` build args. |
| `severity` | `CRITICAL,HIGH` | Severities the scan reports on. |
| `ignore-unfixed` | `true` | Skip vulnerabilities with no fix. |
| `fail-on-findings` | `false` | Fail the run on matching vulnerabilities. |
| `sign` | `true` | Sign published images with cosign. |

No registry secrets are required: publishing uses the built-in `GITHUB_TOKEN`
and signing uses GitHub's OIDC identity.
