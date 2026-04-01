# Phase 7: Build & Compatibility Layer

## Overview

The build system produces an external-compatible CLI binary from Bun's native TypeScript/JSX bundler. It uses a feature flag system for dead code elimination (DCE), shims Bun-specific APIs for compatibility with standard Node.js, and resolves platform-specific native modules via stub packages. The external build strips all internal-only features, leaving only a small set of enabled features for while preserving the full command set available to end users.

 The result is a single self ESM bundle that runs on Bun.

```
Build Pipeline:

bun run scripts/build-external.ts
  └─ Bun.build()
       ├─ Entry:      src/entrypoints/cli.tsx
       ├─ Output:     dist/cli.js (ESM, linked sourcemaps)
       ├─ Target:     bun
       ├─ Plugins:    bun-bundle-shim (replaces bun:bundle)
       ├─ Define:     MACRO.VERSION, MACRO.BUILD_TIME, etc.
       ├─ Loader:     .md → text
       └─ External:   @anthropic-ai/bedrock-sdk, sharp, bun:ffi, etc.
            │
            └─ dist/cli.js + dist/cli.js.map
```

---

## File: `scripts/build-external.ts`

### Build Entry Point

The external build script configures and runs `Bun.build()` to produce the distributable CLI bundle. It defines three key data structures:

### Disabled Features (`EXTERNAL_DISABLED_FEATURES`)

A hardcoded array of ~88 feature names that are **disabled** in the external build. Every feature in this list is excluded from the external binary, and any code gated by `feature('FEATURE_NAME')` is eliminated at build time through dead code elimination.

Major categories of disabled features:

| Category | Features | Purpose |
|----------|----------|---------|
| Internal tooling | `ABLATION_BASELINE`, `AGENT_MEMORY_SNAPSHOT`, `TREE_SITTER_BASH` | Internal testing/analysis |
| Platform features | `BRIDGE_MODE`, `BUDDY`, `DAEMON`, `VOICE_MODE` | Mobile/desktop platform integration |
| Enterprise | `CHICAGO_MCP`, `COMMIT_ATTRIBUTION`, `ANTI_DISTILLATION_CC` | Internal compliance/monitoring |
| Experimental | `EXPERIMENTAL_SKILL_SEARCH`, `SKILL_IMPROVEMENT`, `TOKEN_BUDGET` | Experimental features |
| Remote | `CCR_AUTO_CONNECT`, `CCR_MIRROR`, `CCR_REMOTE_SETUP`, `SSH_REMOTE` | Container/cloud remote features |
| Assistant | `KAIROS`, `KAIROS_BRIEF`, `KAIROS_CHANNELS`, `KAIROS_GITHUB_WEBHOOKS` | Assistant/automation platform |
| Advanced | `ULTRAPLAN`, `ULTRATHINK`, `TORCH`, `PROACTIVE` | Advanced reasoning/planning |

### Enabled Features (`ENABLED_FEATURES`)

A small set of features that **are enabled** in the external build:

| Feature | Default | Purpose |
|---------|---------|---------|
| `AUTO_THEME` | `true` | Automatic terminal theme detection |
| `BREAK_CACHE_COMMAND` | `true` | `/break-cache` debug command availability |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `true` | Built-in explore/plan agent types |

### Feature Shim Plugin (`bunBundlePlugin`)

The build registers a Bun plugin that intercepts all imports of `bun:bundle`:

```typescript
const bunBundlePlugin = {
  name: "bun-bundle-shim",
  setup(build) {
    // Intercept bun:bundle imports
    build.onResolve({ filter: /^bun:bundle$/ }, () => ({
      path: "bun:bundle",
      namespace: "bun-bundle-shim",
    }));
    // Replace with inline feature() function
    build.onLoad({ filter: /.*/, namespace: "bun-bundle-shim" }, () => ({
      contents: featureModuleCode,  // The generated feature() function
      loader: "js",
    }));
  },
};
```

The generated `featureModuleCode` is:

