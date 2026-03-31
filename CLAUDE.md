# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**UIGen** — an AI-powered React component generator with live preview. Users describe components in natural language, Claude generates the code, and a virtual file system renders a live preview.

The app lives entirely in `uigen/`. All commands below should be run from there.

## Commands

```bash
# Install deps, generate Prisma client, run migrations
npm run setup

# Dev server (localhost:3000) with Turbopack
npm run dev

# Background dev server (logs → logs.txt)
npm run dev:daemon

# Build
npm run build

# Tests (Vitest)
npm test

# Run a single test file
npx vitest path/to/test.ts

# Lint
npm run lint

# Reset the database
npm run db:reset
```

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app uses a `MockLanguageModel` that returns static demo components.

## Architecture

### Data flow

1. User sends a message via `ChatInterface` (client)
2. `ChatContext` (`lib/contexts/chat-context.tsx`) streams to `POST /api/chat`
3. The route (`app/api/chat/route.ts`) deserializes the virtual FS, calls Claude with two tools, and streams back tool calls + text
4. `FileSystemContext` (`lib/contexts/file-system-context.tsx`) applies tool calls to the in-memory FS
5. `PreviewFrame` re-renders the component live; `CodeEditor` shows updated files
6. On stream completion, the server saves the project to SQLite (authenticated users only)

### Virtual file system

`lib/file-system.ts` is an **in-memory** FS — nothing is written to disk. Files are serialized to JSON and sent with every chat request so the server is stateless. The root component must be at `/App.jsx`.

### AI tools exposed to Claude

- **`str_replace_editor`** — view, create, str_replace, insert on virtual files
- **`file_manager`** — rename and delete files/directories

The generation prompt lives in `lib/prompts/generation.tsx`. It enforces Tailwind-only styling, `@/` alias imports, and `/App.jsx` as entry point.

### Model / provider

`lib/provider.ts` returns an Anthropic `claude-haiku-4-5-20251001` model, or a `MockLanguageModel` when no API key is set.

### Auth

JWT sessions via `jose` (`lib/auth.ts`). Server-only. Passwords hashed with `bcrypt`. Cookies are HttpOnly + Secure, 7-day expiry.

### Database

Prisma + SQLite. Schema: `User` (email/password) → `Project` (name, messages JSON, data JSON). Run `npx prisma studio` to inspect.

### Preview rendering

`PreviewFrame` renders components in a sandboxed `<iframe srcdoc>`. The flow:
1. `lib/transform/jsx-transformer.ts` transforms all virtual FS files with Babel (JSX + optional TypeScript), producing blob URLs
2. An ES module import map is injected into the iframe HTML; `@/` aliases and extensionless imports are resolved to blob URLs; unknown third-party packages are forwarded to `https://esm.sh/`
3. Tailwind CSS is loaded via CDN (`cdn.tailwindcss.com`) inside the iframe
4. Syntax errors are caught per-file and displayed inline instead of crashing the preview

### Anonymous work tracking

`lib/anon-work-tracker.ts` uses `sessionStorage` to preserve unauthenticated users' messages and file system data across the sign-in flow. On auth completion the saved data is restored so work isn't lost.

### Node.js compat shim

`node-compat.cjs` (required via `NODE_OPTIONS`) deletes the `localStorage`/`sessionStorage` globals that Node 25+ exposes experimentally, preventing SSR crashes where libs detect these globals and assume a browser environment.

### UI layout

`app/main-content.tsx` uses `react-resizable-panels` for a three-panel layout:
- Left: `ChatInterface`
- Right top/bottom toggle: `PreviewFrame` | (`FileTree` + `CodeEditor`)

UI components come from shadcn/ui (Radix primitives + Tailwind), code editor is Monaco.
