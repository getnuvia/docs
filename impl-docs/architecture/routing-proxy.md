# Routing Proxy

This document outlines the design and implementation of the routing proxy component for our Signadot/SLATE clone.

## Overview

The routing proxy is a critical component of the data plane that intercepts and routes traffic based on routing rules. It enables request isolation and directs traffic to appropriate sandbox environments or production services.

```
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│               │      │               │      │               │
│   Client      │──────►  Routing      │──────►  Target       │
│   Request     │      │  Proxy        │      │  Service      │
│               │      │               │      │               │
└───────────────┘      └───────────────┘      └───────────────┘
                              │
                              │
                       ┌──────┴──────┐
                       │             │
                       │  Routing    │
                       │  Rules      │
                       │             │
                       └─────────────┘
```

## Key Features

1. **Header-Based Routing**: Route requests based on HTTP headers
2. **Context Propagation**: Maintain request context across service boundaries
3. **Traffic Splitting**: Support for percentage-based traffic splitting
4. **Protocol Support**: Handle HTTP, HTTP/2, gRPC, and WebSockets
5. **Circuit Breaking**: Protect services from cascading failures
6. **Load Balancing**: Distribute traffic among service instances
7. **Metrics Collection**: Collect metrics for observability

## Design Approach

We have two main options for implementing the routing proxy:

### Option 1: Leverage Existing Service Mesh

Use an existing service mesh like Istio, Linkerd, or Envoy and extend it with custom routing capabilities.

**Pros:**
- Leverages battle-tested proxies
- Reduces implementation effort
- Provides additional features like mTLS, observability

**Cons:**
- Introduces dependency on external project
- May add complexity to setup
- Customizing routing behavior could be challenging

### Option 2: Custom Lightweight Proxy

Implement a lightweight proxy specific to our needs using Go.

**Pros:**
- Full control over implementation
- Minimized resource footprint
- Tailored specifically to our use case

**Cons:**
- Requires more development effort
- Need to implement features that come for free with service meshes
- Need to handle protocol complexities

### Decision

For our implementation, we'll follow a hybrid approach:

1. **Phase 1**: Implement a lightweight custom proxy using Go and Envoy as the base
2. **Phase 2**: Add integration with popular service meshes as an option

This approach gives us control over the core functionality while allowing integration with existing tools.

## Implementation

### Core Components

#### 1. Proxy Server

The proxy server intercepts network traffic and applies routing rules.

```go
type ProxyServer struct {
    routingTable *RoutingTable
    metrics      *Metrics
    config       *Config
}

func NewProxyServer(config *Config) (*ProxyServer, error) {
    // Initialize proxy server
    return &ProxyServer{
        routingTable: NewRoutingTable(),
        metrics:      NewMetrics(),
        config:       config,
    }, nil
}

func (p *ProxyServer) Start(ctx context.Context) error {
    // Start HTTP and gRPC listeners
    go p.startHTTPServer()
    go p.startGRPCServer()
    
    // Wait for shutdown signal
    <-ctx.Done()
    return p.Shutdown()
}

func (p *ProxyServer) Shutdown() error {
    // Graceful shutdown
    return nil
}
```

#### 2. Routing Table

The routing table stores and manages routing rules.

```go
type RoutingTable struct {
    mu    sync.RWMutex
    rules map[string][]*RoutingRule
}

type RoutingRule struct {
    RouteKey        string
    TargetService   string
    PathPrefixes    []string
    HeaderMatchers  map[string]string
    AdditionalHeaders map[string]string
}

func (rt *RoutingTable) AddRule(rule *RoutingRule) {
    rt.mu.Lock()
    defer rt.mu.Unlock()
    
    rt.rules[rule.RouteKey] = append(rt.rules[rule.RouteKey], rule)
}

func (rt *RoutingTable) RemoveRule(routeKey string) {
    rt.mu.Lock()
    defer rt.mu.Unlock()
    
    delete(rt.rules, routeKey)
}

func (rt *RoutingTable) GetTargetForRequest(req *http.Request) (string, map[string]string) {
    rt.mu.RLock()
    defer rt.mu.RUnlock()
    
    // Extract route key from header
    routeKey := req.Header.Get("X-Route-Key")
    if routeKey == "" {
        return getDefaultTarget(req), nil
    }
    
    // Find matching rule
    rules, exists := rt.rules[routeKey]
    if !exists {
        return getDefaultTarget(req), nil
    }
    
    // Apply rules based on path and headers
    for _, rule := range rules {
        if matchesRule(req, rule) {
            return rule.TargetService, rule.AdditionalHeaders
        }
    }
    
    return getDefaultTarget(req), nil
}
```

#### 3. HTTP Handler

Handles HTTP requests and applies routing rules.

