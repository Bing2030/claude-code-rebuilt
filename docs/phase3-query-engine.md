# Phase 3: Query Engine & Streaming

## Overview

The query engine is the heart of the application — the core LLM interaction loop. It uses an async generator pattern to yield streaming events as they arrive from the model, executing tool calls mid-stream and managing context window budget.

```
query() (async generator)
  │
  ├─ queryLoop() — core loop
  │    │
  │    ├─ Context preparation
  │    │    ├─ Memory prefetch (parallel)
  │    │    ├─ Tool result budget
  │    │    ├─ Snip compaction
  │    │    ├─ Microcompact
  │    │    ├─ Context collapse
  │    │    └─ Autocompact
  │    │
  │    ├─ Model call: deps.callModel()
  │    │    ├─ Stream response blocks
  │    │    ├─ Detect tool_use blocks
  │    │    ├─ StreamingToolExecutor.addTool()
  │    │    └─ Yield streaming events
  │    │
  │    ├─ Tool execution
  │    │    ├─ StreamingToolExecutor.getRemainingResults()
  │    │    ├─ OR runTools() fallback
  │    │    ├─ Generate tool use summary
  │    │    └─ Consume prefetched memory/skills
  │    │
  │    └─ Turn management
  │         ├─ Turn count increment
  │         ├─ Max turns check
  │         └─ Recursive next iteration
  │
  └─ Tool execution pipeline (per tool)
       ├─ Find tool + Zod validation
       ├─ validateInput()
       ├─ Pre-tool hooks
       ├─ Permission resolution
       ├─ tool.call()
       ├─ Result mapping
       └─ Post-tool hooks
```

---

## The `query()` Generator

**File:** `src/query.ts`

### Generator Signature (Lines 219-239)

```typescript
async function* query(
  options: QueryOptions,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
> {
```

The generator yields various event types:
- **`StreamEvent`**: Streaming tokens from the model
- **`RequestStartEvent`**: Marks the start of an API call
- **`Message`**: Complete user/assistant messages
- **`TombstoneMessage`**: Placeholder for messages removed during compaction
- **`ToolUseSummaryMessage`**: Compact summary of a batch of tool calls

The `Terminal` return type is yielded when the conversation ends.

### Delegation to `queryLoop()`

The `query()` function sets up the top-level context and delegates to `queryLoop()`:

```typescript
async function* query(options: QueryOptions) {
  // Setup: chain ID, depth tracking, analytics
  const chainId = generateChainId();

  // Delegate to the core loop
  yield* queryLoop(options, chainId);
}
```

---

## Loop Setup (Lines 241-580)

### Memory Prefetch

```typescript
const memoryPrefetch = startRelevantMemoryPrefetch({
  messages: state.messages,
  cwd: getCwd(),
});
```

Memory prefetch runs in parallel with the model call — it searches for relevant memories while the model is generating. The results are consumed after the first turn.

### Token Budget Tracking

```typescript
const budgetTracker = createBudgetTracker({
  model: runtimeModel,
  maxBudget: options.maxBudgetUsd,
});
```

Tracks spending across the entire conversation, including across compaction boundaries.

### Context Preparation Pipeline

Before each model call, the context goes through a series of compaction steps:

```typescript
// 1. Apply tool result budget — truncate old tool results
applyToolResultBudget(state.messages, tokenBudget);

// 2. Snip compaction — remove old tool results entirely
if (snipModule) {
  state.messages = await snipModule.snipCompactIfNeeded(state.messages, ...);
}

// 3. Microcompact — fine-grained removal of verbose outputs
state.messages = await deps.microcompact(state.messages, ...);

// 4. Context collapse — collapse repeated patterns
state.messages = await contextCollapse.applyCollapsesIfNeeded(state.messages, ...);

// 5. Autocompact — full compaction when approaching limits
state.messages = await deps.autocompact(state.messages, ...);
```

### Compaction Strategies

| Strategy | When | What |
|----------|------|------|
| **Tool result budget** | Always | Truncate old tool outputs to budget |
| **Snip** | When approaching 80% | Remove old tool results entirely |
| **Microcompact** | When approaching 85% | Fine-grained removal of verbose outputs |
| **Context collapse** | Opportunistic | Collapse repeated patterns |
| **Autocompact** | When approaching 95% | Full conversation summary compaction |

