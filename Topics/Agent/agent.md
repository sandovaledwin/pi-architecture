
The `Agent` class (`packages/agent/src/agent.ts`) is a stateful wrapper
around the lower-level agent execution loop (`runAgentLoop` and
`runAgentLoopContinue`). It coordinates conversation state, event
dispatching, tool execution, and mid-run message queuing.

## Summary of Functionality

- **Conversation State**: Manages transcript messages (`AgentMessage[]`),
  active tools, model configurations, thinking budgets, and system
  prompt derivation.
- **Run Lifecycle**: Ensures a single active run at any given time.
  Tracks run promises, abort controllers, streaming indicators, and
  pending tool call IDs.
- **Queue System**: Maintains two distinct input queues:
  - *Steering Queue*: Injects messages between turn iterations.
  - *Follow-Up Queue*: Enqueues messages to execute after the agent
    would otherwise stop.
- **Event Dispatch**: Synchronously updates internal state on events
  (`message_start`, `message_update`, `message_end`, `tool_execution_*`,
  `turn_end`, `agent_end`) and invokes listeners in subscription order.
- **Loop Configuration & Hooks**: Exposes lifecycle hooks for
  customizing prompts, LLM request conversion, context transformations,
  turn continuation decisions, and tool execution interception.

## Main Properties

- `state`: Read-only getter returning the current `AgentState` snapshot
  (`messages`, `systemPrompt`, `tools`, `model`, `thinkingLevel`,
  `isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage`).
- `steeringMode`: Queue mode (`"one-at-a-time"` | `"all"`) determining
  how queued steering messages are drained.
- `followUpMode`: Queue mode (`"one-at-a-time"` | `"all"`) determining
  how queued follow-up messages are drained.
- `signal`: Getter for the current run's `AbortSignal`, if active.
- `toolExecution`: Execution mode for multiple tool calls in a turn
  (`"parallel"` or `"sequential"`).
- `transport`: Transport mode passed to LLM stream (`"auto"`, `"sse"`,
  or `"websocket"`).
- `sessionId`: Optional session identifier passed to provider backends.
- `thinkingBudgets`: Optional token budgets per reasoning level.
- `maxRetryDelayMs`: Maximum delay cap for provider retries.
- `streamFunction`: The underlying streaming function (`StreamFn`).
- **Hook Properties**:
  - `convertToLlm`: Formats `AgentMessage[]` into provider messages.
  - `transformContext`: Async context filter/transformer before runs.
  - `beforeToolCall`: Intercepts or overrides tool calls.
  - `afterToolCall`: Intercepts or overrides tool results.
  - `finishTurn`: Custom logic to continue or end a turn.
  - `prepareRequest`: Customizes the LLM call options per request.
  - `prepareNextTurn` / `prepareNextTurnWithContext`: Dynamically
    updates model, tools, or system prompt before the next turn.

## Main Methods

- `subscribe(listener)`: Registers an event listener that receives
  `AgentEvent` and `AbortSignal`. Returns an unsubscribe function.
- `prompt(input, images?)`: Starts an execution run using text, image
  content, or `AgentMessage` objects.
- `continue()`: Resumes the run from the existing transcript (last
  message must be user or tool result) or drains pending queues.
- `steer(message)`: Queues a message to be injected between turns.
- `followUp(message)`: Queues a message to run after the agent loop
  finishes its current task.
- `abort()`: Aborts the active run controller.
- `waitForIdle()`: Returns a promise that resolves when the current run
  and all awaited subscriber handlers settle.
- `reset()`: Resets transcript back to the baseline system prompt and
  clears pending queues.
- `clearSteeringQueue()` / `clearFollowUpQueue()` / `clearAllQueues()`:
  Clears messages from the respective queues.
- `hasQueuedMessages()`: Checks if either queue has pending messages.
- `peekQueuedMessages()`: Returns pending messages without draining.

## Example

```typescript
import { Agent } from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";
import {
	anthropicProvider,
} from "@earendil-works/pi-ai/providers/anthropic";

// 1. Initialize models and agent
const models = createModels();
models.setProvider(anthropicProvider());

const model = models.getModel("anthropic", "claude-sonnet-4-6");

if (!model) throw new Error("Model not found");

const agent = new Agent({
	initialState: {
		systemPrompt: "You are an automated code assistant.",
		model,
	},
	streamFn: models.streamSimple.bind(models),

	// Process one follow-up per turn (default) or drain all ("all")
	followUpMode: "one-at-a-time",
});

// 2. Subscribe to events (stream output to console)
const unsubscribe = agent.subscribe((event) => {
	switch (event.type) {
		case "turn_start":
			console.log("\n--- Turn Start ---");
			break;
		case "message_update":
			if (event.assistantMessageEvent.type === "text_delta") {
				process.stdout.write(event.assistantMessageEvent.delta);
			}
			break;
		case "turn_end":
			console.log("\n--- Turn End ---");
			break;

		case "agent_end":
			console.log("\n--- Agent Run Finished ---");
			break;
	}
});

// 3. Start the initial prompt (runs asynchronously)
const runPromise = agent.prompt("Analyze package.json and summarize it.");

// 4. Queue a follow-up task while the first task is running
// Executes right after the summary completes without interrupting turn.
agent.followUp({
	role: "user",
	content: [
		{ 
			type: "text", 
			text: "Now check if there are any outdated peer dependencies."
		}
	],
	timestamp: Date.now(),
});

// 5. Wait for the agent to finish all work (prompt + followUp)
await runPromise;
  
// 6. Cleanup subscription when done
unsubscribe();
```

