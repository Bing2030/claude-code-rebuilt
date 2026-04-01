# Phase 4: Terminal UI (React + Ink)

## Overview

The terminal UI is built with React components rendered through Ink — a React-to-terminal renderer that uses Yoga (Facebook's flexbox engine) for layout and ANSI escape codes for output. The architecture follows standard React patterns: context providers for state, memoized components for performance, and virtual scrolling for large message lists.

```
Component Hierarchy:

<Root> (Ink)
  └─ <App> (src/components/App.tsx)
       ├─ FpsMetricsProvider
       ├─ StatsProvider
       ├─ AppStateProvider
       └─ <REPL> (src/screens/REPL.tsx)
            ├─ LogoHeader (memoized)
            ├─ StatusNotices
            ├─ VirtualMessageList
            │    └─ <Messages> (src/components/Messages.tsx)
            │         ├─ Message components
            │         └─ Tool result rendering
            └─ <PromptInput> (src/components/PromptInput/PromptInput.tsx)
                 ├─ TextInput / VimTextInput
                 ├─ Typeahead suggestions
                 └─ Permission handling
```

---

## File: `src/components/App.tsx`

### Purpose

Root wrapper component that establishes the context provider hierarchy for the entire application.

### Props Interface

```typescript
type AppWrapperProps = {
  getFpsMetrics: () => FpsMetrics | undefined;
  stats?: StatsStore;
  initialState: AppState;
};
```

### Component Structure

App wraps children in three nested context providers:

1. **FpsMetricsProvider** — Provides FPS frame timing data for performance monitoring
2. **StatsProvider** — Provides application statistics store
3. **AppStateProvider** — Provides the central application state

```typescript
function App({ getFpsMetrics, stats, initialState }: AppWrapperProps) {
  return (
    <FpsMetricsProvider getFpsMetrics={getFpsMetrics}>
      <StatsProvider stats={stats}>
        <AppStateProvider initialState={initialState}>
          {children}
        </AppStateProvider>
      </StatsProvider>
    </FpsMetricsProvider>
  );
}
```

### Performance Optimization

- Uses `React.memo` to prevent unnecessary re-renders
- The component is essentially a pure wrapper — no state, no side effects
- All provider initialization happens in the constructor, not on every render

---

## File: `src/screens/REPL.tsx`

### Purpose

The main interactive screen — handles message display, user input, tool rendering, and streaming updates.

### Props Interface (`REPLProps`)

```typescript
type REPLProps = {
  messages: Message[];
  tools: Tools;
  systemPrompt: string[];
  permissionContext: ToolPermissionContext;
  modelConfig: { model: string; effort?: string; thinking?: string };
  mcpConfig: Record<string, McpServerConfig>;
  // ... many more configuration options
};
```

### Key State Variables

```typescript
// Component-level state
const [inputText, setInputText] = useState('');
const [submittedPrompt, setSubmittedPrompt] = useState<string | undefined>();
const [toolPermissionContext, setToolPermissionContext] = useState(props.permissionContext);

// Refs for imperative access
const messageQueueRef = useRef<MessageQueue>();
const abortControllerRef = useRef<AbortController>();
```

### Key Hooks

| Hook | Purpose |
|------|---------|
| `useTerminalSize()` | Tracks terminal dimensions for responsive layout |
| `useInput()` | Handles keyboard input (key bindings, shortcuts) |
| `useAppState(selector)` | Selective subscription to central state |
| `useSetAppState()` | State updater function |
| `useDeferredValue` | Debounced message rendering for performance |
| `useCallback` | Memoized event handlers |

### Component Layout

```
<Box flexDirection="column" height={rows}>
  {/* Header area */}
  <LogoHeader />
  <StatusNotices />

  {/* Message area - takes remaining height */}
  <Box flexDirection="column" flexGrow={1}>
    <VirtualMessageList
      messages={displayMessages}
      tools={tools}
      verbose={verbose}
    />
  </Box>

  {/* Input area - fixed at bottom */}
  <PromptInput
    value={inputText}
    onChange={setInputText}
    onSubmit={handleSubmit}
    tools={tools}
    permissionContext={toolPermissionContext}
  />
</Box>
```

### Message Processing

REPL applies several transformations to the message list before rendering:

1. **Brief mode filtering**: `filterForBriefTool(messages)` — In brief mode, only shows essential messages
2. **Text dropping**: `dropTextInBriefTurns(messages)` — Removes verbose text in brief turns
3. **Deferred rendering**: `useDeferredValue(processedMessages)` — Debounces rendering for performance

### Streaming Integration

REPL subscribes to streaming events from the query engine:

```typescript
useEffect(() => {
  const subscription = queryEvents.subscribe((event) => {
    if (event.type === 'assistant_message') {
      // Update message list with streaming content
    } else if (event.type === 'tool_use') {
      // Show tool execution progress
    } else if (event.type === 'tool_result') {
      // Display tool results
    }
  });
  return () => subscription.unsubscribe();
}, [queryEvents]);
```

---

## File: `src/components/Messages.tsx`

### Purpose

Handles rendering of individual messages in the conversation, including virtual scrolling for performance.

### Message Types

The Messages component handles several message types:

| Type | Component | Rendering |
|------|-----------|-----------|
| `user` | UserMessage | Markdown with syntax highlighting |
| `assistant` | AssistantMessage | Streaming markdown, tool use blocks |
| `tool_result` | ToolResultMessage | Depends on tool type |
| `tombstone` | TombstoneMessage | Collapsed/removed message indicator |

### Virtual Scrolling

For performance with thousands of messages:

```typescript
function VirtualMessageList({ messages, tools, verbose }) {
  const containerRef = useRef(null);
  const [scrollTop, setScrollTop] = useState(0);
  const [visibleRange, setVisibleRange] = useState({ start: 0, end: 50 });

  // Calculate which messages are visible based on scroll position
  const messageHeightEstimate = 100; // pixels per message (approximate)
  const visibleStart = Math.floor(scrollTop / messageHeightEstimate);
  const visibleEnd = visibleStart + Math.ceil(containerHeight / messageHeightEstimate) + buffer;

  const visibleMessages = messages.slice(
    Math.max(0, visibleStart - buffer),
    Math.min(messages.length, visibleEnd + buffer)
  );

  return (
    <Box ref={containerRef} onScroll={handleScroll}>
      {/* Spacer for messages above visible range */}
      <Box height={visibleStart * messageHeightEstimate} />

      {/* Render only visible messages */}
      {visibleMessages.map(message => (
        <MessageRenderer key={message.uuid} message={message} tools={tools} />
      ))}

      {/* Spacer for messages below visible range */}
      <Box height={(messages.length - visibleEnd) * messageHeightEstimate} />
    </Box>
  );
}
```

### Tool Result Rendering

Each tool defines its own rendering methods:

```typescript
function MessageRenderer({ message, tools }) {
  if (message.type === 'assistant') {
    return message.content.map(block => {
      if (block.type === 'text') {
        return <StreamingMarkdown content={block.text} />;
      }
      if (block.type === 'tool_use') {
        const tool = findToolByName(tools, block.name);
        return tool?.renderToolUseMessage(block.input, { theme, verbose });
      }
    });
  }
  if (message.type === 'tool_result') {
    const tool = findToolByName(tools, block.toolName);
    return tool?.renderToolResultMessage(block.content, { theme, verbose });
  }
}
```

### Markdown Rendering

Streaming markdown is rendered in real-time as tokens arrive:

```typescript
function StreamingMarkdown({ content }: { content: string }) {
  // Renders markdown incrementally:
  // - Code blocks with syntax highlighting
  // - Bold, italic, links
  // - Tables
  // - Lists (ordered, unordered)
  // - Inline code
  return (
    <Box flexDirection="column">
      <MarkdownRenderer content={content} />
    </Box>
  );
}
```

### Message Grouping

Messages can be grouped for display:

```typescript
// Multiple read-only tools in sequence can be collapsed into a group
function shouldGroupMessages(messages: Message[]): boolean {
  return messages.every(m =>
    m.type === 'tool_result' &&
    tools.find(t => t.name === m.toolName)?.isReadOnly()
  );
}
```

---

## File: `src/components/PromptInput/PromptInput.tsx`

### Purpose

Handles user text input with multiple modes (normal, vim), completions, and command processing.

### Props Interface

```typescript
type PromptInputProps = {
  value: string;
  onChange: (value: string) => void;
  onSubmit: (prompt: string) => void;
  tools: Tools;
  permissionContext: ToolPermissionContext;
  disabled?: boolean;
  placeholder?: string;
};
```

### Input Modes

| Mode | Description |
|------|-------------|
| Normal | Standard text input with cursor movement |
| Vim | Vim-like modal input (insert, normal, visual) |

### Key Hooks

| Hook | Purpose |
|------|---------|
| `useInputBuffer` | Manages text input state, cursor position |
| `useTypeahead` | Command/skill completions |
| `usePromptSuggestion` | AI-powered prompt suggestions |
| `useInput` (Ink) | Keyboard event handling |

### Input Buffer Hook

```typescript
function useInputBuffer(initialValue: string) {
  const [buffer, setBuffer] = useState(initialValue);
  const [cursorPos, setCursorPos] = useState(initialValue.length);
  const [history, setHistory] = useState<string[]>([]);
  const [historyIdx, setHistoryIdx] = useState(-1);

  return {
    value: buffer,
    cursorPosition: cursorPos,
    insertText: (text: string) => { ... },
    deleteChar: () => { ... },
    moveCursor: (pos: 'left' | 'right' | 'start' | 'end') => { ... },
    submit: () => { const val = buffer; setBuffer(''); return val; },
    historyUp: () => { ... },
    historyDown: () => { ... },
  };
}
```

### Typeahead Hook

```typescript
function useTypeahead(value: string, commands: Command[]) {
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [selectedIdx, setSelectedIdx] = useState(0);

  useEffect(() => {
    if (value.startsWith('/')) {
      const prefix = value.slice(1).toLowerCase();
      const matches = commands
        .filter(cmd => cmd.name.toLowerCase().startsWith(prefix))
        .map(cmd => `/${cmd.name}`);
      setSuggestions(matches);
    } else {
      setSuggestions([]);
    }
  }, [value, commands]);

  return { suggestions, selectedIdx, selectNext, selectPrev, apply };
}
```

### Command Processing

When the user types a `/` command:

```typescript
const handleSubmit = useCallback((text: string) => {
  if (text.startsWith('/')) {
    // Slash command handling
    const [commandName, ...args] = text.slice(1).split(' ');
    const command = commands.find(c => c.name === commandName);
    if (command) {
      executeCommand(command, args);
    } else {
      showError(`Unknown command: /${commandName}`);
    }
  } else {
    // Normal message submission
    onSubmit(text);
  }
}, [commands, onSubmit]);
```

### Keyboard Bindings

| Key | Action |
|-----|--------|
| Enter | Submit message |
| Shift+Enter | New line (multiline) |
| Up/Down | Navigate history |
| Tab | Accept typeahead suggestion |
| Escape | Cancel current input |
| Ctrl+C | Interrupt current operation |

---

## File: `src/ink/` (Vendored Ink)

### What is Ink?

Ink is a React-to-terminal renderer. The vendored copy in `src/ink/` provides:

1. **Yoga Integration** — Uses Facebook's Yoga flexbox engine for layout calculation
2. **ANSI Output** — Translates layout boxes to ANSI escape sequences for terminal positioning
3. **Screen Management** — Handles terminal resize, alternate screen buffer, cursor management
4. **Component Primitives** — `Box`, `Text`, `Newline` components

### Layout Pipeline

```
React Component Tree
    │
    ▼
Reconciliation (React reconciler)
    │
    ▼
Yoga Node Tree (flexbox layout)
    │
    ▼
Calculated Layout (x, y, width, height for each node)
    │
    ▼
ANSI Output (cursor positioning, colors, text)
    │
    ▼
Terminal (stdout via ANSI escape codes)
```

### Key Exports from `src/ink/`

| Export | Purpose |
|-------|---------|
| `render` | Creates Ink instance and renders React tree |
| `Box` | Flexbox container component |
| `Text` | Text component with color/bold/style support |
| `Newline` | Line break component |
| `useInput` | Keyboard input hook |
| `useTerminalSize` | Terminal dimension hook |
| `useApp` | Ink instance access |

### Terminal Size Handling

```typescript
function useTerminalSize() {
  const [size, setSize] = useState({
    columns: process.stdout.columns ?? 80,
    rows: process.stdout.rows ?? 24,
  });

  useEffect(() => {
    const handler = () => {
      setSize({
        columns: process.stdout.columns ?? 80,
        rows: process.stdout.rows ?? 24,
      });
    };
    process.stdout.on('resize', handler);
    return () => process.stdout.off('resize', handler);
  }, []);

  return size;
}
```

---

## State Management

### AppStateStore (`src/state/AppStateStore.ts`)

The central state store defines the complete application state:

```typescript
interface AppState {
  // Settings
  verbose: boolean;
  model: string;
  effort: string;
  thinking: string;

  // View state
  expandedViews: Set<string>;
  selectionMode: 'normal' | 'vim';

  // Tool permissions
  toolPermissionContext: ToolPermissionContext;

  // Speculation state
  speculationState: SpeculationState | null;

  // Footer
  footerItems: FooterItem[];

  // View modes
  isBriefMode: boolean;
  isTranscriptMode: boolean;

  // Completion boundaries
  completionBoundaries: CompletionBoundary[];
}
```

### AppStateProvider (`src/state/AppState.tsx`)

```typescript
function AppStateProvider({ initialState, children }) {
  const store = useMemo(() => createStore(initialState), [initialState]);

  return (
    <AppStoreContext.Provider value={store}>
      <HasAppStateContext.Provider value={true}>
        {children}
      </HasAppStateContext.Provider>
    </AppStoreContext.Provider>
  );
}
```

### State Access Hooks

```typescript
// Access the store instance
function useAppStore(): Store<AppState> { ... }

// Selective subscription — only re-renders when selected slice changes
function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore();
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
  );
}

// State updater
function useSetAppState(): (updater: (state: AppState) => AppState) => void {
  const store = useAppStore();
  return store.setState;
}
```

### Data Flow

```
User Action (key press, click)
    │
    ▼
PromptInput.onsubmit(text)
    │
    ▼
REPL.handleSubmit(text)
    │
    ▼
query engine (query())
    │
    ▼
Streaming events (yielded)
    │
    ▼
REPL processes events → updates messages
    │
    ▼
useSetAppState() → state update
    │
    ▼
Messages component re-renders (only affected slices)
    │
    ▼
Ink renders to terminal
```

---

## Performance Patterns

### Virtual Scrolling

With potentially thousands of messages, rendering all would be too slow. The `VirtualMessageList` only renders messages in the visible viewport plus a buffer zone.

### React.memo

Expensive components use `React.memo` to prevent cascade re-renders:

```typescript
const LogoHeader = React.memo(function LogoHeader({ ... }) {
  // Only re-renders when props actually change
});
```

### useDeferredValue

For message list processing (filtering, grouping), that shouldn't block the main thread:

```typescript
const processedMessages = useMemo(() => processMessages(messages), [messages]);
const deferredMessages = useDeferredValue(processedMessages);
```

### Selective State Subscription

Components only re-render when their specific state slice changes:

```typescript
// This component only re-renders when 'verbose' changes
const verbose = useAppState(state => state.verbose);
```

---

## Key Questions Answered

**How do React components map to terminal output?**

React component tree → React reconciler → Yoga node tree (flexbox layout) → calculated positions → ANSI escape codes → terminal stdout. Ink translates the React virtual DOM into terminal-compatible output using Yoga for layout.

**How is state managed and propagated?**

Central `AppStateStore` accessed via React context. Components subscribe selectively via `useAppState(selector)` using `useSyncExternalStore`. Updates flow from user actions → event handlers → state updates → selective re-renders.

**How does streaming markdown work?**

The `StreamingMarkdown` component receives content incrementally as tokens arrive from the query engine. Each token is appended to the accumulated text and re-rendered. The markdown renderer handles code blocks, bold, italic, links, tables, and lists.

**How does virtual scrolling work?**

`VirtualMessageList` tracks scroll position and calculates which messages fall within the visible viewport (plus buffer). Only those messages are rendered. Spacer elements above and below maintain correct scroll position.
