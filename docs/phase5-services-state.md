# Phase 5: Services & State Management

## Overview

The services layer provides the infrastructure that the core runtime (query engine, tools) depends on: multi-provider API access, authentication, MCP server integration, and centralized state management with React bindings.

```
Services Architecture:

┌──────────────────────────────────────────────┐
│                   React UI                    │
│         useAppState() / useSetAppState()      │
├──────────────────────────────────────────────┤
│              AppStateProvider                 │
│      (useSyncExternalStore + React Context)   │
├──────────────────────────────────────────────┤
│             AppStateStore                     │
│   (createStore with DeepImmutable state)      │
├──────────┬──────────┬────────────────────────┤
│ API      │ Auth     │ MCP                    │
│ Client   │ Layer    │ Client                 │
├──────────┴──────────┴────────────────────────┤
│  Anthropic │ Bedrock │ Vertex │ Foundry      │
│  SDK       │ SDK     │ SDK    │ SDK          │
└──────────────────────────────────────────────┘
```

---

## File: `src/services/api/client.ts`

### Purpose

Factory function that creates the appropriate Anthropic SDK client based on the configured provider (Direct, Bedrock, Vertex, or Foundry).

### Main Export

```typescript
export async function getAnthropicClient({
  apiKey?: string,
  maxRetries: number,
  model?: string,
  fetchOverride?: ClientOptions['fetch'],
  source?: string,
}): Promise<Anthropic>
```

Returns a unified `Anthropic` client instance regardless of which provider backend is used.

### Multi-Provider Support

The client selects a provider based on environment variables, checked in this order:

#### 1. AWS Bedrock (`CLAUDE_CODE_USE_BEDROCK`)

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
  // Region priority:
  //   1. ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION (for small/fast models)
  //   2. AWS_REGION or AWS_DEFAULT_REGION
  //   3. Default: us-east-1

  // Authentication:
  //   - AWS_BEARER_TOKEN_BEDROCK (API key auth)
  //   - AWS credential chain via refreshAndGetAwsCredentials()
  //   - CLAUDE_CODE_SKIP_BEDROCK_AUTH=1 (skip auth entirely)

  return new AnthropicBedrock(bedrockArgs) as unknown as Anthropic
}
```

Uses `@anthropic-ai/bedrock-sdk`. Supports standard AWS credential providers (environment variables, instance metadata, etc.).

#### 2. Azure Foundry (`CLAUDE_CODE_USE_FOUNDRY`)

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) {
  // Endpoint: ANTHROPIC_FOUNDRY_RESOURCE or ANTHROPIC_FOUNDRY_BASE_URL

  // Authentication:
  //   - ANTHROPIC_FOUNDRY_API_KEY (API key)
  //   - DefaultAzureCredential from @azure/identity (Azure AD)
  //   - CLAUDE_CODE_SKIP_FOUNDRY_AUTH=1 (skip auth)

  return new AnthropicFoundry(foundryArgs) as unknown as Anthropic
}
```

Uses `@anthropic-ai/foundry-sdk`. Falls back to Azure AD when no API key is provided.

