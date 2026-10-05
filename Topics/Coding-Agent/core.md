# Coding-Agent Core Architecture

## Scope and reading guide

Paths below are relative to `packages/coding-agent/src/core` 

## 1. Architectural role

Core is the application layer between Pi's user-facing modes and the
lower-level agent/model packages. It combines coding-specific policy with
provider-neutral execution:

- Session creation and replacement.
- Prompt preparation, tools, extensions, queues, retry, and compaction.
- Durable session trees and model-context reconstruction.
- Model selection, provider composition, and credential storage.
- Settings, project trust, and resource/package discovery.
- Terminal-facing metadata, diagnostics, and exports.

The low-level `Agent` belongs to `@earendil-works/pi-agent-core`, not this
folder. Provider protocols and the underlying `Models` implementation
belong to `@earendil-works/pi-ai`. Interactive widgets and mode dispatch
mostly live outside core, although core's renderers and export code depend
on terminal/UI facilities.

```text
CLI main()                     SDK host
    |                             |
    v                             v
AgentSessionRuntime         createAgentSession()
    |                             |
    +------------+----------------+
                 v
            AgentSession
                 |
    +------------+------------+----------------+
    |            |            |                |
    v            v            v                v
 Agent      SessionManager ModelRuntime   ExtensionRunner
(pi-agent)     |             |                |
    |          v             v                v
    |        JSONL        pi-ai Models    loaded extensions
    v
 executable tools ---> files / shell / injected operations

SettingsManager + ResourceLoader ---> session configuration
Interactive / print / RPC modes <--- session events
```

### The three session abstractions are intentionally different

| Abstraction | Owns | Called by / when |
|---|---|---|
| `AgentSessionRuntime` | Current session and its cwd-bound services; replacement factory | Modes and extension command callbacks on new/resume/fork/import/quit |
| `AgentSession` | One conversation's execution policy, agent hooks, extensions, queues, tools, recovery | Modes and SDK hosts during prompting and session operations |
| `SessionManager` | Entry tree, active leaf, JSONL persistence, canonical projections | Factory during restore; session during events; selectors/exporters during reads |

Problem: replacing history in place can leave tools and extensions bound
to the previous workspace. For example, resuming a session from another
project requires a different `.pi/settings.json` and tool cwd.

Solution: runtime replacement reconstructs the session and services for
the target cwd. Hosts rebind their subscriptions afterward. Tree navigation
within one session instead changes the manager's leaf without replacing the
runtime.

## 2. Construction and external callers

### Standard CLI startup

`../main.ts` defines a `CreateAgentSessionRuntimeFactory` and calls
`createAgentSessionRuntime()`. The same factory is retained for later
session replacement.

```text
main(): choose/open SessionManager, obtain its cwd
    |
    v
createAgentSessionRuntime(factory, initial target)
    |
    v
factory: createAgentSessionServices()
    |       |
    |       +--> ModelRuntime
    |       +--> SettingsManager
    |       +--> DefaultResourceLoader.reload()
    |              +--> optional project-trust bootstrap
    |              +--> packages and resources
    |
    +--> resolve model scope, CLI options, runtime API key
    |
    v
createAgentSessionFromServices()
    |
    v
createAgentSession() --> Agent + CacheWarmer + AgentSession
    |
    v
mode: bindExtensions(), subscribe(), prompt()/commands
```

The CLI resolves explicit resource paths relative to the launch cwd before
the factory captures them. A subsequent session cwd change therefore does
not reinterpret those CLI paths.

### Factory methods

| Function | Responsibility | Caller and timing |
|---|---|---|
| `createAgentSessionServices()` | Normalize cwd/agent directory; create model runtime and settings; reload resources; flush queued extension provider registrations; refresh availability without network; validate extension flags | CLI runtime factory and SDK runtime hosts, before model/tool options are finalized |
| `createAgentSessionFromServices()` | Forward coherent pre-created services and session options to the SDK factory | Runtime factory, after resolving options against the target services |
| `createAgentSession()` | Restore context/model/thinking, choose tools, create agent callbacks/cache warmer, create `AgentSession` | SDK hosts directly, or the services factory |
| `createAgentSessionRuntime()` | Validate stored cwd, call the supplied factory, wrap its result | `main()` and SDK runtime hosts at initial startup |

`AgentSessionServices` is an interface, not a class. Its bundle contains
`cwd`, `agentDir`, `modelRuntime`, `settingsManager`, `resourceLoader`, and
diagnostics. Construction returns diagnostics instead of deciding how to
print errors or exit; that belongs to the host.

### What `createAgentSession()` restores and installs

1. Resolve cwd from explicit options, manager, or process cwd.
2. Use supplied dependencies or create default file-backed services.
3. Call `SessionManager.buildSessionContext()`.
4. Try a saved model, then settings/provider defaults through
   `findInitialModel()`. Report a fallback message when restoration fails.
5. Restore thinking level, apply per-model/global defaults, and clamp it to
   model capabilities. No selected model means thinking is off.
6. Select initial built-ins. Default: `read`, `bash`, `edit`, `write`.
   Allowlist, denylist, `noTools`, and `defaultTools` modify that selection.
7. Construct `Agent`, using the manager's projected messages.
8. Install model streaming through `ModelRuntime.streamSimple()`, current
   request settings, provider attribution headers, and extension callbacks.
9. Create `AgentSession`; its constructor subscribes internally and builds
   the tool/extension runtime.

The SDK factory does **not** itself emit `session_start`. Hosts call
`session.bindExtensions()` after providing mode/UI/command bindings.
An SDK host that needs startup extension lifecycle behavior must do this
explicitly. Ordinary `prompt()` behavior can still use the constructed
runner and its core bindings.

### Mode consumers

| Consumer outside core | Core interaction |
|---|---|
| `../modes/interactive/interactive-mode.ts` | Binds UI and extension actions, renders subscriptions, prompts user input, exposes model/compaction/tree/export commands, manages footer/keybindings |
| `../modes/print-mode.ts` | Binds print/JSON extension context, subscribes for JSON events, sends prompts sequentially, outputs final text, disposes runtime |
| `../modes/rpc/rpc-mode.ts` | Binds RPC UI/actions, maps protocol commands to session/runtime methods, forwards events |
| `../package-manager-cli.ts` | Uses settings, trust, resource loader, and package manager for install/remove/update/list/config without needing a conversation |
| SDK applications | Supply dependencies, call `prompt()`, subscribe, inspect state, dispose |
| `../experimental/` | Reuses selected infrastructure such as models, settings, resolvers, resources, keybindings, and tool renderers; alternative harnesses need not use `AgentSession` |

## 3. `AgentSessionRuntime`: replacement lifecycle

File: `agent-session-runtime.ts`.

Public methods:

- `switchSession(path, options)`: open a target manager; optional cwd override
  and target trust context; replace the current runtime.
- `newSession(options)`: create persisted or in-memory history in the current
  cwd; optional parent link, setup callback, and fresh-context callback.
- `fork(entryId, options)`: create a new session from a selected path. Default
  position is before a user entry, returning its text for reuse. Position
  `at` includes the selected entry.
- `importFromJsonl(path, cwdOverride)`: copy into the session directory with
  collision avoidance, open it, validate cwd, and replace the runtime.
- `setRebindSession(callback)`: host callback for attaching to the new session.
- `setBeforeSessionInvalidate(callback)`: synchronous UI teardown after
  shutdown handlers, before contexts become stale.
- `dispose()`: emit shutdown with reason `quit`, run teardown callback, and
  dispose the session.
- Getters: `session`, `services`, `cwd`, `diagnostics`,
  `modelFallbackMessage`.

