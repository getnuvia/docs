# Getting Started Guide

This guide provides instructions for setting up the development environment and starting implementation of the Signadot/SLATE clone.

## Prerequisites

Before starting development, ensure you have the following installed:

- **Go**: Version 1.18 or higher
- **Docker**: Latest stable version
- **Kubernetes**: Local environment (kind, minikube, k3d)
- **kubectl**: Latest stable version
- **Helm**: Version 3.0 or higher
- **Git**: Latest stable version

## Setting Up Development Environment

### 1. Clone Repository

```bash
# Create and clone the repository
mkdir -p $GOPATH/src/github.com/yourusername
cd $GOPATH/src/github.com/yourusername
git clone https://github.com/yourusername/project-name.git
cd project-name
```

### 2. Create Local Kubernetes Cluster

```bash
# Using kind to create a local Kubernetes cluster
kind create cluster --name dev-cluster

# Verify the cluster is running
kubectl cluster-info
```

### 3. Set Up Project Structure

```bash
# Create the basic project structure
mkdir -p cmd/apiserver cmd/cli
mkdir -p pkg/apis pkg/client pkg/controllers pkg/proxy pkg/util
mkdir -p internal
mkdir -p deploy/helm deploy/kustomize
mkdir -p docs examples test
```

### 4. Initialize Go Modules

```bash
# Initialize Go modules
go mod init github.com/yourusername/project-name

# Add basic dependencies
go get -u k8s.io/client-go@v0.26.0
go get -u sigs.k8s.io/controller-runtime@v0.14.0
go get -u github.com/spf13/cobra@v1.6.1
go get -u github.com/spf13/viper@v1.15.0
go get -u google.golang.org/grpc@v1.53.0
```

## Initial Implementation Steps

### 1. Define API Resources

Start by defining the core API resources in the `pkg/apis` directory:

1. Create a new directory for each API group
2. Define types for each resource
3. Implement validation and defaulting functions
4. Generate client code using code-generator

Example structure:
```
pkg/apis/
└── sandbox/
    ├── v1alpha1/
    │   ├── doc.go
    │   ├── register.go
    │   ├── types.go
    │   └── zz_generated.deepcopy.go
    └── register.go
```

### 2. Implement API Server

The API server will be the central component for managing resources:

1. Create a server in `cmd/apiserver/main.go`
2. Set up API endpoints for resources
3. Implement authentication and authorization
4. Configure validation webhooks

Basic structure:
```go
package main

import (
    "flag"
    "log"
    
    "k8s.io/klog/v2"
)

func main() {
    klog.InitFlags(nil)
    flag.Parse()
    
    // Initialize server
    server := NewServer()
    
    // Start server
    if err := server.Start(); err != nil {
        log.Fatalf("Failed to start server: %v", err)
    }
}
```

### 3. Create Resource Controllers

Implement controllers for managing resources:

1. Create controller for each resource type
2. Implement reconciliation logic
3. Set up watches for resource changes
4. Handle resource lifecycle

Example controller structure:
```go
package controllers

import (
    "context"
    
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
)

type SandboxController struct {
    client client.Client
}

func (r *SandboxController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
    // Reconciliation logic
    return reconcile.Result{}, nil
}
```

### 4. Implement CLI

Start building the command-line interface:

1. Create CLI entry point in `cmd/cli/main.go`
2. Set up command hierarchy using cobra
3. Implement basic commands (create, delete, list)
4. Add authentication to API server

Example CLI structure:
```go
package main

import (
    "os"
    
    "github.com/spf13/cobra"
)

func main() {
    rootCmd := &cobra.Command{
        Use:   "sandboxctl",
        Short: "Command-line tool for managing sandboxes",
    }
    
    // Add commands
    rootCmd.AddCommand(createCmd())
    rootCmd.AddCommand(deleteCmd())
    rootCmd.AddCommand(listCmd())
    
    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

## Development Workflow

### Iterative Development

1. **Implement Core API**: Start with defining resources
2. **Basic Controllers**: Implement simple controller logic
3. **CLI Development**: Build CLI commands for core operations
4. **Routing Implementation**: Develop routing proxy
5. **Context Propagation**: Implement context handling
6. **Integration**: Connect components together

### Testing During Development

1. **Unit Tests**: Write tests for each component
2. **Integration Tests**: Test interactions between components
3. **End-to-End Tests**: Test complete workflows

Example test command:
```bash
# Run unit tests
go test ./pkg/...

# Run integration tests
go test ./test/integration/...
```

### Local Deployment

1. **Build Components**:
   ```bash
   go build -o bin/apiserver ./cmd/apiserver
   go build -o bin/cli ./cmd/cli
   ```

2. **Deploy to Local Cluster**:
   ```bash
   # Using Helm
   helm install sandbox-system ./deploy/helm/sandbox-system

   # Using kubectl
   kubectl apply -f ./deploy/kustomize/base
   ```

3. **Verify Deployment**:
   ```bash
   kubectl get pods -n sandbox-system
   ```

## Next Steps

After setting up the development environment and implementing initial components:

1. **Implement Data Plane**: Develop the routing proxy and context propagation
2. **Develop Sandbox Manager**: Create logic for sandbox management
3. **Implement Local Development Support**: Add support for local-to-remote routing
4. **Add Basic Debugging Tools**: Implement request tracing and logging
5. **Complete CLI Implementation**: Add all required commands

## Troubleshooting

### Common Issues

1. **Kubernetes Connection Issues**:
   - Ensure your kubeconfig is correctly set up
   - Verify the cluster is running: `kubectl cluster-info`

2. **Go Module Issues**:
   - Run `go mod tidy` to clean up dependencies
   - Ensure you're using compatible versions of libraries

3. **API Server Not Starting**:
   - Check for port conflicts
   - Verify RBAC permissions
   - Check logs for detailed errors

4. **Controller Issues**:
   - Enable debug logging for controller-runtime
   - Verify CRD installation

### Getting Help

- Review component documentation
- Check error logs for specific error messages
- Refer to Kubernetes documentation for Kubernetes-specific issues
