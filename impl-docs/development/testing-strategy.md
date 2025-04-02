# Testing Strategy

This document outlines the testing strategy for our Signadot/SLATE clone, ensuring high quality and reliability of the system.

## Testing Objectives

1. **Functionality Verification**: Ensure all components work as expected
2. **Integration Validation**: Verify correct interaction between components
3. **Performance Assessment**: Evaluate system performance under various conditions
4. **Reliability Testing**: Ensure system reliability and stability
5. **Security Validation**: Verify security mechanisms and protections

## Test Types

### 1. Unit Tests

Unit tests verify the functionality of individual components in isolation.

#### Controllers

```go
func TestSandboxController(t *testing.T) {
    // Create a fake client
    client := fake.NewFakeClientWithScheme(scheme)
    
    // Create a reconciler with the fake client
    reconciler := &SandboxReconciler{
        Client: client,
        Log:    logr.Discard(),
        Scheme: scheme,
    }

#### Routing Proxy + Context Propagation

```go
func TestRoutingProxyIntegration(t *testing.T) {
    // Create routing table
    routingTable := NewRoutingTable()
    
    // Add rules
    routingTable.AddRule(&RoutingRule{
        RouteKey:      "test-key",
        TargetService: "test-service:8080",
        PathPrefixes:  []string{"/api"},
    })
    
    // Create test server (target service)
    targetServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Verify headers were propagated
        if r.Header.Get("X-Route-Key") != "test-key" {
            t.Errorf("Expected header X-Route-Key: %s, got %s", "test-key", r.Header.Get("X-Route-Key"))
        }
        
        // Return response
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"message":"success"}`))
    }))
    defer targetServer.Close()
    
    // Override target resolution to use test server
    oldGetTarget := getTargetURL
    defer func() { getTargetURL = oldGetTarget }()
    getTargetURL = func(req *http.Request) string {
        return targetServer.URL + req.URL.Path
    }
    
    // Create proxy server
    proxy := NewProxyServer(&Config{
        RoutingTable: routingTable,
    })
    
    // Create test server (proxy)
    proxyServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        proxy.httpHandler(w, r)
    }))
    defer proxyServer.Close()
    
    // Create test request
    req, err := http.NewRequest("GET", proxyServer.URL+"/api/test", nil)
    if err != nil {
        t.Fatalf("Failed to create request: %v", err)
    }
    req.Header.Set("X-Route-Key", "test-key")
    
    // Send request
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        t.Fatalf("Request failed: %v", err)
    }
    defer resp.Body.Close()
    
    // Verify response
    if resp.StatusCode != http.StatusOK {
        t.Errorf("Expected status %d, got %d", http.StatusOK, resp.StatusCode)
    }
    
    // Read response body
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        t.Fatalf("Failed to read response body: %v", err)
    }
    
    // Verify response body
    if string(body) != `{"message":"success"}` {
        t.Errorf("Expected body %s, got %s", `{"message":"success"}`, string(body))
    }
}
```

#### CLI + API Server

```go
func TestCLIAPIIntegration(t *testing.T) {
    // Start API server
    apiServer := startTestAPIServer()
    defer apiServer.Close()
    
    // Set API server URL in config
    viper.Reset()
    viper.Set("server.url", apiServer.URL)
    viper.Set("token", "test-token")
    
    // Create CLI command
    rootCmd := NewRootCommand()
    
    // Create output buffer
    out := &bytes.Buffer{}
    rootCmd.SetOut(out)
    
    // Execute command
    rootCmd.SetArgs([]string{"sandbox", "create", "test-sandbox", "--ttl", "3600"})
    if err := rootCmd.Execute(); err != nil {
        t.Fatalf("Command execution failed: %v", err)
    }
    
    // Verify output
    if !strings.Contains(out.String(), "Sandbox test-sandbox created successfully") {
        t.Errorf("Expected success message, got: %s", out.String())
    }
    
    // Test listing sandboxes
    out.Reset()
    rootCmd.SetArgs([]string{"sandbox", "list"})
    if err := rootCmd.Execute(); err != nil {
        t.Fatalf("Command execution failed: %v", err)
    }
    
    // Verify sandbox was listed
    if !strings.Contains(out.String(), "test-sandbox") {
        t.Errorf("Expected sandbox to be listed, got: %s", out.String())
    }
}

func startTestAPIServer() *httptest.Server {
    return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        switch {
        case r.Method == "POST" && strings.Contains(r.URL.Path, "/sandboxes"):
            // Handle create sandbox
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusCreated)
            
            // Parse request body
            var sandbox api.Sandbox
            json.NewDecoder(r.Body).Decode(&sandbox)
            
            // Add server fields
            sandbox.Status.Phase = "Pending"
            
            // Return response
            json.NewEncoder(w).Encode(sandbox)
            
        case r.Method == "GET" && strings.Contains(r.URL.Path, "/sandboxes") && !strings.Contains(r.URL.Path, "/status"):
            // Handle list sandboxes
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusOK)
            
            // Create sandbox list
            sandboxList := api.SandboxList{
                Items: []api.Sandbox{
                    {
                        Metadata: api.ObjectMeta{
                            Name:      "test-sandbox",
                            Namespace: "default",
                        },
                        Status: api.SandboxStatus{
                            Phase: "Running",
                        },
                    },
                },
            }
            
            // Return response
            json.NewEncoder(w).Encode(sandboxList)
            
        default:
            // Unknown endpoint
            w.WriteHeader(http.StatusNotFound)
        }
    }))
}
```

