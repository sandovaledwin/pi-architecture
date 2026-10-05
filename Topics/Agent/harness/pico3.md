# Pico3 harness architecture

## Summary

Source: `packages/agent/src/harness/pico3/` (all 25 TypeScript files).

Pico3 is an experimental, single-process agent kernel, exported separately
as `@earendil-works/pi-agent-core/experimental/pico3`. It turns user inputs
into durable tasks: model generation, parallel tool execution, continuation,
context compaction, process jobs, and plugin handlers.

The central separation is:

- **Session** serializes reads and commits on one queue, called the line.
- **Scheduler** executes task handlers outside that queue. Handlers return
  transitions or completion callbacks, which run inside a commit.
- **Storage** atomically persists records and tracked document changes.
- **ViewManager** publishes safe, commit-granular conversation projections.

This separation lets slow model/tool calls run concurrently without letting
concurrent handlers mutate durable state directly. Checkpoints written before
external effects determine what to do after reopening storage.

## Architecture diagram

```text
Host application / Micro controller / Chord conversation service
                            |
                            v
                 Harness + ConversationHandle
                    |                    |
             reads / commits          resume()
                    v                    v
             Session: serial line <--- Scheduler
                    ^                initial / phase / abort
                    |                    |
                 rt.commit() <--- Task handlers (off-line)
                    |              |         |          |
                    |            Models    Tools    ProcessHost
                    v
            Storage.commit(batch)
             /                 
    MemoryStorage          JsonlStorage
                           sidecars -> main marker
                    |
       update indexes + ViewManager (on-line)
                    |
          deliver Envelope (off-line)
                    v
         Watch -> UI or Chord view bridge
```

## Who calls what, and when

### 1. Opening and starting

1. The host opens `MemoryStorage` or calls `JsonlStorage.open(directory)`.
   JSONL opening repairs torn tails and replays published commits.
2. The host calls `Harness.open(storage, options, ctx)`. Its constructor
   installs built-in kinds and registries, creates Session, ViewManager,
   hook runners, and Scheduler, and wires commit listeners.
3. `Harness.init()` loads conversations and pending/running tasks. It also
   loads terminal owner tasks needed to reconstruct ownership ancestry.
   If storage is empty, it creates the root conversation, ID 1.
4. The host configures conversations, namespaces, hooks, and watches.
   Opening does **not** start task execution.
5. The host calls `harness.resume()`. Harness terminalizes tasks whose kinds
   are missing as `orphaned`, then calls `Scheduler.resume()`.

A concrete caller is `packages/coding-agent/src/experimental/micro/runtime.ts`:
`openMicro()` opens JSONL storage and Harness, sets tools, registers system
instruction hooks, captures/starts a root watch, then calls `resume()`.
Its controller routes prompts, steering, follow-ups, compaction, and abort
commands to the root conversation handle.

### 2. Sending an input

The host calls `ConversationHandle.send()`, which uses a kernel-authorized
`Session.commit()` to call `TxImpl.send()`.

- A repeated `requestId` returns the existing input ID.
- If idle, Session places queued boundary items, appends `pi.user`, records
  the input as `placed`, creates `pi.generation`, and emits turn events.
- If busy, input is queued as `steer` or `followUp`; the default is follow-up.
  `whenBusy: "reject"` throws `ConversationBusy` instead.
- Busy admission means a live, non-background kind with `turn: true`.
  It is not identical to `waitForIdle()`, which counts all non-background
  live tasks, including ordinary jobs/plugins.

The post-commit Scheduler listener calls `kick()` when tasks change.
`reserveEligible()` runs on the Session line and selects tasks with no active
invocation and terminal `after` dependencies. It marks pending tasks running.
`dispatch()` then calls the kind's initial/current-phase handler off-line.

### 3. The model/tool turn

```text
send(input)
    |
    v
pi.generation
    |
    +-> snapshot -> systemInstructions -> validate snapshot
    |                    |
    |              pi.system + prepared checkpoint
    |
    +-> beforeRequest -> requesting checkpoint -> Models.stream
    |                                                |
    |                                  frames -> sticky.turn.message
    |
    +-> afterResponse -> classify
             |
             +-- tool calls --> pi.tool tasks (parallel)
             |                      |
             |          beforeTool -> execute -> afterTool
             |                      |
             |                 pi.tool_result
             |                      |
             |        pi.post_tools waits for ALL tools
             |                      |
             |          afterTools + queued steering
             |                      |
             |              next pi.generation
             |
             +-- final answer --> onYield -> final boundary
                                     |
                            settle inputs / continue
```

**Generation preparation:** `generation.initial()` calls `takeSnapshot()`
in a commit, then `prepareDraft()` outside the line. System instruction hooks
edit sections and optionally override tools. Its transition callback takes a
second snapshot; if relevant state/registry revisions changed, preparation
retries. Otherwise it writes a managed `pi.system` baseline/delta if needed
and checkpoints model, tools, retry policy, and a fixed transcript cutoff.

