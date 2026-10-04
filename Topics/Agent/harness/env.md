# Execution Environment (`harness/env`)

The `harness/env` module defines and implements the environment
abstraction for agents. It provides a controlled interface to the host
operating system, isolating filesystem access and shell process
execution behind the `ExecutionEnv` interface.

## Architecture Diagram

```
===========================================================================
                      EXECUTION ENVIRONMENT (env)
===========================================================================

                              ExecutionEnv
                   ┌───────────────┴───────────────┐
                   ▼                               ▼
               FileSystem                        Shell
    • read / write / append           • exec (spawn process)
    • stat / list / exists            • streams stdout/stderr
    • canonicalPath / resolve         • timeout & abort signals
    • createTempDir / File            • cwd & env variables
                   ▲                               ▲
                   └───────────────┬───────────────┘
                                   │
                                   ▼
+─────────────────────────────────────────────────────────────────────────+
|                           NodeExecutionEnv                              |
|                    (Node.js OS & Process Runtime)                       |
|                                                                         |
|  +───────────────────────────────────+ +─────────────────────────────+  |
|  |     Node.js fs / fs/promises      | |     node:child_process      |  |
|  |  • Native async file I/O          | |  • spawn(command, {shell})  |  |
|  |  • Result<T, FileError> mapping   | |  • Graceful SIGTERM/SIGKILL |  |
|  |  • Temp dirs / files tracking     | |  • Timeout management       |  |
|  +───────────────────────────────────+ +──────────────┬──────────────+  |
|                                                       │                 |
|                                                       ▼                 |
|                                        +─────────────────────────────+  |
|                                        |    OutputCapture Bridge     |  |
|                                        |  • Head/Tail line buffer    |  |
|                                        |  • Byte & line limit caps   |  |
|                                        |  • Spillover to temp disk   |  |
|                                        +─────────────────────────────+  |
+─────────────────────────────────────────────────────────────────────────+
```

## Core Abstractions (`packages/agent/src/harness/types.ts`)

The execution environment is split into two composable capabilities:

1. **`FileSystem`**:
   - Manages non-throwing, `Result<T, FileError>`-based file and
     directory operations.
   - Normalizes path resolution, canonical path lookups, and temporary
     file/directory creation.
2. **`Shell`**:
   - Executes arbitrary terminal commands with environment variable
     inheritance, working directory overrides, timeouts, and cooperative
     abort cancellation.
   - Provides real-time bounded output capture and streaming updates.
3. **`ExecutionEnv`**:
   - Combines both interfaces (`interface ExecutionEnv extends
     FileSystem, Shell {}`).

## Implementation: `NodeExecutionEnv` (`nodejs.ts`)

`NodeExecutionEnv` is the concrete Node.js implementation of
`ExecutionEnv`.

### 1. Filesystem Operations
- **Path Resolution**: Resolves relative paths against `this.cwd`,
  expands `~` to `os.homedir()`, and normalizes `file://` URLs.
- **Error Mapping**: Translates raw Node.js `ErrnoException` errors
  into strongly-typed `FileError` codes (`not_found`,
  `permission_denied`, `already_exists`, `is_directory`, `aborted`).
- **File I/O**:
  - `readFile` & `readBinaryFile`: Reads text or raw byte buffers.
  - `writeFile` & `appendFile`: Writes content, creating parent folders
    when necessary.
  - `stat`, `exists`, `canonicalPath`: Queries file metadata and resolves
    symlinks via `fs.promises.realpath`.
  - `listDirectory`, `makeDirectory`, `remove`, `rename`: Full directory
    tree management.
- **Resource Tracking**: Tracks all temporary files and directories
  created via `createTempFile` and `createTempDir`.

### 2. Shell & Process Execution
- **Process Spawning**: Spawns processes using `node:child_process`
  with configurable environment variables and working directory.
- **Bounded Output Capture (`OutputCapture`)**:
  - Intercepts stdout and stderr streams.
  - Enforces `maxBytes` and `maxLines` limits with `"head"` or `"tail"`
    retention strategies.
  - Delivers real-time incremental output events (`replace`, `append`,
    `slide`, `metadata`) through the `onUpdate` callback.
