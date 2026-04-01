# Phase 1: Boot Sequence & Entry Points

## Overview

The CLI bootstraps through a layered process designed to minimize startup time. Fast-paths exit before loading the heavy module graph, while the full CLI goes through initialization, Commander.js routing, and REPL rendering.

```
process.argv
  └─ src/entrypoints/cli.tsx  main()
       ├─ fast-path? → handle and exit
       └─ src/main.tsx  main()
            ├─ preAction hook
            │    ├─ MDM + Keychain prefetch
            │    ├─ init()
            │    ├─ initSinks()
            │    ├─ runMigrations()
            │    └─ loadRemoteManagedSettings()
            └─ default command action
                 ├─ permission setup
                 ├─ MCP config
                 ├─ tool assembly
                 └─ launchRepl()
                      └─ src/replLauncher.tsx
                           ├─ dynamic import App
                           ├─ dynamic import REPL
                           └─ renderAndRun()
```

---

## File: `src/entrypoints/cli.tsx`

### Module-Level Side Effects (Lines 1-26)

Before `main()` even runs, the file executes these side-effects at import time:

```typescript
// Line 5: Prevent corepack from polluting package.json
process.env.COREPACK_ENABLE_AUTO_PIN = '0';

// Lines 9-14: Bump heap for CCR (container) environments
if (process.env.CLAUDE_CODE_REMOTE === 'true') {
  process.env.NODE_OPTIONS = existing
    ? `${existing} --max-old-space-size=8192`
    : '--max-old-space-size=8192';
}

// Lines 21-26: Ablation baseline for harness testing
// Sets SIMPLE, DISABLE_THINKING, DISABLE_AUTO_COMPACT, etc.
// feature() gate DCEs this from external builds
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  for (const k of [...]) { process.env[k] ??= '1'; }
}
```

**Why here?** `BashTool`, `AgentTool`, and `PowerShellTool` capture `DISABLE_BACKGROUND_TASKS` into module-level constants at import time. `init()` runs too late. The `feature()` gate ensures dead code elimination in external builds.

### `main()` Function (Lines 33-299)

#### 1. `--version` Fast-Path (Lines 37-42)

```typescript
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v' || args[0] === '-V')) {
  console.log(`${MACRO.VERSION} (Claude Code)`);
  return;
}
```

- **Zero dynamic imports** — `MACRO.VERSION` is inlined at build time by Bun's `define` option.
- This is the fastest possible exit: no module loading beyond this file.

#### 2. Startup Profiler (Lines 44-48)

```typescript
const { profileCheckpoint } = await import('../utils/startupProfiler.js');
profileCheckpoint('cli_entry');
```

All non-`--version` paths load the profiler first. `profileCheckpoint()` records timestamps for performance analysis.

#### 3. `--dump-system-prompt` Fast-Path (Lines 53-71)

- Ant-only feature flag (`feature('DUMP_SYSTEM_PROMPT')`)
- Used by prompt sensitivity evals to extract the rendered system prompt
- Enables configs, gets model, calls `getSystemPrompt()`, prints and exits

#### 4. Chrome MCP / Native Host (Lines 72-93)

Three Chrome-related fast-paths:
- `--claude-in-chrome-mcp`: Runs the Chrome extension MCP server
- `--chrome-native-host`: Runs the native messaging host
- `--computer-use-mcp`: Computer use MCP (feature-gated `CHICAGO_MCP`)

All use dynamic imports to avoid loading Chrome-specific code unless needed.

#### 5. Daemon Worker (Lines 95-106)

```typescript
if (feature('DAEMON') && args[0] === '--daemon-worker') {
  const { runDaemonWorker } = await import('../daemon/workerRegistry.js');
  await runDaemonWorker(args[1]);
  return;
}
```

- Must come before the `daemon` subcommand check
- Workers are lean: no `enableConfigs()`, no analytics sinks
- If a worker needs configs/auth, it calls them inside its `run()` function

#### 6. Bridge / Remote Control (Lines 108-162)

Handles `remote-control`, `rc`, `remote`, `sync`, `bridge` subcommands.

Auth flow:
1. Check OAuth tokens exist (required before GrowthBook check)
2. `getBridgeDisabledReason()` — awaits GrowthBook init
3. Version check via `checkBridgeMinVersion()`
4. Policy limits check: `isPolicyAllowed('allow_remote_control')`
5. Call `bridgeMain(args.slice(1))`

#### 7. Daemon Supervisor (Lines 164-180)

```typescript
if (feature('DAEMON') && args[0] === 'daemon') {
  enableConfigs();
  initSinks();  // Attach logging sinks
  await daemonMain(args.slice(1));
}
```

The daemon supervisor is a long-running process that manages workers.

#### 8. Background Sessions (Lines 182-209)

Handles `ps`, `logs`, `attach`, `kill`, `--bg`, `--background` — all delegate to `cli/bg.js`.