### 3. End-to-End Tests

End-to-end tests verify the complete system workflow.

```go
func TestSandboxWorkflow(t *testing.T) {
    // Skip if not running in integration test environment
    if os.Getenv("INTEGRATION_TEST") != "true" {
        t.Skip("Skipping integration test")
    }
    
    // Set up kubectl command
    kubectl := func(args ...string) (string, error) {
        cmd := exec.Command("kubectl", args...)
        out, err := cmd.CombinedOutput()
        return string(out), err
    }
    
    // Clean up any existing resources
    kubectl("delete", "sandbox", "e2e-test", "--ignore-not-found")
    
    // Create sandbox
    out, err := kubectl("apply", "-f", "testdata/e2e-sandbox.yaml")
    if err != nil {
        t.Fatalf("Failed to create sandbox: %v\nOutput: %s", err, out)
    }
    
    // Wait for sandbox to be ready
    err = waitForCondition(func() (bool, error) {
        out, err := kubectl("get", "sandbox", "e2e-test", "-o", "jsonpath={.status.phase}")
        if err != nil {
            return false, err
        }
        return out == "Running", nil
    }, 60*time.Second)
    if err != nil {
        t.Fatalf("Sandbox did not become ready: %v", err)
    }
    
    // Deploy test service
    out, err = kubectl("apply", "-f", "testdata/e2e-service.yaml")
    if err != nil {
        t.Fatalf("Failed to deploy service: %v\nOutput: %s", err, out)
    }
    
    // Wait for test service to be ready
    err = waitForCondition(func() (bool, error) {
        out, err := kubectl("get", "deployment", "e2e-service", "-o", "jsonpath={.status.readyReplicas}")
        if err != nil {
            return false, err
        }
        return out == "1", nil
    }, 60*time.Second)
    if err != nil {
        t.Fatalf("Service did not become ready: %v", err)
    }
    
    // Set up port forwarding
    stopCh := make(chan struct{})
    portForwardCmd := exec.Command("kubectl", "port-forward", "svc/e2e-service", "8080:80")
    portForwardCmd.Start()
    defer func() {
        close(stopCh)
        portForwardCmd.Process.Kill()
    }()
    
    // Wait for port forwarding to be established
    time.Sleep(2 * time.Second)
    
    // Send test request
    client := &http.Client{}
    req, err := http.NewRequest("GET", "http://localhost:8080/api/test", nil)
    if err != nil {
        t.Fatalf("Failed to create request: %v", err)
    }
    req.Header.Set("X-Route-Key", "e2e-test")
    
    // Wait for service to be accessible
    err = waitForCondition(func() (bool, error) {
        resp, err := client.Do(req)
        if err != nil {
            return false, nil // Not ready yet
        }
        defer resp.Body.Close()
        return resp.StatusCode == http.StatusOK, nil
    }, 30*time.Second)
    if err != nil {
        t.Fatalf("Service not accessible: %v", err)
    }
    
    // Test successful request
    resp, err := client.Do(req)
    if err != nil {
        t.Fatalf("Request failed: %v", err)
    }
    defer resp.Body.Close()
    
    // Verify response
    if resp.StatusCode != http.StatusOK {
        t.Errorf("Expected status %d, got %d", http.StatusOK, resp.StatusCode)
    }
    
    // Clean up
    kubectl("delete", "sandbox", "e2e-test")
}

func waitForCondition(condition func() (bool, error), timeout time.Duration) error {
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        done, err := condition()
        if err != nil {
            return err
        }
        if done {
            return nil
        }
        time.Sleep(1 * time.Second)
    }
    return fmt.Errorf("timed out waiting for condition")
}
```