### StreamingToolExecutor Setup

```typescript
let streamingToolExecutor: StreamingToolExecutor | undefined;
if (config.gates.streamingToolExecution) {
  streamingToolExecutor = new StreamingToolExecutor(
    toolDefinitions,
    canUseTool,
    toolUseContext,
  );
}
```

Conditionally created based on the `streamingToolExecution` feature gate. When disabled, tools execute only after the full model response completes.

---

## Model Streaming (Lines 659-864)

### The Streaming Loop

```typescript
for await (const message of deps.callModel({
  model: runtimeModel,
  messages: state.messages,
  tools: toolDefinitions,
  systemPrompt: renderedSystemPrompt,
  // ... extensive config options
})) {
  // Process each streaming block
}
```

`deps.callModel()` returns an async iterable of streaming blocks.

### Tool Use Detection

```typescript
if (message.type === 'assistant' && message.content) {
  for (const block of message.content) {
    if (block.type === 'tool_use') {
      toolUseBlocks.push(block);
      needsFollowUp = true;

      // Forward to streaming executor immediately
      streamingToolExecutor?.addTool(block, message);
    }
  }
}
```

Tool use blocks are detected as they stream in and immediately forwarded to the `StreamingToolExecutor`, which starts executing them in parallel with the ongoing model response.

### Backfilling Observable Input

```typescript
// Before yielding to SDK, backfill observable input fields
if (tool.backfillObservableInput) {
  tool.backfillObservableInput(parsedInput);
}
```

Tools can mutate the observable copy of their input (for SDK stream, transcript, permission checks) without affecting the original API-bound input (which would break prompt cache).

### Error Handling

```typescript
// Withhold recoverable errors until recovery can succeed
if (error.type === 'prompt_too_long') {
  // Queue for recovery after context compaction
  withheldError = error;
  break;
}

if (error.type === 'max_output_tokens') {
  // Continue the conversation with a note
  needsFollowUp = true;
}
```

### Tombstoning

When streaming fallback occurs (the initial model call fails and a fallback model is used), orphaned messages from the failed attempt are replaced with tombstones:

```typescript
yield {
  type: 'tombstone',
  replacedMessageIds: orphanedIds,
};
```

### Yielding Streaming Results

```typescript
// Yield completed tool results as they arrive
if (streamingToolExecutor) {
  for await (const result of streamingToolExecutor.getCompletedResults()) {
    if (result.message) yield result.message;
    if (result.newContext) {
      updatedToolUseContext = result.newContext;
    }
  }
}
```

---

## Tool Execution Dispatch (Lines 1360-1728)

### After Model Response Completes

```typescript
// Get remaining tool results
const toolUpdates = streamingToolExecutor
  ? await streamingToolExecutor.getRemainingResults()
  : await runTools(toolUseBlocks, ...);
```

When `StreamingToolExecutor` is enabled, it collects any remaining results. Otherwise, `runTools()` executes all tool calls sequentially.

### Tool Use Summary

```typescript
// Async summary generation (non-blocking)
const summaryPromise = generateToolUseSummary({
  toolUseBlocks,
  toolResults: toolUpdates,
}).catch(() => null);

// ... later
const summary = await summaryPromise;
if (summary) {
  yield { type: 'tool_use_summary', summary };
}
```

The summary is generated in the background and yielded when ready, without blocking the next API call.

### Memory/Skill Consumption

```typescript
// Consume prefetched memories
const memories = await memoryPrefetch;
if (memories.length > 0) {
  state.messages.push(...memories);
}

// Consume prefetched skill discoveries
const skillDiscovery = await skillPrefetch;
if (skillDiscovery) {
  state.messages.push(skillDiscovery);
}
```

### Turn Management

```typescript
state.turnCount++;

// Check max turns limit
if (options.maxTurns && state.turnCount >= options.maxTurns) {
  yield { type: 'max_turns_reached', turnCount: state.turnCount };
  return;
}

// Check budget
if (budgetTracker.isOverBudget()) {
  yield { type: 'budget_exceeded' };
  return;
}

// Continue to next iteration
// ... prepare state for next turn
```

---

## Tool Execution Pipeline

**File:** `src/services/tools/toolExecution.ts`

### `runToolUse()` — Full Pipeline

