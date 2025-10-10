# MAAP CI/CD Templates

GitLab CI/CD pipeline templates for building and deploying OGC Application Packages to the MAAP (Multi-Mission Algorithm and Analysis Platform) HySDS environment.

## Overview

This repository provides two reusable GitLab CI templates:

1. **[build-ogc-app-pack.yml](gitlab/build-ogc-app-pack.yml)** - Builds Docker images from algorithm repositories and generates OGC-compliant CWL workflow files
2. **[deploy-ogc-hysds.yml](gitlab/deploy-ogc-hysds.yml)** - Converts and deploys OGC Application Packages to HySDS infrastructure

## Quick Start

### Building OGC Application Packages

Add this template to build Docker images and generate CWL files:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/MAAP-Project/maap-ci-templates/main/gitlab/build-ogc-app-pack.yml'
```

### Deploying to HySDS

Add this template to deploy existing OGC packages:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/MAAP-Project/maap-ci-templates/main/gitlab/deploy-ogc-hysds.yml'
```

## Pipeline Stages

### Build Pipeline (build-ogc-app-pack.yml)

The build pipeline consists of two stages:

1. **Build** - Builds Docker image from algorithm repository
   - Uses Docker-in-Docker (dind) service
   - Clones algorithm repository at specified branch
   - Builds Docker image with configurable base image and build command
   - Pushes image to OGC Application Package registry
   - Authenticates using deploy token

2. **Generate** - Generates OGC Application Package CWL file
   - Uses algorithm configuration (base64-encoded JSON)
   - Generates CWL workflow using ogc-app-pack-generator
   - Publishes CWL file to configured endpoint via GitLab API
   - Creates or updates CWL file using POST/PUT requests

### Deployment Pipeline (deploy-ogc-hysds.yml)

The deployment pipeline consists of four stages:

1. **Validate** - Downloads and validates OGC Application Package CWL files against OGC standards
2. **Download Artifacts** - Manages Docker image artifacts via local S3-backed registry
3. **Convert to HySDS** - Converts OGC CWL specifications to HySDS format
4. **Deploy to HySDS** - Deploys the converted specifications to HySDS infrastructure

## Required Environment Variables

### Build Pipeline Variables (build-ogc-app-pack.yml)

- `IMAGE_NAME` - Name for the Docker image (e.g., "my-algorithm")
- `IMAGE_TAG` - Tag for the Docker image (e.g., "v1.0.0")
- `OGC_APP_PACK_REGISTRY` - Docker registry URL for OGC Application Packages
- `OGC_APP_PACK_DEPLOY_TOKEN` - Authentication token for registry access
- `REPOSITORY_URL` - Git repository URL containing the algorithm code
- `BRANCH_REF` - Branch or tag reference to clone from repository
- `BUILD_CMD` - Command to execute for building the algorithm (e.g., "./build.sh")
- `BASE_IMAGE_NAME` - Base Docker image to use (default: condaforge/miniforge3:25.3.1-0)
- `ALGO_CONFIG_JSON_B64` - Base64-encoded JSON configuration for algorithm
- `OGC_PROCESS_FILE_PUBLISH_URL` - GitLab API URL to publish the generated CWL file
- `OGC_APP_PACK_WRITE_TOKEN` - GitLab API token with write permissions

### Deployment Pipeline Variables (deploy-ogc-hysds.yml)

Configure these in your GitLab CI/CD settings:

#### Pipeline Configuration
- `HYSDS_DOCKER_REGISTRY` - Target HySDS Docker registry URL
- `MAAP_CODE_BUCKET` - S3 bucket for storing artifacts and CWL files
- `S3_REGION` - S3 region (default: us-west-2)
- `CWL_URL` - URL to the OGC Application Package CWL file to deploy
- `PROCESS_NAME_HYSDS` - Name to use for the HySDS job specification

#### MAAP Executor Configuration
- `MAAP_OGC_EXECUTOR_CONTAINER_NAME` - Name of the MAAP OGC executor container
- `MAAP_OGC_EXECUTOR_CONTAINER_VERSION` - Version of the MAAP OGC executor container
- `MAAP_OGC_EXECUTOR_CONTAINER_URL` - Full URL to the MAAP OGC executor container image
- `MAAP_PRIVATE_DOCKER_REGISTRY` - Private Docker registry URL for MAAP images
- `MAAP_PRIVATE_REGISTRY_TOKEN` - Authentication token for private registry access

#### HySDS Configuration
- `MOZART_REST_URL` - HySDS Mozart REST API endpoint
- `STORAGE` - Storage backend URL for HySDS
- `GRQ_REST_URL` - HySDS GRQ (General Request Queue) REST API endpoint

