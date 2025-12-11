# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude (via Anthropic API) to generate React components in real-time within a virtual file system, with no files written to disk.

**Tech Stack:**
- Next.js 15 with App Router and Turbopack
- React 19
- TypeScript
- Tailwind CSS v4
- Prisma with SQLite
- Anthropic Claude AI (claude-haiku-4-5)
- Vercel AI SDK

## Common Commands

### Development
```bash
npm run dev              # Start dev server with turbopack
npm run dev:daemon       # Start dev server in background, logs to logs.txt
npm run build            # Production build
npm run start            # Start production server
```

### Testing
```bash
npm run test             # Run all tests with vitest
```

### Database
```bash
npm run setup            # Install deps + generate Prisma client + run migrations
npm run db:reset         # Reset database (force migration reset)
npx prisma generate      # Regenerate Prisma client only
npx prisma migrate dev   # Run database migrations
```

### Linting
```bash
npm run lint             # Run ESLint
```

## Architecture

### Virtual File System (VFS)
The core architecture revolves around a **VirtualFileSystem** class (`src/lib/file-system.ts`) that manages an in-memory file tree. This is NOT a real filesystem - all files exist only in memory and are serialized to the database.

- Files are represented as a tree of `FileNode` objects stored in a `Map`
- The VFS supports standard file operations: create, read, update, delete, rename
- Projects are persisted by serializing the VFS state to the `Project.data` JSON field
- The VFS is reconstructed from serialized data on project load

### AI Component Generation Flow

1. **API Route** (`src/app/api/chat/route.ts`):
   - Receives chat messages and current VFS state from frontend
   - Reconstructs VirtualFileSystem from serialized nodes
   - Streams responses using Vercel AI SDK's `streamText`
   - Uses up to 40 steps (or 4 for mock provider) for multi-turn tool calling
   - Saves conversation and VFS state to database on completion

2. **AI Tools** (`src/lib/tools/`):
   - `str_replace_editor`: File operations (create, view, str_replace, insert)
   - `file_manager`: File management operations
   - Both tools operate on the same VirtualFileSystem instance

3. **Provider System** (`src/lib/provider.ts`):
   - Uses Anthropic Claude (claude-haiku-4-5) if `ANTHROPIC_API_KEY` is set
   - Falls back to `MockLanguageModel` that generates static Counter/Form/Card components
   - Mock provider simulates multi-step tool calling for demo purposes

4. **System Prompt** (`src/lib/prompts/generation.tsx`):
   - Instructs AI to create React components with Tailwind CSS
   - Requires `/App.jsx` as entry point (exported as default)
   - All non-library imports must use `@/` alias (e.g., `@/components/Calculator`)

### Frontend Architecture

**Preview System** (`src/components/preview/PreviewFrame.tsx`):
- Transforms JSX to executable code using Babel standalone
- Creates import maps and blob URLs for modules
- Renders components in sandboxed iframe with `srcdoc`
- Auto-detects entry point: `/App.jsx`, `/App.tsx`, `/index.jsx`, etc.
- Hot-reloads when VFS changes via `refreshTrigger` from context

**File System Context** (`src/lib/contexts/file-system-context.tsx`):
- Wraps VirtualFileSystem in React context
- Provides `refreshTrigger` to signal preview updates
- Manages client-side VFS instance that syncs with server

**Chat Interface** (`src/components/chat/`):
- Uses Vercel AI SDK's `useChat` hook
- Streams AI responses and tool calls in real-time
- Updates VFS state from tool results

### Database Schema

```prisma
User {
  id, email, password, projects[]
}

Project {
  id, name, userId?, messages (JSON), data (JSON)
}
```

- `messages`: Serialized chat history (array of AI SDK messages)
- `data`: Serialized VirtualFileSystem state (Record<path, FileNode>)
- Projects can be anonymous (userId = null) or owned by authenticated users

### Path Aliases

The project uses `@/*` as an import alias that maps to `./src/*` (configured in `tsconfig.json`).

**Example:**
```typescript
import { VirtualFileSystem } from "@/lib/file-system";
import { ChatInterface } from "@/components/chat/ChatInterface";
```

### Testing

- Uses Vitest with jsdom environment
- Test files located in `__tests__` directories next to source files
- React Testing Library for component tests
- Run specific test: `npm run test -- <test-file-pattern>`

## Important Implementation Notes

### When Working with the Virtual File System
- Always use normalized paths (starting with `/`)
- The VFS automatically creates parent directories
- Files must be serialized/deserialized when crossing client-server boundary
- Use `serialize()` to convert to JSON, `deserializeFromNodes()` to reconstruct

### When Modifying AI Tools
- Both tools must accept and modify the same VirtualFileSystem instance
- Tool parameters are validated with Zod schemas
- Tool execution is synchronous but wrapped in async for AI SDK compatibility

### When Changing the Preview System
- Preview transformations happen entirely client-side using Babel standalone
- Blob URLs must be revoked to prevent memory leaks
- Import maps require `allow-same-origin` sandbox attribute
- Entry point detection falls back through multiple patterns

### Database Migrations
- Prisma client output is in `src/generated/prisma` (not standard location)
- Always run `npx prisma generate` after schema changes
- Use `npm run db:reset` to start fresh (destroys data)