- **Spillover to Disk**:
  - When output exceeds bounded memory buffers and `spill: true` is
    requested, streams complete raw output to an execution-local
    temporary file on disk.
  - Provides `spillPath` in output metadata so models and callers can
    inspect untruncated output.
- **Timeout & Cooperative Cancellation**:
  - Sets up timers to enforce execution timeouts.
  - Listens to `context.abortSignal` to kill processes gracefully
    (SIGTERM followed by SIGKILL if necessary).

### 3. Resource Cleanup
- `cleanup()`: Best-effort release method that terminates all lingering
  child processes, closes active spill write streams, and deletes
  registered temporary directories and files.

## Callers and Invocation Lifecycle

### 1. Who Calls `ExecutionEnv`

- **Built-in Harness Tools (`harness/tools/`)**:
  - `read`: Reads file contents and images via `readFile` and
    `readBinaryFile`.
  - `write`: Writes or creates files via `writeFile`.
  - `edit`: Reads existing files and applies multi-edits via `readFile`
    and `writeFile`.
  - `bash`: Executes shell commands via `exec`, captures bounded output,
    and manages temp spill files.
  - `file-mutation-queue`: Computes canonical path keys via
    `canonicalPath` to lock concurrent mutations.
- **Session Storage Engines (`harness/session/`)**:
  - `JsonlStorage` and `JsonlSessionRepo` call `FileSystem` methods to
    create, read, and append session `.jsonl` files and commit snapshots.
- **Skills & Prompt Templates Loaders (`skills.ts`,
  `prompt-templates.ts`)**:
  - Discover and load workspace files via `listDirectory`, `stat`, and
    `readFile`.
- **Session Workers & Servers (`packages/coding-agent`)**:
  - Instantiate `new NodeExecutionEnv({ cwd })` to bind filesystem and
    process execution to the target project directory.
- **SDK Consumers**:
  - Instantiate `NodeExecutionEnv` to adapt built-in harness tools into
    standard `AgentTool` instances for `Agent` or `agentLoop`.

### 2. When It Is Called

```
+─────────────────────────────────────────────────────────────────────────+
| Phase: STARTUP / DISCOVERY (Session Init)                               |
| • Methods: listDirectory(), stat(), readFile(), makeDirectory()         |
| • Action: Scans workspace for skills & templates; initializes JSONL dir |
+────────────────────────────────────┬────────────────────────────────────+
                                     │
                                     ▼
+─────────────────────────────────────────────────────────────────────────+
| Phase: TOOL EXECUTION (read, write, edit, bash tool turns)              |
| • Methods: readFile(), readBinaryFile(), writeFile(), canonicalPath()  |
|            exec(), createTempFile()                                     |
| • Action: Atomic file I/O within path locks; process execution with     |
|           stream capture and disk spillover                             |
+────────────────────────────────────┬────────────────────────────────────+
                                     │
                                     ▼
+─────────────────────────────────────────────────────────────────────────+
| Phase: SESSION PERSISTENCE (End of Turn / Commit Boundary)              |
| • Methods: appendFile(), writeStream(), createTempFile(), rename()      |
| • Action: Appends message entries and atomic commit snapshots to disk   |
+────────────────────────────────────┬────────────────────────────────────+
                                     │
                  ┌──────────────────┴──────────────────┐
                  │ (Normal Shutdown)                   │ (AbortSignal)
                  ▼                                     ▼
+───────────────────────────────────+ +───────────────────────────────────+
| Phase: TEARDOWN                   | | Phase: CANCELLATION               |
| (Session / Lane Close)            | | (Operation Aborted)               |
| • Method: cleanup()               | | • Signal Handler on exec()        |
| • Action: Terminates child        | | • Action: Sends SIGTERM / SIGKILL |
|   processes, closes open streams, | |   to child processes, terminates  |
|   and deletes scratch temp files  | |   running exec() calls            |
+───────────────────────────────────+ +───────────────────────────────────+
```