## Runner Requirements

### Build Pipeline Requirements

- **docker** runners with Docker-in-Docker (dind) support
- Access to Docker registry for pushing images
- GitLab container registry credentials

### Deployment Pipeline Requirements

- **docker** runners - For validation and conversion stages (Python 3.12 environment)
- **shell** runners - For download and deployment stages (must have Docker daemon access and AWS credentials)

## Features

### Build Pipeline Features
- **Automated Docker image building** from algorithm repositories
- **Configurable base images** (default: condaforge/miniforge3)
- **Shallow git cloning** for faster builds
- **OGC CWL workflow generation** from algorithm configuration
- **Automatic CWL publishing** to configured endpoints via GitLab API
- **Manual and API-triggered pipelines** with rule-based execution

### Deployment Pipeline Features
- **OGC Application Package validation** using `ogc_ap_validator`
- **Automated version extraction** from CWL `s:version` field
- **Docker image retagging** and registry migration
- **Digest-based push verification** with automatic retry on failure
- **HySDS specification generation** via `hysds-ogc-container-builder`
- **Self-contained deployment** with inline scripts for portability
- **Absolute path handling** for reliable artifact passing between stages

## Performance Optimizations

- Pip dependency caching between pipeline runs
- Shallow git clones (`--depth 1`) for faster repository checkout
- Conditional Docker pulls (only if not cached locally)
- Early validation to fail fast on errors
- Direct registry URLs instead of tar files for faster deployment
- Optimized Docker digest comparison for reliable push verification

## Artifacts

### Build Pipeline Artifacts

- `image.env` - Dotenv artifact containing `FULLY_QUALIFIED_IMAGE_URL`
- `cwl_workflows/*` - Generated CWL workflow files
- `algo_config.yaml` - Parsed algorithm configuration in YAML format

### Deployment Pipeline Artifacts

- CWL file (renamed with version tag)
- `docker_url.txt` - Registry URL of the pushed Docker image
- `hysds-io.json*` - HySDS input/output specification
- `job-spec.json*` - HySDS job specification
- `build.env` - Environment variables passed via dotenv

## File Naming Convention

CWL files are renamed to: `${PROCESS_NAME_HYSDS}.${TAG}.process.cwl` where TAG is extracted from the `s:version:` field in the CWL file.

## Dependencies

### Build Pipeline Dependencies

- [ogc-app-pack-generator](https://github.com/MAAP-Project/ogc-app-pack-generator) - OGC CWL workflow generator
- `condaforge/miniforge3:25.3.1-0` - Default base image
- Docker registry v2
- GitLab API for CWL file publishing

### Deployment Pipeline Dependencies

- [ogc_ap_validator](https://pypi.org/project/ogc-ap-validator/) v0.5.0 - OGC Application Package validation
- [hysds-ogc-container-builder](https://github.com/MAAP-Project/hysds-ogc-container-builder) - HySDS specification generator
- `hysds/verdi:v5.2.0` - HySDS deployment container
- Docker registry v2

## Recent Improvements

### Version 1.1 (October 2025)

#### Build Pipeline
- **Added build-ogc-app-pack.yml template**: New template for building Docker images and generating OGC CWL files (commit 4c61ee4)
- **Shallow git cloning**: Uses `--depth 1 --single-branch` for faster algorithm repository checkout (commit 1d90320)
- **Inline Dockerfile generation**: Builds Docker images with configurable base image, repository URL, branch, and build command
- **Automated CWL publishing**: Publishes generated CWL files to GitLab via API with POST/PUT fallback
- **Rule-based execution**: Supports both web UI and API triggers, blocks direct git push events

#### Deployment Pipeline
- **Improved Docker push verification**: Implemented digest-based verification by comparing local and registry digests instead of relying on `docker manifest inspect` (commit 8238226, de88942)
- **Absolute path handling**: Added `$CI_PROJECT_DIR` prefix to all artifact paths for reliable artifact passing between stages in GitLab CI (commit b3ac036)
- **Artifact debugging**: Added pre-script artifact listing in conversion and deployment stages to help diagnose issues (commit 34fe5ac)
- **Updated converter call**: Updated `cwl_to_hysds` invocation to use the latest API with `--docker-uri` flag (commit e4a6306)
- **PYTHONPATH configuration**: Explicitly set `PYTHONPATH` for converter module imports
- **Logging improvements**: Removed colons from inline script echo statements to prevent GitLab CI collapsible section formatting issues (commit e5b0369, b635271)
- **Build reliability**: Fixed typos and improved error messages throughout the pipeline (commit e6eedcb)

## License

[Add your license here]

## Support

For issues and questions, please open an issue in this repository.
