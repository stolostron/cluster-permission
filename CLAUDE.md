# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes controller project for Open Cluster Management (OCM) that manages RBAC permissions across managed clusters. The project implements a `ClusterPermission` custom resource that automatically distributes RBAC resources (Roles, ClusterRoles, RoleBindings, ClusterRoleBindings) to managed clusters using OCM's ManifestWork API.

## Architecture

### Core Components

- **API Layer** (`api/v1alpha1/`): Defines the ClusterPermission CRD and its types
- **Controller** (`controllers/`): Main reconciliation logic for ClusterPermission resources
- **Generated Client** (`client/`): Auto-generated Kubernetes client code for the ClusterPermission API
- **Configuration** (`config/`): Kubernetes manifests for CRDs, RBAC, and deployment

### Key Concepts

- **ClusterPermission**: Custom resource that specifies RBAC rules to be distributed to managed clusters
- **ManifestWork**: OCM resource used to deploy the RBAC manifests to target clusters
- **ManagedServiceAccount Integration**: Supports MSA as binding subjects for cross-cluster authentication
- **Validation**: Optional validation of referenced roles/clusterroles on managed clusters

## Development Commands

### Building and Testing
```bash
# Install CRDs to cluster
make install

# Run controller locally (requires access to OCM hub cluster)
make run

# Build the binary
make build

# Run tests
make test

# Run end-to-end tests (deploys OCM first)
make test-e2e
```

### Code Generation
```bash
# Generate manifests (CRDs, RBAC, etc.)
make manifests

# Generate deepcopy methods and client code
make generate

# Combined pre-build tasks (deps, manifests, generate, fmt, vet)
make pre-build
```

### Dependencies and Linting
```bash
# Update Go dependencies
make deps

# Format code
make fmt

# Run go vet
make vet
```

### Docker and Deployment
```bash
# Build Docker image
make docker-build

# Deploy to cluster
make deploy

# Remove from cluster
make undeploy
```

## Testing Framework

- Uses Ginkgo/Gomega for testing
- Controller tests in `controllers/clusterpermission_controller_test.go`
- Test suite setup in `controllers/suite_test.go`
- E2E tests in `e2e/` directory
- Tests require OCM environment for full functionality

## Code Structure Patterns

### API Types
- Follow Kubernetes API conventions
- Use `+optional` markers for optional fields
- Condition types defined as constants
- Validation logic integrated into the controller

### Controller Pattern
- Standard controller-runtime reconciler pattern
- Uses ManifestWork to deploy RBAC to managed clusters
- Implements validation logic for referenced roles
- Status conditions track reconciliation state

### Client Generation
- Uses Kubernetes code-generator tools
- Generated clients in `client/` directory
- Update with `make generate` after API changes

## Important Files

- `main.go`: Controller manager entry point with scheme registration
- `api/v1alpha1/clusterpermission_types.go`: Core API types and specifications
- `controllers/clusterpermission_controller.go`: Main reconciliation logic
- `controllers/helper.go`: Utility functions for manifest generation
- `config/samples/`: Example ClusterPermission resources for testing

## Dependencies

- Built with controller-runtime and Kubebuilder v3
- Requires Open Cluster Management APIs
- Optional integration with ManagedServiceAccount
- Uses ManifestWork for cross-cluster deployment

## Development Notes

- The controller must run in an OCM hub cluster context
- ClusterPermission resources must be created in managed cluster namespaces
- Generated ManifestWork resources handle the actual RBAC deployment
- Validation is optional and uses separate ManifestWork resources