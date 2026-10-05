# Coding-Agent CLI architecture

## Summary

This folder is the command-line boundary, not the agent execution engine.
It parses arguments, prepares prompts/files, checks or prints credentials,
lists models, and supplies short-lived terminal dialogs for startup, trust,
session selection, and resource configuration. `main.ts`, outside the folder,
coordinates these helpers and creates the agent runtime and selected mode.

There are two distinct parsers:

- `args.ts`: normal agent options, positional messages, `@files`, and unknown
  long options preserved for extensions.
- `experimental/command.ts`: typed server/client command tree with explicit
  option declarations, builders, and injected execution callbacks.

## Architecture diagram

ASCII diagram lines are no wider than 75 characters.

```text
src/cli.ts
    |
    +-> setupCli() -> process identity + HTTP dispatcher
    |
    v
main(argv)
    |
    +-> auth command? -> auth parser -> ModelRuntime -> stdout
    |
    +-> package/config command? -> package-manager-cli.ts
    |                                  |
    |                         trust UI / selectConfig()
    |
    +-> parseArgs() -> diagnostics / version / export
    |
    +-> resolve app mode + migrations + startup settings
    |                                  |
    |                         optional first-time setup
    |
    +-> createSessionManager() -> optional selectSession()
    |
    +-> target cwd -> trust context -> runtime services
    |
    +-> help / listModels? -> output and exit
    |
    +-> stdin + processFileArguments() + buildInitialMessage()
    |
    v
InteractiveMode / runPrintMode / runRpcMode
    |
    v
Agent session/runtime (outside this folder)
```

## Normal CLI: who calls what, and when

### 1. Process setup and early command dispatch

`src/cli.ts` calls `setupCli()` and then `main(process.argv.slice(2))`.

`setupCli()` sets the process title, `PI_CODING_AGENT=true`, and `AI_AGENT=pi`,
suppresses `process.emitWarning`, and configures the HTTP dispatcher before
provider requests. Settings-specific HTTP configuration is applied later.

`main()` first recognizes offline mode, then handles auth commands before
normal settings/bootstrap work. After install cleanup and global proxy
bootstrap, it dispatches package commands and `config` through
`package-manager-cli.ts`. These do not pass through normal `parseArgs()`.

For a regular agent invocation, `main()` calls `parseArgs()`, prints collected
diagnostics, and exits on errors. Version and HTML export are early exits.

### 2. Parsing normal arguments

`parseArgs()` returns structured `Args`; it does not start the agent or print
errors itself.

- Parses model/provider/thinking, session, tools, resource paths, prompts,
  output mode, theme, trust, and offline controls.
- Collects positional text in `messages` and strips `@` into `fileArgs`.
- `--` ends option parsing, but trailing `@...` still denotes a file.
- `-p` can consume the next prompt directly, including a `---` prefix.
- Preserves unknown long flags as boolean/string values for extension
  registration; `--flag=value` is supported for this unknown-flag path.
- Reports unknown short options as errors. Invalid thinking produces a
  warning; invalid mode/TUI mode and missing theme/name values produce errors.

Unknown long flags can consume the next non-flag/non-file argument. Therefore
an unrecognized option may take text that otherwise would have been a prompt.
Do not interpret this parser as uniform strict validation: many recognized
value options only check whether another argument exists.

`printHelp()` is called by `main()` **after runtime/resource loading**, so it
can include extension-registered flags. `parseArgs` and `Args` are also exported
from the package root; `core/model-resolver.ts` uses `isValidThinkingLevel()`.

### 3. Mode, startup UI, session, and trust

`main()` selects RPC for `--mode rpc`, JSON for `--mode json`, print mode for
`-p` or non-TTY stdin/stdout, otherwise interactive mode. Noninteractive output
is guarded unless this is a plain help/model-list command. RPC rejects `@file`
arguments and reserves stdin for its protocol.

After migrations, startup settings are loaded. For interactive startup only,
`shouldRunFirstTimeSetup()` permits setup when all these conditions hold:

- Official package/application/config-directory identity.
- Experimental features enabled.
- No custom agent-directory environment override.
- Settings file absent.

`showFirstTimeSetup()` previews/detects terminal theme and persists theme and
analytics choice on submission; cancellation does not persist a result.