#### 3. Google Vertex AI (`CLAUDE_CODE_USE_VERTEX`)

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
  // Region priority (model-specific overrides):
  //   1. VERTEX_REGION_CLAUDE_3_5_HAIKU
  //   2. VERTEX_REGION_CLAUDE_HAIKU_4_5
  //   3. VERTEX_REGION_CLAUDE_3_5_SONNET
  //   4. VERTEX_REGION_CLAUDE_3_7_SONNET
  //   5. CLOUD_ML_REGION (global fallback)
  //   6. Default: us-east5

  // Authentication:
  //   - GoogleAuth with cloud-platform scope
  //   - Service account or ADC (Application Default Credentials)
  //   - CLAUDE_CODE_SKIP_VERTEX_AUTH=1 (skip auth)

  // Project ID: ANTHROPIC_VERTEX_PROJECT_ID or standard GCP env vars
  return new AnthropicVertex(vertexArgs) as unknown as Anthropic
}
```

Uses `@anthropic-ai/vertex-sdk`. Project ID falls back to `ANTHROPIC_VERTEX_PROJECT_ID` when no standard GCP project env vars are set (avoids the 12-second metadata server timeout).

#### 4. Anthropic Direct API (Default)

```typescript
// Standard Anthropic API client
const clientConfig = {
  apiKey: isClaudeAISubscriber() ? null : apiKey || getAnthropicApiKey(),
  authToken: isClaudeAISubscriber()
    ? getClaudeAIOAuthTokens()?.accessToken
    : undefined,
  // Staging OAuth support (ant-only)
  ...(USE_STAGING_OAUTH ? { baseURL: getOauthConfig().BASE_API_URL } : {}),
}
return new Anthropic(clientConfig)
```

### Custom Headers

```typescript
async function configureApiKeyHeaders(
  headers: Record<string, string>,
  isNonInteractiveSession: boolean,
): Promise<void> {
  // Sets Authorization: Bearer <token>
  // Token from ANTHROPIC_AUTH_TOKEN or apiKeyHelper command
}

function getCustomHeaders(): Record<string, string> {
  // Parses ANTHROPIC_CUSTOM_HEADERS env var
  // Format: "Name: Value" (curl style), multiple headers separated by newlines
}
```

### Common Client Arguments

All providers share base configuration:

```typescript
const ARGS = {
  maxRetries,
  defaultHeaders: {
    'x-app': 'cli',
    'User-Agent': getUserAgent(),
    // Session tracking headers
    // Custom headers from ANTHROPIC_CUSTOM_HEADERS
    // Auth headers from configureApiKeyHeaders
  },
  ...(fetchOverride && { fetch: fetchOverride }),
}
```

### Provider Selection Flow

```
getAnthropicClient()
  │
  ├─ CLAUDE_CODE_USE_BEDROCK? ──→ AnthropicBedrock
  │
  ├─ CLAUDE_CODE_USE_FOUNDRY? ──→ AnthropicFoundry
  │
  ├─ CLAUDE_CODE_USE_VERTEX?  ──→ AnthropicVertex
  │
  └─ (default)                ──→ Anthropic (Direct API)
       ├─ claude.ai subscriber? → OAuth accessToken
       └─ otherwise            → API key or OAuth token
```

---

## File: `src/utils/auth.ts`

### Purpose

Manages authentication resolution — determines how the client authenticates with the Anthropic API. Supports multiple auth sources with a clear priority chain.

### Key Types

```typescript
export type ApiKeySource =
  | 'ANTHROPIC_API_KEY'      // Direct env var
  | 'apiKeyHelper'           // External helper command
  | '/login managed key'     // OAuth-managed key from `claude login`
  | 'none'                   // No key found
```

### Core Functions

| Function | Returns | Purpose |
|----------|---------|---------|
| `isAnthropicAuthEnabled()` | `boolean` | Should Anthropic auth be used? |
| `getAuthTokenSource()` | `{ source, hasToken }` | Determine token source |
| `getAnthropicApiKey()` | `null \| string` | Get API key (no source info) |
| `hasAnthropicApiKeyAuth()` | `boolean` | Is API key auth available? |
| `getAnthropicApiKeyWithSource(opts)` | `{ key, source }` | Get key + source metadata |

### Auth Enabled Check (`isAnthropicAuthEnabled()`)

Returns `false` (meaning: don't use Anthropic auth) when:

```typescript
// Bare mode (--bare flag): API key only, no OAuth
if (isBareMode()) return false

// Third-party services handle their own auth
if (CLAUDE_CODE_USE_BEDROCK) return false
if (CLAUDE_CODE_USE_VERTEX) return false
if (CLAUDE_CODE_USE_FOUNDRY) return false