```text
new / resume / fork / import
    |
    v
session_before_switch or session_before_fork
    |                       |
    | allowed               +--> cancel: keep current session
    v
prepare target manager + validate target cwd where required
    |
    v
await old session.abort()
    |
    v
session_shutdown --> host pre-invalidation teardown
    |
    v
old session.dispose(): invalidate contexts and disconnect
    |
    v
factory(target cwd, manager, session_start metadata)
    |
    v
apply result --> host rebind --> optional withSession(ctx)
```

The target may be prepared before teardown, but creation of the replacement
services happens after teardown. This is not an atomic rollback mechanism:
if replacement construction fails after disposal, the error propagates to
the host; the previous session is not restored automatically.

Subscriptions remain attached to an `AgentSession` object, not the runtime.
All three standard modes install rebind handling. Interactive mode also
uses pre-invalidation teardown for extension UI components.

Captured extension contexts become invalid after replacement or reload.
`withSession` receives a fresh `ReplacedSessionContext` for work that must
continue after a replacement.

## 4. `AgentSession`: orchestration and methods

File: `agent-session.ts`. Created by `sdk.ts`.

### Public method groups

| Methods / properties | Responsibility | Caller and timing |
|---|---|---|
| `prompt()` | Dispatch extension command, run input transforms, expand skill/template, queue or start a run | Modes and SDK hosts on input |
| `steer()`, `followUp()` | Prepare and queue input with explicit delivery semantics | Modes, RPC commands, SDK hosts while work is active |
| `sendUserMessage()`, `sendCustomMessage()` | Extension-origin input; custom messages may be context-only, queued, or trigger a run | Runner-bound actions and SDK integrations |
| `clearQueue()`, queue getters | Track user-visible queued text and clear agent queues | UI/RPC controls and status displays |
| `subscribe()` | Add public event listener; return unsubscribe function | Modes/SDK hosts before prompting, and after replacement |
| `abort()`, `waitForIdle()` | Cancel/wait across run and compaction lifecycle | Hosts and command contexts before conflicting operations |
| `dispose()`, `refreshContext()` | Disconnect/invalidate/cleanup; rebuild finalized messages from the manager | Runtime teardown; hosts after manager-based setup |
| `setModel()`, `cycleModel()` | Validate/select model, append model metadata, optionally persist defaults | Model selectors, keybindings, RPC, extensions |
| `setThinkingLevel()`, `cycleThinkingLevel()` | Clamp reasoning level and record changes; optional settings persistence | UI/RPC/extensions on reasoning changes |
| `getAvailableThinkingLevels()`, `supportsThinking()` | Capability checks | Selectors and status UI |
| `setScopedModels()` | Replace model cycling scope | Hosts and model-scope UI |
| `setSteeringMode()`, `setFollowUpMode()` | Update queue policy and settings | Settings UI/RPC |
| `getAllTools()`, `getToolDefinition()`, `getActiveToolNames()` | Inspect registered definitions versus active implementations | UI, extensions, export |
| `setActiveToolsByName()` | Select registered tools and rebuild prompt options | Extensions and host controls; applied on the next turn |
| `compact()`, `abortCompaction()`, `setAutoCompactionEnabled()` | Manual compaction/cancellation and automatic-policy setting | Commands, RPC, extension context |
| `navigateTree()`, `abortBranchSummary()` | Move within current history with optional abandoned-branch summary | Tree selector and command contexts |
| `getUserMessagesForForking()` | Enumerate fork candidates | Fork selectors |
| `executeBash()`, `recordBashResult()`, `abortBash()` | User-issued shell execution and transcript recording | `!`/`!!` UI and RPC; extension-handled shell results |
| `bindExtensions()`, `reload()` | Bind host facilities and start lifecycle; replace loaded resources/runner | Mode startup/rebind and reload commands |
| `setSessionName()` | Append display metadata and emit change events | Name commands and extensions |
| `getSessionStats()`, `getContextUsage()` | Historical accounting versus active-context estimate | Status UI, RPC, extensions |
| `exportToHtml()`, `exportToJsonl()` | Export history/tree or active branch | Export commands and SDK hosts |
| `summarizeForBugReport()`, `getLastAssistantText()` | Optional model-generated report summary; final-text utility | Bug-report/copy commands and SDK hosts |
| `createReplacedSessionContext()` | Fresh post-replacement context including awaited message helpers | Runtime's `withSession` callback |

Read-only access includes `state`, `messages`, `model`, `thinkingLevel`,
`systemPrompt`, `sessionId`, `sessionFile`, `sessionName`, `promptTemplates`,
`resourceLoader`, `extensionRunner`, scoped models, queue modes, recovery,
compaction, shell, and cache-warming state.

### Prompt-to-settlement trace

```text
mode / SDK --> AgentSession.prompt(text)
    |
    +--> registered extension command? --> command handler --> return
    |
    v
input handlers --> skill expansion --> prompt-template expansion
    |
    +--> run active? --> steer/followUp queue --> return
    |
    v
model/auth preflight --> possible pre-prompt compaction
    |
    v
before_agent_start --> images --> prompt/tool transcript delta
    |
    v
_runAgentPrompt() --> Agent.prompt()
    |                   |
    |                   +--> model requests / tools / message events
    v
_handlePostAgentRun()
    +--> retry / overflow recovery / compaction / queued work
    |             |
    |             +--> Agent.continue() --> repeat
    v
agent_before_settle boundary --> optional continuation
    |
    v
flush pending context --> agent_settled --> resolve idle wait
```

Input interception happens before skill/template expansion. Registered
extension commands take priority and can execute even while streaming.
Submitting an ordinary prompt during streaming without `streamingBehavior`
rejects rather than choosing a queue implicitly.

Steering is delivered after the current assistant turn and its tools,
before the next model call. Follow-up is delivered when no tool work or
steering remains. Queue-mode settings control all-at-once versus one-at-a-time
delivery. The low-level agent drains the actual queues; `AgentSession`
maintains text lists for public queue events and display.

Custom messages have additional delivery semantics:

- `nextTurn`: hold until the next user prompt.
- Streaming with continuation enabled: agent steering/follow-up queue.
- Streaming with `triggerTurn: false`: defer insertion until tool results
  are complete.
- Idle with `triggerTurn: true`: start a run.
- Idle without a trigger: append context without contacting the model.

Deferring context-only custom messages and shell results is necessary:
an insertion between an assistant tool call and its matching result can
make replay invalid for providers that enforce message ordering.

### Agent hooks and internal event handling

The constructor installs hooks once; callbacks consult the current runner,
so replacing extensions does not require reinstalling every agent hook.

| Internal method | Installed boundary / effect |
|---|---|
| `_installAgentToolHooks()` | `beforeToolCall` invokes `tool_call`; `afterToolCall` chains `tool_result` and normalizes images |
| `_installAgentRequestProjection()` | `prepareRequest` refreshes request messages from canonical session projection, and current executable tools/model/thinking |
| `_installAgentNextTurnRefresh()` | Before the next assistant response, check projected token pressure and append prompt/tool changes |
| `_installAgentBoundaryHooks()` | `finishTurn` dispatches actionable `turn_end` once finalized entries exist |
| `_installAgentForcedPromptProjection()` | After context transforms, render a forced run-specific prompt as the leading system message without storing that opaque replacement |
| `_handleAgentEvent()` | Update visible queues, dispatch extension events, notify public subscribers, persist finalized messages, track recovery state |
| `_refreshFinalizedContext()` | Replace finalized agent messages with the manager's projection and associate messages with source entry IDs |
| `_buildBoundaryContext()` | Build an in-memory manager preview with proposed entries and determine whether continuation is runnable |
| `_commitBoundaryDrafts()` | Append validated boundary entries, refresh context, emit entry events |

Important event ordering: for `message_end`, extension replacement runs
first, public listeners are notified next, and `SessionManager` append
happens afterward in the internal handler. A subscriber should not assume
that the just-emitted message is already findable in the manager during
that callback. Later turn boundaries operate after completed messages and
tool results have been appended.

