# Local Development Integration

This document outlines the implementation of local development integration for our Signadot/SLATE clone, enabling developers to connect local services to the sandbox environment.

## Overview

Local development integration allows developers to run services locally on their development machines while connecting them to services running in sandbox environments. This provides a powerful workflow for testing changes without deploying to the cluster.

```
┌─────────────────┐       ┌─────────────────┐      ┌─────────────────┐
│                 │       │                 │      │                 │
│  Local Service  │◄─────►│  Local Agent    │◄────►│  Routing Proxy  │
│                 │       │                 │      │                 │
└─────────────────┘       └─────────────────┘      └─────────────────┘
                                                          │
                                                          ▼
                                                   ┌─────────────────┐
                                                   │                 │
                                                   │  Remote         │
                                                   │  Services       │
                                                   │                 │
                                                   └─────────────────┘
```

## Key Features

1. **Local-to-Remote Connectivity**: Connect local services to remote dependencies
2. **Secure Tunneling**: Establish secure tunnels between local and remote environments
3. **Bidirectional Routing**: Route traffic in both directions
4. **Header Propagation**: Maintain routing context across service boundaries
5. **Resource Mapping**: Map local resources to sandbox resources

## Implementation Components

### 1. Local Agent

The local agent runs on the developer's machine and facilitates communication between local services and the remote environment.

```go
type LocalAgent struct {
    config      *Config
    tunnels     map[string]*Tunnel
    routingKey  string
    client      *client.Client
    stopCh      chan struct{}
}

func NewLocalAgent(config *Config) (*LocalAgent, error) {
    client, err := client.NewClient(config.ServerURL, config.Token)
    if err != nil {
        return nil, err
    }
    
    return &LocalAgent{
        config:  config,
        tunnels: make(map[string]*Tunnel),
        client:  client,
        stopCh:  make(chan struct{}),
    }, nil
}

func (a *LocalAgent) Start() error {
    // Register with server
    workload, err := a.client.CreateLocalWorkload(a.config.Namespace, a.config.LocalService, &api.LocalWorkloadSpec{
        SandboxRef: api.ObjectRef{
            Name: a.config.SandboxName,
        },
        Service: api.ServiceSpec{
            Name:  a.config.ServiceName,
            Ports: a.config.Ports,
        },
    })
    if err != nil {
        return err
    }
    
    a.routingKey = workload.Status.RoutingKey
    
    // Set up tunnels for each port
    for _, port := range a.config.Ports {
        tunnel, err := NewTunnel(port.LocalPort, port.RemotePort, a.routingKey)
        if err != nil {
            return err
        }
        
        a.tunnels[port.Name] = tunnel
        go tunnel.Start()
    }
    
    // Start heartbeat
    go a.heartbeat()
    
    return nil
}

func (a *LocalAgent) Stop() {
    close(a.stopCh)
    
    // Close all tunnels
    for _, tunnel := range a.tunnels {
        tunnel.Stop()
    }
    
    // Deregister with server
    a.client.DeleteLocalWorkload(a.config.Namespace, a.config.LocalService)
}

func (a *LocalAgent) heartbeat() {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            if err := a.client.UpdateLocalWorkloadStatus(a.config.Namespace, a.config.LocalService); err != nil {
                log.Printf("Failed to update heartbeat: %v", err)
            }
        case <-a.stopCh:
            return
        }
    }
}
```

### 2. Tunneling

The tunneling component establishes secure communication channels between local and remote environments.

