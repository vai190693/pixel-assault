# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This App Is

UIGen is an AI-powered React component generator. Users describe UI in natural language, Claude generates the code using tool calls, and a sandboxed iframe renders a live preview. The app supports optional auth (JWT) and project persistence (SQLite via Prisma).

## Commands

```bash
npm run dev        # Start dev server (Next.js 15 + Turbopack) at http://localhost:3000
npm run build      # Production build
npm run lint       # ESLint
npm run test       # Vitest (run all tests)
npx vitest run src/path/to/file.test.ts  # Run a single test file
npm run setup      # First-time setup: install + prisma generate + migrate
npm run db:reset   # Wipe and re-migrate the SQLite database
```

## Environment Variables

- `ANTHROPIC_API_KEY` — Optional. If absent, the app falls back to `MockLanguageModel` (static hardcoded component).
- `JWT_SECRET` — Optional. Defaults to `"development-secret-key"` if not set.

No `.env.example` exists; create `.env` manually with those two keys.

## Architecture

### Request Flow

1. User types in `ChatInterface` → `ChatProvider` calls `POST /api/chat` with messages + serialized VFS state
2. `/api/chat/route.ts` calls `streamText()` (Vercel AI SDK) with Claude Haiku 4.5 and two tools
3. Claude uses tools to modify the **VirtualFileSystem** (in-memory, no disk I/O)
4. `FileSystemContext` applies tool results client-side; `PreviewFrame` watches for changes
5. `PreviewFrame` runs Babel standalone to transform JSX, builds an import map, and renders in a sandboxed `<iframe>` via blob URL
6. If authenticated, the completed chat + VFS state is saved to Prisma (SQLite) on stream finish

### AI / Tool Layer (`src/lib/`)

- **`provider.ts`** — Selects model. Uses `claude-haiku-4-5` when `ANTHROPIC_API_KEY` is set; falls back to `MockLanguageModel` (returns hardcoded static components) for development without a key.
- **`prompts/generation.tsx`** — System prompt instructing Claude to build React + Tailwind components, using `@/` import aliases.
- **`tools/str-replace.ts`** — `str_replace_editor` tool: `view`, `create`, `str_replace`, `insert` on VFS files.
- **`tools/file-manager.ts`** — `file_manager` tool: `rename`, `delete` on VFS files.
- **`lib/file-system.ts`** — `VirtualFileSystem` class (in-memory tree). Serializes to `Record<string, FileNode>` for DB storage (Maps become plain objects). Path normalization: always `/`-prefixed, parent dirs auto-created.

### Contexts (`src/lib/contexts/`)

- **`file-system-context.tsx`** — Holds VFS state; exposes methods consumed by tool handlers and PreviewFrame. Tool call results from `chat-context` are applied here.
- **`chat-context.tsx`** — Wraps Vercel AI SDK's `useChat`; passes serialized VFS on every message; routes tool results back to `FileSystemContext`.

### Preview (`src/components/preview/PreviewFrame.tsx`)

- Detects entry point: `/App.jsx`, `/App.tsx`, `/index.jsx`, etc.
- Transforms JSX client-side with Babel standalone (React automatic runtime, TypeScript support; CSS imports stripped).
- Builds import map: React 19 + ReactDOM from esm.sh CDN; local `@/` imports become blob URLs of transformed files.
- Renders in a sandboxed `<iframe>` with `allow-scripts allow-same-origin allow-forms`.

### Auth (`src/lib/auth.ts`, `src/actions/index.ts`)

- JWT sessions via JOSE, stored in HttpOnly cookies (7-day expiry). Payload: `{ userId, email, expiresAt }`.
- `middleware.ts` guards `/api/projects` and `/api/filesystem` (returns 401 if no valid session).
- Anonymous users get sessionStorage tracking via `anon-work-tracker.ts`; their work can be saved after sign-up.

### Database (`prisma/schema.prisma`)

The structure of the data is defined in `prisma/schema.prisma` — refer to that file every time you need to understand the database structure.

Two models: `User` (email + bcrypt password) and `Project` (name + `messages` JSON + `data` JSON + optional userId). SQLite at `prisma/dev.db`. Prisma client auto-generated into `src/generated/prisma/`.

## Code Style

- Use comments sparingly. Only comment complex/non-obvious code.

## Key Patterns

- **Prompt caching**: System message in `/api/chat` uses Anthropic ephemeral cache control to reduce token costs on repeated requests.
- **Mock mode**: No API key → `MockLanguageModel` → static component template. Full UI still works; useful for frontend work. `maxSteps` is 4 in mock vs. 40 in real mode.
- **Path alias**: `@/*` maps to `./src/*` (tsconfig + Next.js).
- **Turbopack + node-compat**: Dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` (already in the `dev` script) due to Turbopack edge runtime compatibility shims.
- **UI components**: Radix UI primitives wrapped in `src/components/ui/` via shadcn/ui conventions (`components.json`, `baseColor: "neutral"`).
- **Tests**: Co-located in `__tests__/` subdirectories next to source files. Uses Vitest + jsdom + `vite-tsconfig-paths`.
- **No autosave**: Project is saved to Prisma only on `onFinish` of the AI stream, and only when the user is authenticated and a `projectId` exists.
- **`/api/chat` limits**: `maxTokens: 10000`, `maxSteps: 40`, route `maxDuration: 120s`.