`agent_end` means one low-level run ended. It is not final application
settlement: retries, recovery, queued work, and boundary continuation can
follow. `agent_settled` is the final notification once automatic work is
done. Actions submitted during its notification are deferred until dispatch
finishes.

### Prompt and tool state

`system-prompt.ts` supplies `normalizeBuildSystemPromptOptions()`,
`buildSystemPromptSections()`, `buildSystemPromptState()`,
`buildSystemPrompt()`, and `diffSystemPromptSections()`.

The session combines loaded prompts/context/skills with active tool snippets
and guidelines. Independently named sections are recorded as system-message
patches; tool declarations are also replayable transcript state. This lets
model context reproduce changes instead of relying only on mutable global
prompt text.

A `before_agent_start` extension can alter structured options or force exact
prompt text. Forced text is request-local; structured prompt changes remain
in the transcript. `setActiveToolsByName()` chooses executable implementations
and rebuilds the base options; the next-turn path declares the corresponding
model-facing changes.

## 5. `SessionManager`: durable tree and canonical context

File: `session-manager.ts`.

### Method groups

| Methods | Caller / timing |
|---|---|
| Static `create()`, `open()`, `continueRecent()`, `inMemory()`, `forkFrom()` | CLI/SDK before session construction or runtime replacement |
| Static `findById()`, `list()`, `listAll()` | CLI resume resolution and session pickers; list operations support progress and cancellation |
| `newSession()`, `setSessionFile()` | Manager-level lifecycle operations; normal application replacement is coordinated by runtime |
| `appendMessage()`, `appendModelChange()`, `appendThinkingLevelChange()` | Session event persistence, startup metadata, user model/thinking changes |
| `appendCompaction()`, `branchWithSummary()` | Completed manual/automatic summaries and tree navigation |
| `appendCustomEntry()`, `appendCustomMessageEntry()` | Extension state and model-visible injected content |
| `appendContextEdit()` | Actionable boundaries and recovery omissions; validate target belongs to the active branch and is editable |
| `appendUsage()` | Cache warmer or other non-message model usage |
| `appendLabelChange()`, `appendSessionInfo()` | Labels and session naming |
| `getBranch()`, `getTree()`, `getChildren()`, `getEntry()`, leaf getters | Selectors, extensions, branch summary preparation, runtime fork |
| `getEntries()`, `getHeader()`, metadata getters | Accounting, exports, diagnostics, restore |
| `buildContextEntries()`, `buildSessionProjection()`, `buildSessionContext()` | Restore and every canonical-context refresh/request |
| `branch()`, `resetLeaf()` | Navigate in the same file without deleting history |
| `createBranchedSession()` | Runtime fork: copy a selected path and applicable labels into a new session |

Each appended entry has an ID and parent ID. The manager advances its active
leaf on append; a branch operation moves that leaf. The header is file
metadata rather than a tree node.

```text
Raw append-only entry tree

root --> user A --> assistant A --> user B --> assistant B
                       |
                       +--> branch summary --> user C <-- leaf

Only the root-to-leaf path feeds active context.
Other branches remain available to the tree, export, and accounting.
```

### The context pipeline

```text
getEntries() + active leaf
    |
    v
walk parent links: active branch
    |
    v
buildContextEntries(): latest compaction + retained tail
    |
    v
buildSessionProjection(): apply active context_edit entries
    |                      preserve sourceEntry -> message mapping
    v
buildSessionContext(): messages + model/thinking metadata
    |
    v
Agent request transforms --> convertToLlm() --> provider
```

The latest compaction contributes a complete system checkpoint and summary,
then retained non-system entries starting at `firstKeptEntryId`, then entries
following compaction. Older summarized entries are not destroyed.

`context_edit` is append-only. `replacement: null` omits a target from future
model context; a replacement changes content without rewriting raw history
or message metadata. The latest selected edit wins. Returning to a branch
position before an edit restores the earlier contribution.

Different reads answer different questions:

- `getEntries()`: raw history across all branches.
- `getBranch()`: one root-to-leaf path.
- `buildContextEntries()`: compaction-aware selected raw entries.
- `buildSessionProjection()`: selected entries plus their edited/omitted
  message contributions and provenance.
- `buildSessionContext()`: finalized runtime context without the provenance
  wrapper.

Assigning `agent.state.messages` is not a durable history replacement. The
next request refreshes from the manager. Hosts restoring external history
must provide manager entries and call context refresh where appropriate.

### Persistence timing

New setup-only sessions remain in memory until a user or assistant message
exists. The first conversation append flushes the accumulated header/setup
entries using exclusive file creation; later appends write one JSONL line.
Thus opening and closing without chatting leaves no transcript file, while
a submitted prompt survives even if the first reply fails.

The file format is version 3. Loading migrates older formats and can rewrite
them; normal conversation changes are append-only. Discovery uses bounded
header scans and best-effort filtering. Full loading is authoritative, skips
malformed JSON lines, verifies a header, and repairs a missing final newline
on a valid session. Listing limits concurrent scans and publishes progress.

## 6. Compaction, branch summaries, and retry

### Responsibilities and callers

| File / functions | Purpose | Caller / timing |
|---|---|---|
| `compaction/compaction.ts`: `estimateTokens()`, `estimateContextTokens()`, `estimateProjectedContextTokens()`, `shouldCompact()` | Estimate active context and test token pressure | Session status, pre-request checks, post-run recovery |
| `findCutPoint()`, `prepareCompaction()` | Choose retained history while respecting turn/tool boundaries; prepare previous-summary and split-turn inputs | Manual/auto compaction before extension interception and generation |
| `compact()`, `generateSummaryWithUsage()`, `generateSummary()` | Generate history summary, optionally separate ongoing-turn prefix summary; collect file-operation metadata and usage | Session's shared default compaction path; SDK utility callers |
| `completeSummarization()` | Shared retry-enabled single-model-call boundary; disable prompt caching; use separate routing when needed | Compaction, branch summary, bug-report summary |
| `compaction/branch-summarization.ts`: `collectEntriesForBranchSummary()` | Find common ancestor and abandoned path | `navigateTree()` before navigation |
| `prepareBranchEntries()`, `generateBranchSummary()` | Budget and summarize the abandoned branch, preserving file metadata | Tree navigation when requested and not extension-supplied |
| `compaction/utils.ts` | Track read/edited/written files; serialize conversation as text; shared summarizer prompt | Both summarization implementations |

### Manual versus automatic compaction

`AgentSession.compact()` first aborts/waits for active work, prepares the
branch, emits `session_before_compact`, accepts cancellation or an extension
result, otherwise calls the default generator, appends the compaction, and
refreshes canonical context. Public start/end and extension success/failure
events report the operation.

Automatic checks run before a new prompt, before the next assistant response,
and after a low-level run. They distinguish:

1. Threshold pressure: compact without replaying a completed response.
2. Successful response beyond the context window: compact, do not duplicate
   that response.
3. Context overflow or recoverable truncation: append omission edits for the
   failed attempt, compact, then continue once. A repeated recovery failure
   stops rather than entering an unlimited compact/retry loop.

Both manual and automatic routes converge on the same default generator.
Summaries may update an older summary and separately summarize a split turn.
The resulting entry records summary usage for accounting.

### Transient error retry

`_handlePostAgentRun()` detects retryable assistant errors, excluding context
overflow. `_prepareRetry()` uses settings-controlled exponential backoff,
emits retry events, durably omits the failed attempt from model context, and
returns permission to continue. Raw failed messages remain available for
inspection and accounting. Abort cancels backoff and terminates continuation.

Provider/SDK request retry and application-level assistant retry are distinct
budgets. Summarization calls use `completeSummarization()` with the configured
assistant retry policy and separate summarization progress callbacks; they do
not execute through ordinary agent message/tool events.

### Tree navigation is not a runtime fork