```go
type Tunnel struct {
    localPort    int
    remotePort   int
    routingKey   string
    listener     net.Listener
    connections  map[string]net.Conn
    stopCh       chan struct{}
    mu           sync.Mutex
}

func NewTunnel(localPort, remotePort int, routingKey string) (*Tunnel, error) {
    listener, err := net.Listen("tcp", fmt.Sprintf(":%d", localPort))
    if err != nil {
        return nil, err
    }
    
    return &Tunnel{
        localPort:   localPort,
        remotePort:  remotePort,
        routingKey:  routingKey,
        listener:    listener,
        connections: make(map[string]net.Conn),
        stopCh:      make(chan struct{}),
    }, nil
}

func (t *Tunnel) Start() error {
    defer t.listener.Close()
    
    // Accept connections
    go func() {
        for {
            conn, err := t.listener.Accept()
            if err != nil {
                select {
                case <-t.stopCh:
                    return
                default:
                    log.Printf("Failed to accept connection: %v", err)
                    continue
                }
            }
            
            // Handle connection
            connID := uuid.New().String()
            t.mu.Lock()
            t.connections[connID] = conn
            t.mu.Unlock()
            
            go t.handleConnection(connID, conn)
        }
    }()
    
    <-t.stopCh
    return nil
}

func (t *Tunnel) Stop() {
    close(t.stopCh)
    t.listener.Close()
    
    // Close all connections
    t.mu.Lock()
    defer t.mu.Unlock()
    
    for _, conn := range t.connections {
        conn.Close()
    }
}

func (t *Tunnel) handleConnection(id string, conn net.Conn) {
    defer func() {
        conn.Close()
        t.mu.Lock()
        delete(t.connections, id)
        t.mu.Unlock()
    }()
    
    // Connect to remote service
    remoteConn, err := net.Dial("tcp", fmt.Sprintf("tunnel-server:%d", t.remotePort))
    if err != nil {
        log.Printf("Failed to connect to remote service: %v", err)
        return
    }
    defer remoteConn.Close()
    
    // Add routing header
    // This is simplified - actual implementation would depend on protocol
    if httpConn, ok := remoteConn.(*http.Conn); ok {
        httpConn.Header.Set("X-Routing-Key", t.routingKey)
    }
    
    // Copy data bidirectionally
    errCh := make(chan error, 2)
    go func() {
        _, err := io.Copy(remoteConn, conn)
        errCh <- err
    }()
    
    go func() {
        _, err := io.Copy(conn, remoteConn)
        errCh <- err
    }()
    
    // Wait for either direction to finish
    <-errCh
}
```

### 3. Local Workload Controller

The server-side component that manages local workload connections.

```go
// LocalWorkloadReconciler reconciles a LocalWorkload object
type LocalWorkloadReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
}

// Reconcile handles LocalWorkload resources
func (r *LocalWorkloadReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("localworkload", req.NamespacedName)
    
    // Fetch the LocalWorkload resource
    var localWorkload sandboxv1alpha1.LocalWorkload
    if err := r.Get(ctx, req.NamespacedName, &localWorkload); err != nil {
        if errors.IsNotFound(err) {
            // Resource deleted - no requeue
            return ctrl.Result{}, nil
        }
        // Error reading - requeue
        return ctrl.Result{}, err
    }
    
    // Check if workload is being deleted
    if !localWorkload.ObjectMeta.DeletionTimestamp.IsZero() {
        // Handle cleanup
        return r.handleCleanup(ctx, &localWorkload)
    }
    
    // Handle initial setup
    if localWorkload.Status.Phase == "" {
        return r.handleInitialSetup(ctx, &localWorkload)
    }
    
    // Check for heartbeat timeout
    if localWorkload.Status.Phase == sandboxv1alpha1.LocalWorkloadConnected {
        lastHeartbeat := localWorkload.Status.Connection.LastHeartbeat.Time
        if time.Since(lastHeartbeat) > 30*time.Second {
            // Connection timed out
            localWorkload.Status.Phase = sandboxv1alpha1.LocalWorkloadDisconnected
            if err := r.Status().Update(ctx, &localWorkload); err != nil {
                return ctrl.Result{}, err
            }
            
            // Requeue to check again later
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
    }
    
    // Requeue to check again later
    return ctrl.Result{RequeueAfter: 15 * time.Second}, nil
}

// handleInitialSetup handles the initial setup of a LocalWorkload
func (r *LocalWorkloadReconciler) handleInitialSetup(ctx context.Context, workload *sandboxv1alpha1.LocalWorkload) (ctrl.Result, error) {
    // Get sandbox
    var sandbox sandboxv1alpha1.Sandbox
    if err := r.Get(ctx, types.NamespacedName{
        Name:      workload.Spec.SandboxRef.Name,
        Namespace: workload.Namespace,
    }, &sandbox); err != nil {
        if errors.IsNotFound(err) {
            // Sandbox not found
            workload.Status.Phase = sandboxv1alpha1.LocalWorkloadFailed
            workload.Status.Conditions = append(workload.Status.Conditions, metav1.Condition{
                Type:    "Ready",
                Status:  metav1.ConditionFalse,
                Reason:  "SandboxNotFound",
                Message: fmt.Sprintf("Sandbox %s not found", workload.Spec.SandboxRef.Name),
            })
            
            if err := r.Status().Update(ctx, workload); err != nil {
                return ctrl.Result{}, err
            }
            
            return ctrl.Result{}, nil
        }
        
        return ctrl.Result{}, err
    }
    
    // Generate routing key
    routingKey := generateRoutingKey(sandbox.Name, workload.Name)
    
    // Update status
    workload.Status.Phase = sandboxv1alpha1.LocalWorkloadConnected
    workload.Status.RoutingKey = routingKey
    workload.Status.Connection = sandboxv1alpha1.ConnectionStatus{
        ConnectedAt:   metav1.NewTime(time.Now()),
        LastHeartbeat: metav1.NewTime(time.Now()),
        TunnelID:      uuid.New().String(),
    }
    
    workload.Status.Conditions = append(workload.Status.Conditions, metav1.Condition{
        Type:    "Ready",
        Status:  metav1.ConditionTrue,
        Reason:  "LocalWorkloadReady",
        Message: "Local workload is connected",
    })
    
    if err := r.Status().Update(ctx, workload); err != nil {
        return ctrl.Result{}, err
    }
    
    // Create or update route group for the workload
    if err := r.setupRouting(ctx, workload, routingKey); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{RequeueAfter: 15 * time.Second}, nil
}

// handleCleanup handles the cleanup when a LocalWorkload is deleted
func (r *LocalWorkloadReconciler) handleCleanup(ctx context.Context, workload *sandboxv1alpha1.LocalWorkload) (ctrl.Result, error) {
    // Remove finalizer
    controllerutil.RemoveFinalizer(workload, "localworkload.sandbox.example.com")
    if err := r.Update(ctx, workload); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}

// setupRouting creates or updates a route group for the local workload
func (r *LocalWorkloadReconciler) setupRouting(ctx context.Context, workload *sandboxv1alpha1.LocalWorkload, routingKey string) error {
    // Create or update route group
    // Implementation depends on routing mechanism
    return nil
}
```