// External auth token present (non-managed context)
if (ANTHROPIC_AUTH_TOKEN && !isManagedContext) return false

// External API key present (non-managed context)
if (ANTHROPIC_API_KEY && !isManagedContext) return false
```

### Token Source Priority (`getAuthTokenSource()`)

```
getAuthTokenSource()
  │
  ├─ Bare mode? ──→ apiKeyHelper only
  │
  ├─ ANTHROPIC_AUTH_TOKEN? ──→ auth token (if not managed)
  │
  ├─ CLAUDE_CODE_OAUTH_TOKEN? ──→ OAuth token
  │   (file descriptor or env var)
  │
  ├─ apiKeyHelper configured? ──→ helper command output
  │
  └─ claude.ai OAuth? ──→ getClaudeAIOAuthTokens()
```

### API Key Resolution (`getAnthropicApiKeyWithSource()`)

```
getAnthropicApiKeyWithSource(opts)
  │
  ├─ Homespace? ──→ ignore ANTHROPIC_API_KEY (use Console key)
  │
  ├─ 3P preference + ANTHROPIC_API_KEY? ──→ { key, 'ANTHROPIC_API_KEY' }
  │
  ├─ CI / test? ──→
  │   ├─ file descriptor key ──→ { key, 'ANTHROPIC_API_KEY' }
  │   ├─ ANTHROPIC_API_KEY ──→ { key, 'ANTHROPIC_API_KEY' }
  │   └─ CLAUDE_CODE_OAUTH_TOKEN ──→ { null, 'none' }
  │
  ├─ Approved ANTHROPIC_API_KEY? ──→ { key, 'ANTHROPIC_API_KEY' }
  │
  ├─ File descriptor key? ──→ { key, 'ANTHROPIC_API_KEY' }
  │
  ├─ apiKeyHelper? ──→
  │   ├─ skipRetrievingKeyFromApiKeyHelper? ──→ { null, 'apiKeyHelper' }
  │   └─ cached helper output ──→ { key, 'apiKeyHelper' }
  │
  ├─ Config / macOS keychain? ──→ { key, '/login managed key' }
  │
  └─ (nothing found) ──→ { null, 'none' }
```

### Special Cases

- **Homespace**: On Anthropic's internal infrastructure, `ANTHROPIC_API_KEY` is ignored to force use of the Console-managed key
- **CI/Test**: File descriptor (`CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`) is checked before env vars for secure token passing
- **API Key Helper**: External command that returns an API key. Cache may be cold on first call — callers needing a real key must `await getApiKeyFromApiKeyHelper()` first
- **Managed Context**: When running under management (MDM, enterprise), external auth tokens don't disable Anthropic auth

### Auth Flow Diagram

```
                    ┌────────────────────┐
                    │ isAnthropicAuth    │
                    │ Enabled()?         │
                    └──────┬─────────────┘
                           │
                    ┌──────▼─────────────┐
                    │ getAnthropicApiKey │
                    │ WithSource()       │
                    └──────┬─────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼──┐   ┌────▼────┐  ┌───▼──────┐
        │ API Key│   │ OAuth   │  │ apiKey   │
        │ Helper │   │ Token   │  │ (direct) │
        └─────┬──┘   └────┬────┘  └───┬──────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────▼─────────────┐
                    │ getAnthropicClient │
                    │ ({ apiKey })       │
                    └──────┬─────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼──┐   ┌────▼────┐  ┌───▼──────┐
        │Direct  │   │Bedrock/ │  │Vertex/   │
        │API     │   │Foundry  │  │          │
        └────────┘   └─────────┘  └──────────┘
