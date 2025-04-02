# Resource Controllers

This document outlines the design and implementation of resource controllers for our Signadot/SLATE clone.

## Controller Architecture

Resource controllers follow the Kubernetes controller pattern, implementing a reconciliation loop that drives the actual state toward the desired state specified in resource definitions.

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│                 │      │                 │      │                 │
│  API Server     │─────▶│  Controllers    │─────▶│  Kubernetes     │
│                 │      │                 │      │  Resources      │
└─────────────────┘      └─────────────────┘      └─────────────────┘
        ▲                        │                        │
        │                        │                        │
        └────────────────────────┴────────────────────────┘
                          Watch/Notify
```

## Controller Components

### 1. Manager

The manager is responsible for:
- Setting up shared dependencies (client, scheme, etc.)
- Starting and managing controllers
- Managing webhooks
- Handling signals and graceful shutdown

```go
func main() {
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:             scheme,
        MetricsBindAddress: metricsAddr,
        LeaderElection:     enableLeaderElection,
        LeaderElectionID:   "controller-leader-election",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }
    
    // Add controllers to the manager
    if err := (&controllers.SandboxReconciler{
        Client: mgr.GetClient(),
        Log:    ctrl.Log.WithName("controllers").WithName("Sandbox"),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "Sandbox")
        os.Exit(1)
    }
    
    // Start manager
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

### 2. Reconciler

Each controller implements a Reconciler interface that handles reconciliation logic:

```go
type Reconciler interface {
    Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error)
}
```

The reconciliation loop follows these steps:
1. Fetch the resource
2. Check if resource is being deleted
3. Handle resource finalization if needed
4. Update resource status
5. Create or update required resources
6. Return reconciliation result

## Controller Implementation

### Sandbox Controller

The Sandbox Controller manages the lifecycle of sandbox environments.

```go
// SandboxReconciler reconciles a Sandbox object
type SandboxReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
}

// Reconcile reconciles a Sandbox resource
func (r *SandboxReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("sandbox", req.NamespacedName)
    
    // Fetch the Sandbox resource
    var sandbox sandboxv1alpha1.Sandbox
    if err := r.Get(ctx, req.NamespacedName, &sandbox); err != nil {
        if errors.IsNotFound(err) {
            // Resource deleted - no requeue
            return ctrl.Result{}, nil
        }
        // Error reading - requeue
        return ctrl.Result{}, err
    }
    
    // Check if sandbox is being deleted
    if !sandbox.ObjectMeta.DeletionTimestamp.IsZero() {
        // Handle finalization
        return r.handleFinalization(ctx, &sandbox)
    }
    
    // Add finalizer if not present
    if !containsString(sandbox.Finalizers, sandboxFinalizer) {
        sandbox.Finalizers = append(sandbox.Finalizers, sandboxFinalizer)
        if err := r.Update(ctx, &sandbox); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // Update sandbox status
    if sandbox.Status.Phase == "" {
        sandbox.Status.Phase = sandboxv1alpha1.SandboxPending
        if err := r.Status().Update(ctx, &sandbox); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // Process sandbox
    result, err := r.processSandbox(ctx, &sandbox)
    if err != nil {
        log.Error(err, "Failed to process sandbox")
        
        // Update status to reflect failure
        sandbox.Status.Phase = sandboxv1alpha1.SandboxFailed
        if updateErr := r.Status().Update(ctx, &sandbox); updateErr != nil {
            log.Error(updateErr, "Failed to update sandbox status")
        }
        
        return result, err
    }
    
    return result, nil
}

// processSandbox handles the main sandbox reconciliation logic
func (r *SandboxReconciler) processSandbox(ctx context.Context, sandbox *sandboxv1alpha1.Sandbox) (ctrl.Result, error) {
    // Implementation steps:
    // 1. Create or update service overrides
    // 2. Configure routing rules
    // 3. Set up resource limits
    // 4. Update sandbox status
    
    return ctrl.Result{}, nil
}

// handleFinalization handles sandbox cleanup when it's being deleted
func (r *SandboxReconciler) handleFinalization(ctx context.Context, sandbox *sandboxv1alpha1.Sandbox) (ctrl.Result, error) {
    // Implementation steps:
    // 1. Clean up resources created for the sandbox
    // 2. Remove finalizer to allow deletion
    
    return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager
func (r *SandboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&sandboxv1alpha1.Sandbox{}).
        Owns(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Complete(r)
}
```

### RouteGroup Controller

The RouteGroup Controller manages routing rules for sandboxes.