### 4. CLI Integration

The CLI provides commands for managing local development connections.

```go
func NewLocalConnectCommand() *cobra.Command {
    var sandbox string
    var service string
    var ports []string
    
    cmd := &cobra.Command{
        Use:   "connect",
        Short: "Connect local service to sandbox",
        RunE: func(cmd *cobra.Command, args []string) error {
            namespace, _ := cmd.Flags().GetString("namespace")
            
            // Parse port mappings
            var portMappings []api.PortMapping
            for _, portStr := range ports {
                parts := strings.Split(portStr, ":")
                if len(parts) != 2 {
                    return fmt.Errorf("invalid port mapping: %s (use local:remote format)", portStr)
                }
                
                localPort, err := strconv.Atoi(parts[0])
                if err != nil {
                    return fmt.Errorf("invalid local port: %s", parts[0])
                }
                
                remotePort, err := strconv.Atoi(parts[1])
                if err != nil {
                    return fmt.Errorf("invalid remote port: %s", parts[1])
                }
                
                portMappings = append(portMappings, api.PortMapping{
                    Name:        fmt.Sprintf("port-%d", localPort),
                    LocalPort:   localPort,
                    RemotePort:  remotePort,
                })
            }
            
            // Create config
            config := &agent.Config{
                ServerURL:    viper.GetString("server.url"),
                Token:        viper.GetString("token"),
                Namespace:    namespace,
                SandboxName:  sandbox,
                ServiceName:  service,
                LocalService: fmt.Sprintf("local-%s", service),
                Ports:        portMappings,
            }
            
            // Create and start agent
            agent, err := agent.NewLocalAgent(config)
            if err != nil {
                return err
            }
            
            fmt.Printf("Connecting local service %s to sandbox %s...\n", service, sandbox)
            if err := agent.Start(); err != nil {
                return err
            }
            
            fmt.Printf("Local service connected! Port mappings:\n")
            for _, port := range portMappings {
                fmt.Printf("  localhost:%d -> %s.%s:%d\n", port.LocalPort, service, sandbox, port.RemotePort)
            }
            
            fmt.Println("\nPress Ctrl+C to disconnect")
            
            // Wait for interrupt
            c := make(chan os.Signal, 1)
            signal.Notify(c, os.Interrupt, syscall.SIGTERM)
            <-c
            
            fmt.Println("\nDisconnecting...")
            agent.Stop()
            fmt.Println("Disconnected!")
            
            return nil
        },
    }
    
    cmd.Flags().StringVar(&sandbox, "sandbox", "", "Sandbox name to connect to")
    cmd.MarkFlagRequired("sandbox")
    cmd.Flags().StringVar(&service, "service", "", "Service name to override")
    cmd.MarkFlagRequired("service")
    cmd.Flags().StringSliceVar(&ports, "port", nil, "Ports to expose (local:remote)")
    cmd.MarkFlagRequired("port")
    
    return cmd
}
```

## Protocol Support

The local development integration should support various protocols:

### HTTP/HTTPS