```typescript
const bg = await import('../cli/bg.js');
switch (args[0]) {
  case 'ps':    await bg.psHandler(args.slice(1)); break;
  case 'logs':  await bg.logsHandler(args[1]); break;
  case 'attach': await bg.attachHandler(args[1]); break;
  case 'kill':  await bg.killHandler(args[1]); break;
  default:     await bg.handleBgFlag(args);
}
```

#### 9. Template Jobs / Environment Runner / Self-Hosted Runner (Lines 212-245)

Additional fast-paths for template operations, BYOC runner, and self-hosted runner — all feature-gated.

#### 10. Tmux Worktree (Lines 248-274)

```typescript
if (hasTmuxFlag && (args.includes('-w') || args.includes('--worktree') || ...)) {
  // ... exec into tmux before loading full CLI
}
```

Prevents full CLI loading when the user wants to exec into a tmux session for worktree mode.

#### 11. Flag Corrections (Lines 277-285)

- `--update` / `--upgrade` → redirects to `update` subcommand
- `--bare` → sets `CLAUDE_CODE_SIMPLE=1` early so gates fire during module eval

#### 12. Full CLI Load (Lines 287-298)

```typescript
const { startCapturingEarlyInput } = await import('../utils/earlyInput.js');
startCapturingEarlyInput();  // Start buffering stdin
const { main: cliMain } = await import('../main.js');
await cliMain();
```

`startCapturingEarlyInput()` begins buffering user keystrokes while `main.tsx` loads (~135ms of imports).

---

## File: `src/main.tsx`

### Module-Level Side Effects (Lines 1-20)

Before `main()` runs, three side-effects execute in parallel with the import graph:

```typescript
profileCheckpoint('main_tsx_entry');   // Mark import start
startMdmRawRead();                     // Fire MDM subprocesses (plutil/reg query)
startKeychainPrefetch();               // Fire keychain reads (OAuth + legacy API key)
```

These run during the ~135ms of module evaluation, so by the time `main()` executes, the subprocess results are likely ready.

### `main()` Function (Lines 585-856)

#### Security Setup (Lines 588-606)

```typescript
// Prevent Windows PATH hijacking
process.env.NoDefaultCurrentDirectoryInExePath = '1';

// Initialize warning handler
initializeWarningHandler();

// Reset cursor on exit
process.on('exit', () => resetCursor());

// SIGINT handler (skipped in print mode)
process.on('SIGINT', () => {
  if (process.argv.includes('-p') || process.argv.includes('--print')) return;
  process.exit(0);
});
```

#### URL Handling (Lines 612-677)

- **`cc://` / `cc+unix://` URLs**: Rewrites argv so the main command handles it
- **Deep link URIs**: `--handle-uri` flag for OS protocol handler
- **macOS URL handler**: Detects `__CFBundleIdentifier === 'com.anthropic.claude-code-url-handler'`

#### Assistant / SSH Mode (Lines 679-795)

- `claude assistant [sessionId]` — strips `assistant` from argv, stashes session ID
- `claude ssh <host> [dir]` — strips `ssh` from argv, extracts SSH flags

#### Interactive Detection (Lines 797-855)

```typescript
const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY;
setIsInteractive(isInteractive);
initializeEntrypoint(isNonInteractive);

// Client type detection
const clientType = (() => {
  if (isEnvTruthy(process.env.GITHUB_ACTIONS)) return 'github-action';
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'sdk-ts') return 'sdk-typescript';
  // ... etc
  return 'cli';
})();
setClientType(clientType);

// Settings and run
eagerLoadSettings();
await run();
```

### `run()` Function (Lines 884+)

Creates the Commander.js program:

```typescript
const program = new CommanderCommand()
  .configureHelp(createSortedHelpConfig())
  .enablePositionalOptions();
```

#### `preAction` Hook (Lines 907-967)

Runs before every command action handler:

1. **`ensureMdmSettingsLoaded()` + `ensureKeychainPrefetchCompleted()`**
   - These were started as side-effects at import time (lines 12-20)
   - Awaiting here is nearly free — subprocesses completed during imports

2. **`init()`** — Core initialization:
   - Telemetry setup
   - Auth initialization
   - Settings loading

3. **Terminal title**: `process.title = 'claude'`

4. **`initSinks()`**: Attach logging sinks so `logEvent`/`logError` work for subcommands

5. **`--plugin-dir` handling**: Wires inline plugins via `setInlinePlugins()`

6. **`runMigrations()`**: Settings schema migrations:
   - `migrateAutoUpdatesToSettings`
   - `migrateBypassPermissionsAcceptedToSettings`
   - `migrateFennecToOpus` / `migrateOpusToOpus1m` / `migrateSonnet1mToSonnet45`
   - etc.