`navigateTree()` rejects while streaming or compacting. It collects the
abandoned path, emits `session_before_tree`, optionally generates or accepts
a summary, moves the leaf, appends that summary at the target position, and
refreshes context/tools. Selecting a user/custom message moves to its parent
and returns its text for editing. `session_tree` reports completion.

No new file or cwd-bound services are required. `AgentSessionRuntime.fork()`
is the operation that creates a separate session and replaces the runtime.

## 7. Extensions: loading, binding, and dispatch

```text
ResourceLoader.reload()
    |
    v
loader.ts: jiti import / inline factory
    |
    +--> Extension registrations: tools, commands, handlers, renderers
    +--> ExtensionRuntime: pre-bind stubs + queued providers
    |
    v
services: register providers before model selection
    |
    v
AgentSession._buildRuntime() --> ExtensionRunner
    |
    +--> bindCore(): session/model actions
    +--> wrapRegisteredTools(): executable adapters
    |
    v
mode calls bindExtensions()
    +--> UI and command-context actions
    +--> session_start --> resources_discover
```

### Loading (`extensions/loader.ts`)

`createExtensionRuntime()` creates shared actions that initially reject
session operations. Registration is allowed during load; provider
registrations are queued until a model runtime is available.

`loadExtensionFromFactory()` supports inline factories;
`loadExtensions()` and `loadExtensionsCached()` import file modules and await
factory completion; `discoverAndLoadExtensions()` supplies conventional
filesystem discovery. `clearExtensionCache()` invalidates cached generations
on reload/cwd change.

The per-extension API separates registration from runtime action delegation.
Pending factory changes are committed only after successful loading;
failed factories discard pending state/subscriptions. `jiti` enables local
TypeScript loading, with host package mapping/virtual modules to share Pi
implementations. Static and regular jiti entry modules support different
packaging environments.

Factories are not a safe place for session-scoped background services:
extensions can load during help, configuration, or package operations that
never start a conversation. Such resources belong in `session_start` or the
command/tool that needs them, with idempotent `session_shutdown` cleanup.

### `ExtensionRunner` method groups

File: `extensions/runner.ts`; constructed by `AgentSession`.

| Methods | Responsibility and timing |
|---|---|
| `bindCore()` | Connect shared extension actions to session/model operations; flush pending providers if still present |
| `bindCommandContext()` | Install host new/fork/resume/reload/wait handlers during mode binding |
| `setUIContext()`, `getUIContext()`, `hasUI()` | Select mode-specific UI; prompt wrappers emit UI lifecycle events |
| `createContext()`, `createCommandContext()` | Construct guarded current-session contexts when dispatching handlers/commands/tools |
| `getAllRegisteredTools()`, `getToolDefinition()` | Resolve tool registration for session registry refresh |
| `getRegisteredCommands()`, `getCommand()`, command diagnostics | Resolve invocation names/collisions for command dispatch and autocomplete |
| `getFlags()`, flag value methods, `getShortcuts()`, shortcut diagnostics | CLI flag integration and configurable keyboard registration |
| Renderer/transformer getters | Custom message, entry, and Markdown presentation lookups by the UI |
| `emit()` | General ordered notification/session-before dispatch; cancellation may short-circuit |
| `emitInput()` | Chain input transforms; `handled` stops input submission |
| `emitBeforeAgentStart()` | Chain structured prompt edits and collect injected messages before a run |
| `emitContext()` | Request-local transforms: conversation-only phase with system restoration, then full-system phase |
| `emitBeforeProviderRequest()`, `emitBeforeProviderHeaders()` | Chain payload replacement or in-place header mutation at request time |
| `emitMessageEnd()` | Chain finalized-message replacement while preserving role |
| `emitToolCall()`, `emitToolResult()`, `emitUserBash()` | Tool interception; result transforms; user-shell dispatch |
| `emitBoundary()` | Chain draft entries and continuation requests, rebuilding a canonical preview after each handler |
| `emitResourcesDiscover()` | Collect extension skill/prompt/theme paths after session start/reload |
| `emitCacheWarmingDecision()` | Last explicit action override wins before a cache refresh |
| `onError()`, `emitError()`, `hasHandlers()` | Host error reporting and cheap handler checks |
| `invalidate()`, `shutdown()` | Reject stale APIs and delegate host shutdown |

Dispatch snapshots handlers in extension-load/registration order. Changes
made while dispatching do not alter that active dispatch. Most handler
failures report an extension error and continue. `tool_call` errors block
execution; `user_bash` errors also block rather than falling through to
local execution.

`context` transforms do not rewrite persisted entries. For durable changes,
use boundary draft entries such as `context_edit` or custom messages. Boundary
continuation is validated against runnable context, preventing a request
from being continued without an appropriate message/queue state. Extensions
still must avoid unconditional continuation loops.

`extensions/wrapper.ts` and `tools/tool-definition-wrapper.ts` adapt a
registered `ToolDefinition` to an `AgentTool` and supply a fresh runner
context. They do **not** implement tool interception; that occurs in the
session's agent hooks.

`event-bus.ts` provides `createEventBus()` (`emit`, `on`, `clear`) for arbitrary
extension-to-extension channels. It is distinct from typed session lifecycle
events. Runtime invalidation releases tracked subscriptions.

## 8. Model runtime, provider composition, and credentials

```text
CLI auth/model UI       Agent request         extension model calls
         |                    |                       |
         +--------------------+-----------------------+
                              v
                         ModelRuntime
                              |
          +-------------------+--------------------+
          |                   |                    |
          v                   v                    v
   ModelConfig         RuntimeCredentials      pi-ai Models
   models.json                |                    |
          |                   v                    v
          |               AuthStorage      composed Provider
          |               auth.json         /     |      \
          |                              chat   image  classifier
          v
 composeModelProvider() <--- built-in / native / extension layers
          |
          v
 remote catalog overlay <--> FileModelsStore (models-store.json)

ModelRegistry = extension-facing facade over the same ModelRuntime
```

### `ModelRuntime`

File: `model-runtime.ts`. Canonical model/auth runtime; implements `Models`.
Created during services or direct SDK construction; reused by a session's
agent, cache warmer, summaries, selectors, and extension facade.

- `create()`: load credentials/config/catalog storage; wrap built-ins with
  remote-catalog behavior; compose providers; optionally refresh.
- `getProviders()`, `getProvider()`, `getModels()`, `getModel()` and typed/all
  variants: synchronous catalog reads.
- `getAvailable()`, typed/all availability, `checkAuth()`: asynchronous
  auth-aware queries. `getAvailableSnapshot()` supports synchronous UI reads.
- `getAuth()`: resolve request-time auth; apply configured model headers and
  optional overrides.
- `stream()`, `streamSimple()`, `complete()`, `completeSimple()`: normalize
  context, prepare auth/headers/base URL, dispatch to the chosen provider.
- `streamDeferred()`, `fetchDeferred()`, `cancelDeferred()`: optional provider
  deferred-response operations.
- `generateImages()`, `classify()`: typed non-chat model operations.
- `login()`, `logout()`, `setRuntimeApiKey()`, `removeRuntimeApiKey()`:
  serialize changes per provider, then synchronize provider composition,
  persisted model state, and availability.
- `refresh()`: reload models config, recompose selected/all providers, refresh
  catalog state, and update availability; respect network/abort options.
- `registerProvider()`, `registerNativeProvider()`, `unregisterProvider()`:
  extension contributions; recompose and refresh without model network.
- Auth/status/error and registered-provider getters: diagnostics and selectors.

Availability refreshes have sequence guards so old asynchronous results do
not overwrite newer credential state. `CredentialSynchronizationError`
distinguishes a completed credential mutation from failure to synchronize
subsequent runtime state.

