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

DocumentDB lacks the visibility into active database operations needed for effective troubleshooting and monitoring. The existing currentOp implementation is missing critical fields that MongoDB users expect, limiting their ability to:

* Debug application issues by correlating operations with application logs via session IDs
* Monitor transaction state across distributed operations
* Identify blocking operations and understand cursor lifecycle
* Track client connections and driver metadata for performance analysis

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
* **Keep Logic in Extension Layer**: Avoid adding currentOp logic in the gateway, since aggregation-based currentOp is only possible in the extension layer where aggregation pipeline execution happens. By storing metadata in shared memory, the pg_documentdb extension will have direct access to this information when executing both command-based and aggregation-based currentOp queries

### What are the non-goals?
* **Complete MongoDB Field Parity** (30-40+ additional fields): Focus on establishing infrastructure and essential fields first before expanding to all MongoDB parameters
* **Historical Tracking**: Only active operations tracked; long-term operation history is out of scope
* **Distributed Tracing Infrastructure**: While operation correlation is enabled via lsid, full distributed tracing system integration is deferred
* **Client Metadata Capture Mechanism**: While client metadata fields will be stored, the detailed mechanism for capturing and persisting this data will be addressed separately

### What behavior are you intending to emulate? What are the quirks there?
* **MongoDB currentOp Command**: Standard format and field semantics from MongoDB 
* **Aggregation-based currentOp**: Support for `db.aggregate([{$currentOp: {}}])` syntax

#### MongoDB currentOp Quirks to Handle:

* Command truncation at 1KB (must still be valid BSON)
* Session ID format: `{ "id": UUID }`
* Transaction timing in microseconds, not milliseconds
* OpID format must be parseable for kill operations

---

## Approach
### What is your proposed solution?
The proposed solution is to use User Defined Functions (UDFs) in each path of an operation to set, update, and remove metadata from our currentOp data structure. The currentOp metadata will use a shared memory array to store operation metadata in the extension for easy access when clients query different currentOp parameters with minimal performance overhead.

We will create `OpMetadataArray[MaxBackends]` in shared memory, where:

* Each PostgreSQL backend has exactly one dedicated array slot
* Backends write metadata to their own slot (`array[MyBackendId]`)
* currentOp reads across all slots to gather operation information
* The array persists for the lifetime of the PostgreSQL instance

### Why is this approach better than alternatives?
This approach keeps the operation metadata in the extension layer specifically for the currentOp aggregation stages. Implementing this in the gateway would make the aggregation stage much more complicated, as the extension requires access to this data. Therefore, keeping it native in the extension is the optimal solution for this use case.
### What are the key benefits and tradeoffs?

**Benefits:**
* This implementation allows us to add more parameters in the future to achieve MongoDB parity
* Enables users to utilize parameters they expect from MongoDB
* Provides a scalable foundation for extending monitoring capabilities

**Tradeoffs:**
* The gateway must not change the original command request from the client—any query optimization or transformation must happen in the extension layer, not the gateway
* If future gateway transformations are needed, they must be carefully coordinated with currentOp metadata collection


---

## Detailed Design

### Technical Details
The currentOp metadata tracking system captures operation-specific information in shared memory to support the currentOp command. The design maximally leverages PostgreSQL's existing BackendStatusArray infrastructure to avoid data duplication while storing only truly unique operation metadata.

#### Data Structures

__OpMetadata Structure__:

```c
typedef struct OpMetadata
{
	uint32_t commandLength;                   // Length of BSON data stored
	char commandData[MAX_OP_COMMAND_LENGTH];  // Raw command BSON (1KB)
	char lsid[MAX_LSID_LENGTH];              // Logical session ID (128 bytes)
} OpMetadata;
```

__Global Shared Memory Array:__

```c
OpMetadata *OpMetadataBackendArray;  // Sized by MaxBackends
```

__Linked with:__

```c
PgBackendStatus *BackendStatusArray;  // PostgreSQL's existing array
```

Both arrays use the same backend index (0 to MaxBackends-1) for 1:1 correspondence.

More parameters to be added.

- Stores ONLY data not available in PostgreSQL's BackendStatusArray

- All process information (PID, timestamps, state, opId) accessed from existing infrastructure

- Fixed-size structure for efficient shared memory access

- Raw BSON storage with 1KB limit matching mongoDB



__Synchronized Dual-Array Design:__

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

__Write Algorithm (Registration):__