**Request:** the `prepared` handler resolves the model, derives context at the
cutoff, chains `beforeRequest`, checks token capacity, and durably checkpoints
`requesting` before calling `Models.stream()`. Stream frames update the turn
view: first content promptly, then batches at roughly 100 ms or 256 units of
pending frame size. `afterResponse` observes terminal provider messages.

**Tool calls:** generation's completion callback appends the assistant entry,
creates tool slots and one `pi.tool` task per call, then creates `pi.post_tools`
with `after` pointing to every tool task. Tools can execute concurrently;
results are stored in completion order. Request projection reorders them into
assistant call order.

**Tool invocation:** `tool.initial()` verifies the offered tool set, registry,
and TypeBox arguments, runs `beforeTool`, protects call identity, and validates
arguments again. It checkpoints the final call as `started` before calling
`ToolDeclaration.execute()`. Tools receive bounded streaming, progress fields,
memos, child-conversation creation, and typed task APIs. `afterTool` may replace
the result; the completion callback bounds text and stores `pi.tool_result`.
Tool errors normally become error results, not failed task outcomes.

**After tools:** Scheduler calls `postTools.initial()` only after all tool
tasks are terminal. It reads results and invokes `afterTools`. Its completion
callback supplies missing results, applies `addTools`, `terminate`, or `handoff`
controls, admits boundary inputs, and usually creates the next generation.
Handoff appends a new self-head; terminate/handoff settle the current inputs.

**Final answer:** generation runs `onYield` hooks, appends the answer, and
places final-boundary items. A hook continuation is used only if no queued
triggers were placed and the boundary did not terminate the turn. Otherwise
inputs finish as `done`, with the assistant entry as their answer; queued
triggers can start a new turn. The hook is called before that boundary decision.

### 4. Boundary placement and context

`TxImpl.boundary()` is called by idle admission, generation completion/failure,
and post-tools completion. It places passive writes and steering at post-tools
boundaries; follow-ups are additionally admitted at final boundaries. Each
input class uses its configured `all` or `one-at-a-time` policy.
Queued self-head writes make older conversational inputs stale.

`deriveContext()` is called through transaction/runtime context reads. It:

1. Finds the newest fork-visible head at or before the requested cutoff.
2. Scans the active range and collects newest-wins entry edits.
3. Places the current head first and excludes other head entries.
4. Omits/replaces edited messages and concatenates remaining `entry.model`.
5. Orders tool results and synthesizes missing results after fork cuts.

Display-only error/aborted assistants and usage entries contribute no model
messages. The UI transcript is different: `captureActiveTranscript()` retains
all display entries in the active range rather than applying model edits.

A host fork uses `Session.fork()` to validate the historical entry and inherit
rewindable state as of its atomic commit. Sticky state starts fresh. Transcript
inheritance follows `parent`; task scope and subtree hooks follow `owner`.
These are distinct relationships.

### 5. Compaction, jobs, and plugins

- **`pi.collapse`:** created by manual `conversation.collapse()`, generation's
  token threshold check, or request overflow. `chooseThrough()` preserves a
  recent token budget and keeps assistant/tool-result exchanges together.
  `beforeCollapse` can decline, change instructions, or supply a summary.
  Otherwise the task checkpoints `summarizing` before a model request with
  tools removed. It checks the expected head before publishing `pi.summary`;
  a moved head produces `stale`. The summary advances context's head but does
  not delete stored history. Overflow creates a successor generation dependent
  on collapse; threshold collapse can overlap other turn work.
- **`pi.job`:** explicitly created by a host/task/tool; not automatically
  created for shell tools. It calls the supplied `ProcessHost.start/status`
  using `taskId:occurrence` keys. It supports delayed/repeated execution,
  polling, live output slots, and optional passive `pi.notice` entries.
  Abort sends SIGTERM, waits five seconds, then sends SIGKILL.
- **`pi.plugin`:** explicitly created with a registered handler name.
  It checkpoints `started`, then calls `PluginHandler(input, taskApi, ctx)`.
  Reopening reruns the handler. Missing/throwing handlers produce declared
  failures. This task kind is separate from namespace-bound hooks/state.

### 6. Commit, observation, and shutdown

`Session.commit()` queues the operation, checks invocation authority, preloads
tracked documents, runs the callback, collects record writes/document deltas,
and calls `Storage.commit()` without a caller abort signal. After publication
it updates in-memory indexes and invokes on-line view listeners; ordinary
listeners and watch delivery run after the line operation returns.

Callback/validation failure evicts touched document trackers. Persistence
failure faults Session and closes storage: it cannot safely keep using tracked
state already adopted by the legacy tracker. Listener errors are reported via
`onReport`, not surfaced as commit failures. Transaction methods and document
proxies are invalid after the callback/transaction; snapshots are plain copies.

