# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenTofu is an open-source Infrastructure as Code (IaC) tool for building, changing, and versioning infrastructure safely and efficiently. It's a community-driven fork that uses a high-level configuration syntax (HCL) to describe infrastructure as code.

**Language**: Go 1.25.3
**License**: Mozilla Public License v2.0

## Common Development Commands

### Building
```bash
# Build the tofu binary in the current directory
go build ./cmd/tofu
# Or use make
make build

# Test the binary
./tofu --version
```

### Testing
```bash
# Run all unit tests
go test ./...
# Or use make
make test

# Run tests for a specific package
go test ./internal/command/...
go test ./internal/addrs

# Run tests with coverage
make test-with-coverage

# Run acceptance tests (requires TF_ACC=1)
TF_ACC=1 go test ./internal/initwd
```

### Integration Tests
```bash
# List all available integration tests
make list-integration-tests

# Run backend-specific integration tests
make test-s3        # S3 backend (requires AWS credentials)
make test-pg        # PostgreSQL backend (requires Docker)
make test-consul    # Consul backend (requires Docker)
make test-kubernetes # Kubernetes backend (requires Docker)

# Run all integration tests
make integration-tests
```

### Linting and Code Generation
```bash
# Run golangci-lint
make golangci-lint

# Generate code (run after modifying generated sources)
go generate ./...
# Or use make
make generate

# Regenerate protobuf stubs (only needed when modifying .proto files)
make protobuf

# Check licenses of dependencies (requires GITHUB_TOKEN)
export GITHUB_TOKEN=your_token_here
make license-check
```

### Debugging
Use the debug configurations in contributing/DEVELOPING.md for VSCode or IntelliJ. The `debug-opentofu` script in `scripts/` can run OpenTofu in debug mode on port 2345.

Set `TF_LOG=trace` environment variable for detailed logging.

## Architecture

OpenTofu uses a **graph-based execution model** where operations are represented as directed acyclic graphs (DAGs). The high-level architecture is documented in `docs/architecture.md`.

### Core Components

**CLI Layer** (`internal/command/`)
- Entry point for all user commands
- Maps command names to implementations in `cmd/tofu/commands.go`
- Constructs `backend.Operation` objects describing the action to take

**Backend System** (`internal/backend/`)
- Determines where state snapshots are stored
- The `local` backend executes operations locally and wraps other backends
- Remote backends (s3, gcs, azure, consul, etc.) only handle state storage
- Each backend uses a state manager (`internal/states/statemgr/`)

**Configuration Loader** (`internal/configs/`)
- Parses HCL configuration files
- Loads root module and all child modules
- Produces `configs.Config` representing the entire configuration tree
- Some expressions remain as `hcl.Body` or `hcl.Expression` for later evaluation

**Core Context** (`internal/tofu/`)
- `tofu.Context` is the main orchestrator for operations
- Builds and executes graphs for plan, apply, destroy, etc.
- Uses graph transformers to build operation-specific graphs
- Walks graphs respecting dependency edges, evaluating vertices concurrently where possible

**Graph Execution**
- Graph vertices represent resources, modules, providers, etc.
- Graph edges represent "happens after" dependencies
- Graph builders use transforms (`tofu.GraphTransformer`) to construct graphs
- Key transforms: `ConfigTransformer`, `StateTransformer`, `ReferenceTransformer`, `ProviderTransformer`
- Graph walker (`tofu.ContextGraphWalker`) evaluates vertices respecting dependencies

**Expression Evaluation** (`internal/lang/`)
- Evaluates HCL expressions during graph walk
- Uses `lang.Scope` to provide evaluation context
- Produces `cty.Value` objects representing values in the OpenTofu language
- Coordinates access to state, variables, and built-in functions

**State Management** (`internal/states/`)
- `states.State` represents a state snapshot
- `states.SyncState` provides thread-safe concurrent access
- State managers serialize/deserialize state (typically as JSON)

### Key Packages

- `internal/addrs` - Address types for resources, modules, providers, etc.
- `internal/plans` - Plan representation and diff logic
- `internal/providers` - Provider plugin protocol and RPC
- `internal/provisioners` - Provisioner plugin protocol
- `internal/encryption` - State encryption functionality
- `internal/getproviders` - Provider installation and registry interaction
- `internal/configs/configschema` - Schema definitions for providers

### Entry Point

The main entry point is `cmd/tofu/main.go`, which immediately delegates to command implementations based on the mapping in `cmd/tofu/commands.go`.

## Development Workflow

### Before Starting
1. Read the DCO (Developer Certificate of Origin) at https://developercertificate.org/
2. Configure git with your name and email matching your GitHub account
3. Do NOT use AI coding assistants - they may emit BSL-licensed Terraform code

### Making Changes
1. Find an issue with `accepted` and `help wanted` labels
2. Comment on the issue and wait for assignment
3. Write code yourself (no copy/paste from external sources)
4. Update `CHANGELOG.md` with your changes
5. Run tests: `go test ./internal/path/to/package`
6. Commit with DCO sign-off: `git commit -s -m "Your message"`
7. Submit a PR and complete the checklist

### Testing Your Changes
- Run unit tests for the specific package you're modifying
- If your change affects providers or backends, run relevant integration tests
- Acceptance tests (`TF_ACC=1`) test interactions with external services
- Integration tests (`make test-*`) test specific backend implementations

### Commit Messages
- Use descriptive messages focusing on "why" not "what"
- Always use `git commit -s` to add DCO sign-off
- Follow the style of recent commits (check with `git log`)
- Ensure git `user.name` and `user.email` match your GitHub settings

## Important Constraints

### Copyright and Licensing
- **Critical**: Never copy code from HashiCorp Terraform (BSL-licensed)
- Only contribute code you wrote yourself
- Add `Co-authored-by` if including code from others (with permission)
- Check licenses before copying from external sources
- Violations will disqualify your PR and may bar future contributions

### Code Quality
- Dependencies must use approved licenses (see `.licensei.toml`)
- Follow existing code style and conventions
- Use the provided linter configuration (`.golangci.yml`)
- Some packages are frozen for compatibility (see `.golangci.yml` exclusions)

### Generated Code
Some files are generated and should not be edited directly:
- Run `go generate ./...` after modifying generation sources
- Run `make protobuf` after modifying `.proto` files
- Use `git diff` to verify generated changes

## Terminology

See `docs/glossary.md` for project-specific terminology, including:
- **Attribute vs Argument**: Attribute = key in object type, Argument = setting in config block
- **Resource vs Resource Instance**: Resource = config block, Resource Instance = actual instance (with count/for_each)
- **Data source vs Data resource**: Data source = remote thing being read, Data resource = `data` block
- **Unknown value**: Result of expressions with unknown inputs (computed at apply time)
- **Mark/Value mark**: Annotations on cty.Values without modifying the underlying value

## Documentation

- Architecture overview: `docs/architecture.md`
- Development guide: `contributing/DEVELOPING.md`
- Contribution guide: `CONTRIBUTING.md`
- Release process: `CONTRIBUTING.RELEASE.md`
- Diagnostics style: `docs/diagnostics.md`
- Glossary: `docs/glossary.md`