`main.createSessionManager()` resolves ephemeral, fork, explicit session,
resume, continue, exact-ID, or new-session selection. For `--resume`, it calls
`selectSession()` with current-project and all-project loaders. The picker
passes progress and cancellation support to `SessionSelectorComponent` and
returns a path or `null`. It stops its TUI on selection/cancellation;
`createSessionManager()` stops the theme watcher in `finally`.

The selected session's cwd is established **before** cwd-bound settings,
resources, providers, and models are created. A missing saved cwd prompts
interactive users with `showStartupSelector()`; noninteractive startup fails.

During service resource loading, `main()` supplies `createProjectTrustContext()`
to core trust resolution when needed. Package/config command settings use the
same helper. This context adapts select/confirm/input to startup dialogs only
when `hasUI` and interactive mode are both true. Otherwise it returns no
selection/input and false confirmation; noninteractive notifications use
stderr. It does not itself decide or persist project trust.

### 4. Help/model listing and prompt preparation

After runtime creation, `main()` exits through `printHelp()` or `listModels()`
if requested. `listModels()` asks ModelRuntime for available models, optionally
fuzzy-filters provider plus ID, sorts by provider/ID, and prints context window,
maximum output, reasoning, and image capability columns. Load errors become
warnings; empty/unmatched results get explanatory text.

For an actual run, `main()` reads and trims piped stdin unless in RPC mode.
Its `prepareInitialMessage()` calls `processFileArguments()` when files exist,
then calls `buildInitialMessage()`.

`processFileArguments()` processes files sequentially:

1. Resolve against `process.cwd()`, expanding read-path conveniences such as
   tilde and screenshot Unicode spaces.
2. Check access and skip zero-byte files.
3. Detect supported image content; process it into image attachments and
   textual file references/hints.
4. Otherwise read UTF-8, strip BOM, and wrap text in `<file name="...">`.

Missing/unreadable text files terminate the process with code 1. Unsuccessful
image processing contributes an explanatory file block instead. Its default
is automatic image resizing, but normal startup explicitly disables resizing
here: AgentSession does it after extension hooks choose the request model.

`buildInitialMessage()` concatenates stdin, file text, and the **first** CLI
message using `join("")`, with no inserted separator. It shifts that first
message out of `parsed.messages`; later messages remain for the selected mode.
It returns images only when nonempty. Although its comment says noninteractive,
`main()` also uses this preparation for interactive startup.

Then `main()` initializes themes, handles diagnostics/model availability, and
calls InteractiveMode, print mode, or RPC mode. Agent requests and tool
execution are responsibilities of those modes/runtime, not this folder.

## Authentication commands

```text
main() -> runAuthCommand(argv)
                 |
                 +-> auth help -> printAuthCommandHelp()
                 |
                 +-> parseAuthCommand() -> parseArgs(common options)
                               |
                +--------------+----------------+
                |                               |
          auth check                      print credential
                |                               |
 ReadOnlyAuthStorage or AuthStorage        ModelRuntime.create()
                |                          15-second timeout
 createAuthCheckModelRuntime()                   |
                |                   resolveCredentialForPrint()
 checkProviderAuth(refresh?)                     |
                |                     exactly one credential
 optional getProviderCredential()                |
                +---------------+---------------+
                                |
                    stdout result / secret + exit code
```

### Parsing and shared helpers

`parseAuthCommand()` recognizes `check`, `print-api-key`, and
`print-bearer-token`. It extracts check-only `--json`, `--credentials`, and
`--no-refresh`; bearer-only `--min-expiry` accepts integer `ms/s/m/h` durations.
Remaining options go through normal `parseArgs()`.

`validateAuthCommandArgs()` requires provider or model and rejects unknown
flags, explicit API-key overrides, messages, and files. Despite its error text,
it does not reject every other recognized normal CLI flag.

`getAuthCredential()` extracts `auth.apiKey`, or a case-insensitive
Authorization header containing a Bearer token. Besides CLI auth callers,
experimental Radius authentication and interactive sharing/bug-report helpers
reuse it.

### Readiness checking

`runAuthCommand()` creates a read-only credential store for `--no-refresh`,
otherwise a normal AuthStorage. `createAuthCheckModelRuntime()` disables model
network loading and refresh-on-create and uses an in-memory models store.
This avoids catalog startup work; explicit OAuth refresh can still occur.