```

---

## File: `src/services/mcp/client.ts`

### Purpose

Implements the Model Context Protocol (MCP) client — connects to external MCP servers, registers their tools, and manages authentication and session lifecycle.

### Error Classes

```typescript
export class McpAuthError extends Error        // Authentication failure
export class McpSessionExpiredError extends Error  // Session token expired
export class McpToolCallError extends Error      // Tool execution failure
export function isMcpSessionExpiredError(error: Error): boolean
```

### Transport Types

MCP supports five transport mechanisms:

| Transport | Class | Use Case |
|-----------|-------|----------|
| **Stdio** | `StdioClientTransport` | Local subprocess communication |
| **SSE** | `SSEClientTransport` | Server-Sent Events over HTTP |
| **HTTP** | `StreamableHTTPClientTransport` | HTTP streaming |
| **WebSocket** | `WebSocketTransport` | WebSocket with TLS/proxy |
| **Claude.ai Proxy** | (custom) | Claude.ai integration proxy |

### Connection Process

```
connectToMcpServer(config)
  │
  ├─ Determine transport type from config
  │
  ├─ Create transport instance
  │   ├─ stdio: spawn subprocess
  │   ├─ sse/http: create HTTP transport with fetch wrapper
  │   ├─ websocket: create WS transport with TLS options
  │   └─ claudeai-proxy: create proxy transport
  │
  ├─ Create MCP Client
  │   const client = new Client({ name, version, capabilities })
  │   await client.connect(transport)
  │
  ├─ Discover tools
  │   const { tools } = await client.listTools()
  │
  ├─ Register tools
  │   for (const tool of tools) {
  │     // Normalize name, wrap in MCPTool class
  │     registeredTools.push(new MCPTool(tool, client))
  │   }
  │
  └─ Return MCPServerConnection
       { status, client, tools, resources, ... }
```

### Authentication Handling

#### Auth Cache

```typescript
const MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000 // 15 minutes

// Memoized file read — N concurrent calls share single file read
let authCachePromise: Promise<McpAuthCacheData> | null = null

async function isMcpAuthCached(serverId: string): Promise<boolean> {
  const cache = await getMcpAuthCache()
  const entry = cache[serverId]
  if (!entry) return false
  return Date.now() - entry.timestamp < MCP_AUTH_CACHE_TTL_MS
}
```

Cache is stored in `~/.claude/mcp-needs-auth-cache.json`. Writes are serialized through a promise chain to prevent read-modify-write races.

#### Step-Up Auth

When a server returns an authentication error:

```typescript
function handleRemoteAuthFailure(
  name: string,
  serverRef: ScopedMcpServerConfig,
  transportType: 'sse' | 'http' | 'claudeai-proxy',
): MCPServerConnection {
  // 1. Log analytics event
  logEvent('tengu_mcp_server_needs_auth', { transportType, ... })

  // 2. Cache the auth requirement
  setMcpAuthCacheEntry(serverId)

  // 3. Return connection result with needs_auth status
  return { status: 'needs_auth', ... }
}
```

#### Session Expiration Detection

```typescript
// Detects expired sessions via:
// - HTTP 404 status
// - JSON-RPC error code -32001
function isMcpSessionExpiredError(error: Error): boolean
```

### Tool Registration

Each MCP tool is wrapped with the `MCPTool` class that implements the standard `Tool` interface:

```typescript
class MCPTool implements Tool {
  name: string                    // Normalized via normalizeNameForMCP()
  inputSchema: z.ZodType          // Derived from JSON Schema
  inputJSONSchema: ToolInputJSONSchema  // Original JSON Schema
  isMcp: true                     // Marks as MCP tool
  mcpInfo: { serverName, toolName }

