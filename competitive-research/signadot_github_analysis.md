# Signadot GitHub Repositories Analysis

## Go SDK Repository

The Signadot Go SDK repository (https://github.com/signadot/go-sdk) serves as a Signadot API library generated with Swagger. It provides the foundation for interacting with Signadot's services programmatically.

### Repository Structure

- **client/**: Contains client implementations for various Signadot services
  - **artifacts/**: Client for managing artifacts
  - **cluster/**: Client for cluster operations
  - **job_logs/**: Client for accessing job logs
  - **jobs/**: Client for job management
  - **resource_plugins/**: Client for resource plugins
  - **route_groups/**: Client for route group management
  - **runner_groups/**: Client for runner groups
  - **sandboxes/**: Client for sandbox operations
  - **test_executions/**: Client for test execution operations
  - **tests/**: Client for test management
  - **signadot_api_client.go**: Main API client implementation

- **generate/**: Contains sources for code generation using Swagger
- **models/**: Data models used throughout the SDK
- **transport/**: Transport layer implementation for API communication
- **utils/**: Utility functions and helpers

### Key Features

1. **API Client**: The SDK provides a comprehensive client for interacting with all Signadot services.
2. **Sandbox Support**: Special support for local workloads in sandboxes, which requires specific handling.
3. **Swagger Generation**: The code is generated from Swagger specifications, ensuring API compatibility.
4. **Transport Configuration**: Customizable transport configuration for different environments.

### Implementation Notes

- The SDK is still in development and may have breaking changes.
- It's consumed by the Signadot CLI and supports local workloads in sandboxes.
- Local workloads require additional supporting code and root access.
- The SDK must include local workload specs in the swagger.json, unlike other SDKs.
- The source of truth for the swagger.json is in the `signadot` repository.

## CLI Repository

The Signadot CLI repository (https://github.com/signadot/cli) provides the command-line interface for interacting with Signadot services.

### Repository Structure

- **cmd/signadot/**: Contains the main CLI entry point
  - **main.go**: Main CLI implementation

- **internal/**: Internal packages used by the CLI
  - **buildinfo/**: Build information
  - **clio/**: CLI input/output utilities
  - **command/**: Command implementations
    - **artifact/**: Artifact commands
    - **bug/**: Bug reporting commands
    - **cluster/**: Cluster management commands
    - **jobrunnergroup/**: Job runner group commands
    - **jobs/**: Job management commands
    - **local/**: Local development commands
    - **locald/**: Local daemon commands
    - **logs/**: Log viewing commands
    - **resourceplugin/**: Resource plugin commands
    - **routegroup/**: Route group commands
    - **sandbox/**: Sandbox management commands
    - **test/**: Test commands
  - **config/**: Configuration management
  - **hack/**: Development utilities
  - **jsonexact/**: JSON parsing utilities
  - **locald/**: Local daemon implementation
  - **poll/**: Polling utilities
  - **print/**: Output formatting
  - **repoconfig/**: Repository configuration
  - **sdtab/**: Table formatting
  - **shallow/**: Shallow copy utilities
  - **spinner/**: Progress indicators
  - **utils/**: General utilities

- **scripts/**: Installation and utility scripts
- **tests/**: Test fixtures and utilities

### Key Features

1. **Command Structure**: Well-organized command hierarchy for different Signadot features
2. **Local Development**: Support for local development workflows
3. **Installation Script**: Easy installation via curl script
4. **Job Management**: Commands for submitting and managing jobs
5. **Sandbox Management**: Commands for creating and managing sandboxes
6. **Test Integration**: Commands for running and managing tests

### Implementation Notes

- The CLI is built on top of the Go SDK
- It uses libconnect for connectivity
- Starting with the next release, it has a dependency on a private repository
- Previous releases should continue to work
- The CLI can be installed via a script or built from source

## Relationship Between Repositories

The CLI repository depends on the Go SDK for interacting with Signadot services. The SDK provides the API client implementation, while the CLI provides the user interface and command structure. This separation of concerns allows for a clean architecture where:

1. The SDK handles API communication, data models, and transport
2. The CLI handles user interaction, command parsing, and output formatting

This design is similar to many other cloud-native tools and follows good software engineering practices.

## Key Insights for Implementation

1. **API-First Design**: Both repositories follow an API-first design, with the SDK generated from Swagger specifications.
2. **Command Pattern**: The CLI uses a command pattern for organizing functionality.
3. **Modular Architecture**: Both repositories have a modular architecture with clear separation of concerns.
4. **Local Development Support**: Special attention is given to local development workflows.
5. **Sandbox Isolation**: The sandbox concept is central to both repositories, allowing for isolated environments.

These insights will be valuable for designing our own implementation of a similar system.
