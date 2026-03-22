# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server (localhost:3000) with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run all Vitest tests
npm test -- <file>  # Run a single test file
npm run setup        # First-time setup: install, generate Prisma client, run migrations
npm run db:reset     # Force reset the database (destructive)
```

## Environment

- `ANTHROPIC_API_KEY` in `.env` — optional. If absent, a `MockLanguageModel` is used instead of Claude, which generates deterministic counter/form/card components.
- `JWT_SECRET` defaults to `"development-secret-key"` — should be set in production.
- Database: SQLite at `prisma/dev.db`.

## Architecture

UIGen is an AI-powered React component generator with three main panels: chat, code editor, and live preview.

**Request flow for component generation:**
1. User sends a message → `POST /api/chat` streams a response via Vercel AI SDK
2. Claude calls two tools: `str_replace_editor` (create/view/edit files) and `file_manager` (rename/delete)
3. Tool calls update the **VirtualFileSystem** (in-memory, `src/lib/file-system.ts`)
4. The JSX transformer (`src/lib/transform/jsx-transformer.ts`) compiles files with Babel and builds an import map using esm.sh CDN
5. The preview iframe renders the compiled output with Tailwind CSS via CDN

**State management** lives in two React contexts:
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — conversation state, AI streaming via `useChat`
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — virtual FS state, active file selection

**Persistence:** Project messages and file system are JSON-serialized into `Project.messages` and `Project.data` columns (SQLite). Projects can be anonymous (`userId` is nullable).

**Authentication:** JWT sessions via Jose, stored in an httpOnly cookie (7-day expiry). `src/lib/auth.ts` manages sessions; `src/actions/index.ts` has `signUp`/`signIn`/`signOut`/`getUser`. Anonymous users get localStorage tracking via `src/lib/anon-work-tracker.ts`.

## Key files

| File | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | Streaming chat endpoint — tool definitions, Claude call, DB persistence |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — full CRUD + serialization |
| `src/lib/transform/jsx-transformer.ts` | Babel compilation + import map generation for preview iframe |
| `src/lib/prompts/generation.tsx` | System prompt sent to Claude for component generation |
| `src/lib/provider.ts` | Returns real Claude model or `MockLanguageModel` based on API key presence |
| `src/lib/tools/str-replace.ts` | `str_replace_editor` tool implementation |
| `src/lib/tools/file-manager.ts` | `file_manager` tool implementation |
| `prisma/schema.prisma` | Source of truth for all database models — always reference this when working with data shapes or DB queries |

## Testing

Tests use Vitest + React Testing Library with a jsdom environment (`vitest.config.mts`). Test files live in `__tests__/` directories adjacent to the code they test. Coverage spans: `ChatContext`, `FileSystemContext`, `VirtualFileSystem`, `JSXTransformer`, and key UI components.
