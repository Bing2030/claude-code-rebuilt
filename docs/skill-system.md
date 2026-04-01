# Skill System: Discovery, Selection & Execution

## Overview

Skills are reusable prompt templates that the model can invoke by name. The system discovers skills from multiple sources at startup and at runtime, presents them to the model through the system prompt and a `Skill` tool, and the model decides which skill to use based on the user's request. There is no keyword-based matching or relevance scoring — the model is the sole decision-maker.

```
Skill Lifecycle:

  Definition          Discovery              Presentation          Execution
  ──────────          ──────────             ───────────           ─────────
  SKILL.md file  →   loadSkillsDir()  →    System prompt     →   SkillTool.call()
  Bundled code   →   registerBundled() →   SkillTool desc    →   command.prompt()
  Plugin manifest →  loadPlugins()    →                        →   inject into conversation
  MCP server     →   connectToMcp()   →                        →   return result
  Dynamic paths  →  discoverDirs()   →
```

---

## File: `src/types/command.ts`

### Skill-Related Types

Skills are a subset of the `Command` type:

```typescript
type PromptCommand = {
  type: 'prompt'

  // Identity
  name: string                       // Slash command name (e.g., "commit")
  aliases?: string[]                 // Alternative names
  description: string                // What the skill does
  hasUserSpecifiedDescription: boolean

  // Model guidance
  whenToUse?: string                 // Hint for when the model should use this skill
  argumentHint?: string              // Example arguments
  argumentNames?: string[]           // Named arguments ($ARG1, $ARG2)
  allowedTools: string[]             // Tools the skill can use

  // Behavior
  userInvocable: boolean             // Can user type /name?
  disableModelInvocation: boolean    // Can the model invoke it?
  model?: string                     // Specific model to use
  context?: 'inline' | 'fork'       // Execution context
  agent?: string                     // Agent to use

  // Content
  contentLength: number              // Markdown content length
  source: 'bundled' | 'userSettings' | 'projectSettings' | 'mcp'
  loadedFrom: 'bundled' | 'skills' | 'commands_DEPRECATED' | 'plugin' | 'mcp'

  // Advanced
  paths?: string[]                   // Conditional path patterns (gitignore-style)
  hooks?: HooksSettings              // Pre/post hooks
  isEnabled?: () => boolean          // Runtime enable check
  isHidden: boolean                  // Hidden from SkillTool listing?
  progressMessage?: string           // Spinner text while running

  // The actual prompt factory
  getPromptForCommand: (
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}
```

---

## File: `src/skills/bundledSkills.ts`

### Purpose

Registers skills that are compiled into the CLI binary and available to all users.

### BundledSkillDefinition

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

  // Reference files extracted to disk on first invocation
  files?: Record<string, string>

  // The prompt factory function
  getPromptForCommand: (
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}
```

### Registration Flow

```typescript
registerBundledSkill(definition) {
  // 1. If skill has 'files', set up lazy extraction to disk
  //    - First invocation extracts reference files
  //    - Prepends "Base directory for this skill: <dir>" to prompt
  //
  // 2. Create Command object
  const command = {
    type: 'prompt',
    name: definition.name,
    source: 'bundled',
    loadedFrom: 'bundled',
    isHidden: !(definition.userInvocable ?? true),
    // ... all other fields
  }
  //
  // 3. Push to internal registry
  bundledSkills.push(command)
}
```

### File Extraction

Bundled skills can ship with reference files that are extracted to disk on first use:

```
~/.claude/bundled-skills/<skill-name>/
  ├── reference1.md
  ├── reference2.md
  └── ...