  async call(args, context, canUseTool, parentMessage, onProgress) {
    // Execute via MCP client with timeout
    // Handle auth errors, session expiry
    // Return ToolResult
  }
}
```

#### Name Normalization

MCP tool names are normalized to avoid conflicts:

```typescript
// Server "github" with tool "create_issue" → "mcp__github__create_issue"
function normalizeNameForMCP(serverName: string, toolName: string): string
```

#### Description Limits

Tool descriptions are capped at `MAX_MCP_DESCRIPTION_LENGTH` (2048 characters) to avoid oversized prompts.

### Special MCP Tools

| Tool | Purpose |
|------|---------|
| `ListMcpResourcesTool` | Lists resources from MCP servers |
| `ReadMcpResourceTool` | Reads specific MCP resources |
| `McpAuthTool` | Handles MCP authentication flow |

### Connection State

```typescript
type MCPServerConnection = {
  status: 'connected' | 'disconnected' | 'needs_auth' | 'error'
  client: Client
  tools: Tool[]
  resources: ServerResource[]
  serverConfig: ScopedMcpServerConfig
  error?: Error
}
```

---

## File: `src/state/AppStateStore.ts`

### Purpose

Defines the central application state shape (`AppState` type) and the supporting types for completion boundaries, speculation state, and UI elements.

### Core Types

#### CompletionBoundary

Tracks meaningful completion points in the conversation:

```typescript
export type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
```

#### SpeculationState

Tracks speculative (pre-computed) execution:

```typescript
export type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }
    }
```

#### FooterItem

```typescript
export type FooterItem =
  | 'tasks' | 'tmux' | 'bagel' | 'teams' | 'bridge' | 'companion'
```

### AppState Interface

The complete application state, wrapped in `DeepImmutable<>` for safety:

```typescript
export type AppState = DeepImmutable<{
  // ── Settings ──────────────────────────────
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting

  // ── UI State ──────────────────────────────
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  showTeammateMessagePreview?: boolean
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null
  spinnerTip?: string

  // ── Permissions ───────────────────────────
  toolPermissionContext: ToolPermissionContext

  // ── Agent System ──────────────────────────
  agent: string | undefined
  kairosEnabled: boolean
  agentNameRegistry: Map<string, AgentId>

  // ── Remote Session ────────────────────────
  remoteSessionUrl: string | undefined
  remoteConnectionStatus:
    | 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  remoteBackgroundTaskCount: number

  // ── REPL Bridge ───────────────────────────
  replBridgeEnabled: boolean
  replBridgeExplicit: boolean
  replBridgeOutboundOnly: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  replBridgeReconnecting: boolean
  replBridgeConnectUrl: string | undefined
  replBridgeSessionUrl: string | undefined
  replBridgeEnvironmentId: string | undefined
  replBridgeSessionId: string | undefined
  replBridgeError: string | undefined
  replBridgeInitialName: string | undefined
  showRemoteCallout: boolean

  // ── Task Management ───────────────────────
  tasks: { [taskId: string]: TaskState }
  foregroundedTaskId?: string
  viewingAgentTaskId?: string

  // ── MCP Integration ──────────────────────
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }

  // ── Plugin System ─────────────────────────
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: {
      marketplaces: Array<{
        name: string
        status: 'pending' | 'installing' | 'installed' | 'failed'
        error?: string
      }>
    }
  }

  // ── Companion ─────────────────────────────
  companionReaction?: string
  companionPetAt?: number
}>
```

### State Categories

| Category | Key Fields | Purpose |
|----------|-----------|---------|
| **Settings** | `settings`, `verbose`, `mainLoopModel` | User preferences and model config |
| **UI** | `expandedView`, `isBriefOnly`, `footerSelection` | Visual state of the terminal UI |
| **Permissions** | `toolPermissionContext` | Tool execution permissions |
| **Agents** | `agent`, `kairosEnabled`, `agentNameRegistry` | Agent system configuration |
| **Remote** | `remoteSessionUrl`, `remoteConnectionStatus` | Remote session management |
| **REPL Bridge** | `replBridgeEnabled`, `replBridgeConnected` | VS Code / IDE bridge |
| **Tasks** | `tasks`, `foregroundedTaskId` | Background task tracking |
| **MCP** | `mcp.clients`, `mcp.tools`, `mcp.resources` | MCP server state |
| **Plugins** | `plugins.enabled`, `plugins.commands` | Plugin lifecycle |

---

## File: `src/state/AppState.tsx`

### Purpose

React integration layer for `AppStateStore` — provides context providers and hooks for subscribing to state changes within the component tree.

### Contexts

```typescript
const AppStoreContext = React.createContext<AppStateStore | null>(null)
const HasAppStateContext = React.createContext<boolean>(false)
```

Two contexts are used:
- `AppStoreContext` — Holds the store instance (null when outside provider)
- `HasAppStateContext` — Detects nested providers (prevents accidental nesting)

### AppStateProvider Component

```typescript
type Props = {
  children: React.ReactNode
  initialState?: AppState
  onChangeAppState?: (args: {
    newState: AppState
    oldState: AppState
  }) => void
}

