# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** is an AI-powered React component generator built with Next.js 15 (App Router). Users describe React components in natural language and Claude generates them with live preview.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run Vitest unit tests
npm run db:reset     # Reset SQLite database to initial state
```

To run a single test file:
```bash
npx vitest run src/path/to/file.test.tsx
```

## Architecture

### Core Concepts

**Virtual File System** — `src/lib/file-system.ts` manages an in-memory file tree (`VirtualFileSystem` class). Files never write to disk; they're serialized as JSON and persisted in the database `Project.data` column. The AI edits files through two tools: `str_replace_editor` (create/view/edit) and `file_manager` (rename/delete), defined in `src/lib/tools/`.

**AI Integration** — `/api/chat` streams responses using the Vercel AI SDK (`ai` package) with `@ai-sdk/anthropic`. If `ANTHROPIC_API_KEY` is unset, `src/lib/provider.ts` falls back to a `MockLanguageModel` that returns static components. The system prompt lives in `src/lib/prompts/`.

**React Contexts** — Two contexts wrap the main UI:
- `FileSystemContext` (`src/lib/contexts/`) — virtual file state, active file, editor content
- `ChatContext` — chat messages, AI streaming, project persistence

**Authentication** — JWT tokens in httpOnly cookies (7-day expiry), handled by `src/lib/auth.ts` and `src/middleware.ts`. Projects support both authenticated users and anonymous usage (`userId` is nullable).

### Data Flow

1. User types in `ChatInterface` → message sent to `/api/chat`
2. Server streams Claude's response; Claude calls tools to create/modify virtual files
3. Tool results update `FileSystemContext` on the client
4. `PreviewFrame` renders the virtual `/App.jsx` in real-time

### Database

SQLite via Prisma. Schema defined in `prisma/schema.prisma`: `User` (email + bcrypt password) → many `Project` (name, messages JSON, data JSON). Run `npx prisma studio` to inspect.

### Key Path Aliases

`@/*` maps to `src/*` (configured in `tsconfig.json` and `components.json`).

### UI Components

Uses shadcn/ui ("new-york" style) with Radix UI primitives, Tailwind CSS v4, Monaco Editor for code editing, and lucide-react for icons.
