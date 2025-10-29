# State Management Skill (internal/states/)

## Overview

You are an expert in OpenTofu's state management system. This skill covers:
- Infrastructure state snapshots and lifecycle
- Thread-safe concurrent access patterns
- State serialization, versioning, and encryption
- State managers and storage backends
- Best practices for working with state

## Core Concepts

### State Hierarchy

OpenTofu's state is organized in a hierarchical structure:

```
State
├── Modules (map[string]*Module)
│   ├── Resources (map[string]*Resource)
│   │   └── Instances (map[InstanceKey]*ResourceInstance)
│   │       ├── Current (ObjectSrc)
│   │       └── Deposed (map[DeposedKey]*ObjectSrc)
│   ├── OutputValues (map[string]*OutputValue)
│   └── LocalValues (map[string]cty.Value)
└── CheckResults (*CheckResults)
```

**Key Types:**

- **State**: Top-level container for all state data
- **Module**: State for a specific module instance
- **Resource**: Container for all instances of a resource
- **ResourceInstance**: State for a single resource instance
- **ResourceInstanceObjectSrc**: Serialized form of resource attributes

### State Metadata

Every state has critical metadata:

- **Lineage**: UUID set at creation, never changes. Identifies a unique state lineage.
- **Serial**: Monotonic version counter. Increments on every modification.
- **TerraformVersion**: Version that last modified the state.