`checkProviderAuth()` resolves provider/model, detects invalid runtime state
or missing provider/credentials, and optionally calls `getAuth()` to refresh.
The helper's default is **no refresh**, but the CLI explicitly enables refresh
unless `--no-refresh` was supplied. This is credential readiness checking,
not a model-completion probe against a provider.

With `--credentials`, `getProviderCredential()` extracts the usable credential.
For no-refresh OAuth, it reads the saved access token directly. Missing usable
credentials changes readiness to `not_ready`.

CLI output is status, credential, or JSON, with exit codes:

- `ready`: 0
- `not_ready`: 1
- `invalid` or check argument/resolution failure: 2

### Printing a credential

`resolveCredentialForPrint()` lists configured credential types and resolves
an explicit provider/model or searches configured providers for a model match.
API-key printing skips OAuth credentials; bearer printing requires OAuth.
It calls normal ModelRuntime auth resolution, which can refresh/persist OAuth.
Bearer printing requests at least 30 minutes of validity by default, overridable
with `--min-expiry`.

Exactly one usable credential succeeds. Zero matches or multiple matching
providers fail, asking for an explicit provider where necessary. `main()` writes
the secret plus newline to stdout; errors go to stderr and set exit code 1.
These commands intentionally emit secrets; auth checks do so only when asked.

## Short-lived UI helpers

`createStartupTui()` is shared by session selection, trust dialogs, and setup.
It loads enabled **global** themes with project trust disabled, skips missing
package installation, ignores broken startup themes, sets capability overrides
and keybindings, and creates a main-screen TUI with cursor/shrink settings.

`startStartupTui()` starts the terminal and asynchronously checks automatic
terminal theme, with a 100 ms detection timeout. Selector/input helpers clear,
render, wait briefly, and stop before resolving; input also disposes its
component. First-time setup handles its own initial detection/preview flow.

`selectConfig()` uses a separate TUI path. `handleConfigCommand()` resolves
trust and global/project package resources, rejects untrusted local edits,
and passes scope/availability to it. The selector initializes the theme,
focuses the resource list, and stops TUI/theme watching on close. Resource
editing behavior belongs to `ConfigSelectorComponent`, outside this folder.

## Experimental server/client CLI

This path is development-only, not imported by the published `src/cli.ts`.

```text
src/experimental/cli.ts
             |
          setupCli()
             |
 runExperimentalCommand(argv)
             |
       feature enabled AND first arg is server/client?
             |
       +-----+-----------------------+
       | yes                         | no
       v                             v
 cli.execute(argv, context)        main(argv)
       |
 Command selects child -> parses options -> builder validates
       |
       +-> server action -> context.runServer(command)
       |                          |
       |                 startForegroundServer()
       |                 wait signal/close -> cleanup
       |
       +-> client action -> context.runClient(command)
                                  |
                     no prompt + TTY -> client TUI
                     otherwise -> client runtime + stdout
```

`experimental/cli.ts` composes server/client children beneath a named
`experimental` Command. The actual dispatcher passes arguments beginning with
`server` or `client`, not an extra `experimental` token.

`Command.parse()` selects a child by first argument. `execute()` parses and
validates before invoking its registered action with injected context. Errors
from parsing/building are returned; thrown action errors are caught by the
external dispatcher.

The option parser supports `--name=value`, repeatable options, duplicate checks,
and valueless flags. It stops at the first unknown argument or `--`, leaving
the remainder for the command builder. Unlike normal parsing, it retains the
`--` marker. Thus recognized options must precede positional prompt text.

Shared command options validate:

- Inline auth token versus token-file path: mutually exclusive; token-file
  contents are not read by this parser.
- `radius://<server-id>`: canonical address with lowercase UUIDv4 server ID,
  no credentials, port, query, fragment, or additional path.
- `unix:///absolute/path`: no authority/query/fragment, canonical URL shape,
  decoded absolute POSIX path, no NUL.

**Server builder:** accepts server ID, session directory, provider/model,
repeatable `-e` plugin packages, and auth input. Provider requires model;
trailing/unsupported options are rejected. Its action calls `context.runServer`.
The external dispatcher starts a foreground server, prints server/socket and
relay status, waits for SIGINT/SIGTERM or runtime closure, then closes runtime.

