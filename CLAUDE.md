# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (with Turbopack)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Reset the database
npm run db:reset
```

Tests use Vitest with jsdom and React Testing Library.

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app runs using a `MockLanguageModel` (`src/lib/provider.ts`) that returns static hardcoded components — useful for local development without API costs.

## Architecture

UIGen is a Next.js 15 (App Router) application where users chat with Claude to generate React components that are previewed live in-browser.

### Core Data Flow

1. **User sends a message** → `ChatContext` (`src/lib/contexts/chat-context.tsx`) serializes the in-memory `VirtualFileSystem` and sends it along with the message history to `POST /api/chat`.
2. **API route** (`src/app/api/chat/route.ts`) reconstructs the `VirtualFileSystem` server-side, calls the Claude model via Vercel AI SDK's `streamText`, and exposes two tools to the model:
   - `str_replace_editor` — view/create/edit files via string replacement or line insertion
   - `file_manager` — rename/delete files
3. **Tool calls stream back** to the client, where `FileSystemContext.handleToolCall` (`src/lib/contexts/file-system-context.tsx`) applies the mutations to the client-side `VirtualFileSystem`.
4. **PreviewFrame** (`src/components/preview/PreviewFrame.tsx`) watches `refreshTrigger` from `FileSystemContext`. On each change it transforms all files via Babel (`src/lib/transform/jsx-transformer.ts`), generates blob URLs, builds an import map, and injects everything into an `<iframe srcdoc>`. Third-party npm packages are resolved from `esm.sh`.
5. **Persistence** — on `onFinish`, if the user is authenticated and a `projectId` exists, the API route saves the full message history and serialized file system to the `Project` row in SQLite via Prisma.

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree of `FileNode` objects. It never writes to disk. The client holds the canonical copy; the server deserializes a snapshot sent with each request. `serialize()` / `deserializeFromNodes()` convert between the `Map`-based internal structure and a plain `Record<string, FileNode>` for JSON transport.

### Preview Pipeline

`createImportMap` in `src/lib/transform/jsx-transformer.ts`:
- Transforms JSX/TSX files with Babel standalone (React 19, optional TypeScript preset)
- Creates blob URLs for each transformed file
- Maps `@/` path aliases to root `/`
- Resolves third-party imports to `https://esm.sh/<package>`
- Creates placeholder stub modules for missing local imports
- Injects Tailwind CSS via CDN in the iframe

Entry point resolution order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → first `.jsx`/`.tsx` found.

### Auth

Custom JWT-based auth (`src/lib/auth.ts`) using `jose`. Sessions are stored in an `httpOnly` cookie (`auth-token`), valid 7 days. No third-party auth library. Middleware (`src/middleware.ts`) handles session verification. Anonymous users can work without signing in; their session data is tracked in `sessionStorage` via `src/lib/anon-work-tracker.ts`.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` (email + bcrypt password) and `Project` (stores `messages` and `data` as JSON strings). The Prisma client is generated into `src/generated/prisma`.

### AI Tools Contract

The system prompt (`src/lib/prompts/generation.tsx`) instructs the model to:
- Always create `/App.jsx` as the entry point
- Use Tailwind CSS for all styling (no inline styles)
- Use `@/` prefix for all non-library imports

### Layout

The main UI (`src/app/main-content.tsx`) is a two-panel resizable layout:
- **Left**: Chat interface (`ChatInterface` → `MessageList` + `MessageInput`)
- **Right**: Tabbed Preview / Code view. Code view adds a nested resizable split between `FileTree` and `CodeEditor` (Monaco).