Network catalog refresh is not required for every startup or inference.
Creation requests it only when configured; services explicitly refresh with
`allowNetwork: false`. Interactive catalog refresh can later enable network.
`PI_OFFLINE` disables model-network behavior.

### Related classes

| Class | Main methods | Caller / purpose |
|---|---|---|
| `ModelConfig` (`model-config.ts`) | Static `load()`, provider getters, `getError()` | Runtime create/refresh; parse commented JSON, schema-validate and freeze provider config; return diagnostics for invalid files |
| `ModelRegistry` (`model-registry.ts`) | `refresh()`, `getAll()`, `getAvailable()`, `find()`, auth/request/provider methods | Runner exposes this facade to extensions; delegates to the same runtime, not a second registry |
| `RuntimeCredentials` (`runtime-credentials.ts`) | Runtime-key setters; `read()`, `list()`, `modify()`, `delete()` | Runtime creates it around a credential store; in-memory API keys override reads without rewriting disk |
| `AuthStorage` (`auth-storage.ts`) | Static `create()`, `fromStorage()`, `inMemory()`; `reload()`, `read()`, `modify()`, `delete()`, `list()` | pi-ai auth via runtime; persist API-key/OAuth credentials and refresh revision-aware snapshots |
| `FileAuthStorageBackend` | `withLock()`, `withLockAsync()` | Auth/storage read-modify-write; cross-process file locks, async cancellation/compromised-lock checks |
| `InMemoryAuthStorageBackend` | Same lock callback interface | Tests/SDK custom storage; serialize asynchronous modifications without filesystem |
| `ReadOnlyAuthStorage` | `read()`, `list()`; mutators reject | Restricted CLI operations; validated snapshot with no writes and no command-key execution |
| `FileModelsStore` (`models-store.ts`) | `read()`, `write()`, `delete()` | pi-ai provider refresh; locked persisted dynamic catalogs using the file-lock backend |
| `InMemoryCodingAgentModelsStore` | Same methods | Runtime without catalog files; cloned values in memory |

Auth storage is storage, not the provider login orchestrator. Login and
refresh behavior belongs to `ModelRuntime` and pi-ai. File credentials and
model catalogs use revision-aware/coalesced reads to avoid repeated locked
loads while allowing external changes to become visible.

### Functional collaborators

- `provider-composer.ts`: `composeModelProvider()` composes built-in/native,
  models.json, and extension layers; models.json `modelOverrides` apply as a
  final user override. `validateExtensionProvider()` validates registration
  before mutation. Request-header/auth-status helpers expose resolved config.
- `remote-catalog-provider.ts`: `withRemoteCatalog()` wraps a provider with
  persisted pi.dev catalog overlays, conditional HTTP requests, refresh
  windows, cancellation-safe publish, and static-data freshness rules.
- `model-resolver.ts`: `resolveCliModel()`, scope/pattern resolvers,
  `findInitialModel()`, `restoreModelFromSession()` implement selection
  policy for CLI/SDK/UI, not inference transport.
- `resolve-config-value.ts`: resolve literal, environment, and command-derived
  key/header values; detect missing variables; cache command results; expose
  uncached/error-producing variants. Used by auth storage and composition.
- `provider-attribution.ts`: merge Pi attribution headers subject to settings
  and provider behavior; SDK invokes it during request-header preparation.
- `auth-guidance.ts`: standard model/auth remediation messages.
- `radius.ts`: shared gateway/provider identifiers and URL helper, also used
  by bug-report upload.

## 9. Settings, trust, resources, and packages

### `SettingsManager` and storage

File: `settings-manager.ts`.

`SettingsManager.create()`, `fromStorage()`, and `inMemory()` construct a
merged settings view. Global settings are the base; trusted project values
override them with recursive object merging. Arrays/scalars replace rather
than merge element-by-element.

- `getGlobalSettings()`, `getProjectSettings()`: cloned scope snapshots.
- `isProjectTrusted()`, `setProjectTrusted()`: control project configuration
  inclusion; untrusted project settings are excluded and writes rejected.
- `reload()`: wait for pending writes, reload scopes, preserve valid state
  on parse failure, recompute merged settings.
- `applyOverrides()`: apply runtime-only overrides to the merged view.
- Feature `get...`/`set...` families: model/thinking, tools/resources/packages,
  retry/compaction, shell, images, terminal/UI, transport, HTTP, cache warming,
  telemetry, and other preferences. Consumers read them at construction,
  per request, or from settings UI depending on the feature.
- `flush()`: wait for queued persistence.
- `drainErrors()`: hand nonfatal storage errors to the host.

`FileSettingsStorage` implements `withLock()` to read/write the global/project
JSON path. `InMemorySettingsStorage` implements the same callback contract
without disk. The manager tracks changed fields/nested keys and merges only
those modifications into the latest locked file contents. This reduces
clobbering when different sessions update different preferences. Invalid
scope files are not silently overwritten by subsequent setters.

### Project trust

`ProjectTrustStore` (`trust-manager.ts`) stores decisions in `trust.json`.
`get()`/`getEntry()` find the nearest explicit ancestor decision;
`set()`/`setMany()` perform locked updates. Path canonicalization avoids
separate decisions for equivalent workspace paths.

`hasTrustRequiringProjectResources()` detects project configuration and
project/ancestor `.agents/skills`. `getProjectTrustOptions()` builds local,
parent-folder, and optional session-only choices.

`resolveProjectTrusted()` (`project-trust.ts`) is called by CLI/runtime
resource bootstrap. Precedence: explicit override; no gated resources;
preloaded trusted-user/CLI extension decision; stored decision; default
policy; UI prompt. Without UI, unresolved ask-mode trust is denied.

Trust gates project settings/resources/packages; it is not an OS sandbox.
Explicit CLI extensions and user/global extensions can run before the
project trust decision. Context-file loading is a separate loader behavior;
trust should not be interpreted as rejecting every text file in the project.

### `DefaultResourceLoader`

File: `resource-loader.ts`; implements the injectable `ResourceLoader`.

- Constructor: retain cwd/settings/options and package/event dependencies.
- `reload()`: resolve trust if requested, reload settings, resolve package and
  explicit paths, load extensions/skills/templates/themes, collect metadata
  and diagnostics, load context and system-prompt sources.
- `loadProjectTrustExtensions()`: temporarily exclude project settings,
  loading only the bootstrap-eligible extensions for trust resolution.
- `extendResources()`: add extension-discovered skill/prompt/theme paths after
  startup/reload without replacing the whole session.
- `getExtensions()`, `getSkills()`, `getPrompts()`, `getThemes()`,
  `getAgentsFiles()`, prompt/source getters: snapshots consumed by session/UI.

The trust bootstrap reuses preloaded factories when building the final
extension set, avoiding duplicate execution merely because trust changed.
Resource metadata tracks scope/source/origin; collision diagnostics do not
necessarily prevent every extension from loading.

Supporting functions:

| Module | Responsibility / invocation |
|---|---|
| `skills.ts` | `loadSkills()`/`loadSkillsFromDir()` discover and validate skill metadata; `formatSkillsForPrompt()` advertises skills while full skill content is loaded on invocation |
| `prompt-templates.ts` | Load Markdown templates; `parseCommandArgs()` and `substituteArgs()` support arguments; `expandPromptTemplate()` runs during input preparation |
| `resource-loader.ts`: `loadProjectContextFiles()` | Load user/project instruction files for prompt construction, not tool execution |
| `source-info.ts` | Convert package metadata or synthetic builtin/SDK sources into provenance for tools/commands/resources |
| `diagnostics.ts`, `settings-diagnostics.ts` | Resource collision/error contracts; collect and deduplicate settings diagnostics for hosts |
| `slash-commands.ts` | Built-in command metadata and shared slash-command types; standard built-in command execution lives in modes |

### `DefaultPackageManager`

File: `package-manager.ts`; implements `PackageManager`.

