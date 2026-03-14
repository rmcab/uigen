# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator. Users describe components in natural language, and Claude generates them with a live preview. The AI uses tools to write files into an in-memory virtual file system, which is then rendered in an iframe.

## Commands

```bash
# Initial setup
npm run setup          # Install deps + Prisma generate + migrate

# Development
npm run dev            # Next.js dev server with Turbopack (requires node-compat.cjs shim)
npm run dev:daemon     # Background dev server, logs to logs.txt

# Build & production
npm run build
npm start

# Tests
npm test               # Vitest (all tests)
npx vitest run src/components/chat/__tests__/MessageList.test.tsx  # Single test file

# Database
npm run db:reset       # Reset SQLite database via Prisma

# Lint
npm run lint
```

## Architecture

### AI Pipeline

The core flow: user message → `/api/chat` route → Vercel AI SDK streams tool calls from Claude → tools write files to virtual FS → client renders preview.

- **`src/app/api/chat/route.ts`** — POST handler; calls `streamText` with the model, system prompt, and tools
- **`src/lib/provider.ts`** — Returns real (`AnthropicProvider`) or mock provider based on `ANTHROPIC_API_KEY`. Mock provider used in tests/dev without a key; max 4 steps vs 40 for real
- **`src/lib/prompts/generation.tsx`** — System prompt instructing Claude how to generate components
- **`src/lib/tools/`** — Two AI tools: `str_replace_editor` (create/edit files) and `file_manager` (directory ops)

### Virtual File System

All generated component files live in memory only — never on disk. The virtual FS state is serialized to JSON and persisted in the `data` column of the `Project` SQLite record.

- **`src/lib/file-system.ts`** — Core VFS implementation
- **`src/lib/contexts/file-system-context.tsx`** — React context wrapping VFS state; consumed by editor, preview, and file tree
- **`src/lib/transform/jsx-transformer.ts`** — Transforms JSX/TSX before injecting into the preview iframe

### State Management

Two React contexts carry all app state:
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — Chat messages, streaming state, send handler
- **`FileSystemContext`** — VFS files, selected file, open files

### Database & Auth

- SQLite via Prisma. **The database schema is defined in `prisma/schema.prisma` — reference it anytime you need to understand the structure of data stored in the database.** Prisma client output goes to `src/generated/prisma`.
  - **`User`**: `id`, `email` (unique), `password`, `createdAt`, `updatedAt`, `projects[]`
  - **`Project`**: `id`, `name`, `userId?`, `messages` (JSON string, default `"[]"`), `data` (JSON string, default `"{}"` — stores VFS state), `createdAt`, `updatedAt`, `user?`
- Auth is JWT-based (`jose`) with bcrypt passwords. Logic lives in `src/lib/auth.ts` and server actions in `src/actions/`.
- Anonymous users are tracked via `src/lib/anon-work-tracker.ts` and can have projects without an account.

### Key Directories

| Path | Contents |
|------|----------|
| `src/app/` | Next.js App Router pages and API route |
| `src/actions/` | Next.js Server Actions (CRUD for projects, auth) |
| `src/components/chat/` | Chat UI + tests |
| `src/components/editor/` | Monaco-based code editor + file tree |
| `src/components/preview/` | Iframe preview renderer |
| `src/lib/` | All business logic (AI, VFS, auth, transforms) |
| `src/components/ui/` | shadcn/ui primitives (auto-generated, don't hand-edit) |

## Environment

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude. Without it, the app falls back to a mock provider that returns canned responses (useful for UI development).

## Coding Guidelines

- Use comments sparingly. Only comment complex code.

## Testing

Tests use Vitest + React Testing Library with jsdom. Test files are co-located under `__tests__/` subdirectories next to the components they test.