```

This allows the model to `Read`/`Grep` the reference files during skill execution, providing richer context than a static prompt.

---

## File: `src/skills/loadSkillsDir.ts`

### Purpose

Discovers, loads, deduplicates, and manages all skill sources. This is the largest and most complex file in the skill system (~1087 lines).

### Skill Sources

| Source | Directory | `loadedFrom` | When Loaded |
|--------|-----------|-------------|-------------|
| **Managed** | `<managed>/.claude/skills/` | `'skills'` | Startup |
| **User** | `~/.claude/skills/` | `'skills'` | Startup |
| **Project** | `.claude/skills/` (in project tree) | `'skills'` | Startup |
| **Additional** | `--add-dir` paths | `'skills'` | Startup |
| **Legacy** | `.claude/commands/` | `'commands_DEPRECATED'` | Startup |
| **Bundled** | (compiled in) | `'bundled'` | Startup |
| **Plugin** | Plugin manifests | `'plugin'` | Startup |
| **MCP** | MCP server definitions | `'mcp'` | Startup + runtime |
| **Dynamic** | Discovered from file paths | `'skills'` | Runtime |
| **Conditional** | Path-pattern-filtered | `'skills'` | Runtime |

### File-Based Skill Format

Skills in `.claude/skills/` must follow the directory format:

```
.claude/skills/
  └── commit/
      └── SKILL.md
  └── review-pr/
      └── SKILL.md
```

Each `SKILL.md` has YAML frontmatter followed by the prompt content:

```markdown
---
description: "Create a git commit with staged changes"
when_to_use: "Use this when the user asks to commit changes"
arguments: "message"
allowed-tools: ["Bash", "Read"]
model: "sonnet"
effort: "high"
user-invocable: true
paths:
  - "src/**/*.ts"
  - "test/**/*.ts"
---

You are creating a git commit. Analyze the staged changes and create
a meaningful commit message following the project's conventions.

$ARGUMENT
```

### Frontmatter Fields

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `description` | string | (from markdown) | Skill description shown to model |
| `when_to_use` | string | undefined | Hint for when model should use this |
| `arguments` | string | undefined | Named arguments ($ARG1, $ARG2) |
| `allowed-tools` | string[] | [] | Tools the skill can access |
| `model` | string | undefined | Specific model override |
| `effort` | string | undefined | Effort level |
| `user-invocable` | boolean | `true` | Can user type `/name`? |
| `disable-model-invocation` | boolean | `false` | Can model auto-invoke? |
| `context` | `'inline' \| 'fork'` | undefined | Execution context |
| `agent` | string | undefined | Agent definition |
| `hooks` | object | undefined | Pre/post execution hooks |
| `paths` | string[] | undefined | Conditional activation patterns |
| `shell` | object | undefined | Shell commands to execute in prompt |
| `version` | string | undefined | Skill version |

### Startup Discovery Pipeline

```
getSkillDirCommands(cwd)
  │
  ├── Load from 5 directories in parallel:
  │   ├─ managedSkillsDir  (<managed>/.claude/skills/)
  │   ├─ userSkillsDir     (~/.claude/skills/)
  │   ├─ projectSkillsDirs (.claude/skills/ in project tree)
  │   ├─ additionalDirs    (--add-dir paths)
  │   └─ legacyCommands    (.claude/commands/)
  │
  ├── Flatten + combine all results
  │
  ├── Deduplicate by file identity (realpath)
  │   └─ First-seen wins, logged as "Skipping duplicate skill"
  │
  ├── Separate conditional vs unconditional
  │   ├─ Skills with 'paths' frontmatter → conditionalSkills map
  │   └─ All others → returned as unconditional
  │
  └─ Return unconditional skills
```

### Deduplication

Skills are deduplicated by file identity (resolved via `realpath`):

```typescript
const fileIds = await Promise.all(
  allSkillsWithPaths.map(({ skill, filePath }) =>
    skill.type === 'prompt'
      ? getFileIdentity(filePath)   // realpath for symlink detection
      : Promise.resolve(null),
  ),
)

// First-seen wins by file identity
for (let i = 0; i < allSkillsWithPaths.length; i++) {
  const fileId = fileIds[i]
  if (existingSource = seenFileIds.get(fileId)) {
    logForDebugging(`Skipping duplicate skill '${skill.name}'`)
    continue
  }
  seenFileIds.set(fileId, skill.source)
  deduplicatedSkills.push(skill)
}
```

### Dynamic Skill Discovery

When files are edited during a session, new skill directories are discovered:

```
File edited: /project/src/components/Button.tsx
    │
    ▼ discoverSkillDirsForPaths([filePath], cwd)
    │
    ├─ Walk up from dirname(filePath) to cwd
    │   └─ At each level, check for .claude/skills/ directory
    │       ├─ Skip already-checked dirs (dynamicSkillDirs set)
    │       ├─ Skip gitignored dirs (security boundary)
    │       └─ Collect new directories
    │
    ├─ Sort by path depth (deepest first)
    │   └─ Skills closer to the file take precedence
    │
    └─ addSkillDirectories(newDirs)
        ├─ loadSkillsFromSkillsDir for each new dir
        ├─ Deeper paths override shallower ones
        └─ Emit skillsLoaded signal