```go
func handleHTTPRequest(w http.ResponseWriter, r *http.Request, routingKey string) {
    // Create proxied request
    proxyReq, err := http.NewRequest(r.Method, getTargetURL(r), r.Body)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Copy headers
    for name, values := range r.Header {
        for _, value := range values {
            proxyReq.Header.Add(name, value)
        }
    }
    
    // Add routing header
    proxyReq.Header.Set("X-Routing-Key", routingKey)
    
    // Send request
    resp, err := http.DefaultClient.Do(proxyReq)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadGateway)
        return
    }
    defer resp.Body.Close()
    
    // Copy response headers
    for name, values := range resp.Header {
        for _, value := range values {
            w.Header().Add(name, value)
        }
    }
    
    // Set status code
    w.WriteHeader(resp.StatusCode)
    
    // Copy response body
    io.Copy(w, resp.Body)
}
```

### gRPC

```go
func createGRPCInterceptor(routingKey string) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        // Add routing key to context metadata
        md, ok := metadata.FromOutgoingContext(ctx)
        if !ok {
            md = metadata.New(nil)
        }
        
        md.Set("x-routing-key", routingKey)
        newCtx := metadata.NewOutgoingContext(ctx, md)
        
        // Call the invoker with the new context
        return invoker(newCtx, method, req, reply, cc, opts...)
    }
}
```

### TCP/UDP (raw)

```go
func createTCPTunnel(localPort, remotePort int, routingKey string) (*Tunnel, error) {
    // Create listener
    listener, err := net.Listen("tcp", fmt.Sprintf(":%d", localPort))
    if err != nil {
        return nil, err
    }
    
    tunnel := &Tunnel{
        localPort:   localPort,
        remotePort:  remotePort,
        routingKey:  routingKey,
        listener:    listener,
        connections: make(map[string]net.Conn),
        stopCh:      make(chan struct{}),
    }
    
    return tunnel, nil
}
```

## Connection Management

The system should manage connections effectively:

### Heartbeat Mechanism

```go
func (a *LocalAgent) startHeartbeat() {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            if err := a.sendHeartbeat(); err != nil {
                log.Printf("Failed to send heartbeat: %v", err)
            }
        case <-a.stopCh:
            return
        }
    }
}

func (a *LocalAgent) sendHeartbeat() error {
    return a.client.UpdateLocalWorkloadStatus(
        a.config.Namespace,
        a.config.LocalService,
        &api.LocalWorkloadStatus{
            Phase: "Connected",
            Connection: api.ConnectionStatus{
                LastHeartbeat: metav1.Now(),
            },
        },
    )
}
```

### Reconnection Logic

```go
func (a *LocalAgent) monitorConnection() {
    for {
        select {
        case <-a.stopCh:
            return
        default:
            // Check connection
            if !a.isConnected() {
                log.Println("Connection lost, attempting to reconnect...")
                if err := a.reconnect(); err != nil {
                    log.Printf("Failed to reconnect: %v", err)
                    time.Sleep(5 * time.Second)
                    continue
                }
                log.Println("Reconnected successfully")
            }
            
            time.Sleep(5 * time.Second)
        }
    }
}

func (a *LocalAgent) reconnect() error {
    // Stop existing tunnels
    for _, tunnel := range a.tunnels {
        tunnel.Stop()
    }
    
    // Clear tunnels
    a.tunnels = make(map[string]*Tunnel)
    
    // Recreate connection
    return a.setupConnections()
}
```

## Security Considerations

### Secure Communication

```go
func createSecureTunnel(localPort, remotePort int, routingKey string, tlsConfig *tls.Config) (*Tunnel, error) {
    // Create TLS listener
    listener, err := tls.Listen("tcp", fmt.Sprintf(":%d", localPort), tlsConfig)
    if err != nil {
        return nil, err
    }
    
    tunnel := &Tunnel{
        localPort:   localPort,
        remotePort:  remotePort,
        routingKey:  routingKey,
        listener:    listener,
        connections: make(map[string]net.Conn),
        stopCh:      make(chan struct{}),
    }
    
    return tunnel, nil
}
```

### Authentication

```go
func (a *LocalAgent) authenticate() error {
    // Get token
    token, err := a.client.GetToken()
    if err != nil {
        return err
    }
    
    // Validate token
    if err := a.client.ValidateToken(token); err != nil {
        return err
    }
    
    // Store token for future requests
    a.token = token
    
    return nil
}
```

## Next Steps

1. Implement local agent
2. Create tunneling mechanism
3. Develop server-side controller
4. Implement CLI commands
5. Add support for different protocols
6. Test with real-world scenarios
7. Optimize performance and security