```go
func (p *ProxyServer) httpHandler(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    
    // Get target service and additional headers
    target, additionalHeaders := p.routingTable.GetTargetForRequest(r)
    
    // Create proxy request
    proxyReq, err := http.NewRequest(r.Method, target+r.URL.Path, r.Body)
    if err != nil {
        http.Error(w, "Error creating proxy request", http.StatusInternalServerError)
        return
    }
    
    // Copy original headers
    copyHeaders(r.Header, proxyReq.Header)
    
    // Add additional headers
    for k, v := range additionalHeaders {
        proxyReq.Header.Set(k, v)
    }
    
    // Send proxy request
    resp, err := http.DefaultClient.Do(proxyReq)
    if err != nil {
        http.Error(w, "Error proxying request", http.StatusBadGateway)
        return
    }
    defer resp.Body.Close()
    
    // Copy response headers
    copyHeaders(resp.Header, w.Header())
    
    // Set status code
    w.WriteHeader(resp.StatusCode)
    
    // Copy response body
    io.Copy(w, resp.Body)
    
    // Record metrics
    p.metrics.RecordRequest(r.URL.Path, resp.StatusCode, time.Since(start))
}
```

#### 4. gRPC Interceptor

Intercepts gRPC requests for routing.

```go
func (p *ProxyServer) grpcInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    start := time.Now()
    
    // Extract metadata
    md, _ := metadata.FromIncomingContext(ctx)
    
    // Get target service
    target, additionalMetadata := p.routingTable.GetTargetForGRPC(md)
    
    // Create new context with combined metadata
    newMD := metadata.Join(md, additionalMetadata)
    newCtx := metadata.NewOutgoingContext(ctx, newMD)
    
    // Create connection to target
    conn, err := grpc.Dial(target, grpc.WithInsecure())
    if err != nil {
        return nil, status.Errorf(codes.Unavailable, "failed to connect to target: %v", err)
    }
    defer conn.Close()
    
    // Create new client and forward request
    // (This is simplified - actual implementation would be more dynamic)
    resp, err := forwardGRPCRequest(newCtx, conn, info.FullMethod, req)
    
    // Record metrics
    p.metrics.RecordGRPCRequest(info.FullMethod, status.Code(err), time.Since(start))
    
    return resp, err
}

// Helper to forward gRPC requests to the target service
func forwardGRPCRequest(ctx context.Context, conn *grpc.ClientConn, fullMethod string, req interface{}) (interface{}, error) {
    // Implementation would use reflection or generated code based on service definitions
    // This is a simplified placeholder
    return nil, status.Errorf(codes.Unimplemented, "dynamic forwarding not implemented")
}
```

#### 5. Context Propagation

Handles propagation of context across service boundaries.

```go
// Constants for header names
const (
    HeaderRouteKey = "X-Route-Key"
    HeaderTraceID  = "X-Trace-ID"
    HeaderSpanID   = "X-Span-ID"
)

// PropagateContext ensures context headers are properly propagated
func PropagateContext(src, dst http.Header) {
    // Propagate routing key
    if routeKey := src.Get(HeaderRouteKey); routeKey != "" {
        dst.Set(HeaderRouteKey, routeKey)
    }
    
    // Propagate tracing information
    if traceID := src.Get(HeaderTraceID); traceID != "" {
        dst.Set(HeaderTraceID, traceID)
    }
    
    if spanID := src.Get(HeaderSpanID); spanID != "" {
        dst.Set(HeaderSpanID, spanID)
    }
    
    // Additional context headers can be added here
}

// For gRPC
func PropagateGRPCContext(src, dst metadata.MD) {
    // Similar implementation for gRPC metadata
}
```

### Configuration

The routing proxy should be configurable via configuration files and dynamic updates:

```yaml
# config.yaml
server:
  httpPort: 8080
  grpcPort: 8081
  metricsPort: 9090

routing:
  routeKeyHeader: "X-Route-Key"
  defaultTarget: "default-service"
  
  rules:
    - routeKey: "sandbox-123"
      targetService: "service-a.sandbox-123.svc.cluster.local"
      pathPrefixes:
        - "/api/v1"
      additionalHeaders:
        X-Environment: "sandbox"
        
    - routeKey: "sandbox-456"
      targetService: "service-b.sandbox-456.svc.cluster.local"
      pathPrefixes:
        - "/api/v2"
      headerMatchers:
        Content-Type: "application/json"
```

### Dynamic Updates

The routing table should support dynamic updates from the control plane:

```go
// UpdateRoutingTable updates the routing rules
func (p *ProxyServer) UpdateRoutingTable(rules []*RoutingRule) {
    newTable := NewRoutingTable()
    
    for _, rule := range rules {
        newTable.AddRule(rule)
    }
    
    // Atomic update of routing table
    p.routingTable = newTable
}

// Watch for route group changes
func (p *ProxyServer) WatchRouteGroups(ctx context.Context, client client.Client) {
    for {
        routeGroups := &sandboxv1alpha1.RouteGroupList{}
        err := client.List(ctx, routeGroups)
        if err != nil {
            log.Error(err, "Failed to list route groups")
            time.Sleep(5 * time.Second)
            continue
        }
        
        rules := convertToRoutingRules(routeGroups.Items)
        p.UpdateRoutingTable(rules)
        
        time.Sleep(30 * time.Second)
    }
}
```