`conversation.watch()` captures transcript and subscribes in one line operation
so no commit is lost between the snapshot and subscription. The returned view
and revision are the initial snapshot, not automatically updated objects.
Consumers apply subsequent envelopes with `applyEnvelope()`. Up to 256 envelopes
can buffer before `start()`; overflow or a throwing listener closes the watch.

`attachChordView()` converts a watch into replicated Chord state. Its raw
listener only queues envelopes; a microtask applies each envelope and its
commit events in one `view.change()`. Bridge overflow/publication failure
closes the bridge and calls the owner's optional failure callback.

`harness.abortTask()` durably marks, signals, and joins a run invocation;
Scheduler subsequently invokes the kind's abort handler and commits its abort
closure. `conversation.abort()` withdraws queued conversational inputs and
marks non-background tasks, then waits for relevant work to become idle.
Background tasks survive conversation abort.

`harness.suspend()` stops/signals/joins invocations without terminalizing live
tasks, clears tool waiting indicators, closes watches, then closes Session and
storage. `close()` delegates to suspend; despite its comment, it can write
those waiting-indicator cleanup changes. Reopening with a new Harness is
required; a suspended Harness cannot resume. `hold()` temporarily prevents
dispatch and is accepted only when there are no active invocations.

## Recovery behavior

| Durable phase | Caller after reopen | Behavior |
| --- | --- | --- |
| Generation `prepared` | Scheduler -> phase handler | Re-derive request; rerun request hooks. |
| Generation `requesting` | Scheduler -> phase handler | Treat interrupted effect as a failed attempt; apply retry policy. |
| Generation `retrying` | Scheduler -> phase handler | Sleep until durable backoff deadline; return to prepared. |
| Generation `deferred` | Scheduler -> phase handler | Poll saved provider handle; abort attempts cancellation. |
| Tool `started` | Scheduler -> phase handler | Replay only if stored and current declaration both say safe and args validate; otherwise synthesize error. |
| Collapse `summarizing` | Scheduler -> phase handler | Treat as interrupted and apply retry policy. |
| Collapse `prepared` | Scheduler -> phase handler | Publish saved summary only if expected head still matches. |
| Job `spawning` / `running` | Scheduler -> phase handler | Ask process host for status; rerun unknown work only if enabled. |
| Plugin `started` | Scheduler -> phase handler | Invoke handler again; handler must tolerate replay. |

Handlers explicitly commit in-flight checkpoints before external effects.
Normal `next` transitions into an in-flight phase are rejected. Each handler
and transition/completion callback gets a separate revocable invocation token.
Unexpected exceptions or invalid steps become `faulted`, not a fabricated
kind-declared failure. Missing kinds become `orphaned` at Harness resume.

## Storage layout and guarantees

```text
Session.commit: records + document ops in one batch
                         |
                         v
                JsonlStorage.commit
                         |
        +----------------+----------------+
        |                                 |
 task-ID.jsonl                     sticky-CONV.jsonl
 live intermediate patches         current-state document ops
        |                                 |
        +----------------+----------------+
                         |
                         v
                    main.jsonl
          publication marker + referenced sidecars
          conversations, entries, inputs, task create/end,
          rewindable and session document operations
```

JSONL appends sidecars first and main last. Replay accepts sidecar records only
when a main marker confirms them; unconfirmed/torn tails are discarded.
Terminal task sidecars are removed after publication. Sticky logs can be
rebased/truncated when tasks retire: conversation idle or delta history over
256 KiB. Main history is append-only; rewindable history supports forks at
commit granularity, not at intermediate positions inside a batch.

`fsync` defaults to true. Sidecars are flushed before the main marker, but
parent-directory fsync is absent for file creation/rename/unlink, so this is
not a complete database durability guarantee. With fsync disabled, acknowledged
commits can disappear on machine failure; external idempotency remains needed.
Only one process may own a JSONL directory, and only one Session may own a
Storage instance. This is not a multi-process database or exactly-once executor.

## File map: responsibilities, callers, timing

All paths below are relative to `packages/agent/src/harness/pico3/`.

