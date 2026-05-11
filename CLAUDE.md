# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the main repository for Red Hat OpenShift on Azure (ARO) using the Hosted Control Planes (HCP) architecture. It contains some of the code for the required microservices along with most of the configuration and pipeline definition necessary to deploy it.

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
  - But also two types of EKS clusters: Service Clusters and Management Clusters (`dev-infrastructure/mgmt-pipeline.yaml`)
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

## Architecture Details (Multi-file Understanding Required)

### Request Flow: Customer → HCP Cluster
1. **Customer** makes ARM API call → **Azure ARM** (external)
2. **ARM** → **Frontend** (service cluster) - validates request, integrates with ARM
3. **Frontend** → **Cluster Service** (service cluster) - processes cluster lifecycle requests
4. **Cluster Service** → **Maestro Server** (service cluster) - orchestration layer
5. **Maestro Server** → **Maestro Agent** (management cluster) - transfers Kubernetes manifests
6. **Maestro Agent** → **Hypershift Operator** (management cluster) - creates/manages HCP control planes
7. **Hypershift** provisions actual HCP cluster control plane and worker nodes

### Service Cluster vs Management Cluster
- **Service Cluster**: One per region. Runs RP frontend/backend, Cluster Service, Maestro Server. Handles customer-facing APIs and orchestration. No direct access to ingress - use `kubectl port-forward`.
- **Management Cluster**: Multiple per region (scales based on HCP count). Runs Hypershift, Maestro Agent, ACM. Hosts actual HCP control planes. Independent from service cluster for resilience.

Resource group naming:
- Service: `hcp-underlay-$(regionShort)$(usernameShortPrefix)-svc`
- Management: `hcp-underlay-$(regionShort)$(usernameShortPrefix)-mgmt-1`