### 4. Performance Tests

Performance tests evaluate the system's performance under various conditions.

```go
func TestRoutingProxyPerformance(t *testing.T) {
    // Skip if not running performance tests
    if os.Getenv("PERFORMANCE_TEST") != "true" {
        t.Skip("Skipping performance test")
    }
    
    // Create routing table with many rules
    routingTable := NewRoutingTable()
    for i := 0; i < 1000; i++ {
        routingTable.AddRule(&RoutingRule{
            RouteKey:      fmt.Sprintf("key-%d", i),
            TargetService: fmt.Sprintf("service-%d:8080", i),
            PathPrefixes:  []string{fmt.Sprintf("/api/%d", i)},
        })
    }
    
    // Create test server
    targetServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"message":"success"}`))
    }))
    defer targetServer.Close()
    
    // Override target resolution
    oldGetTarget := getTargetURL
    defer func() { getTargetURL = oldGetTarget }()
    getTargetURL = func(req *http.Request) string {
        return targetServer.URL
    }
    
    // Create proxy server
    proxy := NewProxyServer(&Config{
        RoutingTable: routingTable,
    })
    
    // Create test server (proxy)
    proxyServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        proxy.httpHandler(w, r)
    }))
    defer proxyServer.Close()
    
    // Benchmark proxy performance
    b.Run("RoutingTableLookup", func(b *testing.B) {
        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            // Create test request
            req, _ := http.NewRequest("GET", proxyServer.URL+"/api/500", nil)
            req.Header.Set("X-Route-Key", "key-500")
            
            // Send request
            client := &http.Client{}
            resp, err := client.Do(req)
            if err != nil {
                b.Fatalf("Request failed: %v", err)
            }
            resp.Body.Close()
        }
    })
    
    // Test connection concurrency
    concurrencyTest := func(concurrency int) time.Duration {
        start := time.Now()
        
        // Create wait group
        wg := sync.WaitGroup{}
        wg.Add(concurrency)
        
        // Send concurrent requests
        for i := 0; i < concurrency; i++ {
            go func(i int) {
                defer wg.Done()
                
                // Create test request
                req, _ := http.NewRequest("GET", proxyServer.URL+fmt.Sprintf("/api/%d", i%1000), nil)
                req.Header.Set("X-Route-Key", fmt.Sprintf("key-%d", i%1000))
                
                // Send request
                client := &http.Client{}
                resp, err := client.Do(req)
                if err != nil {
                    t.Errorf("Request failed: %v", err)
                    return
                }
                resp.Body.Close()
            }(i)
        }
        
        // Wait for all requests to complete
        wg.Wait()
        
        return time.Since(start)
    }
    
    // Test different concurrency levels
    t.Log("Concurrency 10:", concurrencyTest(10))
    t.Log("Concurrency 100:", concurrencyTest(100))
    t.Log("Concurrency 1000:", concurrencyTest(1000))
}
```

### 5. Load Tests

Load tests evaluate the system's behavior under high load.

```go
func TestSystemLoad(t *testing.T) {
    // Skip if not running load tests
    if os.Getenv("LOAD_TEST") != "true" {
        t.Skip("Skipping load test")
    }
    
    // Connect to system under test
    baseURL := os.Getenv("LOAD_TEST_URL")
    if baseURL == "" {
        baseURL = "http://localhost:8080"
    }
    
    // Parameters
    duration := 5 * time.Minute
    rampUpTime := 30 * time.Second
    maxConcurrency := 100
    
    // Create client
    client := &http.Client{
        Timeout: 10 * time.Second,
    }
    
    // Create results channel
    results := make(chan *RequestResult, maxConcurrency)
    
    // Start test
    start := time.Now()
    end := start.Add(duration)
    
    // Start worker goroutines
    for i := 0; i < maxConcurrency; i++ {
        go func(id int) {
            // Calculate start time for this worker (for ramp-up)
            workerStart := start.Add(time.Duration(float64(rampUpTime) * float64(id) / float64(maxConcurrency)))
            
            // Sleep until start time
            time.Sleep(time.Until(workerStart))
            
            // Run until end time
            for time.Now().Before(end) {
                // Send request
                reqStart := time.Now()
                req, _ := http.NewRequest("GET", baseURL+"/api/test", nil)
                req.Header.Set("X-Route-Key", fmt.Sprintf("load-test-%d", id%10))
                
                resp, err := client.Do(req)
                result := &RequestResult{
                    Time:     reqStart,
                    Duration: time.Since(reqStart),
                    Error:    err != nil,
                }
                
                if err == nil {
                    result.StatusCode = resp.StatusCode
                    resp.Body.Close()
                }
                
                // Send result
                results <- result
                
                // Throttle rate
                time.Sleep(100 * time.Millisecond)
            }
        }(i)
    }
    
    // Collect results
    go func() {
        for result := range results {
            // Process result
            if result.Error {
                atomic.AddInt64(&stats.Errors, 1)
            } else {
                atomic.AddInt64(&stats.Requests, 1)
                atomic.AddInt64(&stats.TotalDuration, int64(result.Duration))
                
                // Track status codes
                if result.StatusCode >= 500 {
                    atomic.AddInt64(&stats.ServerErrors, 1)
                } else if result.StatusCode >= 400 {
                    atomic.AddInt64(&stats.ClientErrors, 1)
                } else {
                    atomic.AddInt64(&stats.Successes, 1)
                }
                
                // Track latency buckets
                if result.Duration < 100*time.Millisecond {
                    atomic.AddInt64(&stats.Latency100ms, 1)
                } else if result.Duration < 500*time.Millisecond {
                    atomic.AddInt64(&stats.Latency500ms, 1)
                } else if result.Duration < 1*time.Second {
                    atomic.AddInt64(&stats.Latency1s, 1)
                } else {
                    atomic.AddInt64(&stats.LatencySlowHardFWE1s, 1)
                }
            }
        }
    }()
    
    // Wait for test to complete
    time.Sleep(duration)
    
    // Print results
    t.Logf("Load Test Results:")
    t.Logf("  Total Requests: %d", stats.Requests)
    t.Logf("  Errors: %d (%.2f%%)", stats.Errors, float64(stats.Errors)/float64(stats.Requests+stats.Errors)*100)
    t.Logf("  Client Errors: %d (%.2f%%)", stats.ClientErrors, float64(stats.ClientErrors)/float64(stats.Requests)*100)
    t.Logf("  Server Errors: %d (%.2f%%)", stats.ServerErrors, float64(stats.ServerErrors)/float64(stats.Requests)*100)
    t.Logf("  Average Latency: %.2fms", float64(stats.TotalDuration)/float64(stats.Requests)/float64(time.Millisecond))
    t.Logf("  Latency Distribution:")
    t.Logf("    <100ms: %d (%.2f%%)", stats.Latency100ms, float64(stats.Latency100ms)/float64(stats.Requests)*100)
    t.Logf("    100-500ms: %d (%.2f%%)", stats.Latency500ms, float64(stats.Latency500ms)/float64(stats.Requests)*100)
    t.Logf("    500ms-1s: %d (%.2f%%)", stats.Latency1s, float64(stats.Latency1s)/float64(stats.Requests)*100)
    t.Logf("    >1s: %d (%.2f%%)", stats.LatencySlowHardFWE1s, float64(stats.LatencySlowHardFWE1s)/float64(stats.Requests)*100)
    
    // Verify test passed criteria
    if float64(stats.Errors)/float64(stats.Requests+stats.Errors) > 0.01 {
        t.Errorf("Error rate %.2f%% exceeds threshold of 1%%", float64(stats.Errors)/float64(stats.Requests+stats.Errors)*100)
    }
    
    if float64(stats.TotalDuration)/float64(stats.Requests)/float64(time.Millisecond) > 500 {
        t.Errorf("Average latency %.2fms exceeds threshold of 500ms", float64(stats.TotalDuration)/float64(stats.Requests)/float64(time.Millisecond))
    }
}

