# Prerequisite Knowledge and Skills for Gemini-CLI
**Summary**: Prequisite knowledge and skills required in order to be able to understand and modify Gemini-CLI codebase.

**Tags**: #src

---

To understand and modify this codebase, you mainly need these skills, roughly in order of importance.

## Core Prerequisites

1. **TypeScript**

This is a TypeScript monorepo. You should be comfortable with interfaces, generics, async iterators, discriminated unions, module exports, and type narrowing.

2. **Node.js**

You need practical Node knowledge: fs, child_process, streams, signals, environment variables, process relaunching, stdin/ stdout/stderr, path handling, and ESM modules.

3. **Async Programming**

The agent loop is heavily async. You should understand Promise, async/await, AbortController, event emitters, async generators, and streaming APIs.

4. **React + Ink**

The interactive terminal UI is React rendered through Ink. To work on UI, learn React hooks, context providers, component state, rendering performance, and terminal layout constraints.

5. **CLI Architecture**

You should understand how command-line apps parse args, read stdin, write stdout/stderr, handle TTY vs non-TTY modes, manage exit codes, and support both interactive and non-interactive flows.

## Agent-Specific Knowledge
6. **LLM Tool Calling**

This is central. The model emits function/tool calls, the app validates and executes them, then sends tool results back to the model. Key files:

- packages/core/src/core/client.ts
- packages/core/src/core/geminiChat.ts
- packages/core/src/core/turn.ts
- packages/core/src/scheduler/scheduler.ts

7. **Streaming Model APIs**

The Gemini response is streamed. You need to understand partial chunks, finish reasons, retry handling, malformed streams, and how streamed function calls become executable tool requests.

8. **Agent Loops**

The app repeatedly does: user prompt -> model response -> maybe tools -> tool results -> model response -> finish. You should understand loop limits, cancellation, retries, loop detection, context compression, and history management.

9. **Tool Design**

Tools are declarative objects with schemas, validation, confirmation behavior, and execution logic. Learn:
- packages/core/src/tools/tools.ts
- packages/core/src/tools/tool-registry.ts
- packages/core/src/tools/edit.ts
- packages/core/src/tools/write-file.ts
- packages/core/src/tools/shell.ts

10. **Security and Policy**
- This codebase cares about file access, shell execution, approvals, trusted folders, sandboxing, and policy decisions. You need to understand why tools do not simply execute directly.

## Project/Repo Skills
11. **npm Workspaces**

The repo is split into packages. You should understand workspace dependencies, package scripts, local package linking, and build outputs.

12. **Testing With Vitest**

Tests are everywhere. You should know Vitest, mocks, snapshots, integration tests, and how to run targeted tests.

13. **Build Tooling**

The project uses TypeScript, esbuild, npm scripts, linting, formatting, generated schemas/docs, and bundle scripts.

14. **Filesystem Safety**

Many core features read, write, diff, patch, and validate paths. You need good instincts around path resolution, workspace boundaries, symlinks, line endings, and avoiding destructive edits.

## Useful Advanced Knowledge
15. **MCP**

Model Context Protocol support lets external servers expose tools/resources. Useful if working under packages/core/src/mcp or extension/tool discovery.

16. **OAuth/Auth**

Needed for auth work: Google login, API keys, Vertex/Code Assist flows, token storage, and validation errors.

17. **Telemetry/OpenTelemetry**

Needed for metrics, traces, billing/session/tool events.

18. **Sandboxing**

Important for shell/file execution safety. Learn this if touching execution policy or filesystem isolation.

19. **IDE Integration**

Useful for VS Code companion or editor context/diff features:

- packages/vscode-ide-companion
- packages/core/src/ide

## Recommended Learning Path
Start with TypeScript + Node async fundamentals, then read the runtime in this order:

1. packages/cli/index.ts
2. packages/cli/src/gemini.tsx
3. packages/cli/src/interactiveCli.tsx
4. packages/cli/src/ui/hooks/useGeminiStream.ts
5. packages/core/src/core/client.ts
6. packages/core/src/core/geminiChat.ts
7. packages/core/src/core/turn.ts
8. packages/core/src/scheduler/scheduler.ts
9. packages/core/src/scheduler/tool-executor.ts
10. packages/core/src/tools/edit.ts and write-file.ts

After that, pick one vertical slice, like “read a file” or “edit a file,” and trace it end to end. That is the fastest way to
become productive in this repo.