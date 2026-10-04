# Compaction & Summarization

The `harness/compaction` subsystem manages context window optimization
for long-running agent sessions. It shrinks historical messages into
structured summaries while retaining recent turns and recording file
operations.

## Architecture Diagram

```
===========================================================================
                       COMPACTION & SUMMARIZATION
===========================================================================

       Transcript / Session Entries (Messages, Tools, Summaries)
                                   │
                                   ▼
                      Token Count & Threshold Check
                  [calculateContextTokens / shouldCompact]
                                   │
                                   ▼
+─────────────────────────────────────────────────────────────────────────+
|                       SPLIT & EXTRACTION PHASE                          |
|                                                                         |
|   • findCutPoint(): locates turn boundary by token budget               |
|     ├─ Summarized History (older turns to compress)                     |
|     └─ Retained Tail (recent turns preserved intact)                    |
|                                                                         |
|   • FileOperations Extraction (utils.ts):                               |
|     Inspects read, write, edit tool calls -> <read-files>, <modified>   |
+──────────────────────────────────┬──────────────────────────────────────+
                                   │
                                   ▼
+─────────────────────────────────────────────────────────────────────────+
|                        SUMMARIZATION PHASE                              |
|                                                                         |
|   • serializeConversation(): formats messages into plain-text prompt    |
|   • completeSimpleWithRetries(): invokes summarization model with       |
|     retry policy and SUMMARIZATION_SYSTEM_PROMPT                        |
+──────────────────────────────────┬──────────────────────────────────────+
                                   │
                                   ▼
+─────────────────────────────────────────────────────────────────────────+
|                          COMPACT RESULT                                 |
|                                                                         |
|   • summary: concise overview + <read-files> + <modified-files>         |
|   • retainedTail: recent turns preserved without modification           |
|   • Persisted as CompactionEntry in session storage                     |
+─────────────────────────────────────────────────────────────────────────+
```

## Module Overview

### 1. `compaction.ts`

The main driver for context pruning and session compaction.

- **Token Estimation**:
  - `estimateTokens(text)`: Approximates token counts (~4 chars/token).
  - `calculateContextTokens(messages)`: Estimates tokens for a sequence
    of `AgentMessage` objects.
  - `estimateContextTokens(entries)`: Computes tokens across raw session
    entries.
- **Compaction Decision**:
  - `shouldCompact(contextTokens, contextWindow, settings)`: Evaluates
    whether context exceeds `contextWindow - reserveTokens` (default
    reserve is 16,384 tokens).
- **Split Mechanics**:
  - `findTurnStartIndex(messages, targetIndex)`: Snaps cut points to the
    nearest preceding user turn boundary to avoid cutting within turns.
  - `findCutPoint(messages, settings)`: Divides history into turns to
    summarize and a retained tail (governed by `keepRecentTokens`,
    default 20,000).
- **Compaction Execution**:
  - `prepareCompaction(...)`: Builds the split, extracts file operations,
    and returns `CompactionPreparation`.
  - `compact(...)`: Orchestrates preparation, conversation
    serialization, the LLM summarization call with automatic retries,
    and returns a `CompactResult`.

### 2. `branch-summarization.ts`

Handles branch navigation and session tree exploration.

- **Branch Entry Collection**:
  - `collectEntriesForBranchSummary(branch, session, oldTip, targetId)`:
    Finds the common ancestor between the current branch tip and the
    navigation target, gathering entries on the abandoned path.
- **Branch Preparation**:
  - `prepareBranchEntries(entries)`: Extracts messages and accumulated
    file operations across the abandoned branch.
- **Branch Summary Generation**:
  - `generateBranchSummary(...)`: Invokes the LLM to summarize the
    abandoned branch exploration. When users switch to another branch,
    this summary is injected so prior learnings and file operations are
    not lost.

### 3. `utils.ts`

Shared utilities for file tracking and prompt formatting.

- **File Operations Tracking (`FileOperations`)**:
  - Accumulates sets of `read`, `written`, and `edited` files from
    assistant tool calls (`extractFileOpsFromMessage`).
  - `computeFileLists`: Separates files into read-only and modified.
  - `formatFileOperations`: Formats files into XML tags (`<read-files>`
    and `<modified-files>`) appended to summaries.
- **Conversation Serialization (`serializeConversation`)**:
  - Formats LLM messages into plain-text blocks (`[User]`,
    `[Assistant thinking]`, `[Assistant]`, `tool_name(args)`, and
    `[Tool result]`).
  - Truncates individual tool result strings to 2,000 characters to keep
    the summarization prompt within manageable limits.

## Who Calls `compaction.ts` and When

### 1. Who Calls It

- **`AgentLane` & `structural.ts` (`packages/agent/src/harness/runtime`)**:
  - `lane.compact()`: Accepts compaction requests into the serialized
    lane queue.
  - `prepareCompaction()`: Verifies eligibility, validates message count,
    and locates the turn cut-point.
  - `compactWithRequest()` in `structural.ts`: Drives the summarization
    request and commits the resulting `CompactionEntry`.
- **`AgentSession` (`packages/coding-agent/src/core/agent-session.ts`)**:
  - Exposes `session.compact()` for SDK consumers.
  - Automates threshold compaction via `checkAutoCompaction()`.
- **Interactive TUI & CLI (`coding-agent/src/modes/interactive`)**:
  - Dispatches compaction when users run the `/compact` command.
- **RPC Service (`packages/coding-agent/src/modes/rpc`)**:
  - Invokes `session.compact()` when handling remote `compact` RPC calls.
- **Extensions**:
  - Programmatically invoked via `ctx.compact({ customInstructions })`.

### 2. When It Is Called

Compaction executes under three distinct triggers:

1. **Manual Compaction (`reason: "manual"`)**:
   - The user executes the `/compact` slash command.
   - An RPC client submits a compaction request.
   - An extension triggers compaction programmatically.
2. **Threshold Auto-Compaction (`reason: "threshold"`)**:
   - Evaluated between turns when auto-compaction is enabled.
   - Triggered when:
     `contextTokens > contextWindow - reserveTokens`
     (default `reserveTokens` is 16,384).
   - Pauses the loop before the next LLM call to compress older turns.
3. **Overflow Recovery (`reason: "overflow"`)**:
   - Triggered when an LLM call fails due to context window exhaustion.
   - The runtime driver intercepts the error, runs emergency
     compaction, and retries the prompt with compressed context.

