# The Lane Queue

Summary of the article *"The Lane Queue"* by Lince Mathew (Feb 2026).
Source: `https://medium.com/@lince-mathew/the-lane-queue-b90ec8fc3fa1`

---

## Core Problem: Agents Are Stateful

Standard web development paradigms treat requests as stateless and
concurrent. AI agents, however, are inherently **stateful**.

When an agent manages a filesystem, runs shell commands, or interacts
with external environments, relying on ordinary concurrent `async/await`
causes critical issues:

1. **Race Conditions**:
   The agent may attempt to read a file before a previous write or edit
   command has completed.
2. **State Corruption**:
   If a new prompt enters while an LLM is in the middle of a multi-step
   tool loop, the context window mixes unrelated turns, corrupting the
   transcript and reasoning.
3. **Dependency Failures**:
   Build steps, package installations, and compile sequences require
   strict sequential execution. Free-form concurrency breaks these
   tool chains.

---

## The Lane Queue Architecture

The **Lane Queue** pattern enforces **Session-Based Serial Execution**:

- **Queue Assignment (Lanes)**:
  Each user, project, or session is partitioned into an isolated
  "Lane".
- **FIFO Processing**:
  Commands directed to a Lane enter a First-In-First-Out queue.
- **Atomic Execution**:
  The agent runner processes exactly one task at a time. It only drains
  the next command once the previous turn and tool loop have fully
  resolved and system state is finalized.

---

## The Atomic Tool Loop

Rather than treating user requests as single prompt-response interactions,
the Lane Queue wraps operations in an **Atomic Tool Loop**:

1. **Locking**:
   When a command starts, the Lane is locked against concurrent entries.
2. **The Execution Loop**:
   The agent iterates through:
   `Thought` -> `Tool Execution` -> `Tool Result`
   This cycle repeats until the agent decides the task is done.
3. **Commit**:
   Only after the agent signals task completion is the state committed,
   the Lane unlocked, and the next queued item admitted.

This guarantees that the agent's "brain" is never interrupted in the
middle of a multi-step operation (such as refactoring files or running
builds).

---

## Robustness: Crashes, Timeouts, and Durability

Serial queues risk "Hanging Lanes" if a step becomes stuck. The pattern
addresses this through:

- **Heartbeat & Timeout Monitoring**:
  Tracks execution duration. If a tool call or process hangs beyond a
  defined threshold, the runner terminates the process, marks the lane
  with an error state, and clears the lock.
- **Persistent Queueing**:
  Queue state is backed by durable storage (such as SQLite or JSON).
  If the host process crashes or restarts, the Lane Queue resumes from
  the last uncommitted task.

---

## Parallels in Pi Architecture

This architectural pattern aligns directly with Pi's core runtime design:

- **`Agent.activeRun`**: Single-run execution lock preventing concurrent
  prompt processing while allowing mid-run steering and follow-ups.
- **`AgentLane` (`packages/agent/src/harness/runtime/lane.ts`)**:
  Serialized mutation line ensuring operations, commits, and state
  transitions are atomic per session branch.
- **`file-mutation-queue.ts`**: Per-file canonical path locks ensuring
  concurrent tool runs cannot interleave conflicting file edits.
