---
rfc: 0011
title: "Operation Metadata Tracking for currentOp Support"
status: Draft
owner: "@pamulart"
issue: "https://github.com/documentdb/documentdb/issues/XXX"
discussion: "https://github.com/documentdb/documentdb/discussions/XXX"
version-target: 1.0
implementations:
  - "https://github.com/documentdb/documentdb/pull/XXX"
---

# RFC-0011: Operation Metadata Tracking for currentOp Support

## Problem

DocumentDB lacks the visibility into active database operations needed for effective troubleshooting and monitoring. The existing currentOp implementation is missing critical fields that users expect, limiting their ability to:

* Debug application issues by correlating operations with application logs via session IDs
* Monitor transaction state across distributed operations
* Identify blocking operations and understand cursor lifecycle
* Track client connections and driver metadata for performance analysis
* View the actual command being executed (not just the PostgreSQL query)

### Who is impacted by this problem?

* Application developers who need to debug production performance issues
* Database administrators monitoring system health and identifying bottlenecks
* Operations teams troubleshooting transaction conflicts and long-running queries
* Migration teams moving from MongoDB expecting consistent monitoring capabilities

### What are the consequences of not solving it?
* Manual correlation burden, forcing developers to piece together logs from multiple sources
* Migration friction, as teams expect MongoDB-standard monitoring capabilities
* Limited extensibility—without proper infrastructure, adding new monitoring fields remains difficult

### What are the success criteria?
* **MongoDB Compatibility**: Implement essential currentOp fields 
* **Reliability**: Automatic metadata cleanup with no memory leaks under high concurrency
* **Maintain High Performance**: Ensure metadata collection adds minimal overhead to command execution
* **Keep Logic in Extension Layer**: Avoid adding currentOp logic in the gateway, since aggregation-based currentOp is only possible in the extension layer where aggregation pipeline execution happens

### What are the non-goals?
* **Complete MongoDB Field Parity** (30-40+ additional fields): Focus on establishing infrastructure and essential fields first before expanding to all MongoDB parameters
* **Historical Tracking**: Only active operations tracked; long-term operation history is out of scope
* **Distributed Tracing Infrastructure**: While operation correlation is enabled via lsid, full distributed tracing system integration is deferred
* **Client Metadata Capture Mechanism**: While client metadata fields will be stored, the detailed mechanism for capturing and persisting this data will be addressed separately

### What behavior are you intending to emulate? What are the quirks there?
* **MongoDB currentOp Command**: Standard format and field semantics from MongoDB
* **Aggregation-based currentOp**: Support for `db.aggregate([{$currentOp: {}}])` syntax

#### MongoDB currentOp Quirks to Handle:

* Command truncation at 1KB (must still be valid BSON or marked as truncated)
* Session ID format: `{ "id": UUID }`
* Transaction timing in microseconds, not milliseconds
* OpID format must be parseable for kill operations
* BSON documents always store their length in the first 4 bytes (little-endian)

---

## Approach
### What is your proposed solution?
The solution uses shared memory to store operation metadata alongside PostgreSQL's existing BackendStatusArray. Each backend has a dedicated slot in the `OpMetadataBackendArray` where it stores MongoDB-specific operation data that isn't available in PostgreSQL's infrastructure.

Key components:
1. **Shared Memory Array**: `OpMetadataBackendArray[MaxBackends]` storing command BSON and session IDs
2. **Pre-command Hook**: `documentDbPreCommand()` called at the start of each command
3. **CurrentOp Integration**: Enhanced to read from both pg_stat_activity and OpMetadataBackendArray

### Why is this approach better than alternatives?
This approach keeps the operation metadata in the extension layer specifically for the currentOp aggregation stages. Implementing this in the gateway would make the aggregation stage much more complicated, as the extension requires access to this data. Using shared memory with PostgreSQL's existing concurrency protocols ensures thread-safety without custom locking.

### What are the key benefits and tradeoffs?

**Benefits:**
* Leverages PostgreSQL's existing BackendStatusArray infrastructure
* Supports both command-based and aggregation-based currentOp queries
* Automatic cleanup on backend termination

