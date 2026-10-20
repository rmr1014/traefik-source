# Traefik Development Guide for AI Agents

## Architecture Overview

Traefik is a modern HTTP reverse proxy and load balancer written in Go. The architecture follows a provider-based model where configuration sources (Kubernetes, Docker, etc.) dynamically generate routing rules.

### Core Components

- **cmd/traefik** - Main entry point. Uses `paerser` CLI library with loaders for config from flags, env vars, and files. Entry: [cmd/traefik/traefik.go](../cmd/traefik/traefik.go)
- **pkg/provider** - Implements `Provider` interface ([pkg/provider/provider.go](../pkg/provider/provider.go)) with `Provide()` method that pushes `dynamic.Message` to configuration channel. Providers: docker, kubernetes (crd/gateway/ingress), file, consul, etcd, etc.
- **pkg/server** - Core reverse proxy engine managing TCP/UDP entry points and configuration watchers ([pkg/server/server.go](../pkg/server/server.go))
- **pkg/middlewares** - HTTP middleware chain (auth, rate limiting, compression, headers, etc.) in [pkg/middlewares/](../pkg/middlewares/)
- **pkg/config** - Split into `static` (startup config) and `dynamic` (runtime routing config via providers)
- **pkg/safe** - Thread-safe value wrapper and goroutine pool for concurrent provider operations

### Configuration Flow

1. Static config loaded at startup (file/env/flags) → [pkg/config/static/static_config.go](../pkg/config/static/static_config.go)
2. Providers watch their backends and push `dynamic.Configuration` via channels
3. `ConfigurationWatcher` merges provider configs and updates routing dynamically
4. No restarts needed - routes update in real-time

## Development Workflows

### Building

```bash
make generate        # Generate code and WebUI
make binary          # Build traefik binary to dist/
make crossbinary-default  # Build for linux/darwin/windows
```

**Important**: WebUI must be built first (`make generate-webui`). The UI is embedded in Go binary via [webui/embed.go](../webui/embed.go).

### Testing

```bash
make test-unit       # Unit tests: ./pkg/... ./cmd/...
make test-integration  # Integration tests with Docker/K8s (20min timeout)
make test-gateway-api-conformance  # Gateway API conformance suite
make lint            # golangci-lint
```

**Integration tests** ([integration/integration_test.go](../integration/integration_test.go)) use `testcontainers-go` with k3s image. Tests compose Docker services from YAML fixtures and verify routing behavior.

### Code Generation

```bash
go generate  # Runs internal/gen to create config documentation
```

The root [generate.go](../generate.go) triggers code generation for static/dynamic config reference docs.

## Project Conventions

### Logging

Uses **zerolog** for structured logging. Pattern:

```go
log.Error().Err(err).Msg("Context message")
log.Warn().Str("key", value).Msg("Warning")
log.Info().Msg("Info message")
```

Never use `fmt.Println()` except in tests. Context is added via `.Str()`, `.Int()`, etc. before `.Msg()`.

### Error Handling

- Return errors up the stack; log at the boundary (provider/server level)
- Use `fmt.Errorf("context: %w", err)` for wrapping
- Tests use `require.NoError(t, err)` from testify

### Concurrency

Use `safe.Pool` ([pkg/safe/safe.go](../pkg/safe/safe.go)) for goroutine lifecycle management:

```go
pool.GoCtx(func(ctx context.Context) {
    // Provider watch loop
})
```

All providers receive a pool to spawn background tasks safely.

### Testing Patterns

- Unit tests: `*_test.go` files using `testify/suite` or `testify/require`
- Integration: Compose-based fixtures in [integration/resources/](../integration/resources/)
- Use `require.NoError()` consistently, never `t.Fatal()` directly

### Module Structure

- `github.com/traefik/traefik/v3` - v3 major version in module path
- Go 1.24.0+ required (see [go.mod](../go.mod))
- Heavy use of AWS SDK v2, Docker client, Kubernetes client-go

## Key Files & Patterns

### Provider Implementation

Implement `Provider` interface in [pkg/provider/provider.go](../pkg/provider/provider.go):

```go
type Provider interface {
    Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error
    Init() error
}
```

Example: [pkg/provider/docker/](../pkg/provider/docker/), [pkg/provider/kubernetes/](../pkg/provider/kubernetes/)

### Middleware Chain

Middlewares in [pkg/middlewares/](../pkg/middlewares/) follow constructor pattern returning `http.Handler`. Chain using Alice ([containous/alice](https://github.com/containous/alice)).

### Dynamic Configuration

Routes, services, and middleware configured via [pkg/config/dynamic/config.go](../pkg/config/dynamic/config.go) structure:

```go
type Configuration struct {
    HTTP *HTTPConfiguration  // Routers, Services, Middlewares
    TCP  *TCPConfiguration
    UDP  *UDPConfiguration
    TLS  *TLSConfiguration
}
```

## Common Tasks

### Adding a Provider

1. Create package in `pkg/provider/<name>/`
2. Implement `Provider` interface with `Provide()` watching backend
3. Add to static config in `pkg/config/static/static_config.go`
4. Push `dynamic.Message` when routes change
5. Add integration test in `integration/<name>_test.go`

### Adding a Middleware

1. Create package in `pkg/middlewares/<name>/`
2. Implement constructor returning `http.Handler`
3. Register in middleware factory
4. Add config struct to `pkg/config/dynamic/`
5. Document in `docs/content/middlewares/`

### Debugging

- Set `--log.level=DEBUG` for verbose output
- API endpoint (`:8080/api/rawdata`) shows live config
- WebUI dashboard at `:8080/dashboard/` visualizes routes
- Integration tests: Use `-tlog` flag to always show Traefik logs

## External Dependencies

- **Lego v4** - ACME/Let's Encrypt client for TLS certificates
- **Docker/Kubernetes clients** - Native Go clients for provider backends
- **kvtools** - Abstraction for KV stores (Consul/etcd/Redis/Zookeeper)
- **testcontainers-go** - Integration test infrastructure
- **paerser** - CLI/config parsing library (Traefik-maintained)

## Avoid Common Mistakes

- Don't modify files in `webui/static/` - they're generated (see [webui/readme.md](../webui/readme.md))
- Always run `make generate` before building after config struct changes
- Integration tests need Docker - they WILL fail in restricted environments
- Static config changes require restart; dynamic config does not
- Provider names must be unique across all provider types