| File                  | Responsibility                                                                                                 | Who calls it / when                                                                               |
| --------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `index.ts`            | Experimental public exports.                                                                                   | Host integrations importing the Pico3 package subpath.                                            |
| `harness.ts`          | Composition, registries, handles, lifecycle, runtime construction.                                             | Host at open/configure/send/watch/resume/abort/close; Scheduler when constructing runtime leases. |
| `types.ts`            | Durable JSON records, documents, kind/transaction/runtime contracts, errors, typed witnesses.                  | All modules; hosts author tools/tasks/namespaces with these contracts.                            |
| `session.ts`          | Serialized line, transaction implementation, authority, defaults, document cache, admission/boundaries, forks. | Harness, Scheduler, and task runtime reads/commits throughout execution.                          |
| `scheduler.ts`        | Eligibility, dependency gating, phase dispatch, aborts, contract validation, waiters.                          | Harness at resume/lifecycle; Session commit listener when task/input state changes.               |
| `context.ts`          | Model-context projection, edits, heads, ordered/missing tool results.                                          | `TxImpl.context()` during requests, compaction, and explicit context reads.                       |
| `system.ts`           | Section draft/rendering, canonical fold, preparation snapshots, managed system entries, tool loadout fold.     | Generation preparation; collapse to remove effective tools from summary requests.                 |
| `hooks.ts`            | Kind-token matching and global/conversation/owned-subtree hook selection.                                      | Harness constructs runners; task handlers enumerate or chain hooks at their hook points.          |
| `view.ts`             | Safe projections, incremental envelopes, watch lifecycle, immutable envelope application.                      | Harness during watch capture; Session after commits; UI consumers applying updates.               |
| `chord.ts`            | Harness/conversation service definitions, command adapters, replicated view bridge.                            | Chord host/service setup; remote commands; watch updates published via microtasks.                |
| `memory.ts`           | Reference in-memory backend, staged batch writes, fork-aware scans and document history.                       | Session storage calls; JSONL inherits its read/replay path.                                       |
| `jsonl.ts`            | File persistence, publication markers, replay, torn-tail repair, sidecar retirement/truncation.                | Host opening storage; Session committing, retiring tasks, or closing.                             |
| `legacy-tracker.ts`   | Adapts Chord begin/prepare/adopt tracking to flush/rebase.                                                     | Session document cache and ViewManager whenever tracked state is accessed/flushed.                |
| `membrane.ts`         | Revocable document wrappers; reject proxy assignment and unsafe structural changes.                            | TxImpl wraps document/namespace access and revokes wrappers at transaction end.                   |
| `bounded.ts`          | Byte/newline-limited head/tail stream collector with discard counters.                                         | Tool invocation on every streamed output chunk.                                                   |
| `bash.ts`             | Optional unsafe-replay shell tool; merges stdout/stderr into tool stream.                                      | Host creates/registers declaration; `pi.tool` invokes it for an offered bash call.                |
| `kinds/generation.ts` | Model turn, request checkpoints, streaming, retry/deferred handling, tool/continuation/compaction spawning.    | Scheduler after send or a successor task becomes eligible.                                        |
| `kinds/tool.ts`       | Tool checks, hooks, pre-effect checkpoint, bounded results, child abort work.                                  | Scheduler for each generated tool call; recovery/abort dispatch.                                  |
| `kinds/post-tools.ts` | Join tool results, controls, missing-result repair, boundary admission, successor generation.                  | Scheduler after every referenced tool task is terminal.                                           |
| `kinds/collapse.ts`   | Summary task, head validation, collapse cutoff selection.                                                      | Scheduler for manual/threshold/overflow tasks; Harness/generation call cutoff helper.             |
| `kinds/job.ts`        | Delayed/repeated process task, reconciliation, polling, output slot, termination.                              | Scheduler for explicitly created jobs; supplied ProcessHost performs process operations.          |
| `kinds/plugin.ts`     | Named plugin handler execution with durable start marker.                                                      | Scheduler for explicitly created plugin tasks, including replay.                                  |
| `kinds/task-api.ts`   | Shared narrow API for owned conversations and typed tasks.                                                     | Tool invocation and plugin handler setup; those callbacks invoke its methods.                     |
| `kinds/entries.ts`    | Built-in entry-kind tokens and lightweight kind predicates.                                                    | Harness installs tokens; host/task callers use typed entry reads/writes.                          |
| `kinds/frames.ts`     | Apply encoded assistant frames directly to tracked turn state.                                                 | Generation stream flushes while model events arrive.                                              |

## Important boundaries

- Only generation, tool, post-tools, and collapse have core transaction
  authority. Built-in job and plugin tasks have ordinary task authority.
- Ordinary tasks may access their conversation and owned subtrees, write
  passive entries, modify their own slot/checkpoint, and use namespace state.
  They cannot mutate raw core documents/config or manufacture core turn tasks.
- Namespace tokens bind private state across rewindable/sticky/session docs.
  State reaches public views only through registered namespace projections.
  Tool memos and raw task slots/checkpoints are not directly published.
- General hook exceptions are reported/skipped. `beforeTool` is stricter:
  a thrown hook blocks execution. System draft edits from a throwing handler
  are rolled back before proceeding.
- Models and ProcessHost are supplied interfaces. Pico3 coordinates calls and
  persistence; it does not itself implement provider authentication or a
  durable process supervisor.

This document summarizes observed source behavior, not a test run or a
complete conformance/security audit.
