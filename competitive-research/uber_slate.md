# Uber's SLATE Research

## Introduction

SLATE (Short-Lived Application Testing Environment) is a system built at Uber that enables developers to create on-demand, ephemeral testing environments for end-to-end (E2E) testing of microservices. SLATE addresses several challenges with traditional staging environments:

1. **Test Flakiness**: Staging environments may not be as updated as production, leading to unreliable tests
2. **Sharing Environment Among Developers**: Limited ability for multiple developers to test simultaneously
3. **Production Safety**: Difficulty ensuring tests don't impact production data
4. **Cost**: High cost of maintaining separate staging environments for every service

## Core Concepts

### SLATE Environment

- On-demand, ephemeral testing environments with configurable TTL (default: 2 days)
- Multiple SLATE environments can be active simultaneously
- Developers can deploy multiple services in a SLATE environment from git-branches or git-refs
- Services deployed in SLATE are called Service-Under-Test (SUT)
- SUTs use production instances of dependencies by default, ensuring high fidelity while maintaining production safety

### Tenancy-Based Routing

- SLATE introduces the concept of "Tenancy" to distinguish test requests from production requests
- Tenancy is added in request headers as Jaeger Baggage (key: "request-tenancy")
- Jaeger Baggage is transparently propagated in a request call-tree across services
- Every SLATE environment has an associated tenancy of format `uber/test/slate/<slate_env_uuid>`
- Core infrastructure systems (logging, metrics, messaging, alerting) recognize and handle test traffic appropriately

### Routing Override

- Each SLATE environment is associated with a unique tenancy
- When a SUT is deployed, a mapping is created between the tenancy and routing info of SUT(s) in a config-cache
- The routing info (routing-override) contains information about SUT(s), IP, and protocol-specific ports
- Edge-gateway inspects user-id in config-cache and injects routing-overrides information into requests
- User association allows developers to seamlessly test from mobile apps

## Data Isolation

### Isolation of Entities

- SLATE uses Test-Accounts to isolate entities like riders, drivers, trips, etc. from production entities
- Test-Accounts allows developers to create test riders, drivers, and other profiles
- SLATE only allows association with test users, enforcing that SUTs only receive requests for test entities
- Test entities are created in production databases but are individually isolated from production entities

### Isolation in Kafka

- SUTs deployed in SLATE use production Kafka topics by default
- Kafka client is tenancy-aware, ensuring:
  - For producers: messages are auto-tagged with proper tenancy info
  - For SUT consumers: only receive traffic matching their tenancy
  - For production consumers: receive both production and test traffic

### Isolation in Asynchronous Workflows

- For asynchronous requests, Jaeger baggage info is stored in event payload
- Cadence (Uber's workflow orchestrator) task-lists are made configurable via config files or environment variables

### Isolation of Resources

- Some services need isolation of resources (databases, queues) for E2E testing
- SLATE uses environment-based configuration overrides
- Service owners can override E2E resources in a config file called slate.yaml
- Runtime environment of SUT is set to SLATE rather than production
- Most services share common resources across SLATE environments, but environment-specific resources are possible

## SLATE CLI and Server

### CLI Commands

- Simple CLI for developers to manage E2E testing environments:
  - `slate env create <env_name>` - Creates a new environment
  - `slate env clone <env_name>` - Clone an existing environment
  - `slate service deploy <service_name> -b <branch> -e <env_name>` - Deploys a service
  - `slate user associate <user_id> -e <env_name>` - Associate a user with SLATE environment
  - `slate env status <env_name>` - Shows status of commands
  - `slate apply <my_golden_setup>.yaml` - Run a set of commands specified in YAML

### Server Architecture

- SLATE server exposes APIs used by the CLI
- High-level architecture components:
  - Controller: Implements APIs to manage environments, services, and users
  - Environment Worker: Contains components like Reconciler, Automator, Associator, Deployer, and Builder
  - Infrastructure Abstraction layer
  - Integration with Build System, Peloton (deployment system), and Config-Cache

### Environment Worker Components

- **Builder**: Uses Uber's build-system to build service images
- **Deployer**: Uses Peloton to claim instances and deploy services
- **Associator**: Associates/dissociates users in an environment
- **Reconciler**: Maintains the state of SLATE environments and ensures routing-override information is current
- **Automator**: Runs commands specified in YAML, can run tests periodically and send Slack updates

## Additional Features

### Multiple Test Clients

- SLATE allows use of any API client (cURL, scripts, etc.) to issue requests to SUTs
- Mobile apps can be used to test features through user association
- Enables new use cases like sandbox environments for third-party customers and shadow testing

### Golden Environment

- Teams can create "golden" SLATE environments with:
  - Custom test-user profiles
  - Pre-deployed services
  - Sanity tests
  - Periodic regression tests
- Golden environments can be cloned with the `slate clone` command

### Observability

- Services in SLATE have similar logging, tracing, and monitoring as production
- Routing-coverage observability detects edge cases when requests hit production instead of SUT

## Benefits and Future Work

- Significantly improved developer experience and testing velocity
- Enabled testing changes across multiple services against production dependencies
- Efficient use of infrastructure (environments created on demand and reclaimed when not in use)
- Future experiments include multiple instances of services as SUTs and deprecation of staging environments

## Similarities to Signadot

SLATE's approach of using a shared environment with request-level isolation is similar to Signadot's approach. Both systems:

1. Create isolated testing environments within a shared cluster
2. Use request routing and context propagation for isolation
3. Provide lightweight, on-demand environments for testing
4. Enable testing against real dependencies and data