**Tradeoffs:**
* Fixed memory allocation per backend (approximately 1.2KB per slot)
* Command BSON limited to 1KB (matches MongoDB's truncation behavior)

---

## Detailed Design

### Technical Details
The currentOp metadata tracking system captures operation-specific information in shared memory to support the currentOp command. The design maximally leverages PostgreSQL's existing BackendStatusArray infrastructure to avoid data duplication while storing only truly unique operation metadata.

#### Data Structures

**OpMetadata Structure** (op_metadata.h):

```c
typedef struct OpMetadata
{
    /* Command BSON data (truncated to MAX_OP_COMMAND_LENGTH if larger) */
    uint32_t commandLength;
    char commandData[MAX_OP_COMMAND_LENGTH];  // 1024 bytes

    /* Session ID (lsid) for tracking logical sessions */
    char lsid[MAX_LSID_LENGTH];  // 128 bytes
} OpMetadata;
```

**Global Shared Memory Array**:

```c
OpMetadata *OpMetadataBackendArray;  // Sized by MaxBackends
```

**Integration Points**:

1. **Shared Memory Initialization** (documentdb_init.c):
   - `SharedOpMetadataShmemInit()` called during extension startup
   - Allocates `MaxBackends * sizeof(OpMetadata)` bytes

2. **Command Pre-hook** (commands_common.c):
   ```c
   void documentDbPreCommand(pgbson *commandSpec)
   {
       /* Register operation metadata for currentOp */
       ExtractAndRegisterOpMetadataFromCommand(commandSpec);
   }
   ```

3. **Metadata Registration** (op_metadata.c):
   - Extracts lsid from command BSON
   - Stores raw command BSON (up to 1KB)
   - Uses PostgreSQL's changecount protocol for thread-safe writes


```javascript
Backend #5:
  OpMetadataBackendArray[5]         BackendStatusArray[5]
  ├─ commandLength                  ├─ st_procpid (PID)
  ├─ commandData (BSON)             ├─ st_state (active/idle)
  └─ lsid (session ID)              ├─ st_activity_start_timestamp
                                    └─ st_activity_raw (SQL text)
         └────────── Linked by backend index ──────────┘
```

### Algorithms

**Write Algorithm (Registration)**:

```javascript
1. Extract metadata from command BSON:
   - Parse lsid.id UUID if present
   - Format as hex string (xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
   - Store raw BSON up to MAX_OP_COMMAND_LENGTH

2. Protected write using changecount protocol:
   beentry = MyBEEntry

   PGSTAT_BEGIN_WRITE_ACTIVITY(beentry)
     ├─ changecount++ (ODD - signals "writing")
     └─ pg_write_barrier()

   memcpy(&OpMetadataBackendArray[MyProcNumber], metadata, sizeof(OpMetadata))

   PGSTAT_END_WRITE_ACTIVITY(beentry)
     ├─ pg_write_barrier()
     └─ changecount++ (EVEN - signals "stable")
```

**BSON Truncation Handling**:

```javascript
// Storage (op_metadata.c):
if (bsonLength > MAX_OP_COMMAND_LENGTH)
    storedLength = MAX_OP_COMMAND_LENGTH  // Truncate but preserve BSON header

// Retrieval (current_op.c):
uint32_t originalLength = *(uint32_t*)commandData  // Read BSON length header
if (storedLength < originalLength)
    // BSON was truncated - mark with $truncated: true
```

**CurrentOp Read Algorithm**:

```javascript
1. Query pg_stat_activity for backends

2. For each backend:
   - Check if state == "active" (CRITICAL: skip idle backends)
   - If idle: skip entirely (stale metadata ignored)
   - If active:
     - Get backend index from proc number
     - Read OpMetadataBackendArray[index] with changecount protection
     - Parse stored BSON if not truncated
     - Extract command fields and determine operation type
```

### Architecture Patterns

**1. Per-Backend Slot Allocation**
- One fixed slot per backend in shared memory
- Indexed by ProcNumber (PG17+) or BackendId-1 (older versions)
- No explicit cleanup needed - slot reused on next command

**2. Concurrency Control**
- Uses PostgreSQL's changecount protocol (lock-free)
- Counter is ODD during updates, EVEN when stable
- Readers retry if they see ODD or changing counter
- Memory barriers ensure proper CPU ordering

**3. Implicit Cleanup via Backend State**
- **No explicit cleanup required** - leverages PostgreSQL's existing state management
- When command completes: backend state changes from "active" to "idle" in pg_stat_activity
- CurrentOp query logic: `if (state != "active") skip backend`
- Stale metadata in idle slots is never read, effectively "cleaned up"
- Slot reused when backend handles next command (overwrites old data)
- Backend termination handled by PostgreSQL (slot becomes available for new backend)

This design eliminates cleanup overhead and race conditions by using PostgreSQL's authoritative state tracking.

### API Changes

**New currentOp Fields Implemented**:
- `lsid`: Logical session ID for operation correlation
- `command`: Actual command BSON (truncated if >1KB)
- `effectiveUsers`: User context for the operation
- `connectionId`: User's connection id
- `client`: Name of client who made request
- `desc`: Description of client/connection
- `microsecs_running`: Microseconds request has been running


**Planned Fields**:
- `clientMetadata`: Driver and application information
- Transaction parameters
- Cursor parameters

### Configuration Changes

**New GUC Parameter**:
```
documentdb.enable_op_metadata_collection = false  # Default
```

When enabled:
- Allocates shared memory for OpMetadataBackendArray
- Activates metadata collection in command handlers
- Enables enhanced currentOp fields

### Testing Strategy

**Unit Tests**:
- Metadata extraction from various command types
- BSON truncation handling and validation
- Shared memory read/write concurrency

**Integration Tests**:
- Command field display for all operation types

**Performance Tests**:
- Overhead measurement under concurrent load
- Memory usage validation
- Truncation impact on large commands

### Migration Path

**Backwards Compatibility**:
- Feature disabled by default via GUC
- No changes to existing currentOp fields
- Additional fields are additive only

**Upgrade Path**:
1. Deploy new extension version
2. Enable GUC if desired: `SET documentdb.enable_op_metadata_collection = true`
3. New fields automatically appear in currentOp output

**Rollback Strategy**:
- Disable GUC to stop metadata collection
- Shared memory cleaned on PostgreSQL restart
- No persistent state to migrate

### Documentation Updates

**User Documentation**:
- New currentOp fields reference
- Session-based debugging guide
- Performance monitoring best practices

**Developer Documentation**:
- Shared memory architecture overview
- Adding new metadata fields guide
- Concurrency protocol explanation

---

## Implementation Tracking

### Implementation PRs

- [ ] PR #1: Core infrastructure - OpMetadata structure and shared memory
  - Files: `op_metadata.h`, `op_metadata.c`
  - Status: In Progress

- [ ] PR #2: Command integration - documentDbPreCommand hook
  - Files: `commands_common.c`, command handlers
  - Status: In Progress (insert.c integrated, others pending)

- [ ] PR #3: CurrentOp enhancements
  - Files: `current_op.c`
  - Status: Pending

- [ ] PR #4: GUC integration and initialization
  - Files: `system_configs.c`, `documentdb_init.c`
  - Status: Pending

- [ ] PR #5: Complete command handler integration
  - Files: `update.c`, `delete.c`, `find_and_modify.c`, etc.
  - Status: Pending