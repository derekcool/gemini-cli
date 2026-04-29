# Gemini-CLI Code Structure
**Summary**: Explain the structure of Gemini-CLI source code
**Tags**: #src

---

## High-Level Shape

- package.json defines an npm workspace monorepo under packages/*.
- Main published CLI package: packages/cli
- Core agent/runtime package: packages/core
- Programmatic SDK: packages/sdk
- Extra packages: a2a-server, devtools, test-utils, and vscode-ide-companion.
- Tests live both beside source files and in top-level suites: integration-tests, evals, memory-tests, and perf-tests.

## Runtime Entry
The executable starts at packages/cli/index.ts. It is intentionally lightweight: it may relaunch itself with larger Node memory settings, then imports the heavier CLI implementation.

The main application startup is packages/cli/src/gemini.tsx. It handles:

- loading settings and trusted folder config
- parsing CLI args
- session creation/resume
- auth validation and refresh
- sandbox relaunch
- policy/hook setup
- telemetry setup
- terminal/theme setup
- dispatch to interactive UI, headless mode, or ACP mode

## Interactive Vs Headless
Interactive mode is started by packages/cli/src/interactiveCli.tsx. It renders the terminal UI with React + Ink, wiring providers for settings, keybindings, mouse, scroll, terminal state, session stats, and vim mode.

Headless mode is handled by packages/cli/src/nonInteractiveCli.ts. It processes a prompt from --prompt or stdin, runs the agent loop, executes tool calls through the scheduler, and emits text, JSON, or stream-JSON output.

## Core Agent Loop
The central runtime is in packages/core/src/core/client.ts. GeminiClient owns the active chat, model routing, context management, loop detection, chat compression, hooks, IDE context injection, retries, and fallback behavior.

A single model interaction is represented by packages/core/src/core/turn.ts. Turn converts Gemini stream chunks into internal events such as:

- text content
- thoughts
- tool call requests
- citations
- finish events
- errors
- retry/invalid stream events

The chat wrapper is packages/core/src/core/geminiChat.ts. It wraps @google/genai, maintains curated/comprehensive history, records sessions, validates streams, retries mid-stream failures, handles before/after model hooks, and consolidates function calls.

## Tool System
Tools are registered in a ToolRegistry in packages/core/src/tools/tool-registry.ts. It tracks built-in tools, MCP tools, and discovered project tools, then exposes Gemini function declarations based on model, policy, plan mode, and tool exclusions.

Built-in tool declaration sets are selected in packages/core/src/tools/definitions/coreTools.ts. The code supports model-family-specific tool descriptions, notably a Gemini 3 set and a legacy/default set.

Tool execution is coordinated by packages/core/src/scheduler/scheduler.ts. The scheduler:

- validates model-requested tool calls
- applies hooks and policy
- requests confirmation when needed
- runs tools in batches, with parallel execution where allowed
- handles cancellation and sandbox expansion requests
- records status through the message bus

Built-in tools include file reads/writes, grep/ripgrep, glob/list directory, shell, web fetch/search, memory, todos, MCP resource access, skill activation, and ask-user.

## Config, Policy, Hooks, Context
packages/core/src/config contains the core config object and model/settings behavior. packages/cli/src/config adapts CLI settings, arguments, auth, sandbox config, trusted folders, and policy updates.

### Important systems:

- policy: allow/deny/ask behavior for tools and execution.
- hooks: lifecycle hooks around session/model/tool execution.
- context: context management, compression, memory, history shaping, and output masking.
- mcp: MCP client, OAuth, and token storage support.
- ide: IDE companion connection and editor context.
- telemetry: metrics, traces, billing/session/tool events.
- sandbox: execution isolation support.

## SDK
packages/sdk/src/index.ts exports a smaller programmatic API around agents, sessions, tools, and skills. packages/sdk/src/agent.ts creates/resumes Gemini CLI sessions using the same core storage/session machinery.

## Supporting Packages

- packages/vscode-ide-companion: VS Code extension that exposes IDE workspace state and diff workflows to the CLI.
- packages/devtools: small web/devtools client built with React and WebSockets.
- packages/a2a-server: Express-based Agent-to-Agent server package wrapping core behavior.
- packages/test-utils: shared fixtures and harnesses for package/integration tests.

## Request Flow
A normal interactive prompt roughly flows like this:

1. packages/cli/index.ts starts the process.
2. packages/cli/src/gemini.tsx loads settings/auth/config and initializes the app.
3. interactiveCli.tsx renders AppContainer.
4. UI submits a user message to GeminiClient.
5. GeminiClient chooses model/context/tools and starts a Turn.
6. GeminiChat streams Gemini responses.
7. Turn emits content or tool-call events.
8. Tool calls go to Scheduler.
9. Scheduler applies policy/confirmation/hooks, executes tools, returns function responses.
10. Function responses are sent back into the model loop until no more tools are requested.

In short: cli owns process/UI/UX, core owns the agent runtime, tools, model calls, policy, hooks, and persistence, while sdk and companion packages reuse that core in other integration surfaces.

## File Mutation Flow
The actual file mutation happens in two tool implementations:

- packages/core/src/tools/edit.ts:903
EditToolInvocation.execute() calculates a replacement, creates parent dirs, normalizes line endings, then calls:

await this.config.getFileSystemService().writeTextFile(this.resolvedPath, finalContent);
- packages/core/src/tools/write-file.ts:345
WriteFileToolInvocation.execute() overwrites or creates the whole file, then calls:

await this.config.getFileSystemService().writeTextFile(this.resolvedPath, finalContent);

That delegates to the filesystem service:

- Normal mode: packages/core/src/services/fileSystemService.ts:38

async writeTextFile(filePath: string, content: string): Promise<void> {
    await fs.writeFile(filePath, content, 'utf-8');
}
- Sandboxed mode: packages/core/src/services/sandboxedFileSystemService.ts:114
It validates the path, prepares a sandbox __write command, then writes the new content to the child process stdin.

The call path is roughly:

model tool call -> Turn emits ToolCallRequest -> Scheduler.schedule() -> ToolExecutor.execute() -> executeToolWithHooks(...) -> specific tool invocation execute() -> writeTextFile(...).

So if you’re looking for “where source files get changed,” start with edit.ts:903 and write-file.ts:345; if you’re looking for the final disk primitive, it’s fileSystemService.ts:39.

## References: Flow of "Fix a Bug"
High-level flow for a prompt like “Fix a bug”:

1. Process starts
    packages/cli/index.ts is the executable entry. It may relaunch Node with better memory settings, then imports the real CLI.
2. CLI initializes
    packages/cli/src/gemini.tsx loads settings, parses args, initializes auth, sandboxing, policy, session storage, telemetry, and
    decides whether to run interactive or headless mode.
3. Interactive UI captures your prompt
    packages/cli/src/interactiveCli.tsx renders the Ink terminal app. The main UI lives under:
    packages/cli/src/ui
4. Prompt is sent into the agent loop
    The UI stream hook, especially packages/cli/src/ui/hooks/useGeminiStream.ts, sends your message to the core client.
5. Core client prepares model request
    packages/core/src/core/client.ts owns the main agent loop. It adds context, memory, tools, IDE context, history, model routing,
    hooks, retries, and loop detection.
6. Gemini API stream starts
    packages/core/src/core/geminiChat.ts calls @google/genai, streams model chunks, records history, and handles retry/fallback
    behavior.
7. One model turn is interpreted
    packages/core/src/core/turn.ts converts stream chunks into events: text, thoughts, finish events, errors, or tool call requests.
8. Model asks to inspect code
    If the model needs context, it emits tool calls like read file, grep, glob, list directory, or shell. Tool definitions live in:
    packages/core/src/tools
9. Tool calls are scheduled
    packages/core/src/scheduler/scheduler.ts validates tool calls, checks policy, asks for confirmation if needed, batches parallel-
    safe tools, and coordinates execution.
10. Tool is executed
    packages/core/src/scheduler/tool-executor.ts invokes the selected tool and converts its result back into a Gemini function
    response.
11. Model receives tool results
    The tool result goes back through packages/core/src/core/client.ts, continuing the loop. For “Fix a bug,” this usually repeats:
    inspect files, reason, maybe run tests, inspect errors, edit files.
12. File changes happen
    Actual edits happen in:
    packages/core/src/tools/edit.ts for targeted replacements
    packages/core/src/tools/write-file.ts for full-file writes

Both delegate to:
packages/core/src/services/fileSystemService.ts
or sandboxed writes via:
packages/core/src/services/sandboxedFileSystemService.ts

13. Validation: tools may run
- The model may call shell via: packages/core/src/tools/shell.ts

That is how commands like tests, typecheck, or lint are run.

14. Loop finishes:
    When the model stops requesting tools and returns final text, Turn emits a finished/content event, the UI hook updates the
    terminal, and the assistant’s final answer is shown.

In short: cli gathers your prompt and renders the session; core/client.ts runs the agent loop; geminiChat.ts talks to Gemini;
turn.ts detects text/tool calls; scheduler.ts and tool-executor.ts run tools; edit.ts and write-file.ts actually mutate files.

