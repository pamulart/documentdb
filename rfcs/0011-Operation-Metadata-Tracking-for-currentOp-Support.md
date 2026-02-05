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

*This section MAY BE REQUIRED before moving from Proposed to Accepted status. This section MUST be completed and approved to move to Implementing status.*

**Purpose:** Provide comprehensive technical details needed for implementation.

**Complete this section when:** Your solution approach has been validated and you're ready to commit to specific implementation details.

**Guidance:** This is where you get specific. Include enough detail that someone could implement this RFC without having to make major design decisions.

### Technical Details

*Describe the technical implementation specifics*
- Data structures
- Algorithms
- Architecture patterns
- Performance considerations

The currentOp metadata will be stored in the existing shared memory space in an opmetadata backend array, similar to an already existing design. 

We will populate the metadata at the beginning of each request once we have entered the extension and clean up our metadata once we have fulfilled our operation.

### API Changes

The currentOp command will have a number of new parameters added. List of currently proposed parameters (more will be added in future iterations):

* `client`
* `clientMetadata`
* `desc`
* `effectiveUsers`
* `killPending`
* `lsid`
* `microsecs_running`
* `cursor`

### Configuration Changes

A new GUC (Grand Unified Configuration) parameter will be added to enable or disable this feature for utilizing the shared memory space. The default value will be disabled (false).

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
-->