| Methods | Caller / timing |
|---|---|
| `resolve()`, `resolveExtensionSources()` | Resource loader during startup/reload; configuration CLI while discovering resources |
| `install()`, `remove()`, `update()` | Package CLI and host package workflows on explicit operations |
| `installAndPersist()`, `removeAndPersist()` | Package commands that also modify configured sources |
| `addSourceToSettings()`, `removeSourceFromSettings()` | Configuration/install workflows updating user/project scope |
| `getInstalledPath()`, `listConfiguredPackages()` | Package list/config/update UI |
| `checkForAvailableUpdates()` | Interactive/package update notification workflows |
| `setProgressCallback()` | Host setup for progress/error presentation |

It resolves npm, git, and local sources, manages user/project/temporary
install locations, discovers conventional or manifest-declared resources,
filters enabled paths, and returns `ResolvedPaths` with provenance.
Missing configured sources may be installed during resolution unless the
missing-source/offline policy says otherwise. Discovery is therefore not
necessarily a pure filesystem read.

Project scope normally takes precedence over duplicate user package
identity. `autoload: false` can instead represent a project filtering delta.
Manifest filters and settings patterns distinguish permitted resources from
enabled resources. `pi-manifest.ts` provides the shared `readPiManifest()`
parser. Installation/update implementation delegates to configured package
commands and git; progress printing is owned by the caller.

## 10. Tools: definitions, execution, concurrency, rendering

Tools are factory-created objects, not a class hierarchy. A definition
contains schema, model-facing description, execution, prompt contributions,
and optional UI renderers. A wrapped `AgentTool` contains the execution
contract consumed by agent-core.

```text
AgentSession._buildRuntime()
    |
    v
createAllToolDefinitions(cwd, options)
    +--> built-ins
    +--> extension + SDK definitions (same-name overrides)
    |
    v
_refreshToolRegistry(): apply allow/deny filters
    |
    +--> definitions: schemas / prompt metadata / renderers
    +--> wrappers: executable AgentTool + current context
    |
    v
Agent executes selected tool calls
    |
    +--> beforeToolCall --> tool_call handlers
    +--> definition.execute() --> operations/fs/shell
    +--> afterToolCall --> tool_result handlers + image processing
    +--> message_end --> durable tool result
```

`tools/index.ts` provides per-name factories, coding defaults,
read-only defaults (`read`, `grep`, `find`, `ls`), and all-tool maps.
All definitions can be registered while only a subset is active. Extension
and SDK definitions can replace a builtin with the same name. Unknown names
passed to active-tool selection are ignored; allow/deny filtering also applies
to extension/custom tools.

### Built-ins

| File / factory | Execution responsibility | Invocation |
|---|---|---|
| `read.ts`: `createReadToolDefinition()` | Read text with offset/limit and head truncation; detect/process images; model-aware resize profile | Model tool call or direct SDK-created tool |
| `bash.ts`: `createBashToolDefinition()` | Execute shell command, stream bounded updates, preserve full truncated output, report timeout/abort/nonzero exit | Model tool call |
| `powershell.ts`: `createPowerShellToolDefinition()` | Reuse shell definition/execution/rendering with PowerShell operations | Explicitly enabled tool call |
| `edit.ts`: `createEditToolDefinition()` | Normalize input; match disjoint edits against original content; preserve BOM/line endings; write and return diff/patch | Model tool call |
| `write.ts`: `createWriteToolDefinition()` | Create parent directories and overwrite/create file | Model tool call |
| `grep.ts`: `createGrepToolDefinition()` | Content search with limits and truncation; local ripgrep-backed execution | Explicitly enabled tool call |
| `find.ts`: `createFindToolDefinition()` | Filename discovery with normalized relative results; local fd-backed execution | Explicitly enabled tool call |
| `ls.ts`: `createLsToolDefinition()` | Directory listing with limits/truncation | Explicitly enabled tool call |

Each `create...Tool()` convenience factory wraps its corresponding definition
for direct agent/SDK use. Operations interfaces permit remote or alternate
filesystem/shell/search behavior; local execution is a default, not hardwired
into all callers. Session execution context supplies the current cwd.

### Execution helpers

- `edit-diff.ts`: fuzzy matching, normalization, non-overlap/uniqueness checks,
  unchanged-line preservation, unified/display diff generation, and preview
  functions `computeEditsDiff()`/`computeEditDiff()` for renderers.
- `file-mutation-queue.ts`: `withFileMutationQueue()` serializes the complete
  operation per canonical file path; unrelated files remain parallel.
  Registration is ordered before queue execution. This is an in-process
  coordination mechanism, not a filesystem transaction or cross-process lock.
- `path-utils.ts`: expand/resolve cwd paths; tolerant read-path resolution.
- `truncate.ts`: head/tail/line truncation and shared limits (2000 lines,
  50 KiB defaults); reports truncation metadata.
- `OutputAccumulator` (`output-accumulator.ts`): `append()` incrementally
  decodes UTF-8, `snapshot()` exposes bounded output, `finish()` flushes
  decoding, `closeTempFile()` settles persisted full output, and
  `getLastLineBytes()` supports truncation notices. Shell tool owns one per
  execution. It retains a rolling tail and spills complete output when needed.
- `exec.ts`: `execCommand()` executes an extension-requested command with
  structured output, timeout/cancellation, and process handling.

Parallel tool execution makes mutation coordination necessary. `edit` and
`write` keep their queue reservation until awaited I/O settles, including
when abort arrives, so another operation cannot start while an aborted write
is still completing.

### User shell is a separate path

`AgentSession.executeBash()` uses `bash-executor.ts`'s
`executeBashWithOperations()`, not an assistant tool-call transcript. It emits
shell output updates and records a `BashExecutionMessage`. `recordBashResult()`
also accepts extension-handled results. During streaming, recording is
deferred to maintain call/result ordering.

`messages.ts` converts shell messages into model-visible user text unless
`excludeFromContext` is true (`!!` behavior). Custom, branch-summary, and
compaction-summary roles also convert to user messages. Native system/user/
assistant/tool-result roles pass through. Display visibility and inclusion
in model context are different concepts.

### Tool renderers

`tools/renderers/` contains `renderCall()`/`renderResult()` implementations
for builtin tools. Interactive tool components call them for streamed/final
rendering; experimental presentation can reuse the same registry.

`createAllToolRenderers()` returns the builtin renderer map;
`withBuiltInRenderers()` supplies missing builtin presentation without
changing execution. `render-utils.ts` centralizes path/text/link formatting.
`BashResultRenderComponent` and `WriteCallRenderComponent` are small terminal
component subclasses supporting streamed output and incremental highlighted
write previews. Edit rendering uses asynchronous diff previews.

Renderers do not execute tools or persist history. Separating presentation
from execution permits print/RPC and custom UI hosts to use the same tools.

## 11. Cache warming and accounting

`CacheWarmer` (`cache-warmer.ts`) is created in `sdk.ts`. The real session
stream wrapper calls `start(request, isCurrent)`; summary requests use
separate routing and do not replace the warmed request.

- `start()`: replace earlier warming, check mode/cache lifetime/replay safety,
  arm a refresh timer.
- `status`: derive current scheduling/economic status for displays.
- `onAgentSettled()`: stop streaming-only warming or enter idle phase.
- `onModeChanged()`: reconcile persisted mode changes.
- `cancel()`: abort active warming during teardown.
- Internal `refresh()`: validate request prefix/model, evaluate expected
  savings, ask extension decision handlers, replay with a one-token cap,
  append `cache_warm` usage, and reschedule if still valid.

This is optional, best-effort paid model traffic, not a local cache. Safety
windows, refresh deadlines, replay restrictions, and economic thresholds
prevent indefinite or unsafe refresh. Default mode is streaming; idle mode
allows shorter-horizon warming between runs. Failures do not fail the agent
run. The timer does not keep the process alive.