```javascript
export function feature(name) {
  const ENABLED = ["AUTO_THEME", "BREAK_CACHE_COMMAND", "BUILTIN_EXPLORE_PLAN_AGENTS"];
  return ENABLED.includes(name);
}
```

This replaces the runtime `FEATURE_MAP` lookup with a compile-time constant array. Bun's bundler can then eliminate all code paths gated by disabled features.

 The `EXTERNAL_DISABLED_FEATURES` array is documentation-only -- the enabled set is what matters at runtime.

### Macro Defines

The build injects compile-time constants via Bun's `define` option:

```typescript
define: {
  "MACRO.VERSION": JSON.stringify(version),
  "MACRO.BUILD_TIME": JSON.stringify(new Date().toISOString()),
  "MACRO.PACKAGE_URL": JSON.stringify("https://www.npmjs.com/package/@anthropic-ai/claude-code"),
  "MACRO.NATIVE_PACKAGE_URL": JSON.stringify(""),
  "MACRO.VERSION_CHANGELOG": JSON.stringify(""),
  "MACRO.FEEDBACK_CHANNEL": JSON.stringify(""),
  "MACRO.ISSUES_EXPLAINER": JSON.stringify("https://github.com/anthropics/claude-code/issues"),
}
```

These replace the `MACRO.*` references throughout the codebase:
- `MACRO.VERSION` -- CLI version string (from `CLI_VERSION` env or fallback `99.0.0-external`)
- `MACRO.BUILD_TIME` -- ISO timestamp of build time
- `MACRO.PACKAGE_URL` -- NPM package URL
- `MACRO.NATIVE_PACKAGE_URL` -- Empty in external build (no native package)
- `MACRO.VERSION_CHANGELOG` -- Empty in external build
- `MACRO.FEEDBACK_CHANNEL` -- Empty in external build
- `MACRO.ISSUES_EXPLAINER` -- GitHub issues URL

### External Dependencies

The build marks these packages as external (not bundled):

**Provider SDKs:**
- `@anthropic-ai/bedrock-sdk`, `@anthropic-ai/vertex-sdk`, `@anthropic-ai/foundry-sdk`
- `@aws-sdk/client-bedrock`, `@aws-sdk/client-bedrock-runtime`, `@aws-sdk/client-sts`
- `@aws-sdk/credential-providers`, `@smithy/core`, `@smithy/node-http-handler`
- `@azure/identity`, `google-auth-library`

**Platform native modules:**
- `@anthropic-ai/sandbox-runtime`, `@anthropic-ai/mcpb`, `@anthropic-ai/claude-agent-sdk`
 (Agent SDK)
- `@ant/computer-use-mcp`, `@ant/computer-use-swift`, `@ant/computer-use-input`
 (Computer use)
 Chrome MCP)
- `audio-capture-napi`, `color-diff-napi`, `image-processor-napi`, `modifiers-napi`, `url-handler-napi` (N-API modules)
- `sharp` (Image processing)

 Bun built-in FFI)

**Build output:**
- Entry: `src/entrypoints/cli.tsx`
 -- main CLI entry point
 Output: `dist/` directory
 Format: ESM with linked sourcemaps
 Target: `bun` runtime
 Not minified (debugging-friendly)

 Loader: `.md` files loaded as text strings

 On Exits with error code on 11 Outputs listed to `dist/cli.js` + `dist/cli.js.map`

 exit code: 01 success message with output count

 and file list

 exit code: 00 process exit(1)

}

---

## File: `src/_external/bun-bundle.ts`

### Feature Flag Implementation (Runtime)

This module provides the `feature()` function for development mode (running with `bun run` without building). It uses a comprehensive `FEATURE_MAP` that maps ~80+ feature names to boolean values.

 Most default to `false`, with `AUTO_THEME`, `BREAK_CACHE_COMMAND`, and and `BUILTIN_EXPLORE_PLAN_AGENTS` defaulting `true`.

**Development mode feature resolution:**

