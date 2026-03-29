# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps + Prisma client + migrations
npm run dev          # Start dev server with Turbopack (localhost:3000)
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run Vitest tests
npm run db:reset     # Reset SQLite database
```

Single test: `npx vitest run <path-to-test-file>`

Environment: copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. Without it, the app uses a `MockLanguageModel` that returns static placeholder code — useful for offline development.

## Architecture

UIGen is an AI-powered React component generator. Users describe a UI in chat; Claude responds with tool calls that mutate an in-memory virtual file system; the result renders live in an iframe via Babel JSX transpilation.

### Request/data flow

1. User sends prompt in `ChatInterface` → `ChatProvider` (`src/lib/contexts/chat-context.tsx`) sends message + serialized VFS state to `POST /api/chat`
2. API route (`src/app/api/chat/route.ts`) calls Vercel AI SDK `streamText()` with the Claude model and two tools:
   - **`str_replace_editor`** (`src/lib/tools/str_replace.ts`) — view, create, str_replace, insert on files
   - **`file_manager`** (`src/lib/tools/file_manager.ts`) — rename, delete files
3. Tool calls mutate the `VirtualFileSystem` (in-memory Map tree, never written to disk)
4. `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) detects changes → refreshes Monaco editor and preview
5. `PreviewFrame` (`src/components/preview/`) runs `jsx-transformer.ts` (Babel standalone) to transpile JSX→JS, builds an import map, and injects into an iframe
6. On stream completion, if the user is authenticated, the project state is persisted to SQLite via Prisma

### Key modules

| Path | Role |
|---|---|
| `src/lib/file-system.ts` | `VirtualFileSystem` class — serializable in-memory file tree |
| `src/lib/contexts/` | React Context providers for chat state and file system state |
| `src/lib/tools/` | AI tool implementations given to Claude |
| `src/lib/transform/jsx-transformer.ts` | Babel transpilation + import map for live preview |
| `src/lib/prompts/generation.tsx` | System prompt that instructs Claude how to generate components |
| `src/lib/provider.ts` | Selects real Claude model or `MockLanguageModel` based on env |
| `src/lib/auth.ts` | JWT sessions (httpOnly cookie, 7-day expiry), bcrypt hashing |
| `src/actions/` | Next.js Server Actions for auth and project CRUD |
| `src/app/api/chat/route.ts` | Streaming AI endpoint (max duration: 120s) |
| `prisma/schema.prisma` | `User` and `Project` models backed by SQLite |

### Path aliases

`@/*` maps to `./src/*`. Generated components should use `@/` imports (the system prompt instructs Claude to do this).

### Authentication

JWT stored in an httpOnly cookie. Anonymous users can generate components; their work is tracked via `localStorage` (`src/lib/anon-work-tracker.ts`) so it can be claimed after sign-up.
