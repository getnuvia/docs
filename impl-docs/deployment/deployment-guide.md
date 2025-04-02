# Deployment Guide

This document provides instructions for deploying the Signadot/SLATE clone in different environments.

## Deployment Options

The system can be deployed in several configurations:

1. **Single-Cluster Deployment**: All components in one Kubernetes cluster
2. **Multi-Cluster Deployment**: Control plane and data plane in separate clusters
3. **Development Deployment**: Lightweight deployment for development/testing

## Prerequisites

Before deploying, ensure you have:

- Kubernetes cluster (v1.22+)
- kubectl configured for your cluster
- Helm v3.0+
- Domain name (for production deployments)
- TLS certificates (for production deployments)

## Single-Cluster Deployment

### 1. Prepare Namespace

```bash
# Create namespace
kubectl create namespace sandbox-system

# Create TLS secret (if using HTTPS)
kubectl create secret tls sandbox-tls \
  --cert=/path/to/tls.crt \
  --key=/path/to/tls.key \
  -n sandbox-system
```

### 2. Install with Helm

```bash
# Add Helm repository
helm repo add sandbox https://example.com/charts
helm repo update

# Install chart
helm install sandbox-system sandbox/sandbox-system \
  --namespace sandbox-system \
  --set apiServer.replicas=2 \
  --set routing.proxy.replicas=3 \
  --set persistence.size=10Gi \
  --set ingress.host=sandbox.example.com \
  --set ingress.tls.enabled=true \
  --set ingress.tls.secretName=sandbox-tls
```

### 3. Verify Installation

```bash
# Check pods
kubectl get pods -n sandbox-system

# Check services
kubectl get svc -n sandbox-system

# Check ingress
kubectl get ingress -n sandbox-system
```

### 4. Configure CLI

```bash
# Install CLI
curl -L https://example.com/install.sh | bash

# Configure CLI
sandboxctl config set server.url https://sandbox.example.com
sandboxctl config set token <your-token>
```

## Multi-Cluster Deployment

### 1. Prepare Control Plane Cluster

```bash
# Create namespace
kubectl create namespace sandbox-control-plane --context control-plane-cluster

# Create kubeconfig secret for data plane access
kubectl create secret generic dataplane-kubeconfig \
  --from-file=kubeconfig=/path/to/dataplane-kubeconfig.yaml \
  -n sandbox-control-plane \
  --context control-plane-cluster
```

### 2. Install Control Plane Components

```bash
helm install sandbox-control-plane sandbox/sandbox-control-plane \
  --namespace sandbox-control-plane \
  --set dataplane.kubeconfig=dataplane-kubeconfig \
  --set apiServer.replicas=2 \
  --set persistence.size=10Gi \
  --set ingress.host=sandbox-api.example.com \
  --set ingress.tls.enabled=true \
  --context control-plane-cluster
```

### 3. Prepare Data Plane Cluster

```bash
# Create namespace
kubectl create namespace sandbox-data-plane --context data-plane-cluster
```

### 4. Install Data Plane Components

```bash
helm install sandbox-data-plane sandbox/sandbox-data-plane \
  --namespace sandbox-data-plane \
  --set controlPlane.url=https://sandbox-api.example.com \
  --set routing.proxy.replicas=3 \
  --context data-plane-cluster
```

### 5. Configure CLI

```bash
sandboxctl config set server.url https://sandbox-api.example.com
sandboxctl config set token <your-token>
```

## Development Deployment

### 1. Set Up Kind Cluster

```bash
# Create Kind configuration
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 8080
    protocol: TCP
EOF

# Create cluster
kind create cluster --config kind-config.yaml --name sandbox-dev
```

### 2. Install Development Version

```bash
# Install CRDs
kubectl apply -f deploy/crds/

# Install components
kubectl apply -f deploy/dev/
```

### 3. Port Forward

```bash
# Port forward API server
kubectl port-forward svc/sandbox-api-server 8080:80
```

### 4. Configure CLI

```bash
sandboxctl config set server.url http://localhost:8080
sandboxctl config set token dev-token
```

## Custom Resource Configurations

### Sandbox

```yaml
apiVersion: sandbox.example.com/v1alpha1
kind: Sandbox
metadata:
  name: my-sandbox
  namespace: default
spec:
  ttl: 86400  # 24 hours
  labels:
    app: my-app
    environment: testing
  services:
    - name: service-a
      targetRef:
        kind: Deployment
        name: service-a
      containers:
        - name: main
          image: myregistry/service-a:feature-branch
          env:
            - name: DEBUG
              value: "true"
  resources:
    cpu: "2"
    memory: "2Gi"
```