```go
// RouteGroupReconciler reconciles a RouteGroup object
type RouteGroupReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
}

// Reconcile reconciles a RouteGroup resource
func (r *RouteGroupReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("routegroup", req.NamespacedName)
    
    // Fetch the RouteGroup resource
    var routeGroup sandboxv1alpha1.RouteGroup
    if err := r.Get(ctx, req.NamespacedName, &routeGroup); err != nil {
        if errors.IsNotFound(err) {
            // Resource deleted - no requeue
            return ctrl.Result{}, nil
        }
        // Error reading - requeue
        return ctrl.Result{}, err
    }
    
    // Process route group
    // Implementation steps:
    // 1. Configure routing proxy
    // 2. Set up route table
    // 3. Update status
    
    return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager
func (r *RouteGroupReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&sandboxv1alpha1.RouteGroup{}).
        Complete(r)
}
```

### LocalWorkload Controller

The LocalWorkload Controller manages connections between local development environments and sandboxes.

```go
// LocalWorkloadReconciler reconciles a LocalWorkload object
type LocalWorkloadReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
}

// Reconcile reconciles a LocalWorkload resource
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
    
    // Process local workload
    // Implementation steps:
    // 1. Set up tunneling
    // 2. Configure routing
    // 3. Update connection status
    
    return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager
func (r *LocalWorkloadReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&sandboxv1alpha1.LocalWorkload{}).
        Complete(r)
}
```

## Controller Patterns

### Owner References

Controllers use owner references to create parent-child relationships between resources:

```go
// Set owner reference
if err := ctrl.SetControllerReference(sandbox, deployment, r.Scheme); err != nil {
    return err
}
```

This ensures that child resources are automatically deleted when the parent is deleted.

### Status Conditions

Controllers update resource status conditions to reflect the current state:

```go
// Update status condition
meta.SetStatusCondition(&sandbox.Status.Conditions, metav1.Condition{
    Type:               "Ready",
    Status:             metav1.ConditionTrue,
    Reason:             "SandboxReady",
    Message:            "Sandbox is ready",
    ObservedGeneration: sandbox.Generation,
})
```

### Finalizers

Controllers use finalizers to handle resource cleanup before deletion:

```go
// Add finalizer
if !containsString(sandbox.Finalizers, sandboxFinalizer) {
    sandbox.Finalizers = append(sandbox.Finalizers, sandboxFinalizer)
    if err := r.Update(ctx, &sandbox); err != nil {
        return err
    }
}

// Remove finalizer
sandbox.Finalizers = removeString(sandbox.Finalizers, sandboxFinalizer)
if err := r.Update(ctx, &sandbox); err != nil {
    return err
}
```

### Events

Controllers emit events to provide visibility into controller actions:

```go
// Record event
r.Recorder.Event(sandbox, corev1.EventTypeNormal, "Created", "Created sandbox resources")
```

## Controller Testing

Controllers should be thoroughly tested using the controller-runtime testing framework:

```go
func TestSandboxController(t *testing.T) {
    // Create a fake client
    fakeClient := fake.NewClientBuilder().WithScheme(scheme).WithObjects(
        // Test objects
    ).Build()
    
    // Create a reconciler
    reconciler := &SandboxReconciler{
        Client: fakeClient,
        Log:    log.Log,
        Scheme: scheme,
    }
    
    // Run reconciliation
    _, err := reconciler.Reconcile(context.Background(), ctrl.Request{
        NamespacedName: types.NamespacedName{
            Name:      "test-sandbox",
            Namespace: "default",
        },
    })
    
    // Assert expectations
    // ...
}
```

## Error Handling

Controllers should handle errors appropriately:

1. **Transient Errors**: Return `Result{Requeue: true}` or `Result{RequeueAfter: time.Second * 10}`
2. **Permanent Errors**: Log error and update status
3. **Resource Not Found**: Return `Result{}, nil` (no requeue)

```go
// Handle transient error
if isTransientError(err) {
    return ctrl.Result{RequeueAfter: time.Second * 10}, nil
}

// Handle permanent error
if isPermanentError(err) {
    log.Error(err, "Permanent error")
    // Update status
    return ctrl.Result{}, nil
}
```

## Resource Watching

Controllers watch resources they need to reconcile:

```go
func (r *SandboxReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&sandboxv1alpha1.Sandbox{}).
        Owns(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Watches(
            &source.Kind{Type: &sandboxv1alpha1.RouteGroup{}},
            handler.EnqueueRequestsFromMapFunc(r.findSandboxesForRouteGroup),
        ).
        Complete(r)
}
```

## Controller Metrics

Controllers should expose metrics for monitoring:

- `reconcile_total`: Total number of reconciliations
- `reconcile_errors_total`: Total number of reconciliation errors
- `reconcile_duration_seconds`: Reconciliation duration
- `work_queue_depth`: Work queue depth
- `work_queue_latency`: Work queue latency

## Next Steps

1. Implement controller manager
2. Create individual controllers for each resource
3. Implement reconciliation logic
4. Add tests for controllers
5. Set up metrics and monitoring
