# Phase 6: Command & Plugin System

## Overview

The command and plugin system provides the multi-layered extensibility architecture for Claude Code. Slash commands come from four sources -- built-in commands, bundled skills, file-based skills from disk, plugin-provided commands, MCP skills, and workflow commands. The plugin layer adds marketplace-based distribution with lifecycle management (install, uninstall, enable, disable, update), policy enforcement, and dependency resolution.

```
Command Resolution Flow:

User types /<name> <args>
  └─ REPL input handler
       └─ findCommand(name, getCommands(cwd))
            │
            ├─ Bundled skills        (loaded at startup, compiled in)
            ├─ Built-in plugins      (shipped with CLI, enabled/disabled via user settings)
            ├─ File-based skills     (discovered from .claude/skills/ directories)
            ├─ Plugin commands      (loaded from marketplace plugins)
            ├─ Plugin skills        (loaded from marketplace plugins)
            ├─ Workflow commands    (feature-gated, from .claude/workflows/)
            ├─ Dynamic skills       (discovered during file operations)
            └─ Built-in commands    (hardcoded in commands.ts)
                 │
                 ├─ type: 'prompt'    → getPromptForCommand() → text injection
                 ├─ type: 'local'     → load().call() → LocalCommandResult
                 └─ type: 'local-jsx' → load().call() → Ink UI render

Plugin Loading Flow:

Startup
  └─ loadAllPlugins()
       ├─ loadInstalledPluginsV2()     (disk-based tracking)
       ├─ loadBuiltinPlugins()          (compiled into CLI)
       ├─ loadPluginsFromSettings()     (user/project/local settings)
       └─ getInlinePlugins()            (--plugin-dir session plugins)
            │
            └─ For each plugin:
                 ├─ loadPluginManifest()       (plugin.json validation)
                 ├─ loadCommands()             (commands/ and manifest.commands)
                 ├─ loadSkills()               (skills/ and manifest.skills)
                 ├─ loadAgents()               (agents/ and manifest.agents)
                 ├─ loadHooks()                (hooks/hooks.json + manifest.hooks)
                 ├─ loadMcpServers()           (.mcp.json + manifest.mcpServers)
                 ├─ loadLspServers()           (manifest.lspServers)
                 └─ loadOutputStyles()         (output-styles/ + manifest.outputStyles)
                      │
                      └─ verifyAndDemote()      (dependency resolution)
                           → PluginLoadResult { enabled, disabled, errors }
```

---

## File: `src/types/command.ts`

### The `Command` Type (Discriminated Union)

Every slash command is a discriminated union of `CommandBase` and one of three execution strategies:

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**CommandBase** -- shared fields across all command types:

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `name` | `string` | required | Slash command name (e.g., `"init"`) |
| `description` | `string` | required | One-line description for help/typeahead |
| `aliases` | `string[]` | optional | Alternate names (e.g., `["q"]`) |
| `availability` | `CommandAvailability[]` | optional | Auth gates: `'claude-ai'` or `'console'` |
| `isEnabled` | `() => boolean` | `true` | Feature flag or env-var gating |
| `isHidden` | `boolean` | `false` | Hide from typeahead/help |
| `source` | `SettingSource \| 'builtin' \| 'mcp' \| 'plugin' \| 'bundled'` | required | Where the command originates |
| `loadedFrom` | `LoadedFrom` | optional | Provenance for skill filtering |
| `disableModelInvocation` | `boolean` | `false` | If true, model cannot invoke this skill |
| `userInvocable` | `boolean` | `true` | If true, users can type `/name` |
| `whenToUse` | `string` | optional | Detailed usage guidance for the model |
| `argumentHint` | `string` | optional | Gray hint text shown after command |
| `immediate` | `boolean` | `false` | Execute immediately, bypassing queue |
| `isSensitive` | `boolean` | `false` | Redact args from conversation history |

### PromptCommand

