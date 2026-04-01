# Claude Code Architecture Documentation

Guided reading path through the `claude-code-rebuilt` codebase — a TypeScript rebuild of Anthropic's Claude Code CLI (~1,971 TS files). Each document covers one subsystem in depth with code excerpts, data flow diagrams, and key patterns.

---

## Reading Paths by Goal

**I want to understand the full request lifecycle:**
> [Phase 1](phase1-boot-sequence.md) → [Phase 3](phase3-query-engine.md) → [Phase 2](phase2-tool-system.md) → [Phase 4](phase4-terminal-ui.md)

**I want to add a new tool:**
> [Phase 2](phase2-tool-system.md) (the "How to Implement a New Tool" section at the end)

**I want to modify the terminal UI:**
> [Phase 4](phase4-terminal-ui.md) → [Phase 5](phase5-services-state.md) (state hooks)

**I want to add a new slash command:**
> [Phase 6](phase6-command-plugin-system.md)

**I want to understand authentication or add a new API provider:**
> [Phase 5](phase5-services-state.md) (API client + auth sections)

**I want to understand the build system or feature flags:**
> [Phase 7](phase7-build-compatibility-layer.md)

**I want to understand how skills are discovered and selected:**
> [Skill System](skill-system.md)

---

## Phase Index

### [Phase 1: Boot Sequence & Entry Points](phase1-boot-sequence.md)

How the CLI starts and routes to different modes.

| Section | Key Files |
|---------|-----------|
| Module-level side effects | `src/entrypoints/cli.tsx` |
| Fast-paths (`--version`, daemon, MCP) | `src/entrypoints/cli.tsx` |
| Security setup, URL handling | `src/main.tsx` |
| Commander.js `preAction` hook | `src/main.tsx` |
| Default command action | `src/main.tsx` |
| REPL launch bridge | `src/replLauncher.tsx` |

---

### [Phase 2: Tool System Architecture](phase2-tool-system.md)

The declarative tool definition pattern, registration, and execution pipeline.

| Section | Key Files |
|---------|-----------|
| `Tool` interface (all methods) | `src/Tool.ts` |
| `TOOL_DEFAULTS` and `buildTool()` factory | `src/Tool.ts` |
| Tool registration (`getAllBaseTools`, `getTools`, `assembleToolPool`) | `src/tools.ts` |
| Example tools (Glob, Grep, Bash) | `src/tools/GlobTool/`, `src/tools/GrepTool/`, `src/tools/BashTool/` |
| Tool orchestration (concurrent vs serial) | `src/services/tools/toolOrchestration.ts` |
| Streaming tool executor | `src/services/tools/StreamingToolExecutor.ts` |
| How to implement a new tool | — |

---

### [Phase 3: Query Engine & Streaming](phase3-query-engine.md)

The core LLM interaction loop — the heart of the application.

| Section | Key Files |
|---------|-----------|
| `query()` async generator | `src/query.ts` |
| Loop setup (memory prefetch, budget, compaction) | `src/query.ts` |
| Model streaming and tool use detection | `src/query.ts` |
| Tool execution dispatch | `src/query.ts` |
| Tool execution pipeline (validation → permissions → hooks → execute) | `src/services/tools/toolExecution.ts` |
| Concurrency model (partitioning algorithm) | `src/services/tools/toolOrchestration.ts` |
| Error recovery (prompt too long, overloaded, fallback) | `src/query.ts` |

---

### [Phase 4: Terminal UI (React + Ink)](phase4-terminal-ui.md)

How React components render to the terminal via Ink and Yoga.

| Section | Key Files |
|---------|-----------|
| Root wrapper and context providers | `src/components/App.tsx` |
| Main REPL screen (messages, input, streaming) | `src/screens/REPL.tsx` |
| Message rendering (virtual scroll, markdown, tool results) | `src/components/Messages.tsx` |
| Input handling (modes, completions, paste) | `src/components/PromptInput/PromptInput.tsx` |
| Vendored Ink renderer (Yoga, ANSI) | `src/ink/` |
| State management (AppStateStore, hooks) | `src/state/AppStateStore.ts`, `src/state/AppState.tsx` |
| Performance patterns (virtual scroll, memo, deferred values) | — |

---

### [Phase 5: Services & State Management](phase5-services-state.md)

The infrastructure layer — API access, authentication, MCP integration, and state.

| Section | Key Files |
|---------|-----------|
| Multi-provider API client (Direct, Bedrock, Vertex, Foundry) | `src/services/api/client.ts` |
| Authentication resolution (OAuth, API key, keychain) | `src/utils/auth.ts` |
| MCP client (transports, tool registration, auth cache) | `src/services/mcp/client.ts` |
| AppState type definition (all fields) | `src/state/AppStateStore.ts` |
| React provider and `useSyncExternalStore` hooks | `src/state/AppState.tsx` |

---

### [Phase 6: Command & Plugin System](phase6-command-plugin-system.md)

Extensibility via slash commands, skills, and plugins.

| Section | Key Files |
|---------|-----------|
| Command types (`prompt`, `local`, `local-jsx`) | `src/types/command.ts` |
| Command registry (built-in, skills, plugins, MCP) | `src/commands.ts` |
| Bundled skills | `src/skills/bundledSkills.ts` |
| File-based skill loading | `src/skills/loadSkillsDir.ts` |
| Plugin types and lifecycle | `src/types/plugin.ts` |
| Plugin loader | `src/utils/plugins/pluginLoader.ts` |
| Plugin operations (install, uninstall, update) | `src/services/plugins/pluginOperations.ts` |
| Plugin policy enforcement | `src/utils/plugins/pluginPolicy.ts` |

---

### [Phase 7: Build & Compatibility Layer](phase7-build-compatibility-layer.md)

How the project builds independently from Anthropic internals.

| Section | Key Files |
|---------|-----------|
| Bun bundle shim (`feature()` DCE) | `src/_external/bun-bundle.ts` |
| Runtime feature flags | `src/_external/preload.ts` |
| Stub packages for private deps | `src/_external/shims/` |
| Package configuration | `package.json` |
| TypeScript configuration | `tsconfig.json` |
| Bun build configuration | `bunfig.toml` |
| Feature flag system (88+ flags) | — |
| Dependency resolution strategy | — |

---

## Architecture Flow

```
User types in terminal
        │
        ▼
[Phase 1] cli.tsx → main.tsx → launchRepl()
        │
        ▼
[Phase 4] PromptInput → REPL.handleSubmit()
        │
        ▼
[Phase 3] query() async generator
        │          │
        │          ▼
        │   callModel() → streaming blocks
        │          │
        │          ▼
        │   [Phase 2] Tool execution
        │   (partition → validate → permission → execute → hooks)
        │          │
        │          ▼
        │   Yield results back to REPL
        │
        ▼
[Phase 5] Services:
        ├─ API client (Anthropic / Bedrock / Vertex / Foundry)
        ├─ Auth (OAuth / API key / keychain)
        ├─ MCP (stdio / SSE / HTTP / WebSocket)
        └─ AppState → React hooks → UI re-render
```