```

### Conditional Skill Activation

Skills with `paths` frontmatter are activated when matching files are touched:

```typescript
// Skill with frontmatter:
// paths: ["src/**/*.ts", "test/**/*.ts"]

activateConditionalSkillsForPaths(filePaths, cwd)
  │
  ├─ For each pending conditional skill:
  │   ├─ Build ignore() matcher from skill.paths
  │   └─ For each filePath:
  │       ├─ Compute relative path from cwd
  │       ├─ gitignore-style pattern matching via ignore library
  │       └─ If matched → activate skill
  │           ├─ Move from conditionalSkills → dynamicSkills
  │           ├─ Add to activatedConditionalSkillNames
  │           └─ Emit skillsLoaded signal
  │
  └─ Return list of newly activated skill names
```

### Bare Mode Behavior

In bare mode (`--bare`), skill discovery is minimal:

```typescript
if (isBareMode()) {
  // Only load from explicit --add-dir paths
  // Skip managed, user, project, and legacy directories
  // Bundled skills still load (registered separately)
}
```

---

## File: `src/tools/SkillTool/ToolSkill.tsx`

### Purpose

The `SkillTool` is the runtime bridge — it's how the model invokes skills during a conversation.

### Tool Definition

```typescript
const SkillTool = buildTool({
  name: 'Skill',
  maxResultSizeChars: 50_000,
  inputSchema: z.object({
    command: z.string().describe(
      "The skill name and slash command arguments..."
    ),
    args: z.string().optional().describe(
      "Arguments to pass to the skill..."
    ),
  }),
})
```

### Skill Name Matching

```typescript
function parseSkillAndArgs(input: string): [Command | null, string] {
  const [command, ...args] = input.split(' ')

  // Try exact match first
  const exact = commands.find(c => c.name === command)
  if (exact) return [exact, args.join(' ')]

  // Try prefix match: "/skillname" or "/skill-name"
  const prefix = command.replace(/^\/, '').replace(/-/g, ' ')
  const match = commands.find(c => {
    if (c.name === prefix) return true
    // Also try aliases
    return c.aliases?.some(alias => {
      const normalized = alias.replace(/^\/, '').replace(/-/g, ' ')
      return normalized === prefix
    })
  })
  if (match) return [match, args.join(' ')]

  // Partial match: if input starts with skill name
  // (handles "/skill-name arg1 arg2" form)
  // ...
  return [null, '']
}
```

### Execution Flow

```typescript
async call(input) {
  // 1. Parse skill name and arguments from input
  const parsed = skillInputSchema.parse(input)
  const [command, args] = parseSkillAndArgs(parsed.command)

  // 2. Find the skill command
  const command = getSkillCommand(input)
  if (!command) return { error: `Unknown skill: ${skillName}` }

  // 3. Get conversation context for the skill
  const context = getContextForSkill(command, context)
  //    ↑ Extracts user + tool_result messages from conversation

  // 4. Execute the skill's prompt
  const prompt = await command.prompt(args, context)
  //    ↑ For file-based: returns SKILL.md content with $ARG substituted
  //    ↑ For bundled: calls getPromptForCommand(args, ctx)

  // 5. Return the prompt result
  return {
    output: promptResult,
    result: { type: 'tool_result', content: promptResult, tool_use_id }
  }
}
```

### Context Extraction

```typescript
function getContextForSkill(command, context) {
  return getSystemPrompt({
    getMessages: getMessages(context),
    // Filter to only user messages and tool results
    filterByPromptMessageType: (m) =>
      m.type === 'tool_result' || m.type === 'user',
  })
}
```

### Prompt Content Generation

For file-based skills (`createSkillCommand`):

```typescript
async getPromptForCommand(args, toolUseContext) {
  let finalContent = baseDir
    ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
    : markdownContent

  // Substitute $ARGUMENT placeholders
  finalContent = substituteArguments(finalContent, args, true, argumentNames)

  // Replace ${CLAUDE_SKILL_DIR} with skill's directory
  if (baseDir) {
    finalContent = finalContent.replace(
      /\$\{CLAUDE_SKILL_DIR\}/g,
      baseDir.replace(/\\/g, '/')
    )
  }

  // Replace ${CLAUDE_SESSION_ID} with current session
  finalContent = finalContent.replace(
    /\$\{CLAUDE_SESSION_ID\}/g,
    getSessionId()
  )

  // Execute shell commands in prompt (!`...` / ```! ... ```)
  // Only for non-MCP skills (security)
  if (loadedFrom !== 'mcp') {
    finalContent = await executeShellCommandsInPrompt(
      finalContent,
      toolUseContext,
      `/${skillName}`,
      shell,
    )
  }

  return [{ type: 'text', text: finalContent }]
}
```

---

## File: `src/tools/SkillTool/skillFinder.ts`

### Purpose

Helper used by the system prompt generation to find skills and format their descriptions.

### SkillFinderResult

```typescript
type SkillFinderResult = {
  name: string
  description: string
  command: Command
  subArguments: string | undefined
  arguments: string | undefined
  matchingHint: string | undefined
}
```

### findSkill()

```typescript
function findSkill(
  input: string,
  commands: Command[],
): SkillFinderResult | null {
  // 1. Extract skill name and arguments from input
  const [command, args] = parseSkillAndArgs(input)
  const skill = getSkillCommand(input)
  if (!skill) return null

  // 2. Return skill info with matching hint
  return {
    name: skill.name,
    description: skill.description,
    command: skill,
    subArguments: args,
    arguments: input.slice(skill.name.length).trim(),
    matchingHint: skill.description,
  }
}
```

---

## File: `src/commands.ts` (lines 563-581)

### Skill Command Filtering

```typescript
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const allCommands = await getCommands(cwd)
    return allCommands.filter(cmd =>
      cmd.type === 'prompt' &&
      !cmd.disableModelInvocation &&         // Model can invoke it
      cmd.source !== 'builtin' &&            // Not a built-in command
      (cmd.loadedFrom === 'bundled' ||        // Is a skill source
       cmd.loadedFrom === 'skills' ||
       cmd.loadedFrom === 'commands_DEPRECATED' ||
       cmd.hasUserSpecifiedDescription ||
       cmd.whenToUse)                         // Or has usage hints
    )
  },
)
```

### System Prompt Skill Formatting

Skills are presented to the model in two ways:

#### 1. SkillTool Description (Dynamic)

The SkillTool's description is dynamically generated to list available skills:

```
- commit: Create a git commit with staged changes
- review-pr: Review a pull request by number
- init: Initialize a new project configuration
```

Budget-aware formatting:
- Default budget: ~8,000 characters (1% of context window)
- Hard cap per entry: 250 characters
- Bundled skills get full descriptions
- Non-bundled skills may be truncated

#### 2. System Prompt Sections

Skills with `necessary: true` are always included as full system prompt sections with their complete description and `whenToUse` hints.

---

## How Relevance Is Determined

The system does NOT do keyword matching, relevance scoring, or semantic search. The model is the sole decision-maker. Here's how it works:

### Tier 1: Always Visible

```
SkillTool description lists ALL available skills:
  ┌─────────────────────────────────────────────────┐
  │ Available skills:                                │
  │ - commit: Create a git commit...                 │
  │ - review-pr: Review a pull request...            │
  │ - init: Initialize a project...                  │
  │ - ...                                            │
  └─────────────────────────────────────────────────┘

