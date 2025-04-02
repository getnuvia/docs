# CLI Design

This document outlines the design and implementation of the Command-Line Interface (CLI) for our Signadot/SLATE clone.

## Overview

The CLI is the primary interface for developers to interact with the system. It provides commands for creating and managing sandboxes, configuring routing, and setting up local development environments.

```
┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐
│                   │     │                   │     │                   │
│  Developer        │────►│  CLI Tool         │────►│  API Server       │
│                   │     │                   │     │                   │
└───────────────────┘     └───────────────────┘     └───────────────────┘
```

## User Experience Goals

1. **Simplicity**: Easy to learn and use
2. **Consistency**: Consistent command patterns and naming
3. **Discoverability**: Help and documentation easily accessible
4. **Efficiency**: Quick to accomplish common tasks
5. **Feedback**: Clear feedback on operations

## Command Structure

The CLI follows a hierarchical command structure using `cobra` for command management:

```
sandboxctl
├── sandbox
│   ├── create
│   ├── delete
│   ├── list
│   ├── get
│   └── update
├── route
│   ├── create
│   ├── delete
│   ├── list
│   └── update
├── local
│   ├── connect
│   ├── disconnect
│   └── status
├── config
│   ├── set
│   ├── get
│   └── view
└── completion
```

## Implementation

### Command Registration

Commands are registered in a hierarchical structure:

```go
package main

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

func main() {
	rootCmd := NewRootCommand()
	if err := rootCmd.Execute(); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}

func NewRootCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "sandboxctl",
		Short: "Manage sandbox environments for microservice testing",
		Long: `
Sandboxctl is a command line tool for creating and managing sandbox 
environments for testing microservices against production dependencies.
`,
	}

	// Add global flags
	cmd.PersistentFlags().String("kubeconfig", "", "Path to the kubeconfig file")
	cmd.PersistentFlags().String("namespace", "default", "Namespace to use")
	
	// Add command groups
	cmd.AddCommand(NewSandboxCommand())
	cmd.AddCommand(NewRouteCommand())
	cmd.AddCommand(NewLocalCommand())
	cmd.AddCommand(NewConfigCommand())
	cmd.AddCommand(NewCompletionCommand())

	return cmd
}
```

### Sandbox Commands

```go
func NewSandboxCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:     "sandbox",
		Aliases: []string{"sb"},
		Short:   "Manage sandbox environments",
	}
	
	cmd.AddCommand(NewSandboxCreateCommand())
	cmd.AddCommand(NewSandboxDeleteCommand())
	cmd.AddCommand(NewSandboxListCommand())
	cmd.AddCommand(NewSandboxGetCommand())
	cmd.AddCommand(NewSandboxUpdateCommand())
	
	return cmd
}

func NewSandboxCreateCommand() *cobra.Command {
	var ttl int64
	var services []string
	var fromFile string
	
	cmd := &cobra.Command{
		Use:   "create [name]",
		Short: "Create a new sandbox environment",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			name := args[0]
			namespace, _ := cmd.Flags().GetString("namespace")
			
			if fromFile != "" {
				return createSandboxFromFile(name, namespace, fromFile)
			}
			
			return createSandbox(name, namespace, ttl, services)
		},
	}
	
	cmd.Flags().Int64Var(&ttl, "ttl", 86400, "Time-to-live in seconds")
	cmd.Flags().StringSliceVar(&services, "service", nil, "Services to include in sandbox (name=image)")
	cmd.Flags().StringVar(&fromFile, "from-file", "", "Create sandbox from YAML file")
	
	return cmd
}
```

### Local Development Commands

