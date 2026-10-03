# Harness Tools

The `harness/tools` directory (`packages/agent/src/harness/tools`)
provides built-in tools for agent execution environments. Unlike core
`AgentTool` definitions, these tools are implemented as
`AgentHarnessTool` instances that require an `ExecutionEnv` (such as
`NodeExecutionEnv`) to access the filesystem and shell.

## Architecture Diagram

```
===========================================================================
                      HARNESS TOOLS ARCHITECTURE
===========================================================================

                              AgentHarness
                                   │
                   Provides ExecutionToolContext { env }
                                   │
                                   ▼
+─────────────────────────────────────────────────────────────────────────+
|                         BUILT-IN HARNESS TOOLS                          |
|                                                                         |
|  +──────────────+ +──────────────+ +──────────────+ +────────────+  |
|  |     read     | |    write     | |     edit     | |    bash    |  |
|  |createReadTool| |  createWrite | |createEditTool| | createBash |  |
|  +──────┬───────+ +──────┬───────+ +──────┬───────+ +─────┬──────+  |
+──────────┼─────────────────┼─────────────────┼────────────────┼─────────+
           │                 │                 │                │
           │                 ▼                 ▼                │
           │       ┌───────────────────────────────────┐        │
           │       │      file-mutation-queue.ts       │        │
           │       │      withFileMutationQueue()      │        │
           │       │  (Serializes concurrent file      │        │
           │       │   mutations per canonical path)   │        │
           │       └─────────────────┬─────────────────┘        │
           │                         │                          │
           ▼                         ▼                          ▼
+─────────────────────+   +─────────────────────+   +─────────────────────+
|    path-utils.ts    |   |    edit-diff.ts     |   |  output-capture.ts  |
| • resolveToolPath() |   | • Exact match check |   | • Stream updates    |
| • Path normalization|   | • LF/CRLF/BOM sync  |   | • 2s checkpoints    |
| • Image detection   |   | • Unified diff gen  |   | • Tail truncation   |
+─────────────────────+   +─────────────────────+   +─────────────────────+
           │                                                    │
           ▼                                                    ▼
+─────────────────────────────────────────────────────────────────────────+
|                       ExecutionEnv (harness/env)                        |
|                                                                         |
|  FileSystem Interface:                    Shell Interface:              |
|   • readFile / readBinaryFile              • exec (stdout/stderr stream)|
|   • writeFile                              • timeout & kill signals     |
|   • stat / canonicalPath                   • cwd & env variables        |
+─────────────────────────────────────────────────────────────────────────+
```

## Built-in Tools Summary

### 1. `read` (`createReadTool`)
- **Primary Function**: Reads file content from the filesystem.
- **Input Parameters**:
  - `path` (string, required): File path to read.
  - `offset` (number, optional): 1-indexed starting line number.
  - `limit` (number, optional): Maximum number of lines to return.
- **Key Features**:
  - **Head Truncation**: Enforces default limits of 2,000 lines or 50KB,
    truncating lines from the top if exceeded.
  - **Image Detection**: Inspects magic bytes via `image.ts` for PNG,
    JPEG, GIF, WebP, and BMP formats. If an image is detected, it reads
    the raw binary and attaches base64 data to the message.
  - **Image Processing**: Supports pluggable `imageProcessor` callbacks
    for automated resizing or format conversions.

### 2. `write` (`createWriteTool`)
- **Primary Function**: Writes or overwrites entire file contents.
- **Input Parameters**:
  - `path` (string, required): File path to write.
  - `content` (string, required): Content to write into the file.
- **Key Features**:
  - **Directory Creation**: Automatically ensures parent directories
    exist before writing.
  - **Concurrency Safety**: Wrapped in `withFileMutationQueue` to
    serialize concurrent writes or edits targeting the same canonical
    path.

### 3. `edit` (`createEditTool`)
- **Primary Function**: Performs targeted search-and-replace edits on
  existing files.
- **Input Parameters**:
  - `path` (string, required): File path to edit.
  - `edits` (array of `{ oldText, newText }` objects, required): One or
    more disjoint text replacements.
- **Key Features**:
  - **Strict Matching**: Every `oldText` must match exactly once in the
    target file to prevent unintended modifications.
  - **Non-Overlapping Edits**: Verifies that edits do not collide or
    overlap in the file buffer.
  - **Format Preservation**: Retains CRLF vs. LF line endings and
    preserves UTF-8 BOM encoding.
  - **Diff & Patch Output**: Generates unified diffs and patch metadata
    using `edit-diff.ts`.
  - **Concurrency Safety**: Wrapped in `withFileMutationQueue` to avoid
    concurrent write collisions.

