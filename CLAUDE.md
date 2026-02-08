# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language via a chat interface, Claude generates React code through tool calls, and components are previewed live in an iframe. All file operations happen in a virtual in-memory file system — nothing is written to disk.

## Commands

```bash
npm run setup          # First-time: install deps, generate Prisma client, run migrations
npm run dev            # Dev server with Turbopack on http://localhost:3000
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Vitest (runs in watch mode)
npx vitest run         # Run tests once without watch
npx vitest run src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run db:reset       # Reset SQLite database
```

The project runs without an Anthropic API key using a mock provider that returns static component examples.

## Architecture

### Request Flow

1. User sends a message in the chat UI
2. `POST /api/chat` (`src/app/api/chat/route.ts`) receives messages + serialized file system
3. Vercel AI SDK streams responses from Claude (claude-haiku-4-5) with two tools available:
   - `str_replace_editor` — create, view, str_replace, insert operations on files
   - `file_manager` — rename/move, delete files
4. Tools operate on a `VirtualFileSystem` instance reconstructed from the request
5. File changes flow back to the client via React Context and update the UI
6. `PreviewFrame` transforms JSX client-side using `@babel/standalone`, builds an import map (resolving React libraries from esm.sh, local files via blob URLs), and renders in an iframe

### Key Modules

- **`src/lib/file-system.ts`** — `VirtualFileSystem` class: in-memory file tree with full CRUD, path normalization, serialization/deserialization. Central to both server-side tool execution and client-side state.
- **`src/lib/provider.ts`** — LLM provider factory. Returns Anthropic model when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that simulates multi-step tool calls.
- **`src/lib/prompts/generation.tsx`** — System prompt defining how Claude should generate components.
- **`src/lib/tools/str-replace.ts`** and **`file-manager.ts`** — AI SDK tool definitions that wrap VirtualFileSystem methods.
- **`src/lib/transform/jsx-transformer.ts`** — Client-side Babel JSX transformation + import map generation for the preview iframe.
- **`src/lib/contexts/chat-context.tsx`** — Chat state (messages, streaming, input) via React Context.
- **`src/lib/contexts/file-system-context.tsx`** — File system state (selected file, file operations) via React Context.
- **`src/lib/auth.ts`** — JWT-based auth using `jose`, sessions stored in HTTP-only cookies (7-day expiry).

### UI Layout

Three-panel resizable layout (`src/app/main-content.tsx`):
- **Left (35%):** Chat interface
- **Right (65%):** Toggle between Preview (iframe) and Code view (file tree + Monaco editor)

### Database

SQLite via Prisma. Schema in `prisma/schema.prisma`:
- `User` — email/password auth
- `Project` — stores name, JSON-stringified messages, and serialized file system data. `userId` is optional (anonymous users supported).

Prisma client is generated to `src/generated/prisma/`.

### Preview System

`PreviewFrame` (`src/components/preview/`) auto-detects entry points (App.jsx/tsx, index.jsx/tsx), transforms all JSX files, builds an import map resolving `@/` aliases, and renders via srcdoc iframe with Tailwind CDN. Includes an ErrorBoundary for runtime errors.

## Conventions

- Path alias: `@/*` maps to `src/*`
- UI components use shadcn/ui (Radix primitives + Tailwind) in `src/components/ui/`
- Server actions in `src/actions/` for data mutations (create/get projects)
- Tests use Vitest + jsdom + React Testing Library, located in `__tests__/` directories alongside source
- Use comments sparingly. Only comment complex code.
