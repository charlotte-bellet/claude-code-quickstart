# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Code Style

Only comment complex or non-obvious code. Skip comments on self-evident logic.

## Commands

```bash
# First-time setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (uses Turbopack)
npm run dev

# Development server in background (logs to logs.txt)
npm run dev:daemon

# Run tests (Vitest)
npm test

# Run a single test file
npx vitest run src/components/editor/__tests__/file-tree.test.tsx

# Lint
npm run lint

# Build for production
npm run build

# Reset database
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Apply new migrations
npx prisma migrate dev
```

The app requires `NODE_OPTIONS='--require ./node-compat.cjs'` (handled by npm scripts). Without `ANTHROPIC_API_KEY` in `.env`, the app falls back to a `MockLanguageModel` that returns static demo code.

## Architecture

UIGen is a Next.js 15 App Router app where users chat with Claude to generate React components that render in a live preview pane.

### Request flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route constructs a `VirtualFileSystem` from the serialized files sent with the request, then calls `streamText` (Vercel AI SDK) with two tools: `str_replace_editor` and `file_manager`
3. Claude streams back text and tool calls. On `onFinish`, if the user is authenticated and has a `projectId`, the updated file system and full message history are persisted to SQLite via Prisma
4. The client (`src/lib/contexts/chat-context.tsx`) processes the stream and dispatches tool calls to `handleToolCall` in `FileSystemContext`. Tool calls are read from message parts (not the `onToolCall` callback) because the Vercel AI SDK doesn't fire `onToolCall` for server-side tools with `execute` functions.

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` is an in-memory tree structure (no disk I/O). It lives in React state via `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`). The context wraps the whole editor and exposes `handleToolCall`, which applies `str_replace_editor` (create/str_replace/insert) and `file_manager` (rename/delete) operations to the in-memory FS and triggers a React re-render.

`VirtualFileSystem` is mutated in-place — React doesn't track changes automatically. Always call `triggerRefresh()` after mutations to update dependent components (PreviewFrame, file tree, etc.).

### Live Preview

`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) reacts to `refreshTrigger` from `FileSystemContext`. On each change it calls `createImportMap` which:
- Transpiles all `.jsx/.tsx/.ts/.js` files with `@babel/standalone`
- Creates blob URLs for each transformed file
- Resolves third-party imports via `https://esm.sh/`
- Creates placeholder modules for missing local imports

The resulting import map is injected into an `<iframe srcdoc>` that mounts the React app (entry point: `/App.jsx` by default).

### Auth

Cookie-based JWT auth (`src/lib/auth.ts`) using `jose`. Sessions last 7 days. The middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem`; `/api/chat` is unprotected at the middleware level — auth is checked inside the handler to decide whether to persist data. Users can also use the app anonymously — projects without a `userId` in the DB are anonymous sessions tracked via `src/lib/anon-work-tracker.ts`.

### AI Provider

`src/lib/provider.ts` — `getLanguageModel()` returns the real Anthropic model (`claude-haiku-4-5`) when `ANTHROPIC_API_KEY` is set, otherwise returns `MockLanguageModel` for offline development. The mock simulates a multi-step agentic loop with static component code. `maxSteps` is 40 for the real provider and 4 for the mock.

### Data model (Prisma / SQLite)

The schema is defined in `prisma/schema.prisma` — always reference it to understand the structure of data stored in the database.

- `User`: email + bcrypt password
- `Project`: belongs to optional `User`, stores full message history as JSON string and VirtualFileSystem state as JSON string in `data` column

The Prisma client is generated to `src/generated/prisma` (not the default `@prisma/client`). Import it as `@/generated/prisma`.

### UI & Styling

Uses **Tailwind v4** — the CSS entry point (`src/app/globals.css`) uses `@import "tailwindcss"` syntax (not the old `tailwind.config.js` object). Theme tokens are defined with `@theme inline` CSS variables.

**shadcn/ui** components are configured via `components.json` (New York style, neutral base color, `lucide-react` icons). Add new shadcn components with `npx shadcn@latest add <component>`.

### Key directories

- `src/app/` — Next.js pages and API routes
- `src/components/chat/` — chat UI (input, message list, markdown renderer)
- `src/components/editor/` — file tree and Monaco-based code editor
- `src/components/preview/` — iframe preview
- `src/lib/contexts/` — React contexts (FileSystem, Chat)
- `src/lib/tools/` — Vercel AI SDK tool definitions (`str-replace.ts`, `file-manager.ts`)
- `src/lib/transform/` — JSX→JS transpilation + import map generation
- `src/lib/prompts/` — system prompt for Claude
- `src/actions/` — Next.js Server Actions for project CRUD
- `prisma/` — schema and SQLite migrations