Text-based commands that inject content into the conversation. Skills are the primary example.

 The `getPromptForCommand(args, context)` method returns `ContentBlockParam[]` (typically text blocks).

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string        // Shown while content loads
  contentLength: number           // Character count for token estimation
  argNames?: string[]             // Named arguments for $ARGUMENTS
  allowedTools?: string[]         // Tools available during invocation
  model?: string                  // Override model for this invocation
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  hooks?: HooksSettings           // Hooks registered during invocation
  skillRoot?: string              // Base dir for CLAUDE_SKILL_DIR substitution
  context?: 'inline' | 'fork'    // Execution context
  agent?: string                  // Agent type in fork context
  effort?: EffortValue             // Effort level override
  paths?: string[]                // Glob patterns for conditional skill activation
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}
```

### LocalCommand

Pure-function commands that return a result without UI rendering:

```typescript
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<{ call: (args, context) => Promise<LocalCommandResult> }>
}
```

`LocalCommandResult` is a discriminated union:
- `{ type: 'text'; value: string }` -- plain text output
- `{ type: 'compact'; compactionResult; displayText? }` -- triggers context compaction
- `{ type: 'skip' }` -- skip message

### LocalJSXCommand

React-rendering commands that display interactive Ink UI:

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<{ call: (onDone, context, args) => Promise<ReactNode> }>
}
```

The `onDone` callback receives optional configuration:
- `display`: `'skip' | 'system' | 'user'` -- how to display the result
- `shouldQuery`: `boolean` -- whether to send to the model after completion
- `metaMessages`: `string[]` -- model-visible but hidden messages
- `nextInput`: `string` -- auto-submit follow-up input

---

## File: `src/commands.ts`

### Command Registry

The central command registry lives in `src/commands.ts`. It assembles commands from all sources into a single ordered list.

**Key constant: `COMMANDS`** -- A memoized function returning the full array of built-in commands:

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color, compact,
 config, copy, desktop, context, contextNonInteractive, cost, diff, doctor, effort, exit, fast, files, heapDump, help, ide, init, keybindings, installGitHubApp, installSlackApp, mcp, memory, mobile, model, outputStyle, remoteEnv, plugin, pr_comments, releaseNotes, reloadPlugins, rename, resume, session, skills, stats, status, statusline, stickers, tag, theme, feedback, review, ultrareview, rewind, securityReview, terminalSetup, upgrade, extraUsage, extraUsageNonInteractive, rateLimitOptions, usage, usageReport, vim, thinkback, thinkbackPlay, permissions, plan, privacySettings, hooks, exportCommand, sandboxToggle, ...loginLogout, passes, tasks, ...CONDITIONAL_COMMANDS
 ])
```

### Feature-Flagged Commands

Several commands are conditionally loaded based on feature flags. These use dynamic `require()` to ensure dead code elimination in external builds:

| Command | Feature Flag | Purpose |
|---------|-------------|---------|
| `proactive` | `PROACTIVE` or `KAIROS` | Proactive assistance |
| `brief` | `KAIROS` or `KAIROS_BRIEF` | Brief summaries |
| `assistant` | `KAIROS` | Assistant mode |
| `bridge` | `BRIDGE_MODE` | Remote bridge mode |
| `remoteControlServer` | `DAEMON && BRIDGE_MODE` | Remote control server |
| `voice` | `VOICE_MODE` | Voice interaction |
| `workflows` | `WORKFLOW_SCRIPTS` | Workflow scripting |
| `web` | `CCR_REMOTE_SETUP` | Remote setup |
| `buddy` | `BUDDY` | Companion UI |
| `fork` | `FORK_SUBAGENT` | Fork subagent |
| `torch` | `TORCH` | Torch mode |
| `peers` | `UDS_INBOX` | Peer inbox |
| `subscribePr` | `KAIROS_GITHUB_WEBHOOKS` | PR subscription |
| `ultraplan` | `ULTRAPLAN` | Ultra plan mode |

### Internal-Only Commands

The `INTERNAL_ONLY_COMMANDS` array lists commands excluded from the external build:

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits,
  bridgeKick, version, resetLimits, resetLimitsNonInteractive,
  onboarding, share, summary, teleport, antTrace,
  perfIssue, env, oauthRefresh, debugToolCall,
  agentsPlatform, autofixPr,
].filter(Boolean)
```

These commands are only included only the `COMMANDS()` result only only when `process.env.USER_TYPE === 'ant'` and not in demo mode.

 They are filtered out in external builds by the build script.

### Command Loading Pipeline

`getCommands(cwd)` is the primary entry point, returning all available commands:

