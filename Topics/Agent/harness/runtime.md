# Harness Runtime Subsystem (`harness/runtime`)

The `harness/runtime` directory contains the stateful, durable runtime
implementation of `AgentHarness` and `AgentLane`. It manages serialized
execution lines, state machine turn stepping, crash recovery, and client
state projection.

## Architecture Diagram

```
===========================================================================
                        HARNESS RUNTIME ARCHITECTURE
===========================================================================

                              createAgentHarness()
                                       │
                                       ▼
+─────────────────────────────────────────────────────────────────────────+
|                        Harness (class in harness.ts)                    |
|                                                                         |
|  • Owns Session storage, Models, HookRegistry, and HarnessEventBus      |
|  • restoreSession (restore.ts): inventories lanes & open operations     |
|  • lanesByName: Map<string, Lane>                                       |
+──────────────────────────────────────┬──────────────────────────────────+
                                       │
                         harness.lane(name)
                                       │
                                       ▼
+─────────────────────────────────────────────────────────────────────────+
|                          Lane (class in lane.ts)                        |
|                                                                         |
|  • Serialized mutation line: accept() -> admissions queue               |
|  • Operation controls: compact(), navigateTree(), requestAbort()        |
|  • Drives execution passes: lane.drive()                                |
+──────────────────────────────────────┬──────────────────────────────────+
                                       │
                       driveOperation() in drive.ts
                                       │
                                       ▼
+─────────────────────────────────────────────────────────────────────────+
|                      Drive State Machine (drive/)                       |
|                                                                         |
|   starting ──> checkpoint ──> assistant.ready ──> tools                 |
|      ▲                              │               │                   |
|      │ (retry / deferred)           ▼               ▼                   |
|   reconcile <── cancel_req    generation.ts      tools.ts               |
+──────────────────────────────────────┬──────────────────────────────────+
                                       │
                                       ▼
                         Emits HarnessEvent stream
                                       │
                                       ▼
+─────────────────────────────────────────────────────────────────────────+
|                    reduceLaneSnapshot() (reducer.ts)                    |
|  • Pure functional reducer: (snapshot, event) => LaneSnapshot           |
|  • Powers live TUI, client views, and remote RPC transcript sync        |
+─────────────────────────────────────────────────────────────────────────+
```

## Module Overview

### 1. `harness.ts` (`class Harness implements AgentHarness`)
- **Factory**: `createAgentHarness(options, context)` validates configs,
  invokes `restoreSession`, and returns `{ harness, open }`.
- **Lane Registry**: `harness.lane(name, ...)` provides atomic
  get-or-create for serialized `Lane` execution lines.
- **Shared Coordination**: Hosts `Session` storage bindings, provider
  `Models`, cross-lane `HookRegistry`, and `HarnessEventBus`.
- **Fault Handling**: Intercepts invariant violations and storage faults,
  transitioning the harness into a protected faulted state.

### 2. `lane.ts` (`class Lane implements AgentLane`)
- **Admission Queue**: `accept(request, context)` admits requests
  (`prompt`, `skill`, `compaction`, `navigation`) into a deterministic
  serialized FIFO queue.
- **Pass Driving**: `drive(options, context)` installs a pass owner and
  executes operations until settlement or a durable wait state.
- **Structural Operations**: Coordinates in-band compaction and
  branch tree navigation (`compact()`, `navigateTree()`).
- **Cooperative Cancellation**: `requestAbort(operationId, context)`
  signals running operations and commits cancellation records.
- **Live Inspection**: `inspectExecution(context)` returns real-time
  status of running tools, active retries, and streaming partials.

### 3. `drive.ts` and `drive/` (Turn State Machine)
Orchestrates turn-by-turn execution of an admitted operation through
specialized procedures:
- `checkpoint.ts`: Initializes turns and creates durable checkpoints.
- `generation.ts`: Prepares context and streams assistant LLM tokens.
- `response.ts`: Streams assistant response frames to storage.
- `tools.ts`: Drives parallel/sequential tool calls and patches results.
- `structural.ts`: Executes branch compaction and tree navigation.
- `deferred.ts`: Handles suspended operations awaiting external resumes.
- `recovery.ts`: Re-attaches to in-flight assistant generations on reboot.
- `reconcile.ts`: Handles cancelled or interrupted operations cleanly.
- `retry.ts`: Manages exponential backoff delays for transient errors.

### 4. `reducer.ts` (`reduceLaneSnapshot`)
- **Pure Functional Reducer**: Transforms incoming `HarnessEvent`s and a
  base `LaneSnapshot` into an updated snapshot.
- Tracks turn counts, running tools, streaming deltas, and settled
  messages without re-reading the storage backend.

### 5. `restore.ts`
- **Side-Effect-Free Restoration**: Reads raw `Session` entries and
  stored state values on startup.
- Inventories existing lanes and unfinished operations (`OpenOperation[]`)
  without executing them.

### 6. `transcript.ts`
- Reconstructs transcript messages and streams assistant frames from
  persisted storage.

## Who Calls `harness/runtime` and When

### 1. `createAgentHarness` (`harness.ts`)
- **Who**: Session worker daemons (`session-worker.ts`), headless mini
  workers (`mini/worker/run.ts`), and test suites.
- **When**: Called once at process startup to initialize the agent
  runtime and discover open operations from prior runs.

### 2. `Lane` (`lane.ts`)
- **Who**: `AgentHarness` and client services (RPC providers, TUI).
- **When**:
  - `harness.lane()` is called when a client attaches to a workspace.
  - `lane.accept()` is called when a prompt, skill, or compaction is
    requested.
  - `lane.drive()` is called to advance the admitted operation.
  - `lane.requestAbort()` is called when an operation is cancelled.

### 3. `driveOperation` (`drive.ts`)
- **Who**: `Lane.drive()`.
- **When**: Invoked continuously while an admitted operation is actively
  progressing through generation, tool execution, and turn commits.

### 4. `restoreSession` (`restore.ts`)
- **Who**: `createAgentHarness`.
- **When**: Invoked during harness construction to inspect durable
  session storage and reconstruct previous lane states.

### 5. `reduceLaneSnapshot` (`reducer.ts`)
- **Who**: Interactive TUI, remote RPC clients, and test harnesses.
- **When**: Invoked upon receiving each `HarnessEvent` to update client
  UI components and progress displays.