export function AppStateProvider({
  children,
  initialState,
  onChangeAppState,
}: Props): React.ReactNode {
  // Prevent nested providers
  const hasAppStateContext = useContext(HasAppStateContext)
  if (hasAppStateContext) {
    throw new Error(
      'AppStateProvider can not be nested within another AppStateProvider',
    )
  }

  // Create store once — stable reference means provider never re-renders
  const [store] = useState(() =>
    createStore<AppState>(
      initialState ?? getDefaultAppState(),
      onChangeAppState,
    ),
  )

  // On-mount check: disable bypass permissions if remote settings say so
  useEffect(() => {
    const { toolPermissionContext } = store.getState()
    if (
      toolPermissionContext.isBypassPermissionsModeAvailable &&
      isBypassPermissionsModeDisabled()
    ) {
      store.setState(prev => ({
        ...prev,
        toolPermissionContext: createDisabledBypassPermissionsContext(
          prev.toolPermissionContext,
        ),
      }))
    }
  }, [])

  // Listen for settings file changes and sync to AppState
  const onSettingsChange = useEffectEvent((source: SettingSource) =>
    applySettingsChange(source, store.setState),
  )
  useSettingsChange(onSettingsChange)

  return (
    <HasAppStateContext.Provider value={true}>
      <AppStoreContext.Provider value={store}>
        <MailboxProvider>
          <VoiceProvider>{children}</VoiceProvider>
        </MailboxProvider>
      </AppStoreContext.Provider>
    </HasAppStateContext.Provider>
  )
}
```

### Provider Wrapping Order

```
<HasAppStateContext.Provider>     ← Nesting guard
  <AppStoreContext.Provider>      ← Store instance
    <MailboxProvider>             ← Messaging system
      <VoiceProvider>             ← Voice mode (conditional, feature-gated)
        {children}
      </VoiceProvider>
    </MailboxProvider>
  </AppStoreContext.Provider>
</HasAppStateContext.Provider>
```

`VoiceProvider` is conditionally compiled — only included in ant builds via `feature('VOICE_MODE')` dead code elimination.

### Exported Hooks

#### `useAppState<T>(selector)` — Selective State Subscription

```typescript
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()

  const get = () => {
    const state = store.getState()
    const selected = selector(state)

    // Ant-only invariant: selector must return a sub-property, not the whole state
    if ("external" === "ant" && state === selected) {
      throw new Error(
        `Your selector must return a property for optimized rendering.`,
      )
    }

    return selected
  }

  return useSyncExternalStore(store.subscribe, get, get)
}
```

**Key behavior**: Uses `useSyncExternalStore` for optimal re-rendering. Only re-renders when the selected value changes (compared via `Object.is`). Call multiple times for independent fields:

```typescript
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)
// Each subscription is independent
```

Do NOT return new objects from the selector:

```typescript
// BAD — new object every time, triggers re-render on every state change
const { text, promptId } = useAppState(s => ({ text: s.promptSuggestion.text, ... }))