7. **`loadRemoteManagedSettings()`**: Non-blocking fetch for enterprise settings

8. **`loadPolicyLimits()`**: Non-blocking fetch for policy limits

#### Command Options (Lines 968-1005)

The default command registers 40+ CLI options including:
- `--debug`, `--verbose`, `--print`, `--bare`
- `--output-format`, `--input-format`
- `--dangerously-skip-permissions`, `--permission-mode`
- `--model`, `--effort`, `--agent`, `--betas`
- `--continue`, `--resume`, `--fork-session`
- `--mcp-config`, `--strict-mcp-config`
- `--system-prompt`, `--system-prompt-file`
- `--add-dir`, `--plugin-dir`
- `--session-id`, `--name`
- `--worktree`, `--tmux`
- `--settings`, `--setting-sources`

#### Default Command Action (Lines 1006+)

The `.action()` handler is the largest function in the codebase (~2800 lines). Key sections:

1. **Bare mode** (1012-1016): Sets `CLAUDE_CODE_SIMPLE=1`
2. **Assistant/Kairos mode** (1033-1089): Trust gate, team init
3. **Options extraction** (1090-1107): Destructure all CLI options
4. **File downloads** (1312-1331): `--file` specs started early
5. **System prompt handling** (1342-1382): `--system-prompt`/`--system-prompt-file`
6. **Permission setup** (1389-1411): `initialPermissionModeFromCLI()`
7. **MCP config parsing** (1413-1523): JSON strings or file paths
8. **Chrome integration** (1525-1577): Claude in Chrome setup
9. **Enterprise MCP check** (1582-1595): Policy enforcement
10. **Resume/continue** (3200-3700): Session recovery
11. **Tool assembly**: `getTools()` + `assembleToolPool()`
12. **REPL launch**: `launchRepl()` with all collected props

---

## File: `src/replLauncher.tsx`

This tiny file (22 lines) is the bridge between the CLI setup and the React terminal UI:

```typescript
type AppWrapperProps = {
  getFpsMetrics: () => FpsMetrics | undefined;
  stats?: StatsStore;
  initialState: AppState;
};

export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>,
): Promise<void> {
  const { App } = await import('./components/App.js');
  const { REPL } = await import('./screens/REPL.js');
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>);
}
```

### Why Dynamic Imports?

- `App.js` and `REPL.js` are the heaviest components (they pull in Ink, React, all tool renderers)
- They're only needed for interactive mode
- Print mode (`-p`) never loads these — it uses `print.ts` directly

### Props Flow

```
AppWrapperProps:
  - getFpsMetrics: FPS counter callback
  - stats: Statistics store
  - initialState: Initial AppState

REPLProps (from screens/REPL.tsx):
  - messages: Conversation history
  - tools: Assembled tool pool
  - systemPrompt: System prompt string
  - permissionContext: Tool permissions
  - modelConfig: Model, effort, thinking
  - mcpConfig: MCP server configs
  - ... many more
```

### Component Hierarchy

```
<Root> (Ink root)
  └─ <App> (context providers)
       ├─ FpsMetricsProvider
       ├─ StatsProvider
       ├─ AppStateProvider
       └─ <REPL> (main screen)
            ├─ LogoHeader
            ├─ StatusNotices
            ├─ VirtualMessageList
            │    └─ Messages
            └─ PromptInput
```

---

## Key Questions Answered

### What fast-paths exist before the full CLI loads?

| Fast-path | Trigger | Modules Loaded |
|-----------|---------|---------------|
| `--version` | Single arg match | None (MACRO inlined) |
| `--dump-system-prompt` | Feature flag + arg | config, model, prompts |
| `--claude-in-chrome-mcp` | Exact argv match | Chrome MCP server |
| `--daemon-worker` | Feature flag + arg | daemon/workerRegistry |
| `remote-control`/`bridge` | Feature flag + arg | auth, bridge, policy |
| `daemon` | Feature flag + arg | config, sinks, daemon |
| `ps`/`logs`/`attach`/`kill` | Feature flag + arg | config, bg |
| `--tmux --worktree` | Both flags present | config, worktree |
| `--bare` | Flag present | None (sets env var) |

### How does `preAction` initialize state before any command runs?

1. Awaits MDM + keychain prefetches (started at module import time)
2. Calls `init()` for telemetry, auth, settings
3. Attaches logging sinks
4. Wires `--plugin-dir` for subcommands
5. Runs settings migrations
6. Loads remote managed settings (non-blocking)
7. Loads policy limits (non-blocking)

### How are arguments routed to REPL vs subcommands?

Commander.js routes based on the first positional argument:
- Known subcommands (`mcp`, `plugin`, `auth`, `doctor`, `update`) → their own action handlers
- No subcommand (default) → the `.action()` handler which sets up the REPL
- Fast-paths in `cli.tsx` intercept before Commander.js even loads
