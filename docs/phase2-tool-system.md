# Phase 2: Tool System Architecture

## Overview

The tool system is built around a declarative pattern: each tool is a plain object conforming to the `Tool` interface, constructed via the `buildTool()` factory. Tools declare their own capabilities (read-only, concurrency-safe, destructive) and the orchestration layer uses these declarations to partition work into concurrent and serial batches.

```
buildTool(def) → Tool object
     │
     ├─ inputSchema (Zod)
     ├─ call() → ToolResult
     ├─ checkPermissions()
     ├─ isReadOnly() / isConcurrencySafe() / isDestructive()
     ├─ renderToolUseMessage() / renderToolResultMessage()
     └─ prompt() / description()
          │
getAllBaseTools() → built-in tools
          │
getTools(permissionContext) → filtered tools
          │
assembleToolPool(permissionContext, mcpTools) → full pool
          │
runTools() → partition → serial/concurrent execution
```

---

## The `Tool` Interface

**File:** `src/Tool.ts:362-695`

Every tool implements this TypeScript interface. The key methods fall into several categories:

### Core Lifecycle Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `call()` | `(args, context, canUseTool, parentMessage, onProgress?) => Promise<ToolResult<Output>>` | Execute the tool. Returns typed result. |
| `description()` | `(input, options) => Promise<string>` | Dynamic description shown to the model |
| `inputSchema` | `z.ZodType` (readonly) | Zod schema for input validation |
| `inputJSONSchema` | `ToolInputJSONSchema` (readonly, optional) | Direct JSON Schema for MCP tools |

### Capability Declaration Methods

| Method | Default | Purpose |
|--------|---------|---------|
| `isEnabled()` | `() => true` | Is this tool available? |
| `isReadOnly(input)` | `() => false` | Does this tool only read? |
| `isConcurrencySafe(input)` | `() => false` | Can this run alongside other tools? |
| `isDestructive(input)` | `() => false` | Does it perform irreversible operations? |
| `interruptBehavior()` | `'block'` | What happens when user sends new message |

### Permission Methods

| Method | Purpose |
|--------|---------|
| `checkPermissions(input, context)` | Per-tool permission check, returns `{ behavior, updatedInput }` |
| `validateInput(input, context)` | Pre-permission validation, returns `ValidationResult` |
| `preparePermissionMatcher(input)` | Prepares hook `if` condition matcher |

### Rendering Methods (React)

| Method | Purpose |
|--------|---------|
| `renderToolUseMessage(input, options)` | Renders the tool input in the transcript |
| `renderToolResultMessage(content, ...)` | Renders the tool output in the transcript |
| `renderToolUseProgressMessage(...)` | Renders progress while tool runs |
| `renderToolUseRejectedMessage(...)` | Custom rejection UI |
| `renderToolUseErrorMessage(...)` | Custom error UI |
| `renderGroupedToolUse(...)` | Renders multiple parallel instances as a group |
| `renderToolUseTag(input)` | Optional tag after tool use (timeout, model, etc.) |

### Metadata Methods

| Method | Purpose |
|--------|---------|
| `prompt(options)` | System prompt fragment describing this tool to the model |
| `userFacingName(input)` | Display name in the UI |
| `userFacingNameBackgroundColor(input)` | Color theme key |
| `getActivityDescription(input)` | Spinner text (e.g., "Reading src/foo.ts") |
| `getToolUseSummary(input)` | Compact summary for condensed views |
| `toAutoClassifierInput(input)` | Compact representation for auto-mode security |
| `extractSearchText(output)` | Text for transcript search indexing |
| `isResultTruncated(output)` | Whether non-verbose view hides content |

### Special Properties

| Property | Type | Purpose |
|----------|------|---------|
| `name` | `string` | Unique tool identifier |
| `maxResultSizeChars` | `number` | Overflow threshold for disk persistence |
| `aliases` | `string[]` | Backward-compatible aliases |
| `searchHint` | `string` | Keyword hint for ToolSearch |
| `shouldDefer` | `boolean` | Requires ToolSearch before use |
| `alwaysLoad` | `boolean` | Never defer, always in initial prompt |
| `isMcp` | `boolean` | Is this an MCP tool? |
| `isLsp` | `boolean` | Is this an LSP tool? |
| `mcpInfo` | `{ serverName, toolName }` | Original MCP server/tool names |
| `strict` | `boolean` | Enable strict API mode |