### 4. `bash` (`createBashTool`)
- **Primary Function**: Executes shell commands within the execution
  environment's working directory (`env.cwd`).
- **Input Parameters**:
  - `command` (string, required): Shell command string.
  - `timeout` (number, optional): Command timeout in seconds.
- **Key Features**:
  - **Real-Time Streaming**: Emits live output updates via `onUpdate`
    with throttled 2-second checkpoints.
  - **Tail Truncation**: Caps captured output to the last 2,000 lines or
    50KB.
  - **Disk Spillover**: Automatically writes full, untruncated output to
    a temporary file on disk when limits are exceeded.
  - **Environment Injection**: Supports custom `commandPrefix` and
    async `prepare` hooks to configure environment variables.

## Supporting Modules

- **`file-mutation-queue.ts`**:
  Maintains a per-environment, per-canonical-path Promise chain. When
  multiple tool calls run in parallel, mutations to the same file are
  strictly serialized to prevent lost updates.
- **`edit-diff.ts`**:
  Contains diffing algorithms, line-ending detection/restoration, and
  unified patch formatting for the `edit` tool.
- **`path-utils.ts`**:
  Resolves relative paths to `env.cwd`, handles tilde (`~`) home
  directory expansion, and resolves canonical symlinks.
- **`image.ts`**:
  Detects binary image headers (PNG, JPEG, GIF, WebP, BMP) and encodes
  binary buffers into base64 image content blocks.
- **`tool-context.ts`**:
  Defines `ExecutionToolContext`, specifying `{ env: ExecutionEnv }` as
  the runtime context required by all harness tools.

### How to instantiate the tools in an Agent

 ```typescript
   import {
     Agent,
     createEditTool,
     createReadTool,
     createWriteTool,
     type AgentHarnessTool,
     type AgentTool,
     type ExecutionToolContext,
   } from "@earendil-works/pi-agent-core";
   import { NodeExecutionEnv } from "@earendil-works/pi-agent-core/node";
   import { createModels } from "@earendil-works/pi-ai";
   import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";

   // 1. Initialize the Node execution environment (filesystem/process boundary)
   const env = new NodeExecutionEnv(process.cwd());
   const toolContext: ExecutionToolContext = { env };

   // 2. Adapter: binds ExecutionEnv and Context into an AgentTool
   function adaptHarnessTool<T extends AgentHarnessTool<ExecutionToolContext, any, any>>(
     harnessTool: T,
   ): AgentTool {
     return {
       name: harnessTool.name,
       description: harnessTool.description,
       parameters: harnessTool.parameters,
       execute: (toolCallId, args, signal, onUpdate) => {
         return harnessTool.execute(
           toolCallId,
           args,
           onUpdate ?? (() => {}),
           toolContext,
           { toolCallId } as any,
           { abortSignal: signal } as any,
         );
       },
     };
   }

   // 3. Instantiate and adapt the harness tools
   const tools: AgentTool[] = [
     adaptHarnessTool(createReadTool()),
     adaptHarnessTool(createEditTool()),
     adaptHarnessTool(createWriteTool()),
   ];

   // 4. Set up provider and model
   const models = createModels();
   models.setProvider(anthropicProvider());
   const model = models.getModel("anthropic", "claude-3-7-sonnet")!;

   // 5. Instantiate the Agent with adapted tools
   const agent = new Agent({
     initialState: {
       systemPrompt: "You are a coding assistant with access to file tools.",
       model,
       tools,
     },
     streamFn: models.streamSimple.bind(models),
   });

   // 6. Subscribe to lifecycle and tool events
   const unsubscribe = agent.subscribe((event) => {
     if (event.type === "tool_execution_start") {
       console.log(`[Tool Start] ${event.toolName}(${JSON.stringify(event.args)})`);
     } else if (event.type === "tool_execution_end") {
       console.log(`[Tool End] ${event.toolName}`);
     } else if (event.type === "message_update") {
       if (event.assistantMessageEvent.type === "text_delta") {
         process.stdout.write(event.assistantMessageEvent.delta);
       }
     }
   });

   // 7. Execute prompt
   await agent.prompt("Read package.json and tell me the version number.");

   await agent.waitForIdle();
   unsubscribe();
 ```