type RequestResult struct {
    Time       time.Time
    Duration   time.Duration
    StatusCode int
    Error      bool
}

type LoadTestStats struct {
    Requests          int64
    Errors            int64
    ClientErrors      int64
    ServerErrors      int64
    Successes         int64
    TotalDuration     int64
    Latency100ms      int64
    Latency500ms      int64
    Latency1s         int64
    LatencySlowHardFWE1s    int64
}
```

## Test Infrastructure

### Continuous Integration

Set up CI pipeline to run tests automatically:

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.18
    - name: Install dependencies
      run: go mod download
    - name: Run unit tests
      run: go test -v ./pkg/... ./cmd/...
      
  integration-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.18
    - name: Install dependencies
      run: go mod download
    - name: Set up test environment
      run: ./scripts/setup-test-env.sh
    - name: Run integration tests
      run: go test -tags=integration -v ./test/integration/...
      
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.18
    - name: Set up KinD
      uses: engineerd/setup-kind@v0.5.0
    - name: Deploy test environment
      run: ./scripts/deploy-test-env.sh
    - name: Run E2E tests
      run: INTEGRATION_TEST=true go test -v ./test/e2e/...
```

### Test Coverage

Track test coverage and enforce minimum coverage:

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html
go tool cover -func=coverage.out
```

Set up coverage reporting in CI:

```yaml
- name: Run tests with coverage
  run: go test -race -coverprofile=coverage.out -covermode=atomic ./...