## Service Mesh Integration

In addition to our custom proxy implementation, we'll provide integration with popular service meshes.

### Istio Integration

```go
type IstioIntegration struct {
    client istioclientset.Interface
}

func (i *IstioIntegration) SyncRouteGroups(routeGroups []sandboxv1alpha1.RouteGroup) error {
    // Convert RouteGroups to Istio VirtualService resources
    for _, rg := range routeGroups {
        vs := convertToVirtualService(rg)
        
        // Create or update virtual service
        _, err := i.client.NetworkingV1alpha3().VirtualServices(rg.Namespace).
            Update(context.Background(), vs, metav1.UpdateOptions{})
        if err != nil {
            if errors.IsNotFound(err) {
                _, err = i.client.NetworkingV1alpha3().VirtualServices(rg.Namespace).
                    Create(context.Background(), vs, metav1.CreateOptions{})
            }
            if err != nil {
                return err
            }
        }
    }
    
    return nil
}
```

### Linkerd Integration

Similar integration can be implemented for Linkerd using Service Profiles.

## Deployment

The routing proxy can be deployed in several ways:

### 1. Sidecar Container

Deploy the proxy as a sidecar container in each pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-service
spec:
  template:
    spec:
      containers:
        - name: app
          image: example-service:latest
        - name: proxy
          image: sandbox-proxy:latest
          ports:
            - containerPort: 8080
```

### 2. Service Mesh Integration

Use an existing service mesh with our custom configuration:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: example-service
spec:
  hosts:
    - example-service
  http:
    - match:
        - headers:
            X-Route-Key:
              exact: sandbox-123
      route:
        - destination:
            host: example-service.sandbox-123.svc.cluster.local
```

### 3. API Gateway

Deploy as an API gateway at the edge of the cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sandbox-gateway
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: proxy
          image: sandbox-proxy:latest
          ports:
            - containerPort: 8080
          env:
            - name: PROXY_MODE
              value: "gateway"
```

## Performance Considerations

1. **Efficient Routing**: Use efficient data structures for route matching
2. **Connection Pooling**: Reuse connections to backend services
3. **Batching**: Batch routing table updates
4. **Caching**: Cache route resolution results
5. **Profiling**: Regularly profile proxy performance

Example connection pool implementation:

```go
type ConnectionPool struct {
    mu    sync.RWMutex
    pools map[string]*http.Client
}

func (cp *ConnectionPool) GetClient(target string) *http.Client {
    cp.mu.RLock()
    client, exists := cp.pools[target]
    cp.mu.RUnlock()
    
    if exists {
        return client
    }
    
    cp.mu.Lock()
    defer cp.mu.Unlock()
    
    // Double-check in case another goroutine created it
    if client, exists = cp.pools[target]; exists {
        return client
    }
    
    // Create new client with connection pooling
    transport := &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    }
    
    client = &http.Client{
        Transport: transport,
        Timeout:   30 * time.Second,
    }
    
    cp.pools[target] = client
    return client
}
```

## Metrics and Monitoring

The proxy should expose metrics for monitoring:

```go
type Metrics struct {
    requestCounter   *prometheus.CounterVec
    requestDuration  *prometheus.HistogramVec
    routingErrors    *prometheus.CounterVec
}

func NewMetrics() *Metrics {
    m := &Metrics{
        requestCounter: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "proxy_requests_total",
                Help: "Total number of requests processed by the proxy",
            },
            []string{"path", "status_code"},
        ),
        requestDuration: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Name:    "proxy_request_duration_seconds",
                Help:    "Request duration in seconds",
                Buckets: prometheus.DefBuckets,
            },
            []string{"path"},
        ),
        routingErrors: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "proxy_routing_errors_total",
                Help: "Total number of routing errors",
            },
            []string{"error_type"},
        ),
    }
    
    // Register metrics
    prometheus.MustRegister(
        m.requestCounter,
        m.requestDuration,
        m.routingErrors,
    )
    
    return m
}

func (m *Metrics) RecordRequest(path string, statusCode int, duration time.Duration) {
    m.requestCounter.WithLabelValues(path, fmt.Sprintf("%d", statusCode)).Inc()
    m.requestDuration.WithLabelValues(path).Observe(duration.Seconds())
}
```

## Next Steps

1. Implement basic HTTP proxy functionality
2. Add gRPC support
3. Implement routing table updates
4. Add context propagation
5. Implement service mesh integration
6. Set up metrics and monitoring
7. Conduct performance testing