### RouteGroup

```yaml
apiVersion: sandbox.example.com/v1alpha1
kind: RouteGroup
metadata:
  name: my-route-group
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-app
  defaultRoutingKey: "production"
  routingKeyHeader: "X-Routing-Key"
  routes:
    - routingKey: "my-sandbox"
      targetSandbox: my-sandbox
      paths:
        - prefix: "/api"
      additionalHeaders:
        X-Environment: "sandbox"
```

### LocalWorkload

```yaml
apiVersion: sandbox.example.com/v1alpha1
kind: LocalWorkload
metadata:
  name: my-local-workload
  namespace: default
spec:
  sandboxRef:
    name: my-sandbox
  service:
    name: service-b
    ports:
      - name: http
        containerPort: 8080
        localPort: 8080
  connection:
    timeout: 300
```

## Production Considerations

### High Availability

For production deployments, ensure high availability:

1. **API Server**: Deploy at least 2 replicas
2. **Routing Proxy**: Deploy at least 3 replicas
3. **Database**: Use a replicated database setup
4. **Load Balancer**: Use a production-grade load balancer

```yaml
apiServer:
  replicas: 3
  resources:
    requests:
      cpu: 1
      memory: 1Gi
    limits:
      cpu: 2
      memory: 2Gi
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - sandbox-api-server
          topologyKey: kubernetes.io/hostname
```

### Security

Enhance security for production deployments:

1. **TLS**: Enable TLS for all communications
2. **Network Policies**: Restrict pod-to-pod communication
3. **RBAC**: Configure fine-grained access control
4. **Pod Security**: Apply pod security policies

```yaml
security:
  tls:
    enabled: true
    certManager: true
  networkPolicies:
    enabled: true
  rbac:
    strictMode: true
  podSecurity:
    level: restricted
```

### Monitoring

Set up monitoring for production:

1. **Prometheus**: Metrics collection
2. **Grafana**: Visualization
3. **Alerting**: Configure alerts for critical conditions

```yaml
monitoring:
  prometheus:
    enabled: true
    scrapeInterval: 15s
  grafana:
    enabled: true
    dashboards:
      - apiServer
      - routingProxy
      - sandboxes
  alerting:
    enabled: true
    slackWebhook: "https://hooks.slack.com/services/..."
```

### Backup and Restore

Configure backup for critical data:

```yaml
backup:
  schedule: "0 2 * * *"  # Daily at 2 AM
  retention: 7           # Keep backups for 7 days
  storage:
    type: s3
    bucket: sandbox-backups
    region: us-west-2
```

## Upgrading

Upgrade procedure:

1. Back up existing data
2. Update Helm repository
3. Upgrade using Helm

```bash
# Backup (if needed)
kubectl exec -n sandbox-system sandbox-db-0 -- pg_dump -U sandbox > backup.sql

# Update Helm repository
helm repo update

# Upgrade
helm upgrade sandbox-system sandbox/sandbox-system \
  --namespace sandbox-system \
  --set apiServer.replicas=2 \
  --set routing.proxy.replicas=3
```

## Troubleshooting

### Common Issues

#### API Server Not Starting

```bash
# Check logs
kubectl logs -n sandbox-system deployment/sandbox-api-server

# Check events
kubectl get events -n sandbox-system

# Verify configuration
kubectl get configmap -n sandbox-system sandbox-config -o yaml
```

#### Routing Issues

```bash
# Check proxy logs
kubectl logs -n sandbox-system deployment/sandbox-routing-proxy

# Check route groups
kubectl get routegroups -A

# Test routing
curl -H "X-Route-Key: my-sandbox" http://example.com/api/test
```

#### Database Connectivity

```bash
# Check database status
kubectl get statefulset -n sandbox-system sandbox-db

# Check database connection
kubectl exec -it -n sandbox-system sandbox-api-server-0 -- \
  curl -v telnet://sandbox-db:5432
```

### Support Information

When reporting issues, include:

1. System version
2. Kubernetes version
3. Deployment configuration
4. Relevant logs
5. Error messages
6. Steps to reproduce

```bash
# Collect diagnostics
sandboxctl diagnostics > diagnostics.zip
```

## Next Steps

After deployment:

1. Configure authentication
2. Set up user accounts
3. Create sandboxes
4. Configure routing
5. Set up CI/CD integration