`cache-stats.ts` exposes `detectCacheMiss()`, `collectCacheMisses()`, and
`computeCacheWaste()` for interactive cost notices/status. These estimate
cache behavior from recorded usage rather than controlling model requests.

`usage-totals.ts` creates/accumulates usage and provides model-attributed
cost breakdowns. `getSessionStats()` aggregates all raw entries, including
compacted and abandoned branches, summary usage, nested tool usage, and
standalone usage entries. `getContextUsage()` instead estimates current
projected context. After compaction it can report unknown token usage until
a trustworthy post-compaction response exists.

## 12. Export, diagnostics, and terminal support

### HTML versus JSONL export

```text
AgentSession.exportToHtml()        exportToJsonl()
    |                                  |
    v                                  v
all entries + active leaf         selected branch
    |                                  |
    +--> custom TUI renderers           v
    |       --> ANSI --> HTML      serializeSessionBranch()
    v                                  |
HTML template + CSS + browser JS        v
    +--> marked / highlight vendor    JSONL file
    |
    v
self-contained HTML file --> browser navigation/rendering
```

- `export-html/index.ts`: `exportSessionToHtml()` exports the stored tree
  and selected leaf; `exportFromFile()` supports CLI standalone export.
  Template assets are resolved through configuration helpers. Session HTML
  export requires an existing persisted conversation file.
- `export-html/tool-renderer.ts`: `createToolHtmlRenderer()` converts custom
  terminal tool rendering into collapsed/expanded HTML.
- `export-html/ansi-to-html.ts`: convert styled terminal output to HTML.
- Browser assets reconstruct/navigate/render exported history; vendor code
  handles Markdown parsing and syntax highlighting.
- `session-export.ts`: `serializeSessionBranch()` emits a versioned header
  and a linearized selected branch, optionally with export-only entries;
  `exportSessionToJsonl()` writes it. JSONL export uses raw branch entries,
  not only the edited/compacted model message projection.

### Diagnostics and reporting

| Module | Caller / timing / responsibility |
|---|---|
| `crash-log.ts` | Interactive crash handling records exceptions and extension-stack matches; later startup retrieves an unnotified crash; read/clear helpers manage the log |
| `bug-report.ts` | Bug-report UI collects redacted metadata and failed-turn diagnostics; creates a bundle; writes ZIP; optional `generateBugReportSummary()` invokes the model when summary sharing is chosen |
| `bug-report-upload.ts` | `uploadBugReport()` posts the selected bundle as multipart data to the gateway after a report workflow requests delivery |
| `settings-diagnostics.ts` | Hosts collect/drain settings errors and deduplicate startup diagnostics |
| `timings.ts` | Startup/resource loader instrumentation: reset named timing streams, record checkpoints, print diagnostics |
| `telemetry.ts` | Policy helper `isInstallTelemetryEnabled()`; not the upload implementation |

Report metadata redaction is distinct from transcript consent. A deliberately
included session transcript is still conversation content; creating a
session does not automatically upload a bug report.

### Terminal/process infrastructure

- `FooterDataProvider` (`footer-data-provider.ts`) is created by interactive
  mode. `getGitBranch()` lazily resolves cached branch state;
  `onBranchChange()` subscribes; filesystem watchers debounce async refresh.
  It handles worktrees/reftable and polling fallbacks. Extension status and
  available-provider methods supply footer data; `setCwd()` rebinds watchers;
  `dispose()` releases them. Extensions receive a read-only view.
- `KeybindingsManager` (`keybindings.ts`) extends the TUI keybinding manager.
  Static `create()` loads user JSON, `reload()` rereads it, and
  `getEffectiveConfig()` returns resolved bindings. Migration and shared
  defaults keep app/editor actions configurable. Interactive modes and
  extension shortcut resolution consume it.
- `output-guard.ts`: `takeOverStdout()` redirects incidental stdout to stderr
  in machine-readable modes. `writeRawStdout()` queues protocol output;
  backpressure/flush methods await delivery; `restoreStdout()` releases the
  takeover. CLI/mode code invokes it; ordinary extension logging must not
  corrupt JSON/RPC output.
- `http-dispatcher.ts`: CLI startup applies proxy settings and configures
  Undici idle timeouts/dispatchers. Interactive HTTP-setting changes can
  reconfigure it. This affects process-level networking, not only one
  session. Existing explicit fetch overrides are preserved.
- `session-cwd.ts`: missing-cwd checks and formatting; runtime validation
  throws `MissingSessionCwdError`, while CLI/UI can offer an override.
- `experimental.ts`: feature-gate helper for experimental callers.
- `defaults.ts`: shared thinking defaults/options, not session state storage.

### Error classes

| Class | Meaning / caller |
|---|---|
| `SessionImportFileNotFoundError` | Runtime import input missing; host displays/remedies it |
| `MissingSessionCwdError` | Persisted workspace unavailable; CLI/runtime/UI decide override behavior |
| `CredentialSynchronizationError` | Credential operation completed but runtime synchronization failed; auth UI distinguishes this from failed credential acquisition |
| Internal `SessionHeaderScanLimitError` | Bounded discovery scan exceeded; discovery skips it, explicit open can fall back to full loading |

## 13. Reload and shutdown boundaries

`AgentSession.reload()` retains the session manager/conversation but changes
the extension/resource generation:

```text
reload command
    |
    v
old session_shutdown(reason=reload)
    |
    v
invalidate old runner --> reload settings + queue policies
    |
    v
reset API providers --> ResourceLoader.reload()
    |
    v
rebuild builtin definitions + runner + tool registry
    |
    v
restore flags/host bindings --> optional host pre-start callback
    |
    v
session_start(reason=reload) --> resources_discover
```

Hosts must invoke reload at an appropriate lifecycle point. Reload is not
itself a replacement of the `AgentSession`, and the method is not a general
transactional rollback mechanism. When host bindings exist, the rebuilt
runner receives the reload start event and discovered resources.

`AgentSession.dispose()` is synchronous cleanup: request cancellation,
invalidate contexts, unsubscribe internal/public listeners, stop cache
warming, and release session-owned model resources. It does not itself emit
an awaited shutdown event. `AgentSessionRuntime.dispose()` handles that
extension shutdown lifecycle first. Runtime replacement also aborts/waits for
active work before shutdown so outgoing finalized messages can persist.

## 14. Architectural conclusions

1. **Canonical history is the key boundary.** Stored entries drive future
   requests; live agent messages are a derived execution view. This enables
   durable context edits and recovery without erasing failed attempts.
2. **Session replacement and branch navigation solve different problems.**
   Replacement changes object/service lifetime; navigation changes a path
   within one persistent tree.
3. **The extension runner is a policy adapter, not the agent loop.** It
   exposes actions and transforms at explicit boundaries; agent-core executes
   requests/tools and core adds application-level continuation/recovery.
4. **Model configuration, credentials, and availability are distinct.**
   Provider composition is credential-independent; request auth is resolved
   late; UI snapshots update asynchronously with stale-result protection.
5. **Resources are executable infrastructure as well as text.** Loading can
   install packages and run extension factories; trust/bootstrap ordering is
   therefore part of architecture, not merely a startup dialog.
6. **There are deliberate process-global surfaces.** HTTP dispatch, stdout
   takeover, extension caches, and some API registrations are not isolated
   per session. In-process SDK hosts should account for these lifetimes.
7. **Presentation is mostly outside execution, but not entirely outside core.**
   Tool renderers, footer/keybindings, themes, and HTML export create UI
   dependencies inside this folder. Alternative hosts can reuse execution
   boundaries without adopting the standard terminal mode.

## 15. Complete file map

### Session and orchestration

