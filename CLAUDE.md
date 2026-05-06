# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the main repository for Red Hat OpenShift on Azure (ARO) using the Hosted Control Planes (HCP) architecture. It contains some of the code for the required microservices along with most of the configuration and pipeline definition necessary to deploy it.

## High-Level Architecture

ARO-HCP uses a three-tier architecture deployed across Azure regions:

**Regional Scope** - Each region operates independently with two cluster types:
- **Service Cluster**: Hosts customer-facing services (Resource Provider frontend/backend, Cluster Service, Maestro Server). One per region. Deployed in its own subscription with AKS, DNS zones, Key Vaults, Cosmos DB, Event Grid.
- **Management Clusters**: Execute HCP cluster workloads via Hypershift. Multiple per region for horizontal scaling. Each in its own subscription with AKS and Key Vaults.

**Global Scope** - Shared infrastructure with built-in redundancy:
- Azure Container Registry with geo-replication (SVC ACR for services, OCP ACR mirroring quay.io)
- Azure Front Door for OIDC artifact distribution

**Geography Scope** - Shared per geography:
- Kusto clusters for centralized logging (first region in a geography deploys the geo's Kusto)

**Request Flow**: Customer request → RP (ARM API) → Cluster Service → Maestro Server → Maestro Agent (on Management Cluster) → Hypershift Operator → HCP cluster creation

See [Architecture Overview](docs/high-level-architecture.md) for detailed diagrams and component descriptions.

## Common Development Commands

### Build and Test
- `make test` - Run all tests across the Go workspace (20 min timeout)
- `make test-unit` - Run unit tests only
- `make lint` - Run linting across all modules with build tag `E2Etests`
- `make lint-fix` - Auto-fix linting issues
- `make all` - Run tests and lint (default goal)
- `make install-tools` - Install all required development tools (golangci-lint, mockgen, addlicense, yamlfmt, oras, etc.)

### Code Generation
- `make generate` - Run all code generation (deepcopy, mocks, fmt, record E2E specs, tidy)
- `make deepcopy` - Generate deepcopy functions for API types
- `make mocks` - Generate mocks using mockgen
- `make fmt` - Format Go code with goimports (local import prefix: `github.com/Azure/ARO-HCP`)
- `make yamlfmt` - Format YAML files (wraps templated values, formats, then unwraps)
- `make licenses` - Add Apache license headers to Go files

### Verification
- `make verify` - Run all verification checks (deepcopy, json-format, generate, yamlfmt, materialize)
- `make verify-generate` - Verify generated code is up to date
- `make verify-deepcopy` - Verify deepcopy files are up to date
- `make verify-yamlfmt` - Verify YAML formatting
- `make verify-materialize` - Verify config materialization is current

### Local Development Workflow
- `make personal-dev-env` - One-step command to build services, push images, deploy infrastructure, and set up kubconfigs (requires `DEPLOY_ENV=pers`)
- `make build-services` - Build and push all in-repo service images (frontend, backend, admin, sessiongate)
- `make build-hcpctl` - Build the hcpctl CLI tool for HCP management

### Infrastructure Management
- `make infra.region` - Deploy regional infrastructure
- `make infra.svc` - Initialize service cluster infrastructure
- `make infra.mgmt` - Initialize management cluster infrastructure
- `make infra.all` - Deploy all infrastructure
- `make infra.svc.aks.kubeconfig` - Get service cluster kubeconfig
- `make infra.mgmt.aks.kubeconfig` - Get management cluster kubeconfig
- `make infra.cosmos.access` - Set up Cosmos DB access

### Service Deployment
- `make svc.deployall` - Deploy all services to service cluster (backend, frontend, cluster-service, maestro.server, observability.tracing)
- `make mgmt.deployall` - Deploy all services to management cluster (secret-sync-controller, acm, hypershiftoperator, maestro.agent, observability.tracing, route-monitor-operator)
- `make deployall` - Deploy all services to both clusters

### Testing HCP Clusters
The `demo/` directory provides tools for creating and managing HCP clusters:
- `demo/deploy-hcp-bicep.sh` - Deploy customer infrastructure (VNETs, subnets, NSGs, KeyVault), create HCP cluster, and create node pool using Bicep templates
- Configuration via `demo/env_vars` environment file
- Use `az` CLI commands to interact with clusters (see `demo/README.md` for examples):
  - `az resource show --ids "${CLUSTER_RESOURCE_ID}" --api-version 2025-12-23-preview` - Show cluster
  - `az resource invoke-action --ids "${CLUSTER_RESOURCE_ID}" --action requestAdminCredential --api-version 2025-12-23-preview` - Request admin credentials

### E2E Testing
- `make e2e/local` - Run local E2E tests (sets up, builds hcpctl, runs tests)
- `make e2e-local/run` - Run E2E test suite with aro-hcp-tests binary
- `make e2e-local/run-test TEST_NAME="..."` - Run a specific E2E test
- `make record-nonlocal-e2e` - Record non-local E2E test specs

### Environment Variables
- `DEPLOY_ENV` - Target environment (`pers` for personal dev, `cspr` for PR checks, etc.)
- `CONFIG_FILE` - Config file path (default: `config/config.yaml`)
- `LINT_GOTAGS` - Build tags for linting (default: `'E2Etests'`)
- `CONTAINER_RUNTIME` - Container engine to use (`docker` or `podman`)

### Service-Specific Development
Each service (frontend, backend, admin, sessiongate) follows a consistent structure:
- `make build` - Build the service binary
- `make run` - Run the service locally
- `make clean` - Clean build artifacts
- `make image` - Build container image
- `make build-and-push` - Build and push image to ACR (requires `az acr login`)
- `make record-override` - Record image digest to override config file
- `make deploy` - Build, push, and deploy the service

Frontend example for local development:
```bash
cd frontend
make build
./aro-hcp-frontend --location ${LOCATION} \
  --clusters-service-url http://localhost:8000 \
  --cosmos-name ${DB_NAME} \
  --cosmos-url ${DB_URL}
```

## Development Environment

### VSCode Remote Containers
The repository includes a `.devcontainer` configuration for VSCode Remote Containers development:
- Pre-configured with Go, Bicep CLI, Azure CLI, and TypeSpec tooling
- Automatically installs golangci-lint and development tools
- Mounts host `.gitconfig` and SSH keys for seamless git operations
- Includes extensions: Go, EditorConfig, Bicep, Azure CLI, Swagger Viewer

**Mac users**: Use Docker CLI (not Docker Desktop) with Colima:
```bash
brew install docker colima
colima start --cpu 4 --memory 8 --vz-rosetta --vm-type=vz
```

For detailed setup instructions, see [Dev Infrastructure docs](./dev-infrastructure/docs/development-setup.md).

## Target Environments

There's a multi-layered build and deploy system supporting multiple target environments with configs under `config/`):
- **Personal dev**
  - cloud: "dev"; environment: "pers" (`DEPLOY_ENV=pers`)
  - Local development using `make` and properly-configured `az` command
  - Changes must be manually applied by the user by running proper `make` commands
- **CSPR environment** (Clusters Service PR check)
  - cloud: "dev"; environment: "cspr" (`DEPLOY_ENV=cspr`)
  - Uses Prow for CI/CD (jobs defined in [openshift/release](https://github.com/openshift/release/tree/master/ci-operator/jobs/Azure/ARO-HCP))
  - Changes get automatically applied after a PR merge
  - Note: The integrated/shared dev environment (`DEPLOY_ENV=dev`) has been decommissioned. Only global infrastructure (shared ACRs, DNS zones) is still deployed to the dev environment via the `global-pipeline-postsubmit` Prow job.
- **Production systems**
  - cloud: "public"; environments: "int", "stg", "prod"
  - Deployed via Microsoft ADO and EV2 into Microsoft Int, Stage and Prod environments (hosted in Microsoft Azure tenants)
  - For changes to be applied, a PR must be created for `sdp-pipelines` repo and updated pipelines run manually
  - After making changes, remind user about this and point at aka.ms/arohcp-pipelines for next steps


### Access

* "Dev/*" environments can be accessed with a @redhat.com account and are largely read/write. It's likely that `az` is currently logged into it.
* "Public/Int" environment can be accessed with a @microsoft account and is mostly read-only. It's possible to `az login` into, but you need to ask the user explicitly to do that.
* "Public/Stage" and "Public/Prod" require SRE-level access.

## Deployments

Loosely defined categories of deployed objects:
- Native Azure objects (using bicep files in `dev-infrastructure/`).
  - This includes databases, roles, access definitions, automations, monitoring stack "plumbing", etc. (e.g. global from `dev-infrastructure/global-pipeline.yaml`, regional from `dev-infrastructure/region-pipeline.yaml`, etc.)
  - But also two types of AKS clusters: Service Clusters and Management Clusters (`dev-infrastructure/mgmt-pipeline.yaml`)
- Service Clusters
  – One per supported Azure region
  - Hosts the Resource Provider (frontend and backend) and Cluster Service
  - `services_svc_pipelines` in `Makefile` has a list of deployed services (on top of what's in `dev-infrastructure/svc-pipeline.yaml`)
- Management Clusters
  - Multiple per Azure region (number depends on how many HCPs are running in that region)
  - Run all the things required to actually run HCPs
  - `services_mgmt_pipelines` in `Makefile` has a list of deployed services (on top of what's in `dev-infrastructure/mgmt-pipeline.yaml`)

## Components

Loose categorization:
- Deployment pipelines (dirs have `*pipeline.yaml` files present)
  - `dev-infrastructure/` has all the bicep files
  - The rest deploy services to management and service clusters. Most of these do not host the code of the service, just reference already-released images.
- ARO-HCP-specific microservices, like the Resource Provider's `frontend/` and `backend/`:
  - These contain Go code in addition to pipelines configs.
- Additional/helper (`test/`, `hack/`, `tooling/`, etc.)

Incomplete list:
- **Frontend**: ARM REST API endpoint (`frontend/`) - Go service handling Azure ARM API calls
- **Backend**: Internal processing service (`backend/`) - Go service for async operations
- **Cluster Service**: Core cluster management (`cluster-service/`) - Manages HCP cluster lifecycle
- **Maestro**: Multi-cluster orchestration (`maestro/`) - Handles communication between service and management clusters
- **Infrastructure**: Azure infrastructure as code (`dev-infrastructure/`) - Bicep templates for all Azure resources
- **Internal**: Shared libraries (`internal/`) - Common APIs, database, OCM client, tracing utilities
- **demo**: helper scripts to quickly spin up and tear down an HCP cluster

The github.com/Azure/ARO-Tools repo is also a dependency and changes can be suggested for it.

## Additional Build, Configuration and Deployment Info

### Go Workspace
The project uses Go workspaces. All Go modules are defined in `go.work`:
- Main services: `admin`, `backend`, `frontend`, `internal`, `test`
- Tooling: `tooling/hcpctl`, `tooling/templatize`, etc.

### Environment Configuration
- Main config: `config/config.yaml` and overlays
- Environment-specific settings in `dev-infrastructure/configurations/`
- Service configs use Helm charts in each `deploy/` directory

### Pipeline System
Uses a custom templatize system (`tooling/templatize/`) that processes:
- `pipeline.yaml` files define deployment pipelines
- Bicep templates for Azure resource deployment
- Helm charts for Kubernetes service deployment

## Code Organization

### Service Structure
Each service follows consistent patterns:
- `Makefile` - Build, deploy, and run targets
- `deploy/` - Helm charts and Kubernetes manifests
- `pipeline.yaml` - Deployment pipeline definition
- Go services include standard `main.go`, `go.mod`

### Build Tags
- Lint tags: `LINT_GOTAGS='${GOTAGS},E2Etests'`

### Infrastructure as Code
- Bicep templates in `dev-infrastructure/templates/`
- Reusable modules in `dev-infrastructure/modules/`
- Parameter files: `*.bicepparam` (with `.tmpl.bicepparam` templates)

## Tooling

Key development tools (installed via `make install-tools`):
- `golangci-lint` - Go linting
- `mockgen` - Mock generation
- `addlicense` - License header management
- `yamlfmt` - YAML formatting
- `oras` - OCI registry interaction

Custom tools in `tooling/`:
- `hcpctl` - CLI for HCP management and access
- `templatize` - Pipeline template processing
- `secret-sync` - Secret management utilities
- `prometheus-rules` - Monitoring rule generation

## Documentation

Key documentation files:
- [Architecture Overview](docs/high-level-architecture.md)
- [Personal Dev Setup](docs/personal-dev.md)
- [Service Components](docs/service-components.md)
- [Environment Details](docs/environments.md)
- [Deployment Concepts](docs/service-deployment-concept.md)
- [E2E Testing](docs/e2e-testing.md)
- [Configuration](docs/configuration.md)
- [Bicep](docs/bicep.md)

## PR and Contribution Workflow

### Branch Management
**IMPORTANT**: Do NOT create PRs from a fork - GitHub checks will fail. Create branches directly in the ARO-HCP repository instead.

### PR Guidelines
- Start with a draft PR unless ready for review
- Write meaningful commit messages and PR descriptions
- PRs are squashed before merging (unless multi-commit split is needed for `git bisect`)
- Cover new functionality with appropriate tests
- Add documentation for new features
- When working on a Jira issue, document all tradeoffs and decisions in the Jira card
- For new features, consider recording a short demo video and uploading to the team [Drive folder](https://drive.google.com/drive/folders/1RB1L2-nGMXwsOAOYC-VGGbB0yD3Ae-rD?usp=drive_link)

### PR Lifecycle Management
This repo uses automated Prow lifecycle management. PRs/issues progress through stages after inactivity:
- **Stale** (`lifecycle/stale`): 90 days of inactivity
- **Rotten** (`lifecycle/rotten`): Additional inactivity after stale
- **Closed**: Additional inactivity after rotten

Prow commands to manage lifecycle:
- `/remove-lifecycle stale` - Remove stale label and reset timer
- `/remove-lifecycle rotten` - Remove rotten label
- `/lifecycle frozen` - Exempt from automatic closure (use for long-running PRs)
- `/remove-lifecycle frozen` - Remove frozen exemption

Any activity (comments, pushes, label changes) resets the inactivity timer.

### Code Quality
- Follow the existing directory structure (e.g., all API specs in `api/`)
- Use proper build tags when needed (`E2Etests` for E2E test code)
- All Go imports should be organized with local imports prefixed with `github.com/Azure/ARO-HCP`
- License headers: Apache 2.0, Copyright Microsoft Corporation
