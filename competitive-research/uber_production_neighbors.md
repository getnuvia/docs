# Uber's Production Neighbors Debugging with SLATE

## Introduction

This article explores the debugging capabilities built on top of Uber's SLATE (Short-Lived Application Testing Environment) platform. SLATE bridges the gap between local development and production by allowing services under test to be deployed and work alongside production upstream and downstream services. This enables various use cases, including feature development within a production environment and replicating production bugs.

## Debugging Approaches

The article outlines three high-level debugging options developed for SLATE:

1. **Remote debugging a SLATE deployed instance**
2. **Local Debugging in laptop/dev pod machine**
3. **Debug issues by filtered monitoring**

### Traditional Debugging Approaches and Limitations

#### Debug via Logs
- Fundamental practice providing insights into program execution
- Inefficient logging can clutter the system with irrelevant information
- Limited visibility into complex processes

#### Debug via Staging
- Developer-controlled environment mirroring production setup
- May differ from live environment and provide false confidence
- Longer turnaround time

#### Debug Locally
- Essential for faster iteration to test service in isolation
- Challenging to debug user scenarios due to constraints in simultaneously debugging multiple services

## Remote Debugging of SLATE Instance

Remote debugging allows developers to step through statements and monitor variables without needing to add new logs and redeploy, which addresses the limitations of log-based debugging.

### High-level Goals
- Deploy a debuggable binary/code on a SLATE container
- Add breakpoints and tracepoints to a service under test
- See values of different parameters when hitting a breakpoint
- Create a seamless developer experience similar to remote debugging
- Design solutions compliant with security and privacy issues

### Design
SLATE leverages production infrastructure but requires modifications to facilitate debugging:
1. Enabling generation of builds with integrated debugging tools
2. Configuring software execution with remote debugging options
3. Facilitating developer access to remote containers by allocating and exposing ports

### Debuggable Deployment
- Modifications to the deployment pipeline to support debugging for SLATE
- Multiple components must recognize the type of binary and configure features accordingly

### Allocating Ports
- New debug port exposed, similar to gRPC/HTTP port
- Port exposure only for debuggable SLATE deployments
- Security review required for new port exposure

### Access Control
- Restricted to LDAP users (service developers/owners) to ensure minimum security
- Secure SSH connection established between local and remote systems
- SSH authorization using a randomly generated 16-digit password accessible only to service owners

### Debugger Execution
- Debugger runs the application within a dedicated debugging server
- Process blocks, awaiting attachment by the debugger client
- Debugger process listens on a specific TCP/IP network port

### Controlling Program Execution
- Debugging clients (VSCode, GoLand, JetBrains) connect via the debug port
- Clients issue commands for various debugging tasks
- Remote debugging enables debugging on diverse environments, configurations, or architectures

### Limitations
- Remote debugging on production infrastructure limited to read-only operations
- Large iteration time as each change involves build, deploy, and test

## Local Debugging using SLATE Attach

SLATE Attach fills the gap between remote debugging and local development by creating a local debugging experience connected to production upstream and downstream services.

### High-level Goals
- Reduce the code-deploy-test cycle time
- Provide E2E testing with local development instances
- Ensure production isolation and safety

### Iteration Cycle
- The smaller the iteration cycle, the more efficient the use of developers' time
- SLATE Attach significantly reduces the iteration cycle compared to remote debugging

### Need for SLATE Attach
- Iterative development generates a build binary at a faster pace
- Reduced code-deploy-test cycle
- Faster identification and resolution of local, E2E failures
- Faster setup time
- Avoid the need for service changes or onboarding

### Design
- Introduces a SLATE proxy that handles test requests aimed at SLATE instances for local debugging
- Requests redirected to the appropriate local developer machine
- Enables faster iteration and improved developer productivity

### Control Plane
- Enables services running in local laptops or devpods to attach to a SLATE environment
- Local laptop/devpod attaches local environment credentials to a SLATE environment
- Prerequisite is to create a SLATE environment
- Allows mapping updates in routing control DB and local routing DB

### Data Plane
- Handles flow of test requests from different clients (mobile, studio, web)
- Involves routing override header and host tenancy mapping
- Control plane ensures routing override and host mapping maintained in different databases

### Call Flow
1. User initiates SLATE attach from local laptop/devpod
2. SLATE CLI calls Attach() API of SLATE Backend
3. SLATE Backend fetches Proxy information from SLATE Proxy
4. SLATE Backend updates routing override in routing control DB
5. User initiates SSH Session using Cerberus CLI
6. Cerberus gateway adds mapping of deputized tenancy/UUID to laptop credentials

### Routing Control DB
- Maps test tenancy to routing overrides and user account UUIDs to test tenancy
- Stores SLATE Proxy host:port against the service under test
- Ensures requests targeting a particular SLATE environment reach SLATE Proxy
- SLATE Proxy filters and routes requests to development instance in user's machine

### Local Routing DB
- Contains development instance's credentials attached to SLATE environment
- SLATE Proxy interacts with local routing DB to fetch routing credentials
- Routes request to service-under-test running in local environment

### Limitations
- Running a service locally may not be feasible for complex services requiring production-only dependencies
- Limited to test requests to ensure production safety
- Requests timeout on longer wait for a debug request in local

## SLATE Sniffer for Filtered Monitoring

SLATE Sniffer aims to provide better observability on production beyond logs in uMonitor Tool.

### Goals
- Capture request and responses as a filter of a service and UUID
- Support and filter on Production and Test requests

## Impact

- Plug-and-play development environment to improve developer productivity
- Ability to create local experiences that co-work with production
- Increased Developer Velocity: Production debugging helps developers identify and fix issues more efficiently

## Conclusion

SLATE's debugging features strike a balance between security and developer requirements, introducing a new paradigm for developers' code-related activities and service bootstrapping. The platform aims to shift quality left and create visibility on potential issues at the early stage of development.

## Similarities to Signadot

The debugging capabilities built on SLATE are similar to what Signadot aims to provide:
1. Both systems enable testing and debugging against real production dependencies
2. Both use request routing and context propagation for isolation
3. Both provide mechanisms for local development to interact with production services
4. Both aim to improve developer productivity by reducing the iteration cycle
