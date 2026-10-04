# Harness Execution Subsystem (`harness/execution`)

The `harness/execution` directory encapsulates side-effecting operations
within the agent harness. It manages LLM streaming calls, tool
preflighting and execution, and cooperative cancellation gates.

## Architecture Diagram

```
===========================================================================
                     HARNESS EXECUTION SUBSYSTEM
===========================================================================

                              AgentLane / Drive
                                     │
                 ┌───────────────────┼───────────────────┐
                 ▼                   ▼                   ▼
          +─────────────+     +─────────────+     +─────────────+
          | effect-gate |     |  assistant  |     |    tools    |
          +──────┬──────+     +──────┬──────+     +──────┬──────+
                 │                   │                   │
                 ▼                   ▼                   ▼
        [Gate & GateControl]  [LLM Streaming]     [Tool Lifecycle]
         • admit(effect)       • Context prep      • prepareToolCall
         • beginAbort()        • Stream observer   • beforeToolDecision
         • AbortRequested      • Settled message   • executeToolCall
                                                   • finalizeToolCall
                                                         │
                                                         ▼
                                                [ExecutionEnv / Tools]
```

## Module Overview

### 1. `effect-gate.ts` (Admission & Cancellation Barrier)

Manages synchronous effect admission and cooperative cancellation for
drive passes.

- **`createGate()`**: Creates paired `gate` (for executing operations)
  and `control` (for lifecycle owners) handles.
- **`Gate`**:
  - `signal`: Exposes the pass's `AbortSignal`.
  - `admit<T>(invoke: () => T)`: Guard that checks gate status before
    executing a side effect. Throws `AbortRequested` if cancellation has
    begun, or a terminal error if the gate is closed.
- **`GateControl`**:
  - `beginAbort(cancellation)`: Transitions status to `"aborting"` and
    registers the pending cancellation promise.
  - `signalAbort()`: Aborts the underlying `AbortController` with
    `AbortRequested`.
  - `close(error)`: Closes the gate permanently with a failure.
- **`AbortRequested`**: Special control flow error thrown when
  cancellation preempts or interrupts an effect.

### 2. `assistant.ts` (LLM Generation & Stream Consumption)

Coordinates prompt context transformation, LLM streaming, and observer
event propagation.

- **`streamHarnessAssistant(messages, config, context)`**:
  - Applies optional `transformContext` to message history.
  - Converts messages to provider format via `toProviderMessages`.
  - Captures early HTTP status and headers before the stream completes.
  - Invokes `beforePayload` hook if configured.
  - Consumes the stream and returns a `SettledAssistantMessage`.
- **`consumeAssistantStream(stream, observer, afterResponse, context)`**:
  - Iterates over `AssistantMessageEventStream`.
  - Dispatches `observer.start()`, `observer.update()`, and
    `observer.end()` events.
  - Runs the `afterResponse` hook while safely handling `AbortRequested`
    cancellations.

### 3. `tools.ts` (Four-Stage Tool Execution Pipeline)

Drives tool preflight, validation, execution, and result normalization.

1. **Preparation (`prepareToolCall`)**:
   - Resolves tool by name from available `AgentHarnessTool` definitions.
   - Executes tool-specific `prepareArguments` if defined.
   - Validates arguments against schema (`validateToolArguments`).
   - Returns `PreparedToolCall` or synthetic `ImmediateToolOutcome`
     (if missing or invalid).
2. **Hook Filtering (`applyBeforeToolDecision`)**:
   - Applies `beforeToolCall` hook decisions:
     - Mutated arguments: re-validates new parameters.
     - Blocked: produces an immediate error outcome.
   - Produces `ClearedToolCall`.
3. **Execution (`executeToolCall`)**:
   - Runs inside `gate.admit(...)` with `withAbortSignal`.
   - Invokes `tool.execute(callId, args, onUpdate, ...)`.
   - Routes real-time progress through `onUpdate`.
   - Catches unexpected throws into standard error tool results.
4. **Finalization (`finalizeToolCall`)**:
   - Applies `AfterToolPatch` (patches content, details, usage, errors).
   - Produces `FinalizedToolCall`.
5. **Message Conversion**:
   - `createToolResultMessage`: Converts `FinalizedToolCall` to a
     provider-facing `role: "toolResult"` message.
   - `toolResultFromMessage`: Reconstructs canonical `AgentToolResult`
     from transcript messages during replay.

## Who Calls `harness/execution` and When

### 1. `effect-gate.ts`
- **Who**: `AgentLane` (`lane.ts`), runtime driver (`drive.ts`), and
  harness hooks (`hooks.ts`).
- **When**:
  - `createGate()` is called at the start of each drive pass.
  - `gate.admit()` is called before initiating any external effect
    (assistant generation or tool execution).
  - `control.beginAbort()` / `signalAbort()` are called when a user or
    lane aborts an active run.

### 2. `assistant.ts`
- **Who**: Generation driver (`drive/generation.ts`) and deferred stream
  driver (`drive/deferred.ts`).
- **When**:
  - Called during prompt or continuation turns when the agent requires a
    new response from the configured LLM model.
  - Called when resuming a suspended or deferred streaming response.

### 3. `tools.ts`
- **Who**: Tool execution driver (`drive/tools.ts`) and `AgentLane`
  (`lane.ts`).
- **When**:
  - Called immediately after an assistant message returns one or more
    `toolCall` blocks.
  - Preflights and executes tool calls (sequentially or in parallel)
    before returning tool results to the model for the next turn.
  - Called by `lane.ts` when reconstructing tool results from staged
    messages in storage.