---

## `TOOL_DEFAULTS` and `buildTool()`

**File:** `src/Tool.ts:757-792`

### TOOL_DEFAULTS

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,   // Default: NOT safe
  isReadOnly: (_input?: unknown) => false,            // Default: writes
  isDestructive: (_input?: unknown) => false,         // Default: not destructive
  checkPermissions: (input, _ctx?) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}
```

Every tool starts with these defaults. Tools only need to override what's different.

### `buildTool()` Factory

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,  // Override default with tool name
    ...def,                           // Then spread tool-specific definition
  } as BuiltTool<D>
}
```

The order matters: `TOOL_DEFAULTS` first, then `userFacingName` override, then the tool definition. This means:
- Every tool gets `checkPermissions` that allows by default
- Unless the tool definition provides its own `checkPermissions`
- Every tool defaults to NOT concurrency-safe and NOT read-only

---

## Tool Registration

**File:** `src/tools.ts`

### `getAllBaseTools()` (Lines 193-251)

Returns the full array of ~40+ built-in tools:

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool] : []),
    ...(process.env.USER_TYPE === 'ant' ? [TungstenTool] : []),
    ...(SuggestBackgroundPRTool ? [SuggestBackgroundPRTool] : []),
    ...(WebBrowserTool ? [WebBrowserTool] : []),
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
    // ... many more conditionally included tools
    getSendMessageTool(),
    BriefTool,
    ...(getPowerShellTool() ? [getPowerShellTool()] : []),
    ListMcpResourcesTool,
    ReadMcpResourceTool,
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

Key observations:
- Ant-only tools are gated by `process.env.USER_TYPE === 'ant'`
- Some tools are feature-gated (WebBrowserTool, SuggestBackgroundPRTool)
- Glob/Grep are omitted when embedded search tools (bfs/ugrep) are available in the bun binary
- TodoV2 tools are conditionally included based on feature flag
- `getSendMessageTool()` and `getPowerShellTool()` are factory functions that return tool instances

### `getTools(permissionContext)` (Lines 271-327)

Filters the base tools based on mode and permissions:

```typescript
export const getTools = (permissionContext: ToolPermissionContext): Tools => {
  // Simple mode: only Bash, Read, and Edit (or REPL)
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    if (isReplModeEnabled() && REPLTool) {
      return filterToolsByDenyRules([REPLTool, ...], permissionContext);
    }
    return filterToolsByDenyRules([BashTool, FileReadTool, FileEditTool], permissionContext);
  }

  // Full mode: get all base tools, filter by:
  // 1. Remove special tools (ListMcpResources, ReadMcpResource, SyntheticOutput)
  // 2. Filter by deny rules from permission context
  // 3. If REPL mode, hide primitive tools that REPL wraps
  // 4. Check isEnabled() for each tool

  const tools = getAllBaseTools().filter(tool => !specialTools.has(tool.name));
  let allowedTools = filterToolsByDenyRules(tools, permissionContext);

  if (isReplModeEnabled()) {
    allowedTools = allowedTools.filter(tool => !REPL_ONLY_TOOLS.has(tool.name));
  }

  const isEnabled = allowedTools.map(_ => _.isEnabled());
  return allowedTools.filter((_, i) => isEnabled[i]);
}
```

### `assembleToolPool(permissionContext, mcpTools)` (Lines 345-367)

