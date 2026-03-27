# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server (http://localhost:3000) with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run all tests (Vitest)
npm test -- src/components/chat/__tests__/MessageList.test.tsx  # Run single test file
npm test -- --watch  # Watch mode
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset database (destructive)
```

## Architecture Overview

UIGen is a Next.js 15 app where users describe React components in a chat interface, Claude generates them via tool calling, and the result renders live in a sandboxed iframe — all backed by an in-memory virtual file system.

### Request Flow

1. User types a prompt → `ChatInterface` → POST `/api/chat`
2. `/api/chat/route.ts` calls `streamText()` (Vercel AI SDK) with Claude tools
3. Claude responds with tool calls (`str_replace_editor`, `file_manager`) to create/edit virtual files
4. Client-side `FileSystemContext` intercepts tool calls and mutates the in-memory `VirtualFileSystem`
5. `PreviewFrame` re-renders on file system changes — transforms JSX via Babel standalone in a sandboxed iframe
6. On finish, if user is authenticated, messages + serialized file system are saved to SQLite via Prisma

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`)
Pure in-memory tree — no disk writes. Serializes to/from a plain JSON object for DB persistence. Every project's files live here during a session.

**AI Provider** (`src/lib/provider.ts`)
Returns `claude-haiku-4-5` if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that generates a static counter/form/card component across 4 deterministic steps.

**Chat API** (`src/app/api/chat/route.ts`)
Accepts `{ messages, files, projectId? }`. Streams tool calls back to the client. Max 40 steps (real) / 4 steps (mock). Saves to DB in `onFinish` if session exists.

**Preview Rendering** (`src/components/preview/PreviewFrame.tsx` + `src/lib/transform/jsx-transformer.ts`)
Converts virtual FS files to blob URLs, builds an ES module import map, injects Tailwind, and renders in an iframe. Entry point is `/App.jsx` or `/App.tsx`.

**Authentication** (`src/lib/auth.ts`, `src/actions/index.ts`)
JWT in httpOnly cookie (`auth-token`, 7-day expiry). Passwords bcrypt-hashed. Anonymous users can use the app fully — projects only persist for authenticated users.

### Data Model (Prisma/SQLite)

```
User     id, email, password, projects[]
Project  id, name, userId?, messages (JSON[]), data (JSON virtualFS)
```

Prisma client is generated to `src/generated/prisma` (not the default location).

### Path Alias

`@/*` maps to `src/*`. All internal imports use this alias.

### State Management

Two React contexts:
- `FileSystemContext` — owns the `VirtualFileSystem` instance, handles tool call side-effects, exposes file CRUD and `refreshTrigger`
- `ChatContext` — owns message history and streaming state