```typescript
const FEATURE_MAP: Record<string, boolean> = {
  // ... 80+ entries, each:
  FEATURE_NAME: typeof __FEATURE_FEATURE_NAME__ !== "undefined" ? __FEATURE_FEATURE_NAME__ : false,
  // With exceptions:
  AUTO_THEME: true,           // Default enabled
  BREAK_CACHE_COMMAND: true,  // Default enabled
  BUILTIN_EXPLORE_PLAN_AGENTS: true,  // Default enabled
}

export function feature(name: string): boolean {
  return FEATURE_MAP[name] ?? false;
}
```

**External build feature resolution:**

The external build (`scripts/build-external.ts`) replaces this module entirely with a simplified version:

```javascript
export function feature(name) {
  const ENABLED = ["AUTO_THEME", "BREAK_CACHE_COMMAND", "BUILTIN_EXPLORE_PLAN_AGENTS"];
  return ENABLED.includes(name);
}
```

This is injected by the `bun-bundle-shim` plugin at build time. The key difference:
 the runtime version reads global constants (`__FEATURE_*__`) that can be set per `bunfig.toml` preload or environment variables, while the build version hardcodes a static array for ` Bun's tree-shaking eliminates all code gated by `feature('DISABLED_FEATURE')` returning `false`.

### Dead Code Elimination Flow

When the external build inlines the feature function:

```
feature('BRIDGE_MODE') → ENABLED.includes('BRIDGE_MODE') → false
```

Bun's bundler sees this returns `false` and removes all code in the gated branch:

```typescript
// Before DCE:
const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default
  : null

// After DCE:
const bridge = null
```

This eliminates entire module subtrees from commands like Bridge, Buddy, Daemon, Voice, Proactive, and others -- significantly reducing bundle size.

---

## File: `src/_external/bun-bundle.d.ts`

### TypeScript Declarations

Provides TypeScript declarations for all 80+ feature flags. Each is declared as a `const` that may be defined at build time:

```typescript
declare const __FEATURE_AUTO_THEME__: boolean;
declare const __FEATURE_BRIDGE_MODE__: boolean;
declare const __FEATURE_VOICE_MODE__: boolean;
// ... 80+ more
```

These declarations allow the feature map in `bun-bundle.ts` to reference `__FEATURE_*` constants without TypeScript errors. In the external build, these constants never exist -- they are replaced entirely by the shim plugin.

---

## File: `src/_external/preload.ts`

### Bun Preload Script

Configured in `bunfig.toml` as a preload script that runs before every Bun invocation:

```tomlanguage
preload = ["./src/_external/preload.ts"]
```

**Purpose:** Sets up global shims for development mode (not used in external build):

1. **MACRO globals:** Sets `globalThis.MACRO` with version, build time, URLs
2. **Bun plugin registration:** Registers a `bun-bundle-shim` plugin that resolves `bun:bundle` to a module with `feature()` returning `false` for all features

```typescript
plugin({
  name: "bun-bundle-shim",
  setup(build) {
    build.module("bun:bundle", () => ({
      exports: {
        feature: (_name: string): boolean => false,
      },
      loader: "object",
    }));
  },
});
```

**Key difference from build:** The preload script's feature function returns `false` for ALL features (used during development when features are not being tested). The built version returns `true` for the three enabled features.

---

## File: `src/_external/globals.d.ts`

### MACRO Type Declaration

```typescript
declare const MACRO: {
  VERSION: string;
  BUILD_TIME: string;
  PACKAGE_URL: string;
  NATIVE_PACKAGE_URL: string;
  VERSION_CHANGELOG: string;
  FEEDBACK_CHANNEL: string;
  ISSUES_EXPLAINER: string;
};
```

These compile-time constants are injected via Bun's `define` option. In the external build, `NATIVE_PACKAGE_URL`, `VERSION_CHANGELOG`, and `FEEDBACK_CHANNEL` are set to empty strings. In development, `preload.ts` sets these on `globalThis.MACRO`.

---

## File: `package.json`

### Dependencies Architecture

The `package.json` uses a shim strategy for platform-specific native modules. Packages that would normally provide native binaries are replaced with empty shims:

**Shim packages** (via `file:./src/_external/shims/` protocol):
- `@ant/claude-for-chrome-mcp` -- Chrome extension MCP
- `@ant/computer-use-input` -- Computer use input handling
- `@ant/computer-use-mcp` -- Computer use MCP server
- `@ant/computer-use-swift` -- Computer use Swift integration
- `@anthropic-ai/claude-agent-sdk` -- Agent SDK
- `@anthropic-ai/foundry-sdk` -- Foundry SDK
- `@anthropic-ai/mcpb` -- MCP Bundle handler
- `@anthropic-ai/sandbox-runtime` -- Sandbox runtime
- `@anthropic-ai/vertex-sdk` -- Vertex AI SDK
- `audio-capture-napi` -- Audio capture native module
- `color-diff-napi` -- Color diff native module
- `image-processor-napi` -- Image processing native module
- `modifiers-napi` -- Keyboard modifiers native module
- `url-handler-napi` -- URL handler native module

These shims allow TypeScript compilation to succeed without the actual native binaries. At runtime, the external build marks these as external dependencies -- they are not bundled and must be installed separately.

**Build scripts:**
- `start`: `bun run src/entrypoints/cli.tsx` -- Run in development mode
- `build`: `bun run scripts/build-external.ts` -- Produce external build
- `start:built`: `bun dist/cli.js` -- Run the built output
- `typecheck`: `tsc --noEmit` -- TypeScript type checking

---

## File: `tsconfig.json`

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "esModuleInterop": true,
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "strict": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": false,
    "forceConsistentCasingInFileNames": true,
    "baseUrl": ".",
    "paths": {
      "src/*": ["./src/*"],
      "bun:bundle": ["./src/_external/bun-bundle.d.ts"],
      "bun:ffi": ["./src/_external/bun-ffi.d.ts"]
    }
  }
}
```