```typescript
async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)  // Memoized

  const dynamicSkills = getDynamicSkills()              // Runtime discovery

  // Filter by availability and enabled state
  const baseCommands = allCommands.filter(
    _ => meetsAvailabilityRequirement(_) && isCommandEnabled(_)
  )

  // Deduplicate and insert dynamic skills
  return [...baseCommands, ...uniqueDynamicSkills]
}
```

**`loadAllCommands(cwd)`** -- memoized by cwd, loads from five sources in order:

1. `bundledSkills` -- registered at startup via `registerBundledSkill()`
2. `builtinPluginSkills` -- from enabled built-in plugins
3. `skillDirCommands` -- from `.claude/skills/` directories
4. `workflowCommands` -- from workflow scripts
5. `pluginCommands` -- from marketplace plugins
6. `pluginSkills` -- from marketplace plugins
7. `COMMANDS()` -- hardcoded built-in commands

8. `dynamicSkills` -- discovered during file operations (not memoized)

 -- deduplicated and inserted between plugin skills and built-in commands

### Availability Filtering
`meetsAvailabilityRequirement(cmd)` checks auth a command's `availability` array:

- `'claude-ai'` -- available to Claude AI subscribers (Pro/Max/Team/Enterprise)
 users
- `'console'` -- available to direct Console API key users (not 3P, not claude.ai subscriber)

 If no availability is declared, the command is available everywhere.

 re-evaluated fresh on every call because auth state can change mid-session (e.g., after `/login`).

 `isCommandEnabled(cmd)` defaults to `true` when `isEnabled` is not defined.

 The checks GrowthBook feature flags, environment variables, or platform conditions.

### Remote Mode Filtering
Commands are filtered for remote mode using two allowlists:

**`REMOTE_SAFE_COMMANDS`** -- commands safe when connected via remote mode (SSH, CCR):
 `session`, `exit`, `clear`, `help`, `theme`, `color`, `vim`, `cost`, `usage`, `copy`, `btw`, `feedback`, `plan`, `keybindings`, `statusline`, `stickers`, `mobile`

**`BRIDGE_SAFE_COMMANDS`** -- local-type commands safe for mobile/web bridge: `compact`, `clear`, `cost`, `summary`, `releaseNotes`, `files`

`isBridgeSafeCommand(cmd)` determines bridge safety:
- `prompt` commands: always safe (expand to text, sent to model)
- `local-jsx` commands: always blocked (render Ink UI)
- `local` commands: allowed only if in `BRIDGE_SAFE_COMMANDS`

### Cache Management
Command loading is memoized for performance. Cache invalidation occurs via:

| Function | Scope |
|----------|-------|
| `clearCommandMemoizationCaches()` | Clears `loadAllCommands`, `getSkillToolCommands`, `getSlashCommandToolSkills`, skill index |
| `clearCommandsCache()` | Full reset: command caches + plugin caches + skill caches |

Dynamic skills trigger cache invalidation via a signal:
1. `skillsLoaded` signal fires when dynamic skills are added
2. Listeners call `clearCommandMemoizationCaches()` to refresh the command index
3. The skill index in `skillSearch/localSearch.ts` is cleared explicitly

 it sits on top of the lodash memoize layer

---

## File: `src/skills/bundledSkills.ts`

### Bundled Skill Registration

Bundled skills are compiled into the CLI and available to all users. They are registered at startup via `registerBundledSkill()`.

```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // Reference files extracted to disk on first invocation
  getPromptForCommand: (args, context) => Promise<ContentBlockParam[]>
}
 -> void
```

The registration process:
1. If `files` is defined, compute a deterministic extraction directory
2. Wrap `getPromptForCommand` with lazy file extraction
3. Create a `Command` object with `source: 'bundled'` and `loadedFrom: 'bundled'`
4. Push to the `bundledSkills` array

5. `getBundledSkills()` returns a copy of the array

**File Extraction Security**: Files are extracted using `O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW` flags with mode `0o600`, preventing symlink attacks and race conditions. The extraction directory uses a per-process nonce from `getBundledSkillsRoot()`.

---

## File: `src/skills/loadSkillsDir.ts`

### File-Based Skill Discovery

Skills are discovered from `.claude/skills/` directories at a layered search:

**Search Order** (loaded in parallel):
1. `managedSkillsDir` -- `$MANAGED_PATH/.claude/skills/` (enterprise policy)
2. `userSkillsDir` -- `~/.claude/skills/` (user-level)
3. `projectSkillsDirs` -- `.claude/skills/` in project and parent directories
4. `additionalDirs` -- `--add-dir` paths' `.claude/skills/`
5. Legacy `commands/` directories (deprecated format)