```go
func NewLocalCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "local",
		Short: "Local development commands",
	}
	
	cmd.AddCommand(NewLocalConnectCommand())
	cmd.AddCommand(NewLocalDisconnectCommand())
	cmd.AddCommand(NewLocalStatusCommand())
	
	return cmd
}

func NewLocalConnectCommand() *cobra.Command {
	var sandbox string
	var service string
	var ports []string
	
	cmd := &cobra.Command{
		Use:   "connect",
		Short: "Connect local service to sandbox",
		RunE: func(cmd *cobra.Command, args []string) error {
			namespace, _ := cmd.Flags().GetString("namespace")
			return connectLocalService(namespace, sandbox, service, ports)
		},
	}
	
	cmd.Flags().StringVar(&sandbox, "sandbox", "", "Sandbox name to connect to")
	cmd.MarkFlagRequired("sandbox")
	cmd.Flags().StringVar(&service, "service", "", "Service name to override")
	cmd.MarkFlagRequired("service")
	cmd.Flags().StringSliceVar(&ports, "port", nil, "Ports to expose (local:remote)")
	
	return cmd
}
```

## Client Implementation

The CLI uses a client to interact with the API server:

```go
type Client struct {
	httpClient *http.Client
	baseURL    string
	token      string
}

func NewClient(baseURL, token string) *Client {
	return &Client{
		httpClient: &http.Client{
			Timeout: 30 * time.Second,
		},
		baseURL: baseURL,
		token:   token,
	}
}

func (c *Client) CreateSandbox(namespace, name string, spec *api.SandboxSpec) (*api.Sandbox, error) {
	sandbox := &api.Sandbox{
		Metadata: api.ObjectMeta{
			Name:      name,
			Namespace: namespace,
		},
		Spec: *spec,
	}
	
	data, err := json.Marshal(sandbox)
	if err != nil {
		return nil, err
	}
	
	url := fmt.Sprintf("%s/api/v1alpha1/namespaces/%s/sandboxes", c.baseURL, namespace)
	req, err := http.NewRequest("POST", url, bytes.NewBuffer(data))
	if err != nil {
		return nil, err
	}
	
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", "Bearer "+c.token)
	
	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	
	if resp.StatusCode != http.StatusCreated {
		body, _ := ioutil.ReadAll(resp.Body)
		return nil, fmt.Errorf("failed to create sandbox: %s", body)
	}
	
	var result api.Sandbox
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}
	
	return &result, nil
}
```

## Output Formatting

The CLI supports multiple output formats:

```go
type OutputFormat string

const (
	OutputFormatTable OutputFormat = "table"
	OutputFormatJSON  OutputFormat = "json"
	OutputFormatYAML  OutputFormat = "yaml"
)

func printOutput(cmd *cobra.Command, data interface{}) error {
	format, _ := cmd.Flags().GetString("output")
	
	switch OutputFormat(format) {
	case OutputFormatJSON:
		return printJSON(data)
	case OutputFormatYAML:
		return printYAML(data)
	case OutputFormatTable:
		return printTable(cmd, data)
	default:
		return fmt.Errorf("unknown output format: %s", format)
	}
}

func printTable(cmd *cobra.Command, data interface{}) error {
	switch v := data.(type) {
	case *api.SandboxList:
		printSandboxList(v)
	case *api.Sandbox:
		printSandbox(v)
	// Other cases
	default:
		return fmt.Errorf("unsupported data type for table output")
	}
	
	return nil
}

func printSandboxList(list *api.SandboxList) {
	w := tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', 0)
	fmt.Fprintln(w, "NAME\tSTATUS\tAGE\tSERVICES")
	
	for _, sb := range list.Items {
		age := time.Since(sb.Metadata.CreationTimestamp.Time).Round(time.Second)
		services := len(sb.Spec.Services)
		
		fmt.Fprintf(w, "%s\t%s\t%s\t%d\n", 
			sb.Metadata.Name, 
			sb.Status.Phase, 
			age, 
			services,
		)
	}
	
	w.Flush()
}
```

## Configuration Management

The CLI uses `viper` for configuration management:

```go
func initConfig() {
	// Set config file name
	viper.SetConfigName("config")
	
	// Add config file locations
	viper.AddConfigPath("$HOME/.sandboxctl")
	viper.AddConfigPath(".")
	
	// Read config
	if err := viper.ReadInConfig(); err != nil {
		// Config file not found, using defaults
	}
	
	// Set defaults
	viper.SetDefault("server.url", "http://localhost:8080")
	viper.SetDefault("namespace", "default")
	
	// Bind environment variables
	viper.SetEnvPrefix("SANDBOXCTL")
	viper.AutomaticEnv()
}

func getClient() *client.Client {
	token := viper.GetString("token")
	url := viper.GetString("server.url")
	
	return client.NewClient(url, token)
}
```