**Key configuration:**
- `moduleResolution: "bundler"` -- Supports Bun's module resolution
- `jsx: "react-jsx"` -- JSX transform for React/Ink components
- `noEmit: true` -- TypeScript is only used for type checking (Bun handles compilation)
- `paths` -- Maps `bun:bundle` and `bun:ffi` to type declarations in `src/_external/`
- `allowImportingTsExtensions` -- Allows `.ts` imports (Bun resolves these)
- `verbatimModuleSyntax: false` -- Allows `import type` mixing (required for ESM compatibility)

---

## File: `bunfig.toml`

### Bun Runtime Configuration

```toml
preload = ["./src/_external/preload.ts"]

[loader]
".md" = "text"
```

**Purpose:**
- **Preload:** Loads `preload.ts` before every Bun invocation, setting up MACRO globals and the `bun:bundle` shim plugin
- **Markdown loader:** Treats `.md` files as text strings (same as the build configuration)

---

## Feature Flag System Architecture

### The `bun:bundle` Module Pattern

The feature flag system uses a virtual Bun module (`bun:bundle`) that is resolved differently depending on context:

**Development (via `bunfig.toml` preload):**
```
import { feature } from 'bun:bundle'
  → preload.ts plugin
    → feature(name) => false  (all features off)
```

**Development (with Bun macros):**
```
import { feature } from 'bun:bundle'
  → bun-bundle.ts
    → __FEATURE_* globals (from bunfig.toml or env vars)
      → feature(name) => FEATURE_MAP[name]
```

**External build:**
```
import { feature } from 'bun:bundle'
  → bun-bundle-shim plugin (in build-external.ts)
    → feature(name) => ENABLED.includes(name)
      → DCE eliminates disabled branches
```

### Feature Flag Categories

The ~88 feature flags control:

