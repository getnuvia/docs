# API Design

This document outlines the API design for our Signadot/SLATE clone, including resource definitions, API endpoints, and best practices.

## API Principles

The API follows these key principles:

1. **Kubernetes-Native**: Resources follow Kubernetes API conventions
2. **Declarative**: Resources define desired state, controllers reconcile
3. **Versioned**: APIs use versioning to manage changes
4. **Self-Documenting**: OpenAPI specifications for all resources
5. **Idempotent**: Operations can be retried safely

## Core Resources

### Sandbox

Represents an isolated testing environment.

```yaml
apiVersion: sandbox.example.com/v1alpha1
kind: Sandbox
metadata:
  name: my-sandbox
  namespace: default
spec:
  # Timeout in seconds after which sandbox will be automatically deleted
  ttl: 86400  # 24 hours
  # Labels to apply to sandbox resources
  labels:
    app: my-app
    environment: testing
  # Routing key for traffic isolation
  routingKey: "my-unique-routing-key"
  # Service overrides
  services:
    - name: service-a
      # Reference to Kubernetes deployment, statefulset, etc.
      targetRef:
        kind: Deployment
        name: service-a
      # Container overrides
      containers:
        - name: main
          image: myregistry/service-a:feature-branch
          env:
            - name: DEBUG
              value: "true"
  # Resource quotas
  resources:
    cpu: "2"
    memory: "2Gi"
status:
  phase: Running  # Pending, Running, Terminating, Failed
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2023-06-01T12:00:00Z"
      reason: "SandboxReady"
      message: "Sandbox is ready"
  services:
    - name: service-a
      status: Ready
      endpoint: http://service-a.sandbox-my-sandbox.svc.cluster.local
```

### RouteGroup

Defines routing rules for directing traffic to sandbox environments.

```yaml
apiVersion: sandbox.example.com/v1alpha1
kind: RouteGroup
metadata:
  name: my-route-group
  namespace: default
spec:
  # Selector for services to apply routing rules
  selector:
    matchLabels:
      app: my-app
  # Default routing key when no matching route is found
  defaultRoutingKey: "production"
  # Header name containing routing key
  routingKeyHeader: "X-Routing-Key"
  # Individual routes
  routes:
    - routingKey: "my-unique-routing-key"
      # Target sandbox for this routing key
      targetSandbox: my-sandbox
      # HTTP path matching
      paths:
        - prefix: "/api"
        - exact: "/health"
      # Headers to add to requests
      additionalHeaders:
        X-Environment: "sandbox"
status:
  observedGeneration: 1
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2023-06-01T12:00:00Z"
      reason: "RouteGroupReady"
      message: "RouteGroup is ready"
```

### LocalWorkload

Defines a local workload that connects to a sandbox environment.

```yaml
apiVersion: sandbox.example.com/v1alpha1
kind: LocalWorkload
metadata:
  name: my-local-workload
  namespace: default
spec:
  # Reference to sandbox
  sandboxRef:
    name: my-sandbox
  # Service information
  service:
    name: service-b
    # Ports to expose
    ports:
      - name: http
        containerPort: 8080
        localPort: 8080
      - name: debug
        containerPort: 5005
        localPort: 5005
  # Connection information
  connection:
    # Target for tunneling (developer machine)
    target: "dev-machine-id"
    # Connection timeout in seconds
    timeout: 300
status:
  phase: Connected  # Pending, Connected, Disconnected, Failed
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2023-06-01T12:00:00Z"
      reason: "LocalWorkloadReady"
      message: "Local workload is connected"
  # Assigned routing key
  routingKey: "my-unique-routing-key"
  # Connection details
  connection:
    tunnelId: "tunnel-123456"
    connectedAt: "2023-06-01T12:00:00Z"
    lastHeartbeat: "2023-06-01T12:05:00Z"
```

## API Endpoints

The REST API exposes the following endpoints for managing resources:

### Sandbox Endpoints

```
GET /api/v1alpha1/namespaces/{namespace}/sandboxes       # List sandboxes
POST /api/v1alpha1/namespaces/{namespace}/sandboxes      # Create sandbox
GET /api/v1alpha1/namespaces/{namespace}/sandboxes/{name} # Get sandbox
PUT /api/v1alpha1/namespaces/{namespace}/sandboxes/{name} # Update sandbox
DELETE /api/v1alpha1/namespaces/{namespace}/sandboxes/{name} # Delete sandbox
GET /api/v1alpha1/namespaces/{namespace}/sandboxes/{name}/status # Get sandbox status
```

### RouteGroup Endpoints