- name: Upload coverage report
  uses: codecov/codecov-action@v1
  with:
    file: ./coverage.out
    fail_ci_if_error: true
```

### Test Fixtures

Create test fixtures for reproducible tests:

```go
func setupTestFixtures() (*client.Client, *sandboxv1alpha1.Sandbox, func()) {
    // Create fake client
    client := fake.NewFakeClientWithScheme(scheme)
    
    // Create test sandbox
    sandbox := &sandboxv1alpha1.Sandbox{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-sandbox",
            Namespace: "default",
        },
        Spec: sandboxv1alpha1.SandboxSpec{
            TTL: 3600,
            Services: []sandboxv1alpha1.ServiceOverride{
                {
                    Name: "test-service",
                    Containers: []sandboxv1alpha1.ContainerOverride{
                        {
                            Name:  "main",
                            Image: "test-image:latest",
                        },
                    },
                },
            },
        },
    }
    
    // Create sandbox in fake client
    err := client.Create(context.TODO(), sandbox)
    if err != nil {
        panic(fmt.Sprintf("Failed to create test sandbox: %v", err))
    }
    
    // Return cleanup function
    cleanup := func() {
        // Additional cleanup if needed
    }
    
    return client, sandbox, cleanup
}
```

### Test Helpers

Create helper functions for common test operations:

```go
// Wait for condition with timeout
func waitFor(condition func() (bool, error), timeout time.Duration) error {
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        satisfied, err := condition()
        if err != nil {
            return err
        }
        if satisfied {
            return nil
        }
        time.Sleep(100 * time.Millisecond)
    }
    return fmt.Errorf("timed out waiting for condition")
}

// Create temp config file
func createTempConfig(content string) (string, func()) {
    tempFile, err := ioutil.TempFile("", "test-config-*.yaml")
    if err != nil {
        panic(fmt.Sprintf("Failed to create temp file: %v", err))
    }
    
    _, err = tempFile.Write([]byte(content))
    if err != nil {
        os.Remove(tempFile.Name())
        panic(fmt.Sprintf("Failed to write to temp file: %v", err))
    }
    
    err = tempFile.Close()
    if err != nil {
        os.Remove(tempFile.Name())
        panic(fmt.Sprintf("Failed to close temp file: %v", err))
    }
    
    return tempFile.Name(), func() {
        os.Remove(tempFile.Name())
    }
}

// Capture output
func captureOutput(f func()) string {
    old := os.Stdout
    r, w, _ := os.Pipe()
    os.Stdout = w
    
    f()
    
    w.Close()
    os.Stdout = old
    
    var buf bytes.Buffer
    io.Copy(&buf, r)
    return buf.String()
}
```

## Test Organization

Organize tests by type and component:

```
/test
  /unit          # Unit tests
    /controllers # Controller unit tests
    /proxy       # Proxy unit tests
    /cli         # CLI unit tests
  /integration   # Integration tests
    /api         # API integration tests
    /controller  # Controller integration tests
    /proxy       # Proxy integration tests
  /e2e           # End-to-end tests
  /performance   # Performance tests
  /load          # Load tests
  /fixtures      # Test fixtures
  /utils         # Test utilities