## Execution Flow: `prompt()` to `runAgentLoop()`

### Call Hierarchy

```
agent.prompt(input, images?)
  │
  ├── 1. Concurrency Check (rejects if activeRun is present)
  ├── 2. normalizePromptInput(input, images) -> AgentMessage[]
  └── 3. runPromptMessages(messages)
            │
            └── runWithLifecycle(async (signal) => { ... })
                  │
                  ├── Sets activeRun { promise, resolve, abortController }
                  ├── Sets _state.isStreaming = true
                  │
                  └── Calls runAgentLoop(
                        messages,
                        createContextSnapshot(),
                        createLoopConfig(options),
                        (event) => processEvents(event),
                        signal,
                        streamFunction
                      )
```

### 1. `prompt(input, images?)`

- **Concurrency Guard**: Verifies `!this.activeRun`. If a run is in
  progress, it throws an error directing callers to `steer()`,
  `followUp()`, or `waitForIdle()`.
- **Input Normalization**: `normalizePromptInput` converts strings and
  image arrays into structured `AgentMessage[]` user entries with
  timestamps. If `input` is already `AgentMessage` or `AgentMessage[]`,
  it preserves the structure.
- **Delegation**: Calls `this.runPromptMessages(messages)`.

### 2. How `runPromptMessages` Works

`runPromptMessages` binds the `Agent`'s state machine to the lower-level
functional loop:

1. **Lifecycle Management (`runWithLifecycle`)**:
   - Creates an `AbortController` and a settlement promise stored in
     `this.activeRun`.
   - Sets `_state.isStreaming = true`.
   - On unhandled failure or cancellation, `handleRunFailure` creates a
     synthetic terminal error assistant message and emits `turn_end`
     and `agent_end`.
   - In the `finally` block, `finishRun()` resets flags, clears pending
     tool calls, resolves the run promise, and clears `this.activeRun`.

2. **Context Snapshot (`createContextSnapshot`)**:
   - Creates a shallow copy of messages (`_state.messages.slice()`)
     and tools (`_state.tools.slice()`) so the loop operates on an
     isolated state snapshot.

3. **Loop Configuration (`createLoopConfig`)**:
   - Maps runtime settings (model, reasoning level, thinking budgets,
     tool execution mode, transports).
   - Wires up queue closures:
     - `getSteeringMessages`: drains `this.steeringQueue`.
     - `getFollowUpMessages`: drains `this.followUpQueue`.
   - Forwards hook callbacks (`prepareRequest`, `prepareNextTurn`,
     `beforeToolCall`, `afterToolCall`, `finishTurn`).

4. **Event Bridge (`processEvents`)**:
   - Bridges `runAgentLoop` events to internal state:
     - `message_start` / `message_update`: tracks `streamingMessage`.
     - `message_end`: clears `streamingMessage` and pushes the final
       message to `_state.messages`.
     - `tool_execution_start` / `tool_execution_end`: adds or removes
       tool IDs from `_state.pendingToolCalls`.
     - `turn_end`: captures assistant error messages if present.
   - Dispatches events sequentially to registered `listeners`, awaiting
     each subscriber handler before returning to `runAgentLoop`.

## Event Processing & State Reduction (`processEvents`)

The private `processEvents(event)` method acts as the state reducer and
event dispatcher within the `Agent` class. It serves two functions:

1. **Internal State Synchronization**: Mutates `_state` in real time
   based on lifecycle events received from `runAgentLoop`.
2. **Sequential Subscriber Dispatch**: Forwards events to external
   listeners registered via `subscribe()` and awaits them.

### State Reduction per Event Type

- `message_start`:
  Sets `_state.streamingMessage = event.message` to track the incoming
  message shell before text or tool chunks arrive.
- `message_update`:
  Updates `_state.streamingMessage = event.message` with live deltas
  (text tokens, reasoning tokens, or tool call arguments).
- `message_end`:
  Clears `_state.streamingMessage = undefined` and appends the final
  message to `_state.messages` (transcript commit).
- `tool_execution_start`:
  Instantiates a new `Set` containing `event.toolCallId` and assigns it
  to `_state.pendingToolCalls` (immutable set replacement).
- `tool_execution_end`:
  Instantiates a new `Set` without `event.toolCallId` and updates
  `_state.pendingToolCalls`.
- `turn_end`:
  If the assistant turn finished with an error, copies the error text
  to `_state.errorMessage`.
- `agent_end`:
  Safety reset that ensures `_state.streamingMessage` is cleared.

### Listener Dispatch & Backpressure

After updating `_state`, `processEvents` dispatches the event:

```typescript
const signal = this.activeRun?.abortController.signal;
if (!signal) {
  throw new Error("Agent listener invoked outside active run");
}
for (const listener of this.listeners) {
  await listener(event, signal);
}
```

- **Sequential Ordering**: Listeners run one at a time in subscription
  order.
- **Backpressure Barrier**: Because `await listener(...)` is awaited
  inside `processEvents`, `runAgentLoop` pauses until all listeners
  finish. This guarantees UI updates, storage writes, or logging
  settle before subsequent turns or tool executions start.
- **Cancellation Propagation**: Forwards the run's `AbortSignal` to
  subscribers to allow async handlers to abort cooperatively.