**Skill Format** -- directory-based only in `/skills/`:
```
skill-name/
  └── SKILL.md    # Required, contains frontmatter + instructions
```

**Legacy Format** -- file-based in `/commands/` (deprecated):
```
commands/
  ├── command.md         # Single-file command
  └── skill-dir/
       └── SKILL.md      # Directory-based skill command
```

### Frontmatter Parsing

`parseSkillFrontmatterFields()` extracts all frontmatter fields:

| Field | Type | Purpose |
|-------|------|---------|
| `name` | `string` | Display name override |
| `description` | `string` | Required description (fallback: first markdown paragraph) |
| `when_to_use` | `string` | Detailed usage guidance |
| `allowed-tools` | `string[]` | Tool whitelist for invocation |
| `arguments` | `string \| string[]` | Named arguments for `$ARGUMENTS` |
| `argument-hint` | `string` | Typeahead hint |
| `disable-model-invocation` | `boolean` | Block model invocation |
| `user-invocable` | `boolean` | Block user invocation |
| `context` | `'fork'` | Run as sub-agent |
| `agent` | `string` | Agent type in fork mode |
| `effort` | `EffortValue` | Effort level override |
| `model` | `string` | Model override |
| `hooks` | `HooksSettings` | Hooks registered during invocation |
| `shell` | `FrontmatterShell` | Shell command execution context |
| `paths` | `string` | Glob patterns for conditional activation |

### Content Processing Pipeline