```

## Test Documentation

Document test scenarios:

```go
// TestSandboxLifecycle tests the complete lifecycle of a sandbox
// 1. Create sandbox
// 2. Verify sandbox status becomes Running
// 3. Verify resources created by sandbox controller
// 4. Create service override
// 5. Verify service override applied
// 6. Test routing to service
// 7. Delete sandbox
// 8. Verify resources cleaned up
func TestSandboxLifecycle(t *testing.T) {
    
    // Create a test sandbox
    sandbox := &sandboxv1alpha1.Sandbox{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-sandbox",
            Namespace: "default",
        },
        Spec: sandboxv1alpha1.SandboxSpec{
            TTL: 3600,
            Services: []sandboxv1alpha1.ServiceOverride{
                {
                    Name: "test-service",
                    Containers: []sandboxv1alpha1.ContainerOverride{
                        {
                            Name:  "main",
                            Image: "test-image:latest",
                        },
                    },
                },
            },
        },
    }
    
    // Create the sandbox in the fake client
    err := client.Create(context.TODO(), sandbox)
    if err != nil {
        t.Fatalf("Failed to create test sandbox: %v", err)
    }
    
    // Reconcile the sandbox
    request := reconcile.Request{
        NamespacedName: types.NamespacedName{
            Name:      "test-sandbox",
            Namespace: "default",
        },
    }
    
    _, err = reconciler.Reconcile(context.TODO(), request)
    if err != nil {
        t.Fatalf("Reconciliation failed: %v", err)
    }
    
    // Verify the result
    updatedSandbox := &sandboxv1alpha1.Sandbox{}
    err = client.Get(context.TODO(), request.NamespacedName, updatedSandbox)
    if err != nil {
        t.Fatalf("Failed to get updated sandbox: %v", err)
    }
    
    // Check status
    if updatedSandbox.Status.Phase != sandboxv1alpha1.SandboxRunning {
        t.Errorf("Expected phase %s, got %s", sandboxv1alpha1.SandboxRunning, updatedSandbox.Status.Phase)
    }
    
    // Check if deployment was created
    deployment := &appsv1.Deployment{}
    err = client.Get(context.TODO(), types.NamespacedName{
        Name:      "test-service",
        Namespace: "default",
    }, deployment)
    if err != nil {
        t.Fatalf("Failed to get deployment: %v", err)
    }
    
    // Verify deployment
    if deployment.Spec.Template.Spec.Containers[0].Image != "test-image:latest" {
        t.Errorf("Expected image %s, got %s", "test-image:latest", deployment.Spec.Template.Spec.Containers[0].Image)
    }
}
```

#### Routing Proxy

```go
func TestRoutingTable(t *testing.T) {
    // Create a routing table
    routingTable := NewRoutingTable()
    
    // Add a rule
    rule := &RoutingRule{
        RouteKey:      "test-key",
        TargetService: "test-service",
        PathPrefixes:  []string{"/api"},
        HeaderMatchers: map[string]string{
            "Content-Type": "application/json",
        },
    }
    
    routingTable.AddRule(rule)
    
    // Create a test request
    req, err := http.NewRequest("GET", "http://example.com/api/test", nil)
    if err != nil {
        t.Fatalf("Failed to create request: %v", err)
    }
    
    req.Header.Set("X-Route-Key", "test-key")
    req.Header.Set("Content-Type", "application/json")
    
    // Get target for request
    target, headers := routingTable.GetTargetForRequest(req)
    
    // Verify target
    if target != "test-service" {
        t.Errorf("Expected target %s, got %s", "test-service", target)
    }
    
    // Try with non-matching path
    req, _ = http.NewRequest("GET", "http://example.com/other/path", nil)
    req.Header.Set("X-Route-Key", "test-key")
    req.Header.Set("Content-Type", "application/json")
    
    target, _ = routingTable.GetTargetForRequest(req)
    
    // Should not match
    if target == "test-service" {
        t.Errorf("Should not match non-matching path")
    }
}
```

#### CLI Commands

```go
func TestSandboxCreateCommand(t *testing.T) {
    // Set up test server
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Verify request
        if r.URL.Path != "/api/v1alpha1/namespaces/default/sandboxes" {
            t.Errorf("Expected path %s, got %s", "/api/v1alpha1/namespaces/default/sandboxes", r.URL.Path)
        }
        
        if r.Method != "POST" {
            t.Errorf("Expected method %s, got %s", "POST", r.Method)
        }
        
        // Read request body
        body, err := ioutil.ReadAll(r.Body)
        if err != nil {
            t.Fatalf("Failed to read request body: %v", err)
        }
        
        // Parse request
        var sandbox api.Sandbox
        if err := json.Unmarshal(body, &sandbox); err != nil {
            t.Fatalf("Failed to parse request body: %v", err)
        }
        
        // Verify sandbox
        if sandbox.Metadata.Name != "test-sandbox" {
            t.Errorf("Expected name %s, got %s", "test-sandbox", sandbox.Metadata.Name)
        }
        
        if sandbox.Spec.TTL != 3600 {
            t.Errorf("Expected TTL %d, got %d", 3600, sandbox.Spec.TTL)
        }
        
        // Return response
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        
        // Add fields that would be added by the server
        sandbox.Status.Phase = "Pending"
        
        response, _ := json.Marshal(sandbox)
        w.Write(response)
    }))
    defer server.Close()
    
    // Create command
    rootCmd := NewRootCommand()
    
    // Override client to use test server
    oldNewClient := newClient
    defer func() { newClient = oldNewClient }()
    newClient = func(baseURL, token string) client.Interface {
        return client.NewClient(server.URL, "test-token")
    }
    
    // Create buffer to capture output
    out := &bytes.Buffer{}
    rootCmd.SetOut(out)
    
    // Set args
    rootCmd.SetArgs([]string{"sandbox", "create", "test-sandbox", "--ttl", "3600"})
    
    // Execute command
    if err := rootCmd.Execute(); err != nil {
        t.Fatalf("Command execution failed: %v", err)
    }
    
    // Verify output
    if !strings.Contains(out.String(), "Sandbox test-sandbox created successfully") {
        t.Errorf("Expected success message, got: %s", out.String())
    }
}
```

### 2. Integration Tests

Integration tests verify the interaction between components.

#### API Server + Controller

```go
func TestAPIServerControllerIntegration(t *testing.T) {
    // Set up test environment
    env := &envtest.Environment{
        CRDDirectoryPaths: []string{"../../config/crd/bases"},
    }
    
    // Start the test environment
    cfg, err := env.Start()
    if err != nil {
        t.Fatalf("Failed to start test environment: %v", err)
    }
    defer env.Stop()
    
    // Create a client
    client, err := client.New(cfg, client.Options{Scheme: scheme})
    if err != nil {
        t.Fatalf("Failed to create client: %v", err)
    }
    
    // Start controllers
    mgr, err := ctrl.NewManager(cfg, ctrl.Options{
        Scheme: scheme,
    })
    if err != nil {
        t.Fatalf("Failed to create manager: %v", err)
    }
    
    // Add controllers to manager
    if err := (&SandboxReconciler{
        Client: mgr.GetClient(),
        Log:    logr.Discard(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        t.Fatalf("Failed to set up controller: %v", err)
    }
    
    // Start manager
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    go func() {
        if err := mgr.Start(ctx); err != nil {
            t.Errorf("Failed to start manager: %v", err)
        }
    }()
    
    // Create a test sandbox
    sandbox := &sandboxv1alpha1.Sandbox{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "integration-test",
            Namespace: "default",
        },
        Spec: sandboxv1alpha1.SandboxSpec{
            TTL: 3600,
            Services: []sandboxv1alpha1.ServiceOverride{
                {
                    Name: "test-service",
                    Containers: []sandboxv1alpha1.ContainerOverride{
                        {
                            Name:  "main",
                            Image: "test-image:latest",
                        },
                    },
                },
            },
        },
    }
    
    // Create the sandbox
    err = client.Create(ctx, sandbox)
    if err != nil {
        t.Fatalf("Failed to create sandbox: %v", err)
    }
    
    // Wait for sandbox to be reconciled
    time.Sleep(2 * time.Second)
    
    // Get the updated sandbox
    err = client.Get(ctx, types.NamespacedName{
        Name:      "integration-test",
        Namespace: "default",
    }, sandbox)
    if err != nil {
        t.Fatalf("Failed to get sandbox: %v", err)
    }
    
    // Verify status
    if sandbox.Status.Phase != sandboxv1alpha1.SandboxRunning {
        t.Errorf("Expected phase %s, got %s", sandboxv1alpha1.SandboxRunning, sandbox.Status.Phase)
    }
    
    // Verify resources were created
    deployment := &appsv1.Deployment{}
    err = client.Get(ctx, types.NamespacedName{
        Name:      "test-service",
        Namespace: "default",
    }, deployment)
    if err != nil {
        t.Fatalf("Failed to get deployment: %v", err)
    }
}