## Progress Reporting

Long-running operations should provide progress updates:

```go
func createSandbox(name, namespace string, ttl int64, services []string) error {
	fmt.Printf("Creating sandbox %s in namespace %s...\n", name, namespace)
	
	// Create spinner
	spinner := spinner.New(spinner.CharSets[9], 100*time.Millisecond)
	spinner.Suffix = " Creating sandbox resources..."
	spinner.Start()
	
	// Create sandbox
	client := getClient()
	spec := createSpec(ttl, services)
	sandbox, err := client.CreateSandbox(namespace, name, spec)
	
	spinner.Stop()
	
	if err != nil {
		return fmt.Errorf("failed to create sandbox: %v", err)
	}
	
	fmt.Printf("Sandbox %s created successfully!\n", name)
	printSandbox(sandbox)
	
	return nil
}
```

## Command Completion

The CLI provides shell completion support:

```go
func NewCompletionCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "completion [bash|zsh|fish|powershell]",
		Short: "Generate completion script",
		Args:  cobra.ExactValidArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			switch args[0] {
			case "bash":
				return cmd.Root().GenBashCompletion(os.Stdout)
			case "zsh":
				return cmd.Root().GenZshCompletion(os.Stdout)
			case "fish":
				return cmd.Root().GenFishCompletion(os.Stdout, true)
			case "powershell":
				return cmd.Root().GenPowerShellCompletion(os.Stdout)
			default:
				return fmt.Errorf("unsupported shell: %s", args[0])
			}
		},
	}
	
	return cmd
}
```

## Error Handling

The CLI provides clear error messages:

```go
func handleError(err error) {
	if err == nil {
		return
	}
	
	// Check for specific error types
	switch e := err.(type) {
	case *client.APIError:
		fmt.Fprintf(os.Stderr, "API Error: %s\n", e.Message)
		if e.Details != "" {
			fmt.Fprintf(os.Stderr, "Details: %s\n", e.Details)
		}
	case *client.ConnectionError:
		fmt.Fprintf(os.Stderr, "Connection Error: %s\n", e.Error())
		fmt.Fprintf(os.Stderr, "Check your connection and server URL.\n")
	default:
		fmt.Fprintf(os.Stderr, "Error: %s\n", err.Error())
	}
	
	os.Exit(1)
}
```

## Interactive Mode

For complex operations, the CLI can use interactive prompts:

```go
func createSandboxInteractive() error {
	// Prompt for sandbox name
	name, err := prompt.Input("Sandbox name: ", nil)
	if err != nil {
		return err
	}
	
	// Prompt for namespace
	namespace, err := prompt.Input("Namespace: ", func(input string) string {
		if input == "" {
			return "default"
		}
		return input
	})
	if err != nil {
		return err
	}
	
	// Prompt for TTL
	ttlStr, err := prompt.Input("Time-to-live (in hours): ", func(input string) string {
		if input == "" {
			return "24"
		}
		return input
	})
	if err != nil {
		return err
	}
	
	ttl, err := strconv.ParseInt(ttlStr, 10, 64)
	if err != nil {
		return fmt.Errorf("invalid TTL: %v", err)
	}
	
	// Convert to seconds
	ttl = ttl * 3600
	
	// Prompt for services
	var services []string
	addMore := true
	
	for addMore {
		service, err := prompt.Input("Service (name=image): ", nil)
		if err != nil {
			return err
		}
		
		if service != "" {
			services = append(services, service)
		}
		
		addMore, err = prompt.Confirm("Add another service?", false)
		if err != nil {
			return err
		}
	}
	
	// Create sandbox
	return createSandbox(name, namespace, ttl, services)
}
```

## Local Development Integration

The CLI includes functionality to support local development:

