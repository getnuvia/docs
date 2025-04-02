# Contribution Guide

This document provides guidelines for contributing to the Signadot/SLATE clone project. We welcome contributions in the form of bug reports, feature requests, documentation improvements, and code changes.

## Code of Conduct

We expect all contributors to follow our Code of Conduct. Please be respectful and considerate when interacting with other contributors.

## Getting Started

### Prerequisites

Before you begin, ensure you have the following installed:

- Go 1.18 or higher
- Docker
- Kubernetes (local cluster like Kind, Minikube, or k3d)
- kubectl
- Git

### Setting Up Development Environment

1. Fork the repository on GitHub
2. Clone your fork locally:
   ```bash
   git clone https://github.com/yourusername/project-name.git
   cd project-name
   ```

3. Add the original repository as an upstream remote:
   ```bash
   git remote add upstream https://github.com/originalowner/project-name.git
   ```

4. Install development dependencies:
   ```bash
   make setup-dev
   ```

5. Create a local Kubernetes cluster:
   ```bash
   make kind-create
   ```

6. Build and deploy the development version:
   ```bash
   make dev-deploy
   ```

## Development Workflow

### Branching Model

We follow a branching model based on feature branches:

- `main`: The main development branch
- `release-*`: Release branches
- `feature/*`: Feature branches
- `bugfix/*`: Bug fix branches

When working on a new feature or bug fix, create a new branch from `main`:

```bash
git checkout main
git pull upstream main
git checkout -b feature/your-feature-name
```

### Making Changes

1. Make your changes in your feature branch
2. Follow the coding standards and patterns used in the project
3. Add tests for your changes
4. Run tests locally to ensure they pass
5. Update documentation as needed

### Commits

Write clear, concise commit messages:

```
component: short description of the change

More detailed explanation of what the change does and why.
Include any background information or context that would help
reviewers understand the change.

Fixes #123
```

### Testing

Run tests locally before submitting a pull request:

```bash
# Run unit tests
make test

# Run integration tests
make integration-test

# Run end-to-end tests
make e2e-test
```

### Building

Build the project locally:

```bash
# Build all components
make build

# Build specific component
make build-api-server
make build-proxy
make build-cli
```

## Pull Requests

### Creating a Pull Request

1. Push your branch to your fork:
   ```bash
   git push origin feature/your-feature-name
   ```

2. Create a pull request through the GitHub UI
3. Fill in the pull request template with details about your changes
4. Link any related issues

### PR Description

A good pull request description includes:

- What changes you've made
- Why you made these changes
- How you tested the changes
- Any additional context or information that would help reviewers
- Screenshots or demos (if applicable)

### PR Review Process

1. At least one core team member must review and approve the PR
2. CI checks must pass
3. All review comments must be addressed
4. The PR must be up-to-date with the target branch

### After Merge

After your PR is merged, you can clean up your local branches:

```bash
git checkout main
git pull upstream main
git branch -d feature/your-feature-name
git push origin --delete feature/your-feature-name
```

## Code Standards

### Go Code Style

We follow the standard Go style guidelines:

- Use `gofmt` or `goimports` to format your code
- Follow the [Effective Go](https://golang.org/doc/effective_go) guidelines
- Use meaningful variable and function names
- Write comments for exported functions, types, and packages

### Project Structure

The project follows a standard Go project layout:

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
│   ├── e2e/             # End-to-end tests
│   └── integration/     # Integration tests
├── docs/                # Documentation
├── examples/            # Example configurations
└── deploy/              # Deployment manifests
```

### Documentation

Document your code and changes:

1. **Code Comments**: Add comments to explain complex logic or non-obvious behavior
2. **Package Documentation**: Add package-level documentation
3. **User Documentation**: Update user-facing documentation as needed

### Dependency Management

We use Go modules for dependency management:

- Add new dependencies with caution
- Consider the size, maintenance status, and licensing of dependencies
- Update `go.mod` and `go.sum` files

## Issue Tracking

### Reporting Bugs

When reporting bugs, include:

1. Steps to reproduce the bug
2. Expected behavior
3. Actual behavior
4. Environment details (OS, Go version, Kubernetes version, etc.)
5. Logs or error messages
6. Screenshots (if applicable)

### Feature Requests

When requesting features, include:

1. Clear description of the feature
2. Rationale for the feature
3. Example use cases
4. Implementation ideas (if any)

### Issue Labels

We use labels to categorize issues:

- `bug`: Bug reports
- `enhancement`: Feature requests
- `documentation`: Documentation improvements
- `good first issue`: Good for newcomers
- `help wanted`: Looking for contributors
- `wontfix`: Will not be implemented

## Release Process

### Versioning

We follow [Semantic Versioning](https://semver.org/):

- **Major version**: Incompatible API changes
- **Minor version**: Backward-compatible new features
- **Patch version**: Backward-compatible bug fixes

### Release Checklist

Before a release:

1. Update CHANGELOG.md
2. Update version numbers
3. Run full test suite
4. Generate documentation
5. Create release notes

### Creating a Release

1. Create a release branch:
   ```bash
   git checkout -b release-vX.Y.Z
   ```

2. Update version numbers in files:
   ```bash
   make update-version VERSION=X.Y.Z
   ```

3. Commit and push:
   ```bash
   git commit -m "Release vX.Y.Z"
   git push origin release-vX.Y.Z
   ```

4. Create a pull request to `main`
5. After merge, tag the release:
   ```bash
   git tag -a vX.Y.Z -m "Version X.Y.Z"
   git push origin vX.Y.Z
   ```

6. Create a GitHub release with the tag

## Communication

### Community Channels

- GitHub Issues: For bug reports and feature requests
- GitHub Discussions: For questions and discussions
- Slack: For real-time communication
- Mailing List: For announcements and longer discussions

### Meetings

- Community Meeting: Bi-weekly on Wednesdays at 10:00 AM UTC
- Development Meeting: Weekly on Mondays at 2:00 PM UTC

## License

By contributing to this project, you agree that your contributions will be licensed under the project's license.

## Acknowledgments

Thank you for contributing to the project! Your work helps make this project better for everyone.