**Client builder:** accepts connection, session ID, continue/resume aliases,
provider/model, repeated `-e`, auth, and at most one nonempty prompt. Session
ID/continue/resume are mutually exclusive; provider requires model. `--` permits
a dash-prefixed prompt. Its action calls `context.runClient`. The dispatcher
chooses TUI only for no prompt with both stdin/stdout TTY; otherwise it streams
text deltas or prints attached/session-list results. The experimental entrypoint
explicitly exits after client dispatch.

This command layer validates input and delegates execution. It does not itself
open sockets, authenticate Radius, supervise servers, or implement the client.

## File map: responsibility, caller, timing

Paths are relative to `packages/coding-agent/src/cli/`.

| File | Responsibility | Who calls it / when |
| --- | --- | --- |
| `setup.ts` | Process identity, warning suppression, initial HTTP dispatcher. | Normal and experimental executable entrypoints, before dispatch. |
| `args.ts` | Normal argument parser, diagnostics, help, session-name/thinking helpers. | `main()` during normal/auth parsing; help after resources load; SDK exports and model resolver reuse helpers. |
| `auth-command.ts` | Auth subcommand parse/validation/help and credential extraction. | `main.runAuthCommand()` at earliest dispatch; credential helpers and sharing/Radius integrations reuse extraction. |
| `auth-check.ts` | Credential readiness check, optional extraction, lightweight ModelRuntime creation. | `main.runAuthCommand()` for `auth check`, before agent runtime setup. |
| `credential-print.ts` | Resolve one provider API key or OAuth token with validity policy. | Auth dispatcher for credential-print commands, under a 15-second timeout. |
| `file-processor.ts` | Convert `@files` into wrapped text and image attachments. | `main.prepareInitialMessage()` after runtime creation, before mode execution. |
| `initial-message.ts` | Merge stdin/files/first prompt; consume first parsed message. | `main.prepareInitialMessage()` for initial interactive or print-mode prompt preparation. |
| `list-models.ts` | Available-model query, fuzzy search, sorted capability table. | `main()` after runtime/resource loading when `--list-models` is present. |
| `startup-ui.ts` | Global-theme startup TUI, setup gate, selector/input/setup dialogs. | `main()` setup/missing-cwd handling; trust context and session picker during startup. |
| `session-picker.ts` | Resume dialog around injected session loaders. | `main.createSessionManager()` for `--resume`, before target runtime services. |
| `project-trust.ts` | Adapt trust UI contract to startup dialogs or noninteractive fallback. | Main resource loading and package/config settings resolution when trust is evaluated. |
| `config-selector.ts` | TUI wrapper for scoped resource enable/disable selection. | `package-manager-cli.handleConfigCommand()` after trust/resource resolution. |
| `experimental/cli.ts` | Compose typed server/client command tree. | External experimental dispatcher when feature gate and command match. |
| `experimental/command.ts` | Generic option registration, child dispatch, parse/build/action framework. | Experimental command definitions at module load; dispatcher at parse/execute time. |
| `experimental/command-options.ts` | Shared auth/transport parsing and unsupported-remainder validation. | Experimental server/client builders during argument validation. |
| `experimental/commands/server.ts` | Server option declarations, validation, injected action. | Command tree selects `server`; action delegates after successful build. |
| `experimental/commands/client.ts` | Client/session/prompt declarations, validation, injected action. | Command tree selects `client`; action delegates after successful build. |

## Key architectural boundaries

- Main orchestration, SessionManager persistence, resource/trust policy,
  provider authentication, and agent execution are outside this folder.
- Global startup themes are loaded without project trust; target-session cwd
  determines later project-local runtime configuration.
- Help/model-listing are not necessarily cheap parse-only paths: runtime and
  extensions load first. Version/export and auth have earlier exits.
- RPC does not consume prompt stdin and rejects file attachments; normal print
  and interactive startup use the same initial-message builder.
- UI wrappers manage terminal startup/shutdown; imported components implement
  selection, editing, session loading, and interaction behavior.

This is a source-based architecture summary, not a test run or full correctness
or security audit. No source code was changed.