When a skill is invoked via `getPromptForCommand()`:
1. Prepend base directory line if `baseDir` is set
2. Substitute `$ARGUMENTS` and named arguments
3. Replace `${CLAUDE_SKILL_DIR}` with the skill's directory
4. Replace `${CLAUDE_SESSION_ID}` with the current session ID
5. Execute inline shell commands (`!`...`) unless loaded from MCP
6. Return text content blocks

### Deduplication
Skills are deduplicated by resolved file path (using `realpath()` to handle symlinks and overlapping directories). The first-loaded skill wins.

### Dynamic Skill Discovery
During file operations, the system discovers new skill directories by walking up from the operated file path to the cwd:

```
File operation (Read/Write/Edit)
  └─ discoverSkillDirsForPaths(filePaths, cwd)
       └─ Walk from file parent to cwd
            └─ Check .claude/skills/ at each level
                 ├─ Skip gitignored directories
                 └─ Record new directories (deepest first)
                            │
                            └─ addSkillDirectories(dirs)
                                 └─ loadSkillsFromSkillsDir() for each
                                      └─ Merge into dynamicSkills map
                                           └─ skillsLoaded signal
                                                └─ clearCommandMemoizationCaches()
```

### Conditional Skills
Skills with `paths` frontmatter are conditional -- they are not loaded until a matching file is touched:

```
Skill loaded with paths: ["src/**/*.ts"]
  └─ Stored in conditionalSkills map (not in command list)
       │
       File operation touches "src/foo.ts"
         └─ activateConditionalSkillsForPaths(filePaths, cwd)
              └─ For each conditional skill:
                   └─ Check if any filePath matches paths patterns
                        └─ If match: move to dynamicSkills, fire signal
```

---

## File: `src/types/plugin.ts`

### Plugin Type Definitions

**`LoadedPlugin`** -- a fully loaded plugin instance:

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest           // Validated plugin.json
  path: string                        // Installation path
  source: string                      // Marketplace source
  repository: string                  // Marketplace identifier
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string                        // Git commit SHA for versioning
  commandsPath?: string
  commandsPaths?: string[]             // Additional command paths
  commandsMetadata?: Record<string, CommandMetadata>
  agentsPath?: string
  agentsPaths?: string[]
  skillsPath?: string
  skillsPaths?: string[]
  outputStylesPath?: string
  outputStylesPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

**`PluginLoadResult`** -- result of loading all plugins:

```typescript
type PluginLoadResult = {
  enabled: LoadedPlugin[]    // Successfully loaded and enabled
  disabled: LoadedPlugin[]   // Loaded but disabled in settings
  errors: PluginError[]      // Load failures with details
}
```

### Plugin Error System

A discriminated union of 20+ error types providing structured error reporting:

| Error Type | Key Data | When It Occurs |
|------------|----------|----------------|
| `generic-error` | `error: string` | Catch-all for unclassified failures |
| `plugin-not-found` | `pluginId, marketplace` | Plugin not in marketplace |
| `path-not-found` | `path, component` | Plugin directory missing |
| `git-auth-failed` | `gitUrl, authType` | Git clone authentication failure |
| `git-timeout` | `gitUrl, operation` | Git clone/pull timeout |
| `manifest-parse-error` | `manifestPath, parseError` | Invalid JSON in plugin.json |
| `manifest-validation-error` | `manifestPath, validationErrors[]` | Schema validation failure |
| `marketplace-not-found` | `marketplace, availableMarketplaces[]` | Unknown marketplace name |
| `marketplace-blocked-by-policy` | `marketplace, blockedByBlocklist, allowedSources[]` | Enterprise policy blocks |
| `mcp-config-invalid` | `serverName, validationError` | MCP server config schema failure |
| `mcp-server-suppressed-duplicate` | `serverName, duplicateOf` | Duplicate MCP server skipped |
| `dependency-unsatisfied` | `dependency, reason` | Required dependency not enabled/found |
| `mcpb-download-failed` | `url, reason` | MCPB file download failure |
| `lsp-server-start-failed` | `serverName, reason` | LSP server process failed to start |
| `lsp-server-crashed` | `serverName, exitCode, signal` | LSP server crashed mid-session |
| `lsp-request-timeout` | `serverName, method, timeoutMs` | LSP request exceeded timeout |

`getPluginErrorMessage(error)` provides human-readable messages for each error type.

---

## File: `src/utils/plugins/pluginLoader.ts`

### Plugin Loading Architecture

The plugin loader discovers, validates, and materializes plugins from multiple sources:

**Discovery Sources** (in order of precedence):
1. **Marketplace plugins** -- `plugin@marketplace` format in settings
2. **Session-only plugins** -- `--plugin-dir` CLI flag or SDK `plugins` option

**Plugin Directory Structure:**
```
my-plugin/
  ├── .claude-plugin/
  │    └── plugin.json         # Optional manifest
  ├── commands/                # Custom slash commands
  │   ├── build.md
  │   └── deploy.md
  ├── agents/                  # Custom AI agents
  │   └── test-runner.md
  ├── skills/                  # Skill directories
  │   └── my-skill/
  │        └── SKILL.md
  ├── hooks/                   # Hook configurations
  │   └── hooks.json
  ├── output-styles/           # Output style overrides
  ├── .mcp.json                # MCP server configs
  └── .lsp.json                # LSP server configs
```

### Loading Process

1. **`loadAllPlugins()`** -- loads all plugins from all sources
   - Reads `installed_plugins_v2.json` for disk-tracked installations
   - Loads built-in plugins from `BUILTIN_MARKETPLACE_NAME`
   - Loads settings-declared plugins (user, project, local scopes)
   - Loads inline plugins from `--plugin-dir` and SDK
   - Runs `verifyAndDemote()` for dependency resolution
   - Returns `PluginLoadResult { enabled, disabled, errors }`

2. **`cachePlugin(source, manifestHint)`** -- materializes a remote plugin
   - Handles git clone (full or sparse for git-subdir)
   - Handles NPM package extraction
   - Validates manifest against `PluginManifestSchema`
   - Returns `{ path, manifest, gitCommitSha }`

3. **`loadPluginManifest(path, name, source)`** -- validates a plugin.json
   - Reads and parses JSON
   - Validates against `PluginManifestSchema` (Zod)
   - Returns `PluginManifest` or throws descriptive error

---

## File: `src/services/plugins/pluginOperations.ts`

### Plugin CRUD Operations

Pure library functions for plugin management, usable by both CLI commands and interactive UI. Functions never call `process.exit()` or write to console directly -- they return structured result objects.

**Installation Scopes** (most local to most global):

| Scope | Settings File | Install Location |
|-------|--------------|-----------------|
| `local` | `.claude/settings.local.json` | Per-project, personal |
| `project` | `.claude/settings.json` | Per-project, shared with team |
| `user` | `~/.claude/settings.json` | Global, personal |
| `managed` | Managed settings path | Enterprise, read-only |

### `installPluginOp(plugin, scope)`

Settings-first installation:
1. Search materialized marketplaces for the plugin
2. Resolve the plugin via `installResolvedPlugin()`:
   - Check policy (blocked by org?)
   - Check dependencies (blocked dependency?)
   - Write settings (`enabledPlugins[pluginId] = true`)
   - Cache plugin to versioned directory
   - Record in `installed_plugins_v2.json`
   - Resolve dependency closure
3. Return `PluginOperationResult`

### `uninstallPluginOp(plugin, scope, deleteDataDir)`

1. Resolve plugin from loaded plugins or V2 installed data (handles delisted plugins)
2. Find scope-specific installation in V2 data
3. Remove from settings (`enabledPlugins[pluginId] = undefined`)
4. Remove from V2 installed plugins file
5. If last scope, orphan old version and delete plugin data
6. Warn about reverse dependents (but do not block)

### `setPluginEnabledOp(plugin, enabled, scope)`

Enable/disable with scope resolution:
1. Built-in plugins: write to user settings directly
2. External plugins: resolve pluginId and scope from settings
3. Check policy guard (org-blocked plugins cannot be enabled)
4. Check cross-scope hints (plugin installed elsewhere)
5. Write settings and clear caches
6. On disable: capture reverse dependents for warning

7. Return result with dependency warning

### `updatePluginOp(plugin, scope)`

Non-inplace update:
1. Get plugin info from marketplace
2. Get installations from disk (V2 file)
3. For remote plugins: download to temp, calculate version
4. For local plugins: stat source path, calculate version
5. If version unchanged: return `alreadyUpToDate`
6. Copy to new versioned cache directory
7. Update V2 file on disk (memory unchanged until restart)
8. Mark old version orphan if no longer referenced

---

## File: `src/utils/plugins/schemas.ts`

### Plugin Manifest Schema

The plugin system uses Zod schemas for validate plugin manifests, marketplace configurations, and installation tracking. All schemas use `lazySchema()` to avoid circular dependency issues.

**`PluginManifestSchema`** -- validates `plugin.json`:

```
{
  name: string             // Required, kebab-case
  version?: string          // Semantic version
  description?: string
  author?: { name, email?, url? }
  homepage?: string (URL)
  repository?: string (URL)
  license?: string          // SPDX identifier
  keywords?: string[]
  dependencies?: DependencyRef[]

  commands?: string | string[] | Record<string, CommandMetadata>
  agents?: string | string[]
  skills?: string | string[]
  outputStyles?: string | string[]
  hooks?: string | HooksSettings | (string | HooksSettings)[]
  mcpServers?: string | Record<string, McpServerConfig> | (string | ...)[]
  lspServers?: string | Record<string, LspServerConfig> | (string | ...)[]
  channels?: { server, displayName?, userConfig? }[]
  userConfig?: Record<string, UserConfigOption>
  settings?: Record<string, unknown>
}
```

**`PluginMarketplaceSchema`** -- validates `marketplace.json`:

```
{
  name: string               // Unique marketplace name
  owner: { name, email?, url? }
  plugins: PluginMarketplaceEntry[]
  forceRemoveDeletedPlugins?: boolean
  metadata?: { pluginRoot?, version?, description? }
  allowCrossMarketplaceDependenciesOn?: string[]
}
```

**`PluginSourceSchema`** -- defines where a plugin comes from:

| Source Type | Fields | Purpose |
|-------------|-------|---------|
| `string` (relative path) | `./path` | Local to marketplace directory |
| `npm` | `package, version?, registry?` | NPM package |
| `pip` | `package, version?, registry?` | Python package |
| `url` | `url, ref?, sha?` | Git clone from URL |
| `github` | `repo, ref?, sha?` | GitHub shorthand |
| `git-subdir` | `url, path, ref?, sha?` | Monorepo subdirectory |

**`MarketplaceSourceSchema`** -- defines where a marketplace comes from:

| Source Type | Fields | Purpose |
|-------------|-------|---------|
| `url` | `url, headers?` | Direct URL to marketplace.json |
| `github` | `repo, ref?, path?, sparsePaths?` | GitHub repository |
| `git` | `url, ref?, path?, sparsePaths?` | Git repository URL |
| `npm` | `package` | NPM package |
| `file` | `path` | Local file path |
| `directory` | `path` | Local directory path |
| `settings` | `name, plugins[], owner?` | Inline marketplace in settings |
| `hostPattern` | `hostPattern` | Regex for allow marketplace hosts |
| `pathPattern` | `pathPattern` | Regex in allow filesystem paths |

### Security Validations

- **Official name protection**: `BLOCKED_OFFICIAL_NAME_PATTERN` blocks impersonation attempts
- **Non-ASCII rejection**: Prevents homograph attacks via Unicode
- **Reserved names**: `ALLOWED_OFFICIAL_MARKETPLACE_NAMES` are restricted to `anthropics/*` repos
- **Path traversal**: All relative paths must start with `./` and cannot contain `..`

---

## File: `src/utils/plugins/pluginPolicy.ts`

### Policy Enforcement

A leaf module (only imports settings) that provides enterprise policy enforcement:

```typescript
function isPluginBlockedByPolicy(pluginId: string): boolean {
  const policyEnabled = getSettingsForSource('policySettings')?.enabledPlugins
  return policyEnabled?.[pluginId] === false
}
```

Managed settings (`policySettings`) can force-disable any plugin at any scope. This check is used at:
1. Install chokepoint -- blocked plugins cannot be installed
2. Enable operation -- blocked plugins cannot be enabled
3. UI filters -- blocked plugins are hidden from management UI

---

## File: `src/commands/plugin/plugin.tsx`

### Plugin Management UI

The `/plugin` command provides interactive plugin management with subcommands:
- `/plugin` (no args) -- opens plugin browser/management UI
- `/plugin install <name>` -- install from marketplace
- `/plugin uninstall <name>` -- remove plugin
- `/plugin enable <name>` -- enable installed plugin
- `/plugin disable <name>` -- disable installed plugin
- `/plugin update <name>` -- update to latest version

All operations support `--scope user|project|local` for scope control.

---

## File: `src/commands/reload-plugins/reload-plugins.ts`

### Plugin Reload Command

The `/reload-plugins` command refreshes the plugin system without restarting the CLI:

1. In remote mode: re-download user settings from settings sync
2. Call `refreshActivePlugins()` to reload all plugins
3. Report counts: enabled plugins, skills, agents, hooks, MCP servers, LSP servers
 and errors

---

## File: `src/commands/init.ts`

### `/init` Command

The `/init` command analyzes the codebase and creates project configuration through a multi-phase interview:

**Phases:**
1. Ask what to set up (CLAUDE.md files + skills/hooks choice)
2. Explore codebase (manifests, README, CI, existing config)
3. Fill gaps (interview questions for code can't answer)
4. Write CLAUDE.md (project or personal)
5. Write CLAUDE.local.md (personal preferences)
6. Create skills (`.claude/skills/<name>/SKILL.md`)
7. Suggest additional optimizations (GitHub CLI, linting, hooks)
8. Summary and next steps

 including plugin recommendations)

---

## Skill Tool Integration

The command system provides two model-facing entry points for skill discovery:

**`getSkillToolCommands(cwd)`** -- all prompt commands the model can invoke:
- Must be `type: 'prompt'`
- Must not `disableModelInvocation`
- Must not be `source: 'builtin'`
- Must have `loadedFrom` as 'bundled', 'skills', 'commands_DEPRECATED', or have `hasUserSpecifiedDescription` or `whenToUse`

**`getSlashCommandToolSkills(cwd)`** -- skills specifically (excludes built-in commands):
- Must be `type: 'prompt'`
- Must not `source: 'builtin'`
- Must have `hasUserSpecifiedDescription` or `whenToUse`
- Must have `loadedFrom` as 'skills', 'plugin', 'bundled', or `disableModelInvocation`

**`getMcpSkillCommands(mcpCommands)`** -- MCP-provided skills:
- Gated by `feature('MCP_SKILLS')`
- Must be `type: 'prompt'`, `loadedFrom: 'mcp'`, not `disableModelInvocation`

---

## Command Description Formatting

`formatDescriptionWithSource(cmd)` annotates descriptions for user-facing UI:

- Workflow commands: `(workflow)` suffix
- Plugin commands: `(plugin-name)` prefix
- Bundled commands: `(bundled)` suffix
- Built-in/MCP: unmodified
- Skills: `(source-name)` suffix from `getSettingSourceName()`
