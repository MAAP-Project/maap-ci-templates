# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains GitLab CI/CD pipeline templates for deploying OGC Application Packages to the MAAP (Multi-Mission Algorithm and Analysis Platform) HySDS environment. The pipeline automates the conversion of OGC-compliant CWL workflows into HySDS-compatible job specifications and deploys them to the MAAP infrastructure.

## Architecture

The deployment pipeline follows a four-stage process:

1. **Validation Stage**: Downloads and validates OGC Application Package CWL files against OGC standards using `ogc_ap_validator`. Extracts version information and prepares the workspace.

2. **Download Artifacts Stage**: Manages Docker image artifacts by:
   - Running a local Docker registry (port 5050) backed by S3 storage
   - Pulling the Docker image specified in the CWL file
   - Retagging and pushing to the HySDS Docker registry (`${HYSDS_DOCKER_REGISTRY}`)
   - Verifying the push succeeded before proceeding
   - Modifying CWL file to reference the new registry location

3. **Convert to HySDS Stage**: Converts OGC CWL specifications to HySDS format by:
   - Cloning `hysds-ogc-container-builder` repository
   - Running `utils.cwl_to_hysds` to generate `hysds-io.json` and `job-spec.json`
   - Modifying job-spec to use `/app/dps_wrapper.sh` with the CWL file URI

4. **Deploy to HySDS Stage**: Deploys the converted specifications using the `hysds/verdi:v5.2.0` Docker image with an inline deployment script that registers the container and job specifications with HySDS

## Key Environment Variables

Required variables that must be set in GitLab CI/CD settings:

**Pipeline Configuration:**
- `HYSDS_DOCKER_REGISTRY`: Target HySDS Docker registry URL
- `MAAP_CODE_BUCKET`: S3 bucket for storing artifacts and CWL files
- `S3_REGION`: S3 region (default: us-west-2)
- `CWL_URL`: URL to the OGC Application Package CWL file to deploy
- `PROCESS_NAME_HYSDS`: Name to use for the HySDS job specification

**MAAP Executor Configuration:**
- `MAAP_OGC_EXECUTOR_CONTAINER_NAME`: Name of the MAAP OGC executor container
- `MAAP_OGC_EXECUTOR_CONTAINER_VERSION`: Version of the MAAP OGC executor container
- `MAAP_OGC_EXECUTOR_CONTAINER_URL`: Full URL to the MAAP OGC executor container image
- `MAAP_PRIVATE_DOCKER_REGISTRY`: Private Docker registry URL for MAAP images
- `MAAP_PRIVATE_REGISTRY_TOKEN`: Authentication token for private registry access

**HySDS Configuration:**
- `MOZART_REST_URL`: HySDS Mozart REST API endpoint
- `STORAGE`: Storage backend URL for HySDS
- `GRQ_REST_URL`: HySDS GRQ (General Request Queue) REST API endpoint

## Runner Requirements

The pipeline requires two types of GitLab runners:

- **docker** runners: For validation and conversion stages (Python 3.12 environment)
- **shell** runners: For download and deployment stages (must have Docker daemon access and AWS credentials)

## File Naming Convention

CWL files are renamed to: `${PROCESS_NAME_HYSDS}.${TAG}.process.cwl` where TAG is extracted from the `s:version:` field in the CWL file.

## Dependencies

External tools and repositories:
- `ogc_ap_validator==0.5.0`: OGC Application Package validation
- `hysds-ogc-container-builder`: HySDS specification generator (https://github.com/MAAP-Project/hysds-ogc-container-builder)
- `hysds/verdi:v5.2.0`: HySDS deployment container
- Docker registry v2

## Pipeline Artifacts

The pipeline creates and passes these artifacts between stages:
- CWL file (renamed with version tag)
- `docker_url.txt`: Registry URL of the pushed Docker image
- `hysds-io.json*`: HySDS input/output specification
- `job-spec.json*`: HySDS job specification
- `build.env`: Environment variables (CWL_FILE_NAME, TAG) passed via dotenv

## Performance Optimizations

The template includes several performance and reliability improvements:
- **Pip caching**: Dependencies are cached between pipeline runs
- **Shallow git clone**: Uses `--depth 1` for faster repository cloning
- **Conditional Docker pulls**: Only pulls images if not already cached locally
- **Validation**: TAG and DOCKER_IMAGE extraction includes validation to fail fast
- **Push verification**: Docker push is verified before proceeding to next steps
- **Direct registry usage**: Uses registry URLs directly instead of tar files for faster deployment
- **Artifact reuse**: TAG is extracted once and passed via dotenv to avoid duplicate extraction

## Usage with GitLab Include Directive

This template is designed to work with GitLab's `include:` directive. All necessary scripts are embedded inline, making it fully self-contained. Example usage in your `.gitlab-ci.yml`:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/your-org/maap-ci-templates/main/gitlab/deploy-ogc-hysds.yml'
```