Combines built-in tools with MCP tools:

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext);
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext);

  // Sort each partition alphabetically for prompt-cache stability
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name);
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',  // built-ins win on name conflict
  );
}
```

**Why sort?** The server's `claude_code_system_cache_policy` places a global cache breakpoint after the last prefix-matched built-in tool. Interleaving MCP tools between built-ins would invalidate all downstream cache keys.

**Why `uniqBy`?** Built-in tools take precedence over MCP tools with the same name. The `uniqBy` preserves insertion order.

---

## Example Tools

### GlobTool (Simple Read-Only)

**File:** `src/tools/GlobTool/GlobTool.ts`

```typescript
export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME,
  searchHint: "find files by name pattern or wildcard",
  maxResultSizeChars: 100_000,
  // strict: false (default)

  inputSchema: z.object({
    pattern: z.string().describe("The glob pattern..."),
    path: z.string().optional().describe("The directory to search in."),
  }),

  isReadOnly: () => true,
  isConcurrencySafe: () => true,

  checkPermissions: (input, context) =>
    checkReadPermissionForTool(input.path, context),

  async call(args, context, canUseTool, parentMessage, onProgress) {
    const startMs = Date.now();
    const results = await glob({
      pattern: args.pattern,
      path: safeResolvePath(args.path),
      limit: 100,
      signal: context.abortController.signal,
      permissionContext: context.toolPermissionContext,
    });

    return {
      output: {
        filenames: results.map(f => relative(cwd, f)),
        durationMs: Date.now() - startMs,
        numFiles: results.length,
        truncated: results.length >= 100,
      },
    };
  },
  // ... render methods
});
```

**Key patterns:**
- Read-only + concurrency-safe = can run in parallel with other reads
- Uses `checkReadPermissionForTool` for path-based permission check
- Returns relative paths to save tokens
- Self-bounded output (max 100 results)

### GrepTool (Read-Only Search with Multiple Output Modes)

**File:** `src/tools/GrepTool/GrepTool.ts`

```typescript
export const GrepTool = buildTool({
  name: GREP_TOOL_NAME,
  searchHint: "search file contents with regex (ripgrep)",
  maxResultSizeChars: 20_000,
  strict: true,  // Strict API mode

  inputSchema: z.object({
    pattern: z.string(),
    path: z.string().optional(),
    glob: z.string().optional(),
    output_mode: z.enum(['content', 'files_with_matches', 'count']),
    "-B": z.number().optional(),
    "-A": z.number().optional(),
    "-C": z.number().optional(),
    "-i": z.boolean().optional(),
    type: z.string().optional(),
    head_limit: z.number().optional(),
    offset: z.number().optional(),
    multiline: z.boolean().optional(),
  }),

  isReadOnly: () => true,
  isConcurrencySafe: () => true,

  async call(args, context, ...) {
    // Constructs ripgrep command line
    // Executes via ripgrep() utility
    // Processes results based on output_mode
    // Converts to relative paths
    // Applies head_limit and offset
  },
});
```

**Key patterns:**
- Three output modes serve different use cases
- Wraps ripgrep for performance
- Pagination via `head_limit`/`offset`
- Strict mode enabled for better parameter adherence

### BashTool (Complex Write Tool)

**File:** `src/tools/BashTool/BashTool.tsx`

```typescript
export const BashTool = buildTool({
  name: BASH_TOOL_NAME,
  searchHint: "execute shell commands",
  maxResultSizeChars: 30_000,
  strict: true,

  inputSchema: z.object({
    command: z.string(),
    timeout: z.number().optional(),
    description: z.string().optional(),
    run_in_background: z.boolean().optional(),
    dangerouslyDisableSandbox: z.boolean().optional(),
    _simulatedSedEdit: ... // internal for pre-computed edits
  }),

  isReadOnly(input) {
    // Uses checkReadOnlyConstraints() — parses command AST
    // Determines if command is safe (no writes, no cd)
    return checkReadOnlyConstraints(input.command);
  },

  isConcurrencySafe(input) {
    return this.isReadOnly(input);  // Only safe if read-only
  },

  checkPermissions(input, context) {
    // bashToolHasPermission():
    // 1. Parse command security AST
    // 2. Match against permission rules
    // 3. Handle sed edit simulations
    // 4. Validate sandbox requirements
    return bashToolHasPermission(input, context);
  },

  async call(args, context, canUseTool, parentMessage, onProgress) {
    // 1. Handle simulated sed edits (bypasses shell)
    // 2. Spawn shell via runShellCommand() generator
    // 3. Track progress with onProgress callback
    // 4. Handle background tasks
    // 5. Merge stdout/stderr
    // 6. Handle large output persistence (>30K chars → file)
    // 7. Return structured result
  },
});
```

**Security features:**
- Commands can be sandboxed (network/filesystem isolation)
- Sleep patterns >2s are blocked
- Read-only constraints prevent dangerous operations
- Permission system based on command parsing and pattern matching
- Background task management for long-running commands

---

## Tool Execution Pipeline

### `runTools()` Orchestration

**File:** `src/services/tools/toolOrchestration.ts`

```typescript
function partitionToolCalls(toolUseMessages, toolUseContext): Batch[] {
  // Groups tools into batches:
  // - Consecutive read-only, concurrency-safe tools → concurrent batch
  // - Any non-safe tool → serial (exclusive) batch
}

