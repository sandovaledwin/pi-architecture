# AgentHarness Lifecycle & Instantiation

`AgentHarness` (`@earendil-works/pi-agent-core`) is the multi-lane, durable
runtime supervisor. It orchestrates shared resources ([[session]] storage,
`Models`, `ExecutionEnv`, hooks, and telemetry) across isolated execution
lines (`AgentLane`).

## Architecture Diagram

```
===========================================================================
                       AgentHarness RUNTIME MAP
===========================================================================

                  Daemon / Worker Process Bootstrapping
                                   │
                                   ▼
             AgentHarness.create(harnessOptions, context)
            ┌──────────────────────┴──────────────────────┐
            ▼                                             ▼
  [Session Restoration]                         [Inventory Discovery]
  • Attaches storage backend                     • Restores existing lanes
  • Bounded recovery mutation                    • Discovers open ops
            │                                             │
            └──────────────────────┬──────────────────────┘
                                   │
                                   ▼
+─────────────────────────────────────────────────────────────────────────+
|                       ACTIVE AgentHarness INSTANCE                      |
|                                                                         |
|  Shared Resources:                                                      |
|   • session: Durable storage (SQLite / JSONL)                           |
|   • models: LLM provider registry and auth resolution                   |
|   • hooks: Cross-lane lifecycle filters & triggers                      |
|   • events: Global multi-lane event emitter                             |
|                                                                         |
|  Lane Management:                                                       |
|   • harness.lane("main"): Atomic get-or-create execution line           |
|   • harness.watchSession(): Cross-lane transcript observation           |
|   • harness.getStats(): Multi-lane token usage aggregation              |
+──────────────────────────────────┬──────────────────────────────────────+
                                   │
                                   ▼
                  Shutdown / Teardown Boundary
                    [await harness.close(context)]
  • Closes all child lanes cleanly
  • Flushes outstanding commits & releases file locks
```

## Who Calls `AgentHarness`

1. **Background Session Workers**:
   - `coding-agent/src/experimental/session-worker.ts`: Daemon workers
     invoke `AgentHarness.create(...)` to manage workspace sessions
     backed by durable SQLite or JSONL storage.
2. **Headless Worker Services**:
   - `coding-agent/src/experimental/mini/worker/run.ts`: Minimal
     processes running headless agent turns without a local UI.
3. **RPC Facet Providers**:
   - `coding-agent/src/experimental/services/worker.ts`: Expose harness
     capabilities (lanes, model updates, watch) over RPC.
4. **Harness Conformance & Runtime Test Suites**:
   - Integration tests in `packages/agent/test/harness/` that verify
     multi-lane concurrency, crash recovery, and durability.

## When `AgentHarness` Is Called

### 1. Bootstrapping & Crash Recovery
- **Invocation**: `AgentHarness.create(options, context)`
- **When**: Called when a worker daemon or server process boots up.
- **Action**: Connects to the persistent storage backend, inventories
  existing durable lanes, and recovers any interrupted `open` operations
  without automatically driving them.

### 2. Lane Acquisition
- **Invocation**: `harness.lane(name, options?, context)`.
- Review [[runtime]] lane build process.
- **When**: Called when a client, subagent, or user initiates a task.
- **Action**: Atomically gets an existing lane or initializes a new
  durable branch execution line.

### 3. Global Session Observation & Metrics
- **Invocation**: `harness.watchSession()` and `harness.getStats()`
- **When**: Called by dashboards, UI monitors, or telemetry exporters.
- **Action**: Subscribes to cross-lane events and computes aggregate
  token consumption across all lanes in the session.

### 4. Cross-Lane Hook Registration
- **Invocation**: `harness.hooks.on(hookName, handler)`
- **When**: During service setup before admitting tasks.
- **Action**: Configures global policies for tool execution, compaction
  rules, and telemetry spans across all lanes.

### 5. Teardown & Clean Closure
- **Invocation**: `await harness.close(context)`
- **When**: On server shutdown, SIGTERM reception, or worker exit.
- **Action**: Closes all child lanes, cancels pending operations, and
  flushes pending commits to disk.

## Instantiation Example

```typescript
import {
  AgentHarness,
  BACKGROUND_CONTEXT,
} from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";
import {
  anthropicProvider,
} from "@earendil-works/pi-ai/providers/anthropic";

// 1. Initialize provider models
const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-3-7-sonnet")!;

// 2. Create the harness bound to a persistent session
const { harness, open } = await AgentHarness.create(
  {
    session, // Storage-backed Session instance
    models,
    model,
  },
  BACKGROUND_CONTEXT,
);

// 3. Acquire or create a serialized execution lane
const lane = await harness.lane("main", {}, BACKGROUND_CONTEXT);

// 4. Submit an operation to the lane
await lane.accept(
  { kind: "prompt", prompt: "Run workspace setup" },
  BACKGROUND_CONTEXT,
);
```