```javascript
1. Extract metadata from command BSON:
   - Parse lsid.id UUID if present
   - Format as hex string
   - Copy raw BSON (truncate to 1KB if needed)

2. Protected write using BackendStatusArray's changecount:
   beentry = &BackendStatusArray[MyProcNumber]
   
   PGSTAT_BEGIN_WRITE_ACTIVITY(beentry)
     ├─ changecount++ (ODD)
     └─ pg_write_barrier()
   
   memcpy(OpMetadataBackendArray[MyProcNumber], metadata, 1156 bytes)
   
   PGSTAT_END_WRITE_ACTIVITY(beentry)
     ├─ pg_write_barrier()  
     └─ changecount++ (EVEN)
```

__Read Algorithm (currentOp Retrieval):__

```javascript
1. Query pg_stat_activity for active backends

2. For each backend:
   - Check state == "active" (skip if not)
   - Parse backend index from opId
   - Validate backend exists (st_procpid > 0)

3. Safe concurrent read with changecount protocol:
   beentry = &BackendStatusArray[index]
   
   loop:
     before = beentry->st_changecount
     pg_read_barrier()
     
     localCopy = OpMetadataBackendArray[index]  // Copy to local memory
     
     pg_read_barrier()
     after = beentry->st_changecount
     
     if (before == after && before is EVEN):
       break  // Valid read
     retry  // Saw concurrent write
   
   use localCopy (not direct pointer)
```

__Index Correlation Algorithm:__

```javascript
Backend index serves as universal key:
  
Registration:
  MyProcNumber → use as array index → write to OpMetadataBackendArray[index]
  
Retrieval:  
  opId from SQL → parse index → read OpMetadataBackendArray[index]
```

### Architecture Patterns

__1. Per-Backend Slot Allocation__

- One fixed slot per backend in shared memory
- Indexed by backend number (ProcNumber or BackendId-1)

__2. Concurrency Control__

Uses PostgreSQL's __changecount protocol__ - a lock-free technique where:

- Counter is ODD during updates (signals "in progress")
- Counter is EVEN when stable (signals "safe to read")
- Readers check counter before/after reading; retry if it changed or was ODD
- Memory barriers ensure proper ordering on all CPU architectures
- Single counter (`BackendStatusArray[index].st_changecount`) protects both arrays simultaneously

This avoids locks while preventing torn reads of the 1KB+ structure.


__3. Index-Based Array Linking__

- Backend index links OpMetadataBackendArray and BackendStatusArray
- opId encodes backend index for O(1) retrieval

__5. Implicit Lifecycle Management__

- No explicit cleanup required
- Slot overwritten on next request
- Backend termination handled by PostgreSQL



### API Changes

The currentOp command will have a number of new parameters added. For now only the lsid has been implemented to furhter performance impact testing.

### Configuration Changes

A new GUC parameter will be added to enable or disable this feature for utilizing the shared memory space. The default value will be false.

### Testing Strategy

The testing will determine the performance impact of the new overhead involved with storing and retrieving metadata in the shared memory. This will include unit tests for the metadata storage and retrieval functions, integration tests to verify currentOp command compatibility with MongoDB behavior, and performance tests to measure the overhead under various load conditions. Additionally, compatibility tests will ensure that the new fields match MongoDB's field semantics and format requirements.

### Documentation Updates

Documentation that will need to be updated includes the addition of new currentOp parameter support and usage examples.

---

<!--## Implementation Tracking

**Purpose:** Track the implementation progress of this RFC.

**Complete this section when:** Your RFC has been accepted and implementation work begins.

**Guidance:**
- Link to the PRs that implement this RFC. Update as implementation progresses.
- Provide success metrics.

### Implementation PRs

- [ ] PR #XXX: [Brief description of what this PR implements]
- [ ] PR #XXX: [Brief description of what this PR implements]
- [ ] PR #XXX: [Brief description of what this PR implements]

### Status Updates

*Add dated status updates as implementation progresses*

**YYYY-MM-DD:** Initial implementation started in PR #XXX

**YYYY-MM-DD:** [Update on progress, blockers, or changes]

### Open Questions

*Track unresolved questions that arise during implementation*

- [ ] Question: [Description]
  - Discussion: [Link to discussion or resolution]

### Implementation Notes

*Capture important decisions or learnings during implementation*

- **Decision [YYYY-MM-DD]:** [What was decided]
  - **Context:** [Why this decision was made]
  - **Alternatives:** [What else was considered]
--!>