export async function* runTools(...): AsyncGenerator<ToolResult> {
  const batches = partitionToolCalls(toolUseMessages, toolUseContext);

  for (const batch of batches) {
    if (batch.length === 1 && !batch[0].isConcurrencySafe) {
      // Serial execution: run alone with exclusive context access
      yield* runToolsSerially(batch, ...);
    } else {
      // Concurrent execution: all safe, run in parallel
      yield* runToolsConcurrently(batch, ...);
    }
  }
}
```

**Concurrency model:**
- Max 10 parallel tools
- Context modifiers from concurrent tools are batched and applied after the batch completes
- In-progress tool IDs tracked to prevent duplicate execution
- Errors in one tool don't affect others in concurrent batches

### `StreamingToolExecutor`

**File:** `src/services/tools/StreamingToolExecutor.ts`

```typescript
export class StreamingToolExecutor {
  private tools: TrackedTool[] = [];

  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    // 1. Find tool definition by name
    // 2. Parse input via Zod
    // 3. Check if concurrency-safe
    // 4. Start executing immediately if safe, or queue
  }

  getCompletedResults(): AsyncGenerator<MessageUpdate> {
    // Yields results as they complete, in receive order
  }

  getRemainingResults(): Promise<MessageUpdate[]> {
    // Returns all remaining results after model response completes
  }

  discard(): void {
    // Abandon all results (used during streaming fallback)
  }
}
```

**Why streaming execution?** Instead of waiting for the full model response before executing any tool, tools start executing as their `tool_use` blocks stream in. This means:
1. A read-only tool can complete before the model finishes generating the next tool call
2. Results are buffered and emitted in the order tools were received
3. The model's response and tool execution overlap in time

### `runToolUse()` Pipeline

**File:** `src/services/tools/toolExecution.ts`

The complete execution pipeline for a single tool:

```
1. Find tool by name (with alias fallback)
   │
2. Zod schema validation: tool.inputSchema.safeParse(input)
   │
3. Custom validation: tool.validateInput?.(input, context)
   │
4. Pre-tool hooks: runPreToolUseHooks()
   │  Can return: messages, permission decisions, block continuation
   │
5. Permission resolution:
   │  a. Hook permission: resolveHookPermissionDecision()
   │  b. Tool check: tool.checkPermissions(input, context)
   │  c. Can-use-tool gate: canUseTool(decision, input, tool)
   │
6. Tool execution: tool.call(args, context, canUseTool, parentMessage, onProgress)
   │
7. Result mapping: tool.mapToolResultToToolResultBlockParam()
   │
8. Post-tool hooks: runPostToolUseHooks()
   │
9. Error handling: runPostToolUseFailureHooks() on error
```

---

## How to Implement a New Tool

### Minimal Example

```typescript
// src/tools/MyTool/MyTool.ts
import { buildTool } from '../Tool.js';
import { z } from 'zod';

export const MyTool = buildTool({
  name: 'MyTool',
  searchHint: 'describe what this tool does in 3-10 words',

  inputSchema: z.object({
    query: z.string().describe('What to search for'),
    path: z.string().optional().describe('Where to search'),
  }),

  maxResultSizeChars: 50_000,

  isReadOnly: () => true,
  isConcurrencySafe: () => true,

  async call(args, context, canUseTool, parentMessage, onProgress) {
    const startMs = Date.now();
    // ... do work ...
    return {
      output: {
        results: [...],
        durationMs: Date.now() - startMs,
      },
    };
  },

  checkPermissions: (input, context) =>
    checkReadPermissionForTool(input.path, context),

  async description(input) {
    return `Search for "${input.query}" in ${input.path ?? 'cwd'}`;
  },

  async prompt(options) {
    return 'MyTool: searches for things. Usage: MyTool(query="...")';
  },

  // Rendering methods
  renderToolUseMessage(input) {
    return `Searching for: ${input.query}`;
  },

  renderToolResultMessage(output) {
    return `Found ${output.results.length} results`;
  },
});
```

### Registration

Add to `getAllBaseTools()` in `src/tools.ts`:

```typescript
import { MyTool } from './tools/MyTool/MyTool.js';

export function getAllBaseTools(): Tools {
  return [
    // ... existing tools ...
    MyTool,
    // ...
  ];
}
```