System prompt includes necessary skills:
  ┌─────────────────────────────────────────────────┐
  │ The /commit skill creates a git commit. Use when │
  │ the user asks to commit, save changes, etc.      │
  └─────────────────────────────────────────────────┘
```

### Tier 2: Model Decision

```
User: "commit my changes"
    │
    ▼ Model reads SkillTool description
    │ Matches "commit" → SkillTool(command: "commit")
    │
    ▼ SkillTool.call()
    ├─ Finds "commit" skill
    ├─ Gets conversation context
    ├─ Executes commit skill's prompt
    └─ Returns result to model
```

### Tier 3: Conditional Activation (Path-Based)

```
File edited: src/utils/auth.ts
    │
    ▼ activateConditionalSkillsForPaths(["src/utils/auth.ts"], cwd)
    │
    ├─ Check conditional skill "typescript-expert"
    │   paths: ["src/**/*.ts"]
    │   → "src/utils/auth.ts" matches "src/**/*.ts" → ACTIVATE
    │
    └─ Check conditional skill "python-expert"
        paths: ["**/*.py"]
        → "src/utils/auth.ts" does NOT match → skip
```

### Complete Flow Diagram

```
┌─────────────────── STARTUP ───────────────────┐
│                                                 │
│  loadSkillsFromSkillsDir(managed)               │
│  loadSkillsFromSkillsDir(user)                  │
│  loadSkillsFromSkillsDir(project)   ── all in ─┐│
│  loadSkillsFromSkillsDir(additional)  parallel  ││
│  loadSkillsFromCommandsDir(legacy)              ││
│  registerBundledSkill(...)                      ││
│  loadPlugins()                                  ││
│  connectToMcpServer()                           ││
│                                                 ││
│  ──→ Deduplicate by realpath                    ││
│  ──→ Separate conditional vs unconditional      ││
│  ──→ Store conditional skills in map            ││
│  ──→ Return unconditional skills                │┘│
│                                                 │
│  getSkillToolCommands(cwd)                      │
│  ──→ Filter: prompt type, not disabled, etc.    │
│  ──→ Memoized result                            │
│                                                 │
│  System prompt generation:                      │
│  ──→ Format skill list within budget            │
│  ──→ Include necessary skills as full sections  │
│  ──→ Register SkillTool with dynamic desc       │
└─────────────────────────────────────────────────┘