### Infrastructure Scopes
- **Global**: ACRs (svc-acr, ocp-acr with geo-replication), Azure Front Door for OIDC, parent DNS zones. Lives in one subscription, used by all regions.
- **Geography**: Kusto clusters per geography (e.g., "United States" has one Kusto deployed by first region in that geo). See [Azure/ARO-Tools ev2config](https://github.com/Azure/ARO-Tools/blob/main/config/ev2config/config.yaml).
- **Regional**: Per-region resources (Event Grid, DNS zones, databases, Key Vaults). Deployed to both service and management subscriptions.

### Testing Access Patterns
- **Development/CSPR**: @redhat.com accounts, read/write access. `az` likely already logged in.
- **Public/Int**: @microsoft.com accounts, mostly read-only. Requires explicit `az login`.
- **Public/Stage, Public/Prod**: SRE-level access only.

### Frontend Development Workflow
`make run` runs frontend locally. `make deploy` (from `frontend/`) builds a custom dev image, pushes to `arohcpsvcdev` ACR with dev-specific tag, and deploys to personal dev environment. Image naming prevents conflicts between developers. Requires `X-Ms-Identity-Url` header for local testing (can be dummy HTTPS URL ending in `identity.azure.net` for dev environments without real MI data plane).

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
- [E2E Test Best Practices](test/AGENTS.md)
- [Operational Troubleshooting](AGENTS.md) - Essential context for stuck clusters, deletion flows, service cluster APIs, and breakglass access

Operational documentation in `docs/ops/`:
- [Cleanup Stuck Cluster Deletion](docs/ops/cleanup-stuck-cluster-deletion.md)
- [Fix Maestro Stale Resource Bundle](docs/ops/fix-maestro-stale-resource-bundle.md)
- [HCP Cluster Creation Flow](docs/ops/hcp-cluster-creation-flow.md)
- [Postgres Breakglass](docs/ops/postgres-breakglass.md)

## Common Development Commands

### Building and Testing
- `make all` - Run tests and linting (default target)
- `make test` or `make test-unit` - Run all unit tests across Go workspace
- `make test-compile` - Verify all tests compile without running them
- `make lint` - Run golangci-lint across all modules
- `make lint-fix` - Auto-fix linting issues
- `make install-tools` - Install all development tools via bingo
- `make fmt` - Format Go code with goimports
- `make yamlfmt` - Format YAML files
- `make generate` - Generate deepcopy, mocks, format code, and tidy modules
- `make verify` - Run all verification checks (deepcopy, json-format, generate, yamlfmt, materialize)

### E2E Testing
The project uses a custom test runner (`aro-hcp-tests`) built from `test/cmd/aro-hcp-tests/`.

**Build the test binary:**
```bash
make -C test  # Builds test/aro-hcp-tests
```

**List available tests:**
```bash
./test/aro-hcp-tests list | jq '.[].name'
```

**Run a single test by name:**
```bash
./test/aro-hcp-tests run-test "Customer should create an HCP cluster and validate TLS certificates"
```

**Run a test suite:**
```bash
./test/aro-hcp-tests run-suite "integration/parallel"
```

**Run e2e tests locally (requires personal dev environment):**
```bash
make e2e/local  # Sets up subscription and runs full e2e suite
TEST_NAME="test name" make e2e-local/run-test  # Run single test with port-forward
```

**Environment variables for e2e tests:**
- `LOCATION` - Azure region (default: westus3)
- `FRONTEND_ADDRESS` - RP endpoint (default: http://localhost:8443)
- `SKIP_CERT_VERIFICATION` - Skip TLS verification (default: false)
- `ARTIFACT_DIR` - Output directory for test artifacts

### Personal Dev Environment
Personal dev environments are created per-developer and auto-delete after 48 hours unless `PERSIST=true`.

**Deploy full personal dev stack:**
Check [docs/personal-dev.md](docs/personal-dev.md) for detailed instructions. The deployment process creates:
- Regional infrastructure (DNS, Event Grid, databases)
- Service cluster (AKS hosting RP frontend/backend)
- Management cluster (AKS hosting Hypershift for HCP clusters)

**Access your clusters:**
```bash
# Get kubeconfig for service cluster
export KUBECONFIG=$(make infra.svc.aks.kubeconfigfile)

# Get kubeconfig for management cluster
export KUBECONFIG=$(make infra.mgmt.aks.kubeconfigfile)

# Port-forward to frontend
kubectl port-forward svc/aro-hcp-frontend 8443:8443 -n aro-hcp
```

**Deploy individual services:**
- `make -C frontend deploy` - Build and deploy frontend to personal dev
- `make -C backend deploy` - Build and deploy backend to personal dev

### Working with Go Workspace
This repo uses Go workspaces (see `go.work`). All modules share dependency versions.

- `make work-sync` - Sync go.work with module dependencies
- `make tidy` - Run `go mod tidy` on all modules
- `make all-tidy` - Tidy all modules, format code, and add license headers
- `make bump-aro-tools` - Bump github.com/Azure/ARO-Tools to latest main across all modules

### Mocks and Code Generation
- `make mocks` - Regenerate all mocks (uses mockgen with go:generate comments)
- `make deepcopy` - Regenerate Kubernetes deepcopy methods
- `make licenses` - Add Apache 2.0 license headers to Go files

### Image Bumps
For updating container image digests and versions in `config/config.yaml`, see [tooling/image-updater/AGENTS.md](tooling/image-updater/AGENTS.md) for the complete procedure including:
- Bumping single components, multiple components, or all components
- Post-bump steps (materialize configs, update helm charts)
- Branch and commit message conventions
- Repository version upgrades for y-stream updates

## E2E Test Framework Architecture

The e2e test suite uses Ginkgo v2 with custom test organization:

**Key principles:**
- Tests are self-contained and run in parallel
- Use `framework.TestContext` for automatic resource cleanup
- Resource groups created via `tc.NewResourceGroup()` are auto-cleaned after tests
- All clusters in those resource groups are deleted first, then the RGs
- Tests use `labels` package for categorization (Importance: Critical/High/Medium/Low, Positivity: Positive/Negative)
- **Never use Ginkgo Focus (`FIt`, `FDescribe`)** - use CLI filtering instead

**Filtering tests in CI:**
To run specific tests in PR validation against Int/Stage/Prod, edit `test/cmd/aro-hcp-tests/main.go`:
```go
// Uncomment and modify:
// specs = specs.MustFilter([]string{`name.contains("filter")`})
```
**Always revert before merging** to avoid silently skipping tests in CI.

**Test timeouts:**
- Cluster creation: 45 minutes (standard)
- Node pool creation: 45 minutes (standard)
- Use `Eventually` with `WithTimeout()` and `WithPolling()` for async operations

See [test/AGENTS.md](test/AGENTS.md) for complete e2e test design guidelines.

## Important Development Notes

### Lint Tags
When linting or running tests that include e2e code, use build tag: `LINT_GOTAGS='E2Etests'` (already set in Makefile)

### Bicep Templates
- Templates in `dev-infrastructure/templates/`
- Reusable modules in `dev-infrastructure/modules/`
- Parameter files: `*.bicepparam` with `.tmpl.bicepparam` templates processed by templatize tool
- E2E test bicep files auto-generated to `test/e2e/test-artifacts/generated-test-artifacts/` from `demo/bicep/` and `test/e2e-setup/bicep/`

### Nil Checks in Tests
**Critical for e2e tests**: Every `Expect(x).NotTo(BeNil())` **must** include a descriptive message:
```go
// Bad - produces unhelpful "Expected nil not to be nil"
Expect(resp.Properties).NotTo(BeNil())

// Good - clear failure message
Expect(resp.Properties).NotTo(BeNil(), "cluster response Properties was nil")
```

### Test Logging in Eventually/Polling
Use delta-only logging in `Eventually()` blocks - only log when state changes between poll iterations. See `eventuallyVerify` pattern in `test/e2e/cluster_pullsecret.go` for reference. Always log what was expected vs. what was observed on failure.

### Pipeline System
The repo uses a custom `templatize` tool (`tooling/templatize/`) that processes:
- `pipeline.yaml` files → deployment pipelines
- Bicep templates → Azure resource deployment
- Helm charts → Kubernetes service deployment
Environment config from `config/config.yaml` + overlays + `dev-infrastructure/configurations/`

### Production Deployment Workflow
Changes to prod require:
1. Make changes in this repo
2. Create PR for `sdp-pipelines` repo (Microsoft ADO)
3. Manually run updated pipelines in ADO/EV2
4. See aka.ms/arohcp-pipelines for next steps
**Remind user of this workflow after making prod-relevant changes.**