```go
func connectLocalService(namespace, sandbox, service string, ports []string) error {
	fmt.Printf("Connecting local service %s to sandbox %s...\n", service, sandbox)
	
	// Validate ports
	portMappings, err := parsePortMappings(ports)
	if err != nil {
		return err
	}
	
	// Create local workload
	client := getClient()
	spec := &api.LocalWorkloadSpec{
		SandboxRef: api.ObjectRef{
			Name: sandbox,
		},
		Service: api.ServiceSpec{
			Name:  service,
			Ports: portMappings,
		},
	}
	
	workload, err := client.CreateLocalWorkload(namespace, service, spec)
	if err != nil {
		return fmt.Errorf("failed to create local workload: %v", err)
	}
	
	// Set up port forwarding
	fmt.Println("Setting up port forwarding...")
	forwarder := NewPortForwarder(client, workload)
	if err := forwarder.Start(); err != nil {
		return fmt.Errorf("failed to start port forwarding: %v", err)
	}
	
	fmt.Printf("Local service %s connected to sandbox %s!\n", service, sandbox)
	fmt.Println("Press Ctrl+C to disconnect")
	
	// Wait for signal
	waitForSignal()
	
	return nil
}
```

## Testing

The CLI should be thoroughly tested:

```go
func TestSandboxCreateCommand(t *testing.T) {
	// Create test server
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Verify request
		assert.Equal(t, "/api/v1alpha1/namespaces/default/sandboxes", r.URL.Path)
		assert.Equal(t, "POST", r.Method)
		
		// Parse request body
		var sandbox api.Sandbox
		err := json.NewDecoder(r.Body).Decode(&sandbox)
		assert.NoError(t, err)
		
		// Verify sandbox
		assert.Equal(t, "test-sandbox", sandbox.Metadata.Name)
		assert.Equal(t, int64(3600), sandbox.Spec.TTL)
		
		// Return response
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusCreated)
		
		// Add server-side fields
		sandbox.Status.Phase = "Pending"
		sandbox.Metadata.CreationTimestamp = api.Time{Time: time.Now()}
		
		json.NewEncoder(w).Encode(sandbox)
	}))
	defer server.Close()
	
	// Set up viper config for test
	viper.Reset()
	viper.Set("server.url", server.URL)
	
	// Run command
	cmd := NewRootCommand()
	cmd.SetArgs([]string{"sandbox", "create", "test-sandbox", "--ttl", "3600"})
	
	// Capture output
	out := &bytes.Buffer{}
	cmd.SetOut(out)
	
	// Execute command
	err := cmd.Execute()
	assert.NoError(t, err)
	
	// Verify output
	assert.Contains(t, out.String(), "Sandbox test-sandbox created successfully!")
}
```

## Documentation

The CLI includes comprehensive help documentation:

```go
func NewRootCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "sandboxctl",
		Short: "Manage sandbox environments for microservice testing",
		Long: `
Sandboxctl is a command line tool for creating and managing sandbox 
environments for testing microservices against production dependencies.

It allows developers to:
- Create isolated sandbox environments
- Override specific services in the request path
- Connect local development environments to sandboxes
- Test changes without affecting production

Basic usage:
  # Create a new sandbox
  sandboxctl sandbox create my-sandbox

  # Connect local service to sandbox
  sandboxctl local connect --sandbox my-sandbox --service my-service

  # Delete sandbox
  sandboxctl sandbox delete my-sandbox
`,
		Example: `
  # Create a sandbox with a 48-hour TTL
  sandboxctl sandbox create my-sandbox --ttl 172800

  # Create a sandbox with service overrides
  sandboxctl sandbox create my-sandbox --service api=my-registry/api:feature-branch

  # Connect local service to sandbox with port mapping
  sandboxctl local connect --sandbox my-sandbox --service frontend --port 3000:8080
`,
	}
	
	// Add commands and flags
	
	return cmd
}
```

## Next Steps

1. Implement basic command structure
2. Create client for API interaction
3. Implement sandbox management commands
4. Add local development commands
5. Implement output formatting
6. Add configuration management
7. Create comprehensive help documentation
8. Write tests for commands