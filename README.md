# MAAP CI/CD Templates

GitLab CI/CD pipeline templates for deploying OGC Application Packages to the MAAP (Multi-Mission Algorithm and Analysis Platform) HySDS environment.

## Overview

This repository provides a reusable GitLab CI template that automates the conversion and deployment of OGC-compliant CWL (Common Workflow Language) workflows into HySDS-compatible job specifications.

## Quick Start

Add this template to your `.gitlab-ci.yml`:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/MAAP-Project/maap-ci-templates/main/gitlab/deploy-ogc-hysds.yml'
```

## Pipeline Stages

The deployment pipeline consists of four stages:

1. **Validate** - Downloads and validates OGC Application Package CWL files against OGC standards
2. **Download Artifacts** - Manages Docker image artifacts via local S3-backed registry
3. **Convert to HySDS** - Converts OGC CWL specifications to HySDS format
4. **Deploy to HySDS** - Deploys the converted specifications to HySDS infrastructure

## Required Environment Variables

Configure these in your GitLab CI/CD settings:

### Pipeline Configuration
- `HYSDS_DOCKER_REGISTRY` - Target HySDS Docker registry URL
- `MAAP_CODE_BUCKET` - S3 bucket for storing artifacts and CWL files
- `S3_REGION` - S3 region (default: us-west-2)
- `CWL_URL` - URL to the OGC Application Package CWL file to deploy
- `PROCESS_NAME_HYSDS` - Name to use for the HySDS job specification

### MAAP Executor Configuration
- `MAAP_OGC_EXECUTOR_CONTAINER_NAME` - Name of the MAAP OGC executor container
- `MAAP_OGC_EXECUTOR_CONTAINER_VERSION` - Version of the MAAP OGC executor container
- `MAAP_OGC_EXECUTOR_CONTAINER_URL` - Full URL to the MAAP OGC executor container image
- `MAAP_PRIVATE_DOCKER_REGISTRY` - Private Docker registry URL for MAAP images
- `MAAP_PRIVATE_REGISTRY_TOKEN` - Authentication token for private registry access

### HySDS Configuration
- `MOZART_REST_URL` - HySDS Mozart REST API endpoint
- `STORAGE` - Storage backend URL for HySDS
- `GRQ_REST_URL` - HySDS GRQ (General Request Queue) REST API endpoint

## Runner Requirements

The pipeline requires two types of GitLab runners:

- **docker** runners - For validation and conversion stages (Python 3.12 environment)
- **shell** runners - For download and deployment stages (must have Docker daemon access and AWS credentials)

## Features

- **OGC Application Package validation** using `ogc_ap_validator`
- **Automated version extraction** from CWL `s:version` field
- **Docker image retagging** and registry migration
- **Push verification** with automatic retry on failure
- **HySDS specification generation** via `hysds-ogc-container-builder`
- **Self-contained deployment** with inline scripts for portability

## Performance Optimizations

- Pip dependency caching between pipeline runs
- Shallow git clones (`--depth 1`) for faster repository checkout
- Conditional Docker pulls (only if not cached locally)
- Early validation to fail fast on errors
- Direct registry URLs instead of tar files for faster deployment

## Artifacts

The pipeline generates and passes these artifacts between stages:

- CWL file (renamed with version tag)
- `docker_url.txt` - Registry URL of the pushed Docker image
- `hysds-io.json*` - HySDS input/output specification
- `job-spec.json*` - HySDS job specification
- `build.env` - Environment variables passed via dotenv

## File Naming Convention

CWL files are renamed to: `${PROCESS_NAME_HYSDS}.${TAG}.process.cwl` where TAG is extracted from the `s:version:` field in the CWL file.

## Dependencies

- [ogc_ap_validator](https://pypi.org/project/ogc-ap-validator/) v0.5.0 - OGC Application Package validation
- [hysds-ogc-container-builder](https://github.com/MAAP-Project/hysds-ogc-container-builder) - HySDS specification generator
- `hysds/verdi:v5.2.0` - HySDS deployment container
- Docker registry v2

## License

[Add your license here]

## Support

For issues and questions, please open an issue in this repository.
