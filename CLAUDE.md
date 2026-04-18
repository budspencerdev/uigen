# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup
npm run setup          # install + prisma generate + prisma migrate dev

# Development
npm run dev            # Next.js 15 dev server with Turbopack
npm run dev:daemon     # Dev server in background, logs to logs.txt

# Build & production
npm run build
npm run start

# Lint & test
npm run lint           # ESLint (next lint)
npm run test           # Vitest (all tests)
npx vitest run src/path/to/__tests__/file.test.ts  # Single test file

# Database
npx prisma studio      # GUI for SQLite DB
npm run db:reset       # Drop and recreate DB (destructive)
```

Environment: requires `ANTHROPIC_API_KEY` in `.env`. If absent, the app falls back to a mock provider (`src/lib/provider.ts`).

## Architecture

**UIGen** is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates them via tool calls into an in-memory virtual file system, which is rendered live in an iframe.

### Stack

- **Next.js 15** (App Router) + **React 19** + **TypeScript**
- **Vercel AI SDK** (`useChat`) + **Anthropic Claude** (claude-haiku-4-5)
- **SQLite** via **Prisma** (projects persist chat + file system state as JSON strings)
- **JWT** sessions (cookies, 7-day expiry) + **bcrypt** passwords
- **Monaco Editor** for code editing, **Babel standalone** for in-browser JSX transform
- **Tailwind CSS v4** + **shadcn/ui** (new-york style)

### Data flow

```
User chat input
  → useChat (Vercel AI SDK) → POST /api/chat
  → Claude API with tools: str_replace_editor, file_manager
  → Tool calls mutate VirtualFileSystem (in-memory)
  → Streamed response → FileSystemContext refresh
  → PreviewFrame: Babel JSX transform + import map → iframe render
  → CodeEditor / FileTree reflect updated state
```

### Key files

| File | Role |
|------|------|
| `src/app/api/chat/route.ts` | Streams Claude responses; handles tool execution; persists project |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — all file operations live here |
| `src/lib/contexts/file-system-context.tsx` | React context wrapping the VFS, shared across editor/preview/tree |
| `src/lib/contexts/chat-context.tsx` | Vercel AI SDK `useChat` integration + tool call dispatch |
| `src/lib/tools/str-replace.ts` | `str_replace_editor` tool — view / create / str_replace / insert |
| `src/lib/tools/file-manager.ts` | `file_manager` tool — rename / delete |
| `src/lib/prompts/generation.tsx` | System prompt instructing Claude how to generate components |
| `src/lib/transform/jsx-transformer.ts` | Babel transpilation + import map generation for iframe preview |
| `src/lib/provider.ts` | Selects real vs. mock language model provider |
| `src/lib/auth.ts` | JWT sign / verify / delete |
| `src/actions/index.ts` | Server Actions: signUp, signIn, signOut, getUser |
| `src/app/main-content.tsx` | Root UI: resizable panels (chat | preview | editor) |
| `src/middleware.ts` | Protects API routes; redirects unauthenticated requests |

### Project persistence

`Project.messages` and `Project.data` in Prisma are serialized JSON strings — messages hold the full chat history, data holds the virtual file system snapshot. Deserialization happens in `get-project.ts` and serialization in the chat route.

### AI tool limits

- Real provider: up to **40 tool steps**, **10 000 max tokens**, **120 s timeout**
- Mock provider: up to **4 tool steps** (prevents infinite loops in dev)

### Path alias

`@/*` resolves to `./src/*` (configured in `tsconfig.json`).

## Code style

- Omit comments unless the logic is genuinely non-obvious.