**Usage:**
- Two states with different lineages cannot be compared (they're unrelated)
- Higher serial = newer state (only within same lineage)
- Used for conflict detection in concurrent operations

## Thread-Safe Concurrent Access

### SyncState Pattern

**Never access State directly in concurrent code.** Always use `SyncState`:

```go
type SyncState struct {
    state *State
    lock sync.RWMutex
}
```

**Thread-Safety Rules:**

1. **Read operations** acquire read lock and return deep copies
2. **Write operations** acquire write lock and mutate in-place
3. **Callers can safely mutate returned copies** without affecting state
4. **Multiple readers** can access state concurrently
5. **Writers block all other access**

### Common Access Patterns

#### Pattern 1: Reading State (Safe Snapshot)

```go
// Returns a deep copy - safe to mutate
resource := syncState.Resource(addr)
if resource == nil {
    return nil, fmt.Errorf("resource not found")
}

// Iterate safely over snapshot
for key, instance := range resource.Instances {
    // Work with instance without worrying about concurrent changes
    if instance.Current != nil {
        processObject(instance.Current)
    }
}
```

#### Pattern 2: Atomic Modifications

```go
// High-level atomic operations (preferred)
syncState.SetResourceInstanceCurrent(
    addr,
    objectSrc,
    providerConfig,
    providerKey,
)

// Other atomic operations:
syncState.SetOutputValue(addr, value, sensitive, deprecated)
syncState.RemoveResource(addr)
syncState.ForgetResourceInstanceDeposed(addr, deposedKey)
```

#### Pattern 3: Complex Multi-Step Operations

```go
// For operations requiring multiple coordinated changes
syncState.Lock()
defer syncState.Unlock()

state := syncState.state  // Direct access to underlying state

// Perform multiple operations atomically
module := state.EnsureModule(moduleAddr)
module.SetResourceInstanceCurrent(addr, obj, provider)
module.SetOutputValue(outputAddr, value, false, "")
// All changes are atomic as a unit
```

**Important:** Minimize time holding explicit locks. Release as soon as possible.

## State Snapshots and Lifecycle

### Two-Level Storage Model

```
┌─────────────────────────────────────┐
│   Transient Snapshot (Memory)       │  Current working state
│   - Fast in-memory modifications    │  Modified during operations
│   - Not persisted automatically     │  Lost on process exit
└────────────┬────────────────────────┘
             │
             │ RefreshState() ↓
             │ PersistState() ↑
             │
┌────────────┴────────────────────────┐
│  Persistent Snapshot (Durable)      │  Stable, durable state
│  - Stored in backend (disk, S3)     │  Survives process exit
│  - Shared across processes          │  Version controlled
│  - With Serial & Lineage metadata   │
└─────────────────────────────────────┘
```

### State Manager Operations

```go
// Load from persistent storage → transient
err := stateManager.RefreshState(ctx)

// Get current transient state
state := stateManager.State()

// Work with state through SyncState
syncState := state.SyncWrapper()
syncState.SetResourceInstanceCurrent(addr, obj, provider, key)

// Save transient → persistent storage
err := stateManager.PersistState(ctx, schemas)
```

## State Manager Interfaces

### Interface Hierarchy

```go
// Transient: In-memory operations
type Transient interface {
    State() *states.State           // Get current state
    WriteState(*states.State) error // Replace state
}

// Persistent: Durable storage operations
type Persistent interface {
    RefreshState(ctx) error          // Load from storage
    PersistState(ctx, *Schemas) error // Save to storage
}

// Storage: Combined transient + persistent
type Storage interface {
    Transient
    Persistent
}

// Full: Everything including locking
type Full interface {
    Storage
    Locker
}
```

### Key Implementations

**1. Filesystem (Local Files)**

```go
// Create filesystem-based state manager
fsState := statemgr.NewFilesystem("terraform.tfstate", encryption)

// Load initial state
err := fsState.RefreshState(ctx)

// Get/modify state
state := fsState.State()
// ... modifications ...

// Save back to disk
err := fsState.PersistState(ctx, schemas)
```

**Features:**
- Supports separate read/write paths
- Optional backup file creation
- File-based locking for mutual exclusion
- Handles encryption transparently

**2. TransientInMemory (Temporary)**

```go
// For plans, tests, temporary operations
inmemState := statemgr.NewTransientInMemory(initialState)

// Fast, no I/O, not persisted
state := inmemState.State()
```

**3. Remote (Backend Storage)**

```go
// For remote backends (S3, Terraform Cloud, etc.)
remoteState := remote.State(client, schemas, encryption)

// Same interface as filesystem
err := remoteState.RefreshState(ctx)
err := remoteState.PersistState(ctx, schemas)
```

## Resource Lifecycle and Deposed Objects

### Create-Before-Destroy Pattern

When `create_before_destroy` is enabled:

1. **Depose** current object (move aside with generated key)
2. **Create** new replacement as current
3. **Destroy** deposed object after successful creation
4. **Restore** deposed if creation fails

```go
// Step 1: Depose current object
deposedKey := syncState.DeposeResourceInstanceObject(addr)
// Returns: DeposedKey (8-char hex string)

// Step 2: Create new object as current
syncState.SetResourceInstanceCurrent(addr, newObj, provider, key)

// Step 3a: Success - destroy deposed
syncState.ForgetResourceInstanceDeposed(addr, deposedKey)

// Step 3b: Failure - restore deposed
syncState.MaybeRestoreResourceInstanceDeposed(addr, deposedKey)
```

### Object Status

```go
type ObjectStatus byte

const (
    ObjectReady   ObjectStatus = 'R' // Ready for use
    ObjectTainted ObjectStatus = 'T' // Must be replaced
    ObjectPlanned ObjectStatus = 'P' // Transient placeholder
)
```

**Usage:**
- `ObjectReady`: Normal, healthy resource
- `ObjectTainted`: Resource in unrecoverable state, will be replaced
- `ObjectPlanned`: Temporary during plan/refresh, never persisted

## Serialization and Versioning

### State File Format

OpenTofu uses versioned JSON format (currently V4):

```json
{
  "version": 4,
  "terraform_version": "1.x.x",
  "serial": 42,
  "lineage": "uuid-string",
  "outputs": { ... },
  "resources": [
    {
      "module": "module.name[index]",
      "mode": "managed",
      "type": "aws_instance",
      "name": "example",
      "provider": "provider[\"...\"]",
      "instances": [
        {
          "index_key": 0,
          "schema_version": 2,
          "attributes": { ... },
          "sensitive_attributes": [],
          "private": "base64...",
          "dependencies": [ ... ]
        }
      ]
    }
  ]
}
```

### Reading and Writing State Files

```go
// Reading
file, err := statefile.Read(reader, encryption)
if err != nil {
    return err
}
state := file.State  // Extract State object

// Automatic version upgrades
// V0-V3 automatically upgraded to V4 on read

// Writing
file := &statefile.File{
    Lineage:          lineage,
    Serial:           serial,
    TerraformVersion: version.Current(),
    State:            state,
}
err := file.Write(writer, encryption)
```

### Attribute Encoding

Resource attributes are stored as:

```go
type ResourceInstanceObjectSrc struct {
    SchemaVersion uint64        // Schema version when encoded
    AttrsJSON     []byte        // JSON-encoded cty.Value
    Private       []byte        // Provider-opaque data

    AttrSensitivePaths      []cty.PathValueMarks // Sensitive markers
    TransientPathValueMarks []cty.PathValueMarks // Not persisted

    Dependencies        []addrs.ConfigResource
    CreateBeforeDestroy bool
    Status              ObjectStatus
}
```

**Decoding to ResourceInstanceObject:**

```go
// Decode with provider schema
obj, err := objectSrc.Decode(resourceType, schema)
if err != nil {
    return err
}

// obj is now a ResourceInstanceObject with:
// - Value (cty.Value) - decoded attributes
// - Status, Private, Dependencies (copied)
```

## Encryption

### Encryption Interface

```go
type StateEncryption interface {
    EncryptState([]byte) (ciphertext []byte, status EncryptionStatus, err error)
    DecryptState([]byte) (plaintext []byte, status EncryptionStatus, err error)
}
```

### Integration Points

Encryption is transparent at the state manager level:

```go
// Reading with encryption
file, err := statefile.Read(reader, encryptionImpl)
// Automatically decrypts if needed

// Writing with encryption
err := file.Write(writer, encryptionImpl)
// Automatically encrypts if configured

// Filesystem state manager
fsState := statemgr.NewFilesystem(path, encryptionImpl)
// Handles encryption transparently in all operations
```

**Encryption Status Tracking:**

```go
type EncryptionStatus string

const (
    EncryptionUnknown     EncryptionStatus = "unknown"
    EncryptionEncrypted   EncryptionStatus = "encrypted"
    EncryptionUnencrypted EncryptionStatus = "unencrypted"
)
```

## State Locking

### Preventing Concurrent Modifications

State locking ensures only one operation modifies state at a time:

```go
type Locker interface {
    Lock(ctx context.Context, info *LockInfo) (string, error)
    Unlock(ctx context.Context, id string) error
}

type LockInfo struct {
    ID        string    // Unique lock ID
    Operation string    // "plan", "apply", etc.
    Who       string    // user@hostname
    Version   string    // OpenTofu version
    Created   time.Time
    Path      string    // Optional state path
}
```

### Lock Acquisition with Retry

```go
lockInfo := statemgr.NewLockInfo()
lockInfo.Operation = "apply"
lockInfo.Who = fmt.Sprintf("%s@%s", user, hostname)

// Automatic retry with exponential backoff
lockID, err := statemgr.LockWithContext(ctx, locker, lockInfo)
if err != nil {
    if lerr, ok := err.(*statemgr.LockError); ok {
        // Handle lock error with existing lock info
        fmt.Fprintf(os.Stderr, "Lock held by: %s\n", lerr.Info.Who)
    }
    return err
}
defer locker.Unlock(ctx, lockID)

// Perform operations while locked
```

**Lock Error Handling:**

```go
type LockError struct {
    Info *LockInfo  // Current lock holder info
    Err  error      // Underlying error
}

// Check if retry is appropriate
if lerr.Retriable() {
    // Can retry after delay
}
if lerr.RetriableWithoutDelay() {
    // Retry immediately (race condition)
}
```

## Output Values

### Managing Module Outputs

```go
// Set output value
syncState.SetOutputValue(
    addrs.RootModuleInstance.OutputValue("example"),
    cty.StringVal("value"),
    false,      // sensitive
    "",         // deprecated message
)

// Read output value
output := syncState.OutputValue(addr)
if output != nil {
    value := output.Value        // cty.Value
    sensitive := output.Sensitive // bool
    isDeprecated := output.Deprecated != ""
}

// Remove output
syncState.RemoveOutputValue(addr)
```

### OutputValue Structure

```go
type OutputValue struct {
    Value      cty.Value  // The actual output value
    Sensitive  bool       // Hide in UI/logs
    Deprecated string     // Optional deprecation message
}
```

## Check Results

### Recording Assertion Results

Check results track assertion/postcondition outcomes:

```go
// Discard stale check results before modifications
syncState.DiscardCheckResults()

// Record new check results
syncState.RecordCheckResults(checkState)

// Access check results
if state.CheckResults != nil {
    for _, result := range state.CheckResults.ConfigResults.Elems {
        configAddr := result.Key
        aggregate := result.Value

        switch aggregate.Status {
        case checks.StatusPass:
            // All checks passed
        case checks.StatusFail:
            // Some checks failed
        case checks.StatusError:
            // Error evaluating checks
        }
    }
}
```

## Deep Copy and Isolation

### Automatic Deep Copying

All read operations return deep copies for isolation:

```go
// Each call returns independent copy
module1 := syncState.Module(addr)
module2 := syncState.Module(addr)

// Modifying module1 doesn't affect module2
module1.LocalValues["key"] = newValue
// module2 is unchanged
```

### Manual Deep Copying

```go
// Explicit deep copy
stateCopy := state.DeepCopy()

// All nested objects are copied:
// - Modules
// - Resources
// - ResourceInstances
// - OutputValues
// - CheckResults
```

**Note:** Addresses and cty.Value objects are NOT copied (treated as immutable).

## Common Patterns

### Pattern: Resource Lookup and Modification

```go
// Safe pattern for checking and modifying resources
resource := syncState.Resource(addr)
if resource == nil {
    // Resource doesn't exist, create it
    syncState.SetResourceInstanceCurrent(
        addr.Instance(addrs.NoKey),
        newObjectSrc,
        providerConfig,
        addrs.NoKey,
    )
    return
}

// Resource exists, update current instance
instance := resource.Instance(addrs.NoKey)
if instance != nil && instance.Current != nil {
    // Modify based on current state
    updatedObj := modifyObject(instance.Current)
    syncState.SetResourceInstanceCurrent(
        addr.Instance(addrs.NoKey),
        updatedObj,
        providerConfig,
        addrs.NoKey,
    )
}
```

### Pattern: State File Roundtrip

```go
// Read state file
f, err := os.Open("terraform.tfstate")
if err != nil {
    return err
}
defer f.Close()

stateFile, err := statefile.Read(f, encryption)
if err != nil {
    return err
}

state := stateFile.State

// Modify state
syncState := state.SyncWrapper()
syncState.SetResourceInstanceCurrent(addr, obj, provider, key)

// Write back
outFile := &statefile.File{
    Lineage:          stateFile.Lineage,
    Serial:           stateFile.Serial + 1,  // Increment!
    TerraformVersion: version.Current(),
    State:            state,
}

f2, err := os.Create("terraform.tfstate")
if err != nil {
    return err
}
defer f2.Close()

err = outFile.Write(f2, encryption)
```

### Pattern: Pruning Empty State

```go
// Remove resource
syncState.RemoveResource(addr)

// Prune empty modules/resources
state.PruneResourceHusks()

// Removes:
// - Resources with no instances
// - Modules with no resources/outputs/locals
```

### Pattern: Moving Resources

```go
// Move resource to new address
moved := state.MoveAbsResource(oldAddr, newAddr)
if !moved {
    return fmt.Errorf("resource not found at %s", oldAddr)
}

// Or move single instance
moved := state.MoveAbsResourceInstance(oldAddr, newAddr)
```

## Best Practices

### DO

1. **Always use SyncState** for concurrent access
2. **Use high-level atomic operations** (SetResourceInstanceCurrent, etc.)
3. **Increment Serial** when modifying persistent state
4. **Check lineage** before comparing states
5. **Use deep copies** when passing state between goroutines
6. **Lock state** before long operations
7. **Discard check results** before state modifications
8. **Prune empty resources** after removals

### DON'T

1. **Don't access State directly** in concurrent code
2. **Don't hold locks** during I/O operations
3. **Don't modify returned values** assuming they affect state (they're copies)
4. **Don't forget to unlock** (use defer)
5. **Don't compare states** with different lineages
6. **Don't persist ObjectPlanned** status (transient only)
7. **Don't skip schema validation** when decoding objects