```typescript
export async function runToolUse(
  toolUseBlock: ToolUseBlock,
  assistantMessage: AssistantMessage,
  tools: Tools,
  toolUseContext: ToolUseContext,
  canUseTool: CanUseToolFn,
  onProgress?: ToolCallProgress,
): Promise<{ result: ToolResultBlockParam; context?: ToolUseContext }> {
```

#### Step 1: Tool Discovery

```typescript
let tool = findToolByName(tools, toolUseBlock.name);
if (!tool) {
  // Try deprecated aliases
  tool = tools.find(t => t.aliases?.includes(toolUseBlock.name));
}
if (!tool) {
  // Return error result
  return { result: { type: 'tool_result', content: 'No such tool', is_error: true } };
}
```

#### Step 2: Input Validation

```typescript
const parsed = tool.inputSchema.safeParse(rawInput);
if (!parsed.success) {
  return { result: { type: 'tool_result', content: formatZodError(parsed.error), is_error: true } };
}

// Custom validation
if (tool.validateInput) {
  const validation = await tool.validateInput(parsed.data, context);
  if (!validation.valid) {
    return { result: { type: 'tool_result', content: validation.reason, is_error: true } };
  }
}
```

#### Step 3: Pre-Tool Hooks

```typescript
const hookResult = await runPreToolUseHooks({
  toolName: tool.name,
  input: parsed.data,
  toolUseContext,
});

if (hookResult.blocked) {
  return { result: { type: 'tool_result', content: hookResult.reason } };
}

if (hookResult.updatedInput) {
  parsed.data = hookResult.updatedInput;
}
```

#### Step 4: Permission Resolution

```typescript
const hookPermission = resolveHookPermissionDecision(hookResult);
const toolPermission = await tool.checkPermissions(parsed.data, context);
const decision = canUseTool(hookPermission, toolPermission, parsed.data, tool);
```

#### Step 5: Tool Execution

```typescript
const toolResult = await tool.call(
  parsed.data,
  toolUseContext,
  canUseTool,
  assistantMessage,
  onProgress,
);
```

#### Step 6: Result Mapping

```typescript
const resultBlock = tool.mapToolResultToToolResultBlockParam(
  toolResult.output,
  toolUseBlock.id,
);
```

#### Step 7: Post-Tool Hooks

```typescript
await runPostToolUseHooks({
  toolName: tool.name,
  input: parsed.data,
  result: resultBlock,
  toolUseContext,
});
```

---

## Concurrency Model

### Partitioning Algorithm

```
Tool calls: [Read A] [Read B] [Bash] [Read C] [Read D]
                │         │              │         │
                └─────┬───┘              │    ┌────┘
                      │                  │    │
           Batch 1: Concurrent     Batch 2: Serial   Batch 3: Concurrent
           [Read A, Read B]        [Bash]            [Read C, Read D]
```

### Concurrent Execution Rules

1. Tools must declare `isConcurrencySafe(input) → true`
2. Tools must declare `isReadOnly(input) → true`
3. Consecutive safe tools are grouped into a batch
4. Non-safe tools create a serial batch (run alone)
5. Context modifiers from concurrent tools are deferred until batch completes
6. Max 10 concurrent tools

### Streaming vs Non-Streaming

| Aspect | StreamingToolExecutor | runTools() |
|--------|----------------------|------------|
| When tools start | As blocks stream in | After full response |
| Concurrency | Same partitioning | Same partitioning |
| Error handling | discard() on fallback | Standard |
| Feature gate | `streamingToolExecution` | Always available |

---

## Error Recovery

### Prompt Too Long

```typescript
if (error.type === 'prompt_too_long') {
  // 1. Try snip compaction
  // 2. Try microcompact
  // 3. Try autocompact
  // 4. Retry model call with compacted context
  // 5. If still too long, return error
}
```

### Max Output Tokens

```typescript
if (error.type === 'max_output_tokens') {
  // Continue conversation — model will pick up where it left off
  needsFollowUp = true;
}
```

### Model Overloaded

```typescript
if (fallbackModel && isOverloadedError(error)) {
  // Retry with fallback model
  runtimeModel = fallbackModel;
  continue; // Next iteration of the loop
}
```

### Streaming Fallback

```typescript
if (streamingToolExecutor) {
  streamingToolExecutor.discard(); // Abandon all in-progress tools
}
// Yield tombstones for orphaned messages
// Retry with fallback model
```
