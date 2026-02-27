# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, and Claude generates React code displayed in a Monaco editor with a live iframe preview. The virtual file system keeps all files in-memory — nothing is written to disk during generation.

## Commands

```bash
npm run setup          # One-time setup: install deps + prisma generate + migrate
npm run dev            # Dev server with Turbopack at http://localhost:3000
npm run build          # Production build
npm run lint           # ESLint (next lint)
npm run test           # Run all tests with Vitest
npx vitest run <file>  # Run a single test file
npm run db:reset       # Reset SQLite database (destructive)
```

All commands that run Next.js require `NODE_OPTIONS='--require ./node-compat.cjs'` (already baked into the npm scripts).

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude; without it, the app falls back to a `MockLanguageModel` that returns static example components.

## Architecture

### Request Flow

1. User submits a prompt in `ChatInterface`
2. The `ChatContext` (wrapping Vercel AI SDK's `useChat`) sends it to `POST /api/chat` with the current virtual file system state
3. The API route calls Claude with two tools: `str_replace_editor` and `file_manager`
4. Claude invokes tools to create/modify files; tool results update `FileSystemContext`
5. `PreviewFrame` (an iframe) picks up the new virtual FS, transpiles JSX via Babel Standalone, and re-renders

### Key Abstractions

**`VirtualFileSystem`** (`src/lib/file-system.ts`) — In-memory Map-based FS. All file operations go through this class. No real disk I/O. Serializes to/from JSON for project persistence.

**`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — React context holding the VirtualFileSystem instance, selected file, and CRUD methods. Also processes tool calls from Claude (via `onToolCall` in the AI SDK).

**`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — Wraps `useChat`, integrates with FileSystemContext, handles project save/load, and tracks anonymous sessions via localStorage.

**LLM Provider** (`src/lib/provider.ts`) — Returns either the real Anthropic Claude model (`claude-haiku-4-5`) or `MockLanguageModel`. maxTokens: 10,000; maxSteps: 40 (real) / 4 (mock).

**Tools** (`src/lib/tools/`):
- `str_replace_editor` — view, create, str_replace, insert, undo_edit on virtual files
- `file_manager` — rename (move) and delete virtual files

**System Prompt** (`src/lib/prompts/generation.tsx`) — Instructs Claude to create React components with `App.jsx` as the entry point, use Tailwind CSS, and use `@/` import aliases.

**`PreviewFrame`** (`src/components/preview/`) — Sandboxed iframe with `allow-scripts allow-same-origin`. Creates an import map from virtual FS files and uses Babel to transpile JSX at runtime. Looks for `App.jsx`, `index.jsx`, or the first `.jsx` file as the entry point.

### Data Persistence

- SQLite via Prisma (`prisma/dev.db`) — always refer to `prisma/schema.prisma` to understand the database structure
- `User` and `Project` models; `Project.data` (JSON) stores the serialized virtual FS and `Project.messages` stores chat history
- Authentication: JWT sessions in httpOnly cookies (7-day expiry), managed via `src/lib/auth.ts` and `src/actions/`
- Projects are optional for anonymous users (no `userId` required)

### Path Aliases

`@/*` maps to `src/*` (configured in `tsconfig.json` and `vitest.config.mts`).

### Code Style

Use comments sparingly — only comment complex or non-obvious code. Use `//` style comments.

### Testing

Tests live in `__tests__/` subdirectories alongside the code they test. Vitest runs with jsdom. Use `@testing-library/react` and `@testing-library/user-event` for component tests.
