
The `agent-loop.ts` module (`packages/agent/src/agent-loop.ts`) implements
the core multi-turn execution engine for the pi agent runtime. It drives
the conversational cycle between LLM streaming, tool execution, and
runtime message interception.

## Summary of Functionality

- **Native Message Handling**: Operates internally on `AgentMessage[]`
  records and transforms them to LLM `Message[]` only at the provider
  streaming boundary.
- **Dynamic Tool Synchronization**: Automatically reconciles executable
  tools in `AgentContext.tools` with declarations in system messages
  via `declareToolChanges`, emitting delta updates (`toolsAdded` /
  `toolsRemoved`).
- **Two-Tier Turn Engine**:
  - *Inner Loop*: Processes steering messages, prepares next turn state
    (`prepareNextTurn`), adjusts request options (`prepareRequest`),
    streams assistant responses, executes tool calls, and handles turn
    continuation decisions (`finishTurn`).
  - *Outer Loop*: Catches loop completion and polls for queued
    follow-up messages to initiate subsequent turn sequences.
- **Tool Execution Engine**:
  - Supports parallel and sequential execution modes.
  - Automatically switches to sequential execution if any invoked tool
    declares `executionMode: "sequential"`.
  - Supports `beforeToolCall` hooks to validate, mutate, or block calls.
  - Supports `afterToolCall` hooks to inspect or alter tool results.
  - Supports real-time partial tool results (`tool_execution_update`).
  - Truncation safety: if the LLM output stops due to token limits
    (`stopReason === "length"`), tool calls are failed immediately to
    prevent executing truncated arguments.
- **Event Streaming API**: Provides both callback-driven functions
  (`runAgentLoop`, `runAgentLoopContinue`) and event-stream wrappers
  (`agentLoop`, `agentLoopContinue`).

## Main Types and Configurations

- `AgentEventSink`: Callback `(event: AgentEvent) => Promise<void> | void`
  used to receive lifecycle events.
- `AgentContext`: Current transcript messages and executable tool
  definitions:
  - `messages: AgentMessage[]`
  - `tools?: AgentTool<any>[]`
- `AgentLoopConfig`: Runtime configuration controlling loop behavior:
  - `model`: Target LLM model configuration.
  - `reasoning`: Optional thinking level string.
  - `toolExecution`: Mode (`"parallel"` or `"sequential"`).
  - `convertToLlm`: Function converting `AgentMessage[]` to `Message[]`.
  - `transformContext`: Optional context transformer hook.
  - `getSteeringMessages`: Polls inter-turn steering messages.
  - `getFollowUpMessages`: Polls post-turn follow-up messages.
  - `prepareNextTurn`: Prepares context and model settings before turn.
  - `prepareRequest`: Finalizes request settings before LLM stream.
  - `finishTurn`: Decides whether to continue or terminate the loop.
  - `beforeToolCall` / `afterToolCall`: Tool interception hooks.

## Main Functions

- `runAgentLoop(prompts, context, config, emit, signal, streamFn)`:
  Asynchronously executes a new conversation loop with given prompt
  messages. Appends prompts to context, runs the turn loop, and resolves
  with newly generated messages.
- `runAgentLoopContinue(context, config, emit, signal, streamFn)`:
  Asynchronously continues the loop from the current context without
  adding a new prompt. Requires the last message in context to be a user
  or tool result.
- `agentLoop(prompts, context, config, signal, streamFn)`:
  Wrapper around `runAgentLoop` that returns an `EventStream<AgentEvent,
  AgentMessage[]>`.
- `agentLoopContinue(context, config, signal, streamFn)`:
  Wrapper around `runAgentLoopContinue` returning an `EventStream`.

## Example

```typescript
import {
  runAgentLoop,
  getDefaultStreamFn,
} from "@earendil-works/pi-agent";
import type {
  AgentContext,
  AgentLoopConfig,
  AgentMessage,
} from "@earendil-works/pi-agent";
import type { Model } from "@earendil-works/pi-ai";

const model: Model<any> = {
  id: "claude-3-7-sonnet",
  name: "Claude 3.7 Sonnet",
  api: "anthropic-messages",
  provider: "anthropic",
  baseUrl: "https://api.anthropic.com",
  reasoning: false,
  input: ["text"],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 200000,
  maxTokens: 4096,
};

const context: AgentContext = {
  messages: [
    {
      role: "system",
      content: "You are a helpful coding assistant.",
      timestamp: Date.now(),
    },
  ],
  tools: [],
};

const config: AgentLoopConfig = {
  model,
  toolExecution: "parallel",
  convertToLlm: (msgs) =>
    msgs.filter((m) =>
      ["system", "user", "assistant", "toolResult"].includes(m.role),
    ) as any,
  getApiKey: () => process.env.ANTHROPIC_API_KEY,
};

const promptMessage: AgentMessage = {
  role: "user",
  content: [{ type: "text", text: "Explain binary search." }],
  timestamp: Date.now(),
};

const abortController = new AbortController();

const newMessages = await runAgentLoop(
  [promptMessage],
  context,
  config,
  async (event) => {
    if (event.type === "message_update") {
      process.stdout.write(".");
    }
  },
  abortController.signal,
  getDefaultStreamFn(),
);

console.log("Generated messages:", newMessages.length);
```