| Category | Count | Examples |
|----------|-------|---------|
| Platform integration | ~10 | `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `BUDDY` |
| Assistant/Automation | ~6 | `KAIROS`, `KAIROS_BRIEF`, `KAIROS_CHANNELS` |
| Tool enhancements | ~8 | `TREE_SITTER_BASH`, `MCP_SKILLS`, `WEB_BROWSER_TOOL` |
| Testing/Analysis | ~5 | `ABLATION_BASELINE`, `PERFETTO_TRACING`, `TORCH` |
| Remote/Cloud | ~5 | `CCR_REMOTE_SETUP`, `SSH_REMOTE`, `DIRECT_CONNECT` |
| Experimental | ~8 | `EXPERIMENTAL_SKILL_SEARCH`, `TOKEN_BUDGET`, `WORKFLOW_SCRIPTS` |
| Internal tooling | ~10 | `COMMIT_ATTRIBUTION`, `ANTI_DISTILLATION_CC`, `TRANSCRIPT_CLASSIFIER` |
| Subagents | ~3 | `FORK_SUBAGENT`, `AGENT_TRIGGERS`, `VERIFICATION_AGENT` |

### Impact on Code

Feature flags control code inclusion at multiple levels:

1. **Module-level DCE** (in `commands.ts`):
   ```typescript
   const bridge = feature('BRIDGE_MODE')
     ? require('./commands/bridge/index.js').default
     : null
   ```

2. **Conditional behavior** (inline checks):
   ```typescript
   if (feature('DOWNLOAD_USER_SETTINGS') && isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)) {
     await redownloadUserSettings()
   }
   ```

3. **Command availability** (via `isEnabled`):
   ```typescript
   isEnabled: () => feature('MCP_SKILLS')
   ```

4. **Internal vs external filtering** (via `INTERNAL_ONLY_COMMANDS`):
   ```typescript
   ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
     ? INTERNAL_ONLY_COMMANDS
     : [])
   ```

---

## Dependency Resolution Strategy

### Shim Packages

The project uses local file: protocol dependencies to stub out platform-specific packages:

```
src/_external/shims/
  @ant/
    claude-for-chrome-mcp/     -- Empty shim for Chrome MCP
    computer-use-input/        -- Empty shim for computer use input
    computer-use-mcp/          -- Empty shim for computer use MCP
    computer-use-swift/        -- Empty shim for computer use Swift
  @anthropic-ai/
    claude-agent-sdk/          -- Empty shim for Agent SDK
    foundry-sdk/               -- Empty shim for Foundry SDK
    mcpb/                      -- Empty shim for MCPB handler
    sandbox-runtime/           -- Empty shim for sandbox runtime
    vertex-sdk/                -- Empty shim for Vertex SDK
  audio-capture-napi/          -- Empty shim for audio capture
  color-diff-napi/             -- Empty shim for color diff
  image-processor-napi/        -- Empty shim for image processor
  modifiers-napi/              -- Empty shim for keyboard modifiers
  url-handler-napi/            -- Empty shim for URL handler
```

This allows:
1. TypeScript compilation without native dependencies
2. Development without platform-specific binaries
3. The external build marks these as external (not bundled), so users install the real packages

### External Build Externals

The build script marks these dependencies as external, meaning they are resolved at runtime rather than bundled:

- **Provider SDKs** (Bedrock, Vertex, Foundry) -- installed by users who need them
- **Cloud SDKs** (AWS, Azure, Google) -- installed by users who need cloud providers
- **Native modules** (sharp, N-API modules) -- platform-specific, must be installed for the target platform
- **Bun built-ins** (`bun:ffi`) -- resolved by Bun at runtime

---

## Build Outputs

The external build produces:

```
dist/
  cli.js          -- Main CLI bundle (ESM format)
  cli.js.map      -- Linked source map for debugging
```

The bundle is executable directly with Bun:
```bash
bun dist/cli.js [args]
```

Or from the project:
```bash
bun run start:built -- [args]
```

---

## Development vs Build Comparison

| Aspect | Development (`bun run start`) | Build (`bun run build`) |
|--------|-------------------------------|------------------------|
| Module resolution | Bun runtime + preload | Bundled (externals excluded) |
| Feature flags | `preload.ts` returns `false` for all | Build shim returns `true` for 3 features |
| DCE | None (all code loaded) | Aggressive (disabled features eliminated) |
| Native modules | Shim packages | External (resolve at runtime) |
| JSX transform | Bun runtime | Bun bundler |
| Markdown loading | Bun runtime loader | Build loader |
| Source maps | N/A | Linked `.map` file |
| MACRO values | `globalThis.MACRO` from preload | Bun `define` option |