```
GET /api/v1alpha1/namespaces/{namespace}/routegroups       # List route groups
POST /api/v1alpha1/namespaces/{namespace}/routegroups      # Create route group
GET /api/v1alpha1/namespaces/{namespace}/routegroups/{name} # Get route group
PUT /api/v1alpha1/namespaces/{namespace}/routegroups/{name} # Update route group
DELETE /api/v1alpha1/namespaces/{namespace}/routegroups/{name} # Delete route group
GET /api/v1alpha1/namespaces/{namespace}/routegroups/{name}/status # Get route group status
```

### LocalWorkload Endpoints

```
GET /api/v1alpha1/namespaces/{namespace}/localworkloads       # List local workloads
POST /api/v1alpha1/namespaces/{namespace}/localworkloads      # Create local workload
GET /api/v1alpha1/namespaces/{namespace}/localworkloads/{name} # Get local workload
PUT /api/v1alpha1/namespaces/{namespace}/localworkloads/{name} # Update local workload
DELETE /api/v1alpha1/namespaces/{namespace}/localworkloads/{name} # Delete local workload
GET /api/v1alpha1/namespaces/{namespace}/localworkloads/{name}/status # Get local workload status
```

## Custom Resource Definitions (CRDs)

The API resources will be implemented as Kubernetes Custom Resource Definitions (CRDs). Here's an example CRD for the Sandbox resource:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: sandboxes.sandbox.example.com
spec:
  group: sandbox.example.com
  names:
    kind: Sandbox
    listKind: SandboxList
    plural: sandboxes
    singular: sandbox
    shortNames:
      - sb
  scope: Namespaced
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - ttl
              properties:
                ttl:
                  type: integer
                  minimum: 300
                  maximum: 604800
                labels:
                  type: object
                  additionalProperties:
                    type: string
                routingKey:
                  type: string
                services:
                  type: array
                  items:
                    # Service override definition
                resources:
                  type: object
                  properties:
                    cpu:
                      type: string
                    memory:
                      type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum:
                    - Pending
                    - Running
                    - Terminating
                    - Failed
                conditions:
                  type: array
                  items:
                    # Condition definition
                services:
                  type: array
                  items:
                    # Service status definition
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Status
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
```

## API Conventions

### Resource Naming

- Resource names must be lowercase
- Use kebab-case for multi-word names
- Maximum 63 characters
- Must start and end with alphanumeric characters
- May contain dashes and dots

### Field Naming

- Use camelCase for field names
- Boolean fields should use positive naming (e.g., `enabled` instead of `disabled`)
- Field names should be descriptive and unambiguous

### Status Conditions

All resources use a standard condition structure:

```yaml
conditions:
  - type: Ready  # Condition type
    status: "True"  # True, False, Unknown
    lastTransitionTime: "2023-06-01T12:00:00Z"  # Last time the condition changed
    reason: "SandboxReady"  # Machine-readable reason
    message: "Sandbox is ready"  # Human-readable message
```

Common condition types:
- `Ready`: Resource is ready for use
- `Progressing`: Resource is being reconciled
- `Degraded`: Resource is in a degraded state
- `Suspended`: Resource is temporarily suspended

### API Versioning

- `v1alpha1`: Initial unstable version
- `v1beta1`: Stabilizing version with potential changes
- `v1`: Stable version with backward compatibility

## Error Handling

The API uses standard HTTP status codes:
- `200 OK`: Success
- `201 Created`: Resource created
- `400 Bad Request`: Invalid request
- `401 Unauthorized`: Authentication required
- `403 Forbidden`: Permission denied
- `404 Not Found`: Resource not found
- `409 Conflict`: Resource conflict
- `422 Unprocessable Entity`: Validation error
- `500 Internal Server Error`: Server error

Error responses use a standard format:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Sandbox 'my-sandbox' not found",
  "reason": "NotFound",
  "details": {
    "name": "my-sandbox",
    "kind": "Sandbox"
  },
  "code": 404
}
```

## Webhook Validation

The API uses admission webhooks for validating resources:

1. **Validating Webhooks**: Validate resource fields
2. **Mutating Webhooks**: Set default values and transform requests

Example webhook configurations will be provided in the implementation phase.

## OpenAPI Specification

The API will be fully documented using OpenAPI Specification (OAS) 3.0, providing:
- Complete API documentation
- Schema validation
- Client code generation
- Interactive API documentation using Swagger UI

The OpenAPI specification will be generated during build and served by the API server.

## Next Steps

1. Implement Go structs for resource types
2. Create validation functions
3. Generate CRDs from Go structs
4. Implement API endpoints
5. Create OpenAPI documentation