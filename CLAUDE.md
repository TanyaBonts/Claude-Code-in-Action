# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates JSX/TSX files into a virtual file system (no disk writes), and the result renders in a sandboxed iframe using Babel standalone + React 19.

## Setup & Commands

```bash
npm run setup        # Install deps, generate Prisma client, run migrations
npm run dev          # Start Next.js dev server (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run Vitest test suite
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

**Environment:** Copy `.env` and set `ANTHROPIC_API_KEY`. The app works without it — falls back to `MockLanguageModel` returning static demo code.

## Architecture

### Data Flow

1. User types a message → `ChatInterface` → `useChat` hook (Vercel AI SDK) → `POST /api/chat`
2. `/api/chat/route.ts` calls Claude (`claude-haiku-4-5`) via Anthropic SDK with two tools: `str_replace_editor` and `file_manager`
3. Claude responds with tool calls → server applies them to a serialized `VirtualFileSystem` → streams updates back
4. Client receives tool result deltas → `FileSystemContext` updates → `PreviewFrame` re-renders the iframe

### Key Abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`): In-memory file tree. Serializes to/from JSON for DB storage. All file operations go through this class.
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): React context wrapping `VirtualFileSystem` state for the editor and preview.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`, feeds file system state into each request body.
- **`PreviewFrame`** (`src/components/preview/PreviewFrame.tsx`): Renders an `<iframe srcdoc>` using `createPreviewHTML` from `jsx-transformer.ts`, which builds a dynamic import map and injects Babel standalone for client-side JSX compilation.
- **`jsx-transformer.ts`** (`src/lib/transform/jsx-transformer.ts`): Converts the virtual file tree into self-contained iframe HTML. The import map routes bare specifiers (e.g., `react`, `react-dom`) to CDN URLs.
- **`provider.ts`** (`src/lib/provider.ts`): Factory that returns the real Anthropic model or `MockLanguageModel` based on whether `ANTHROPIC_API_KEY` is set.

### AI Tools (used by Claude during generation)

- **`str_replace_editor`** (`src/lib/tools/str-replace.ts`): Supports `view`, `create`, `str_replace`, `insert` commands for file editing.
- **`file_manager`** (`src/lib/tools/file-manager.ts`): Supports `list` and `delete` operations.

The system prompt for generation lives in `src/lib/prompts/generation.tsx`. Anthropic prompt caching is enabled for system prompts in `src/app/api/chat/route.ts`.

### Auth & Persistence

- JWT sessions via httpOnly cookies (7-day expiry), implemented in `src/lib/auth.ts`.
- Server Actions in `src/actions/` handle auth (signUp, signIn, signOut) and project CRUD.
- Prisma + SQLite: schema defined in `prisma/schema.prisma` — reference it whenever you need to understand the data model. `User` and `Project` are the two models. `Project.messages` stores chat history (JSON), `Project.data` stores the serialized virtual file system (JSON). `userId` is optional — anonymous users can work without signing in (tracked in localStorage via `anon-work-tracker.ts`).
- Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem` routes.

### UI Layout

`main-content.tsx` uses `react-resizable-panels` to split the screen:
- Left (35%): `ChatInterface`
- Right (65%): Tabs — "Preview" (`PreviewFrame`) | "Code" (`FileTree` + `CodeEditor`)

Monaco editor (`@monaco-editor/react`) is used for code editing with language auto-detection.

## Path Aliases

TypeScript `@/*` maps to `./src/*`.
