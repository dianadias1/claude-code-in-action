# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Initial setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack at http://localhost:3000
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run all Vitest tests
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

Run tests in watch mode:
```bash
npx vitest
```

## Environment

Copy `.env` and add `ANTHROPIC_API_KEY` to enable Claude AI generation. Without it, the app falls back to a `MockLanguageModel` that generates static component templates.

## Architecture

UIGen is a full-stack AI-powered React component generator. Users describe components in a chat interface; Claude generates them using tool calls against an in-memory virtual file system; the results render in a sandboxed iframe.

### Request Flow

1. User submits a prompt via `ChatInterface`
2. `POST /api/chat` receives messages + serialized file system + projectId
3. Server initializes a `VirtualFileSystem`, selects an LLM provider, and registers two tools:
   - `str_replace_editor` — create/view/edit files (str_replace, insert operations)
   - `file_manager` — rename/delete files
4. `streamText` (Vercel AI SDK) sends the prompt to Claude with the tools
5. Claude calls tools iteratively to write JSX/TSX files into the virtual FS
6. Completed file system is serialized and saved to the `Project` in SQLite (authenticated users only)
7. Client-side `FileSystemContext` receives updates and triggers re-render of the preview

### Preview Rendering (`PreviewFrame.tsx`)

The iframe preview works without a build step:
- JSX files are extracted from the virtual FS
- Babel standalone transpiles JSX → `React.createElement` calls in-browser
- An ES module import map resolves `react`, `react-dom`, etc. from CDN
- The resulting module is injected into a sandboxed `<iframe>`

### Key Files

| File | Purpose |
|---|---|
| `src/app/api/chat/route.ts` | Main generation endpoint — wires LLM, tools, and streaming |
| `src/lib/file-system.ts` | In-memory VirtualFileSystem (no disk I/O); serializable to JSON |
| `src/lib/provider.ts` | Selects Claude or MockLanguageModel based on env vars |
| `src/lib/tools/str-replace.ts` | `str_replace_editor` tool implementation |
| `src/lib/tools/file-manager.ts` | `file_manager` tool implementation |
| `src/lib/prompts/generation.tsx` | System prompt injected into every Claude request |
| `src/lib/transform/jsx-transformer.ts` | Babel-based JSX → JS transformation for preview |
| `src/lib/contexts/file-system-context.tsx` | Client-side FS state; drives preview and editor |
| `src/lib/contexts/chat-context.tsx` | Chat message state and submission logic |
| `src/lib/auth.ts` | JWT session management via httpOnly cookies |
| `src/middleware.ts` | Protects `/api/chat` and project routes |

### Database

Prisma with SQLite (`prisma/dev.db`). Two models:
- `User` — email/password auth
- `Project` — stores chat `messages` (JSON array) and `data` (serialized VirtualFileSystem JSON)

Anonymous users can work without an account; their in-progress work is tracked in `localStorage` via `src/lib/anon-work-tracker.ts`.

### Path Alias

`@/*` maps to `./src/*` (configured in `tsconfig.json` and `vitest.config.mts`).