| File | Responsibility |
|---|---|
| `index.ts` | Core public exports |
| `sdk.ts` | Direct session composition and SDK-facing factories/exports |
| `agent-session-services.ts` | Cwd-bound infrastructure creation and session-from-services factory |
| `agent-session-runtime.ts` | Active session ownership, replacement/import/fork lifecycle |
| `agent-session.ts` | Conversation orchestration, hooks, recovery, extensions, tools |
| `session-manager.ts` | JSONL entry tree, persistence, migration, discovery, canonical projections |
| `session-cwd.ts` | Stored workspace validation and remediation data |
| `session-export.ts` | Selected raw branch serialization/export |
| `messages.ts` | Coding-specific message roles and provider conversion |
| `system-prompt.ts` | Structured prompt construction and section diffs |
| `defaults.ts` | Thinking defaults/options |
| `usage-totals.ts` | Usage aggregation and attributed cost breakdown |
| `cache-warmer.ts` | Optional model cache replay scheduling/policy |
| `cache-stats.ts` | Cache miss/waste accounting |

### Models, auth, and networking

| File | Responsibility |
|---|---|
| `model-runtime.ts` | Canonical model/auth/request runtime |
| `model-registry.ts` | Extension-facing facade over the runtime |
| `model-config.ts` | Validated models.json configuration |
| `model-resolver.ts` | CLI/default/session/scope model selection |
| `provider-composer.ts` | Provider/config/extension layering and request configuration |
| `remote-catalog-provider.ts` | Static-provider dynamic catalog overlay |
| `models-store.ts` | Persistent/in-memory dynamic catalogs |
| `runtime-credentials.ts` | Runtime API-key overlay over credential storage |
| `auth-storage.ts` | File/read-only/in-memory credential stores and locking |
| `auth-guidance.ts` | User-facing auth/model remediation text |
| `resolve-config-value.ts` | Literal/environment/command configuration resolution |
| `provider-attribution.ts` | Request attribution-header policy |
| `http-dispatcher.ts` | Proxy/idle-timeout process HTTP configuration |
| `radius.ts` | Shared Radius gateway configuration access |

### Configuration and resources

| File | Responsibility |
|---|---|
| `settings-manager.ts` | Scoped settings merge, mutation, storage, diagnostics |
| `trust-manager.ts` | Persistent trust decisions and gated-resource detection |
| `project-trust.ts` | Trust-resolution precedence and extension/UI decision flow |
| `resource-loader.ts` | Extensions/skills/prompts/themes/context discovery and reload |
| `package-manager.ts` | Package installation/resolution/filtering/update/provenance |
| `pi-manifest.ts` | Package resource manifest parsing |
| `skills.ts` | Skill discovery, validation, and prompt advertising |
| `prompt-templates.ts` | Markdown prompt discovery and argument expansion |
| `slash-commands.ts` | Builtin command metadata and shared command contracts |
| `source-info.ts` | Source/scope/origin provenance helpers |
| `diagnostics.ts` | Shared resource diagnostic/collision types |
| `settings-diagnostics.ts` | Host-facing settings diagnostic collection/deduplication |

### Extension subsystem

| File | Responsibility |
|---|---|
| `extensions/index.ts` | Extension public exports and shared contracts |
| `extensions/types.ts` | API, contexts, events/results, tools, UI, renderer and runtime types |
| `extensions/loader.ts` | Import/factory registration, cache, runtime bootstrap, discovery |
| `extensions/runner.ts` | Host binding, contexts, event dispatch/transformation/validation |
| `extensions/wrapper.ts` | Registered tool to executable adapter with current context |
| `extensions/jiti-loader.ts` | Regular jiti entry |
| `extensions/jiti-static-loader.ts` | Static jiti entry for packaged runtime |
| `extensions/virtual-modules.ts` | Host-module mapping for standalone/virtual extension imports |
| `event-bus.ts` | Extension-to-extension event channels |
| `exec.ts` | Structured extension subprocess execution |

### Tool execution and helpers

| File | Responsibility |
|---|---|
| `tools/index.ts` | Builtin names, definitions, wrappers, coding/read-only/all factories |
| `tools/read.ts` | Text/image file reading |
| `tools/bash.ts` | Shared shell execution plus Bash definition/operations |
| `tools/powershell.ts` | PowerShell specialization of shared shell tool |
| `tools/edit.ts` | Targeted replacement execution |
| `tools/write.ts` | Create/overwrite execution |
| `tools/grep.ts` | Content search |
| `tools/find.ts` | Filename search |
| `tools/ls.ts` | Directory listing |
| `tools/edit-diff.ts` | Matching, edit application, diff/patch and preview logic |
| `tools/file-mutation-queue.ts` | Per-file in-process mutation serialization |
| `tools/output-accumulator.ts` | Bounded streaming output and complete-output spill |
| `tools/path-utils.ts` | Cwd/path expansion and tolerant read-path lookup |
| `tools/truncate.ts` | Shared output limits and truncation metadata |
| `tools/tool-definition-wrapper.ts` | Definition/executable conversion |
| `tools/render-utils.ts` | Shared terminal path/text/link rendering helpers |
| `bash-executor.ts` | User-issued shell execution outside assistant tool calls |

### Tool presentation

| File | Responsibility |
|---|---|
| `tools/renderers/index.ts` | Renderer map and builtin fallback decoration |
| `tools/renderers/bash.ts` | Shared shell call/output/timing presentation |
| `tools/renderers/read.ts` | Text/image read presentation |
| `tools/renderers/write.ts` | Incremental highlighted write preview |
| `tools/renderers/edit.ts` | Edit/diff preview and result presentation |
| `tools/renderers/grep.ts` | Search-result presentation |
| `tools/renderers/find.ts` | Filename-result presentation |
| `tools/renderers/ls.ts` | Directory-list presentation |

### Compaction

| File | Responsibility |
|---|---|
| `compaction/index.ts` | Summarization public exports |
| `compaction/compaction.ts` | Token estimation, preparation, history/split-turn summary generation |
| `compaction/branch-summarization.ts` | Abandoned-branch collection/budgeting/summary |
| `compaction/utils.ts` | Conversation serialization and file-operation tracking |

### HTML export

| File | Responsibility |
|---|---|
| `export-html/index.ts` | Session/file to self-contained HTML export |
| `export-html/tool-renderer.ts` | Custom terminal tool renderer to HTML bridge |
| `export-html/ansi-to-html.ts` | ANSI styling conversion |
| `export-html/template.html` | Export document shell |
| `export-html/template.css` | Export styling |
| `export-html/template.js` | Browser history/navigation/message/tool rendering |
| `export-html/vendor/marked.min.js` | Bundled Markdown parser |
| `export-html/vendor/highlight.min.js` | Bundled syntax highlighter |

### UI, diagnostics, and process helpers

| File | Responsibility |
|---|---|
| `footer-data-provider.ts` | Git branch watchers and extension/provider footer state |
| `keybindings.ts` | App defaults, user bindings, migration, resolved configuration |
| `output-guard.ts` | Machine-readable stdout isolation and backpressure |
| `crash-log.ts` | Crash persistence, attribution, notification bookkeeping |
| `bug-report.ts` | Report metadata/diagnostics/redaction/bundling/ZIP/optional summary |
| `bug-report-upload.ts` | Explicit multipart report delivery |
| `telemetry.ts` | Install-telemetry policy check |
| `timings.ts` | Named startup/load timing instrumentation |
| `experimental.ts` | Experimental-feature gate |

## References

Primary integration callers: `../main.ts`, `../package-manager-cli.ts`,
`../modes/interactive/interactive-mode.ts`, `../modes/print-mode.ts`, and
`../modes/rpc/rpc-mode.ts`.

Related repository documentation: `packages/coding-agent/docs/sdk.md`,
`docs/session-format.md`, `docs/extensions.md`, and `docs/packages.md`.
Checked runtime usage example:
`packages/coding-agent/examples/sdk/13-session-runtime.ts`.