## File Reference

**Core State:**
- `internal/states/state.go` - State type and operations
- `internal/states/module.go` - Module state
- `internal/states/resource.go` - Resource and instances
- `internal/states/instance_object.go` - Object types
- `internal/states/instance_object_src.go` - Serialization
- `internal/states/sync.go` - Thread-safe wrapper

**Serialization:**
- `internal/states/statefile/file.go` - File metadata
- `internal/states/statefile/read.go` - Deserialization
- `internal/states/statefile/write.go` - Serialization
- `internal/states/statefile/version4.go` - Current format

**State Management:**
- `internal/states/statemgr/statemgr.go` - Core interfaces
- `internal/states/statemgr/filesystem.go` - Local files
- `internal/states/statemgr/locker.go` - Locking
- `internal/states/remote/state.go` - Remote backends

## Task Guidelines

When working with state management:

1. **Understand the context**: Is this transient or persistent state? Concurrent or single-threaded?

2. **Choose the right abstraction**:
   - Simple operations: Use SyncState atomic methods
   - Complex operations: Lock, modify, unlock
   - File operations: Use state managers

3. **Preserve metadata**:
   - Don't lose lineage
   - Increment serial on modifications
   - Track encryption status

4. **Test thoroughly**:
   - Test concurrent access patterns
   - Verify serialization roundtrips
   - Check edge cases (empty modules, deposed objects)

5. **Follow conventions**:
   - Use deep copies for isolation
   - Return errors, don't panic
   - Clean up empty structures

## Common Troubleshooting

**Issue: "state lineage mismatch"**
- States have different lineages (unrelated states)
- Solution: Don't compare/merge states with different lineages

**Issue: "state locked by another process"**
- Another operation holds the lock
- Solution: Wait for lock release or use force-unlock (carefully)

**Issue: "concurrent modification detected"**
- Serial number changed between refresh and persist
- Solution: Refresh state and retry operation

**Issue: "resource not found in state"**
- Resource lookup returned nil
- Solution: Check if resource exists before accessing

**Issue: "deposed object not found"**
- Trying to restore/forget non-existent deposed object
- Solution: Check if deposed key exists before operations

---

This skill provides comprehensive knowledge of OpenTofu's state management system. Use it when working with state-related code, debugging state issues, or implementing new state features.