// GOOD — select existing sub-object reference
const promptSuggestion = useAppState(s => s.promptSuggestion)
```

#### `useSetAppState()` — State Updater (No Subscription)

```typescript
export function useSetAppState(): (
  updater: (prev: AppState) => AppState
) => void {
  return useAppStore().setState
}
```

Returns a stable function that never changes — components using only this hook will never re-render from state changes. Used for imperative state updates.

#### `useAppStateStore()` — Direct Store Access

```typescript
export function useAppStateStore(): AppStateStore {
  return useAppStore()
}
```

Returns the store instance directly. Useful for passing `getState`/`setState` to non-React code.

#### `useAppStateMaybeOutsideProvider<T>(selector)` — Safe Access

```typescript
export function useAppStateMaybeOutsideProvider<T>(
  selector: (state: AppState) => T,
): T | undefined {
  const store = useContext(AppStoreContext)
  return useSyncExternalStore(
    store ? store.subscribe : NOOP_SUBSCRIBE,
    () => store ? selector(store.getState()) : undefined,
  )
}
```

Returns `undefined` when called outside an `AppStateProvider`. Useful for components that may be rendered in contexts where the provider isn't available.

### Data Flow

```
External Event (settings file change, MCP connect, permission update)
    │
    ▼
applySettingsChange() / store.setState()
    │
    ▼
AppStateStore updates immutable state
    │
    ├─ store.subscribe() notifies all listeners
    │
    ├─ useSyncExternalStore detects change
    │   │
    │   ├─ selector(oldState) !== selector(newState)?
    │   │   ├─ YES → component re-renders
    │   │   └─ NO  → component skips render
    │
    └─ onChangeAppState callback (if provided)
        └─ Parent receives { newState, oldState }
```

### Integration Points

| Integration | How it connects to AppState |
|-------------|----------------------------|
| **Settings** | `useSettingsChange` + `applySettingsChange` sync file changes to state |
| **Permissions** | Bypass permissions mode check on mount, updated via `toolPermissionContext` field |
| **MCP** | `mcp` field holds clients, tools, commands, and resources |
| **Plugins** | `plugins` field tracks enabled/disabled plugins, commands, errors |
| **Remote/Bridge** | Multiple `replBridge*` and `remote*` fields for connection state |
| **Tasks** | `tasks` map + `foregroundedTaskId` for background task tracking |

---

## Key Questions Answered

### How does multi-provider API support work?

`getAnthropicClient()` checks environment variables (`CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_FOUNDRY`, `CLAUDE_CODE_USE_VERTEX`) and returns the appropriate SDK client. All clients are typed as `Anthropic` for uniform usage. Provider-specific configuration (region, project ID, credentials) comes from dedicated environment variables with sensible defaults.

### What is the authentication resolution order?

Bare mode uses only API key or apiKeyHelper. Normal mode checks: approved env var key → file descriptor key → apiKeyHelper (cached) → config/macOS keychain → claude.ai OAuth. Third-party providers (Bedrock, Vertex, Foundry) bypass Anthropic auth entirely and use their own credential chains.

### How do MCP servers connect?

Each MCP server config specifies a transport (stdio, SSE, HTTP, WebSocket, or Claude.ai proxy). The client creates the transport, connects, discovers tools via `listTools()`, and wraps each in an `MCPTool` instance. Auth failures are cached for 15 minutes. Session expiration is detected via HTTP 404 or JSON-RPC error code -32001.

### How is state managed in the React tree?

A centralized `AppStateStore` (created via `createStore`) is provided through React context. Components subscribe to state slices via `useAppState(selector)` which uses `useSyncExternalStore` for efficient, selective re-rendering. The store itself is immutable (`DeepImmutable<>`) and updates through `store.setState(updater)`.

### How do settings changes propagate to the UI?

File watchers detect changes to settings files. The `useSettingsChange` hook in `AppStateProvider` calls `applySettingsChange(source, store.setState)` which updates the relevant `AppState` fields. Because components subscribe selectively, only those viewing changed fields re-render.
