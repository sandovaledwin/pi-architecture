# Session Storage Subsystem (`harness/session`)

The `harness/session` directory defines and implements the durable storage
model for the agent harness. It provides a tree-structured, append-only
persistence engine that models conversation histories, branching, and
runtime metadata.

## Architecture Diagram

```
===========================================================================
                       SESSION ARCHITECTURE MAP
===========================================================================

                              SessionRepo
                     (JsonlSessionRepo / Memory)
                                   │
                           open() / create()
                                   │
                                   ▼
+─────────────────────────────────────────────────────────────────────────+
|                         StorageBackedSession                            |
|                                                                         |
|  • Coordinates session tree, branches, metadata, and stats              |
|  • MutationLine: serializes write transactions (mutate())               |
|                                                                         |
|  +───────────────────────────────────+ +─────────────────────────────+  |
|  |            Branch Tree            | |       Key-Value Store       |  |
|  |  • Immutable DAG Entries:         | |  • Namespaced StoredValue   |  |
|  |    message, compaction, summary   | |  • laneState, laneConfig    |  |
|  |  • parentId linked history        | |  • branchTip, operationRes  |  |
|  +─────────────────┬─────────────────+ +──────────────┬──────────────+  |
+────────────────────┼──────────────────────────────────┼─────────────────+
                     │                                  │
                     ▼                                  ▼
+─────────────────────────────────────────────────────────────────────────+
|                           Storage Backend                               |
|                 (JsonlStorage / MemoryStorageState)                     |
|                                                                         |
|  • Atomic commits: prepareStorageCommit / commitBatch                   |
|  • Monotonic seq assignment and disk append serialization               |
+─────────────────────────────────────────────────────────────────────────+
```

## Module Overview

### 1. `types.ts` (Storage & Session Contracts)
- **`Session`**: Authoritative durable session interface. Exposes
  `mutate(callback)`, branch tree querying (`findEntries`, `getEntry`),
  and metadata operations.
- **`Entry`**: Immutable node in a tree-structured transcript. Types
  include `message`, `compaction`, `branch_summary`, and `custom`.
  Every entry carries `id`, `parentId`, monotonic `seq`, and `timestamp`.
- **`Branch`**: A named pointer to an entry ID (`tipId`), representing
  an active conversation path.
- **`SessionStorage`**: Backend storage abstraction providing atomic
  commit batches (`commit`), branch scans, and value queries.
- **`SessionRepo`**: Repository factory managing lifecycle and storage
  for multiple sessions (`open`, `create`, `list`, `fork`, `delete`).

### 2. `session.ts` (`StorageBackedSession`)
- Concrete implementation of `Session` backed by a pluggable `Storage`
  engine.
- **Mutation Line (`mutation-line.ts`)**: Enforces single-writer serial
  execution (`session.mutate()`) so concurrent lane commits never
  corrupt the underlying storage.
- Rejects invalid writes (e.g. attempting to persist an in-flight
  streaming message with `stopReason === "pending"`).

### 3. `values.ts` (Key-Value & List Registry)
Stores typed metadata alongside conversation entries in dedicated
namespaces:
- `branchTip(branchName)`: Maps branch names to their latest entry ID.
- `laneState(laneName)`: Persists lane runtime status and inbox state.
- `laneConfig(laneName)`: Stores model configurations and active tools.
- `operationResultValue(operationId)`: Stores settled turn outcomes.
- Streaming assistant frames and intermediate progress chunks.

### 4. `commit.ts` (Atomic Persistence Engine)
- Translates staged write operations (`EntryWrite`, `UsageWrite`,
  `ValueSetWrite`, `ListAppendWrite`) into committed records.
- Enforces parent-child DAG integrity and assigns monotonically
  increasing sequence numbers (`seq`).

### 5. `context.ts` (Context Message Reconstructor)
- `buildContextEntries`: Gathers entries along an active branch from
  root or latest compaction to tip.
- `sessionEntryToContextMessages`: Normalizes entries into standard
  `AgentMessage[]` arrays for LLM prompts.

### 6. Storage Implementations
- **`jsonl/` (`JsonlStorage`, `JsonlSessionRepo`)**:
  Production-grade append-only newline-delimited JSON storage backed by
  `ExecutionEnv` filesystem operations.
- **`memory.ts` (`MemorySessionRepo`)**:
  In-memory storage backend used for ephemeral sessions and test suites.

### 7. `fork.ts` & `fork-policy.ts` (Branching & Forking)
- Clones session trees across repositories with custom entry projection
  and fork policies (e.g., pruning history, cherry-picking branches).

## Who Calls `harness/session` and When

### 1. Who Calls It

- **`AgentHarness` (`harness.ts`)**:
  - Owns the root `Session` instance.
  - Calls `session.mutate()` during startup recovery (`restoreSession`)
    to read lane state without committing side effects.
  - Queries session statistics via `session.getStats()`.
- **`Lane` (`lane.ts`)**:
  - Executes all persistence writes inside `session.mutate()` closures.
  - Commits new messages, tool results, and lane status records.
- **Background Workers & Servers (`packages/coding-agent`)**:
  - Session workers (`session-worker.ts`, `server.ts`) invoke
    `repo.open()` or `repo.create()` on startup to obtain a `Session`.
- **Compaction & Navigation Subsystems**:
  - `compaction.ts` and `branch-summarization.ts` scan branch
    histories using `branch.findEntries()` to select turns.

### 2. When It Is Called

- **Session Initialization**:
  - `repo.open()` / `repo.create()` initializes storage files.
  - `restoreSession()` scans `laneState` and `branchTip` values to
    rebuild the runtime lane inventory.
- **Turn Settlement & Commits**:
  - After assistant streaming finishes or tool calls complete, the lane
    calls `mutator.commit(writes)` to append entries and advance the tip.
- **Compaction & Branch Summaries**:
  - Persists `CompactionEntry` and updates `branchTip` to point to the
    new compaction boundary.
- **Branch Tree Navigation**:
  - `session.createBranch()` or `session.branch()` is called when
    forking or switching conversation paths.
- **Session Teardown**:
  - `session.close()` flushes pending write streams, terminates the
    mutation line, and releases backend file handles.