┌─────────────── RUNTIME ───────────────────────┐
│                                                 │
│  User edits file: src/foo.ts                    │
│    │                                            │
│    ├─ discoverSkillDirsForPaths([path], cwd)   │
│    │   └─ Walk up looking for .claude/skills/  │
│    │       └─ addSkillDirectories(newDirs)     │
│    │                                            │
│    └─ activateConditionalSkillsForPaths(...)   │
│        └─ Match path patterns via ignore()     │
│            └─ Activate matching skills         │
│                └─ Emit skillsLoaded signal     │
│                    └─ Clear memoization caches  │
│                                                 │
│  Model invokes skill:                           │
│    │                                            │
│    ├─ SkillTool.call(input)                     │
│    │   ├─ parseSkillAndArgs(input)              │
│    │   ├─ getSkillCommand(input)                │
│    │   ├─ getContextForSkill(command, ctx)      │
│    │   ├─ command.getPromptForCommand(args, ctx)│
│    │   │   ├─ Read SKILL.md content             │
│    │   │   ├─ Substitute $ARGUMENT placeholders │
│    │   │   ├─ Replace ${CLAUDE_SKILL_DIR}       │
│    │   │   ├─ Replace ${CLAUDE_SESSION_ID}      │
│    │   │   └─ Execute shell commands in prompt  │
│    │   └─ Return prompt result to model         │
│    │                                            │
│    └─ Model continues with skill result        │
└─────────────────────────────────────────────────┘
```

---

## Key Questions Answered

**How does the system identify skills?**

Skills are identified by their `name` field, which becomes a slash command (e.g., `commit` → `/commit`). Names can have aliases. The matching normalizes `/` prefix and `-` to spaces.

**How does the system determine which skills are necessary for a task?**

It doesn't — the model does. The system's job is to make the model aware of available skills through the SkillTool description and system prompt sections. The model reads the skill names and descriptions, then decides which skill (if any) to invoke based on the user's request.

**What are conditional skills?**

Skills with a `paths` frontmatter field. These are dormant until the user edits files matching the path patterns (gitignore-style matching via the `ignore` library). Once activated, they appear in the SkillTool description like any other skill.

**What are dynamic skills?**

Skills discovered at runtime when files are edited. The system walks up the directory tree from the edited file looking for `.claude/skills/` directories that weren't loaded at startup. Deeper directories (closer to the file) take precedence.

**How does the SkillTool work?**

The model calls `Skill(command: "/commit -m 'message'")`. The tool parses the name and arguments, finds the matching skill command, injects the conversation context, executes the skill's prompt (which returns the SKILL.md content with arguments substituted), and returns the result as a tool result for the model to continue with.
