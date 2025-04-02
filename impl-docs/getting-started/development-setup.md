# Development Setup Guide

This guide provides detailed instructions for setting up the development environment for our Signadot/SLATE clone project.

## System Requirements

- **Operating System**: Linux, macOS, or Windows with WSL2
- **CPU**: 4+ cores
- **Memory**: 8GB+ RAM (16GB recommended)
- **Storage**: 20GB+ free space

## Installing Prerequisites

### 1. Install Go

The project requires Go 1.18 or higher.

#### On macOS

```bash
# Using Homebrew
brew install go

# Verify installation
go version
```

#### On Linux

```bash
# Download Go
wget https://golang.org/dl/go1.19.5.linux-amd64.tar.gz

# Extract
sudo tar -C /usr/local -xzf go1.19.5.linux-amd64.tar.gz

# Set up environment
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

# Verify installation
go version
```

#### On Windows (WSL2)

Follow the Linux instructions above within your WSL2 environment.

### 2. Install Docker

#### On macOS

```bash
# Using Homebrew
brew install --cask docker

# Start Docker Desktop
open /Applications/Docker.app
```

#### On Linux

```bash
# Install Docker
sudo apt-get update
sudo apt-get install -y docker.io

# Add your user to the docker group
sudo usermod -aG docker $USER

# Start Docker
sudo systemctl enable docker
sudo systemctl start docker
```

### 3. Install Kubernetes Tools

#### Install kubectl

```bash
# On macOS
brew install kubectl

# On Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Install Kind

```bash
# On macOS
brew install kind

# On Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### 4. Install Other Tools

```bash
# Install Git (if not already installed)
sudo apt-get install -y git  # Linux
brew install git             # macOS

# Install make
sudo apt-get install -y make  # Linux
brew install make             # macOS

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Setting Up the Repository

### 1. Clone the Repository

```bash
# Fork the repository on GitHub first, then clone your fork
git clone https://github.com/yourusername/project-name.git
cd project-name

# Add the upstream repository
git remote add upstream https://github.com/originalowner/project-name.git
```

### 2. Set Up Go Module

```bash
go mod download
```

### 3. Create Development Cluster

```bash
# Create a local Kind cluster
kind create cluster --name dev-cluster --config deploy/kind/config.yaml

# Verify cluster is running
kubectl cluster-info --context kind-dev-cluster
```

## Project Structure

Here's an overview of the project structure:

```
/
├── cmd/                 # Command-line applications
│   ├── apiserver/       # API server entry point
│   ├── cli/             # CLI tool entry point
│   └── proxy/           # Proxy entry point
├── pkg/                 # Library code
│   ├── apis/            # API definitions
│   ├── client/          # Client libraries
│   ├── controllers/     # Kubernetes controllers
│   ├── proxy/           # Routing proxy
│   └── util/            # Utility functions
├── internal/            # Private packages
├── test/                # Test files
├── docs/                # Documentation
├── examples/            # Example configurations
└── deploy/              # Deployment manifests
```

## Development Workflow

### 1. Building the Project

```bash
# Build all components
make build

# Build individual components
make build-apiserver
make build-proxy
make build-cli
```

### 2. Running Locally

#### Running the API Server

```bash
# Run API server locally
make run-apiserver

# Or manually
go run ./cmd/apiserver/main.go
```

#### Running the Proxy

```bash
# Run proxy locally
make run-proxy

# Or manually
go run ./cmd/proxy/main.go
```

#### Running the CLI

```bash
# Install CLI locally
make install-cli

# Or manually
go install ./cmd/cli
```

### 3. Deploying to Development Cluster

```bash
# Deploy custom resources
kubectl apply -f deploy/crds/

# Deploy components
make deploy-dev

# Verify deployment
kubectl get pods -n sandbox-system
```

### 4. Port Forwarding

```bash
# Forward API server port
kubectl port-forward -n sandbox-system svc/sandbox-api-server 8080:80

# Forward proxy port
kubectl port-forward -n sandbox-system svc/sandbox-proxy 8081:80
```

### 5. Using the CLI

```bash
# Configure CLI to use local API server
sandboxctl config set server.url http://localhost:8080
sandboxctl config set token dev-token

# Test CLI
sandboxctl version
```

## Development Cycle

A typical development cycle looks like this:

1. **Make changes**: Edit code in your editor of choice
2. **Unit test**: Run `make test` to run unit tests
3. **Build**: Run `make build` to build the components
4. **Deploy**: Run `make deploy-dev` to deploy to local cluster
5. **Test**: Test your changes using the CLI or API
6. **Iterate**: Repeat as necessary

## Testing

### Running Tests

```bash
# Run all tests
make test

# Run unit tests only
make test-unit

# Run integration tests
make test-integration

# Run tests with coverage
make test-coverage
```

### Writing Tests

Test files should be placed in the same package as the code they test, with a `_test.go` suffix.

```go
// Example test file: pkg/apis/validators/sandbox_test.go

package validators

import (
    "testing"
)

func TestValidateSandbox(t *testing.T) {
    // Test code here
}
```

## Debugging

### Using Delve

You can use Delve for debugging Go code:

```bash
# Install Delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug API server
dlv debug ./cmd/apiserver/main.go

# Connect to running process
dlv attach $(pgrep apiserver)
```

### Viewing Logs

```bash
# View API server logs
kubectl logs -n sandbox-system deployment/sandbox-api-server

# View proxy logs
kubectl logs -n sandbox-system deployment/sandbox-proxy

# Follow logs
kubectl logs -f -n sandbox-system deployment/sandbox-api-server
```

## Using Makefile

The project includes a comprehensive Makefile with common tasks:

```bash
# Show all available targets
make help

# Common targets
make build          # Build all components
make test           # Run tests
make deploy-dev     # Deploy to development cluster
make clean          # Clean build artifacts
make lint           # Run linters
make generate       # Generate code
```

## Setting Up IDE

### Visual Studio Code

1. Install the Go extension for VS Code
2. Configure settings:

```json
{
    "go.useLanguageServer": true,
    "go.lintTool": "golangci-lint",
    "go.lintFlags": ["--fast"],
    "go.formatTool": "goimports",
    "go.testFlags": ["-v"],
    "editor.formatOnSave": true
}
```

### GoLand

1. Open the project in GoLand
2. Configure GOROOT and GOPATH in settings
3. Enable Go modules in settings

## Containerized Development

You can also develop inside a container with all dependencies pre-installed:

```bash
# Build development container
docker build -t sandbox-dev -f deploy/dev/Dockerfile.dev .

# Run development container
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ${HOME}/.kube:/root/.kube \
  -v ${HOME}/.docker:/root/.docker \
  sandbox-dev
```

## Troubleshooting

### Common Issues

#### Go Module Issues

```bash
# Tidy modules
go mod tidy

# Verify modules
go mod verify
```

#### Kubernetes Connection Issues

```bash
# Check cluster status
kind get clusters
kubectl cluster-info

# Reset Kubernetes context
kubectl config use-context kind-dev-cluster
```

#### Build Errors

```bash
# Clean build cache
go clean -cache

# Rebuild completely
make clean build
```

## Next Steps

After setting up your development environment:

1. Read the architecture documentation in `docs/architecture.md`
2. Look at the implementation roadmap in `docs/implementation-roadmap.md`
3. Check out open issues in the issue tracker
4. Join community discussions

Happy coding!
