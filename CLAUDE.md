# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI (via Anthropic API) to generate React components through a chat interface, with a virtual file system that allows real-time editing and preview without writing to disk.

## Key Commands

### Development
```bash
npm run dev              # Start dev server with Turbopack
npm run dev:daemon       # Start dev server in background, logs to logs.txt
npm run build            # Build for production
npm run start            # Start production server
```

### Testing
```bash
npm run test             # Run all tests with Vitest
```

### Database
```bash
npm run setup            # Install deps + generate Prisma client + run migrations
npm run db:reset         # Reset database (WARNING: deletes all data)
npx prisma migrate dev   # Create and apply new migrations
npx prisma generate      # Regenerate Prisma client after schema changes
npx prisma studio        # Open Prisma Studio (database GUI)
```

### Linting
```bash
npm run lint             # Run ESLint
```

## Architecture

### Core Data Flow

1. **Chat → AI Stream → Tool Calls → Virtual File System → Preview**
   - User sends message via ChatInterface
   - Message routed to `/api/chat/route.ts`
   - Claude AI streams response with tool calls (`str_replace_editor`, `file_manager`)
   - Tool calls modify VirtualFileSystem in memory
   - FileSystemContext syncs changes to UI
   - PreviewFrame renders updated components in real-time

2. **Virtual File System (VFS)**
   - `src/lib/file-system.ts`: Core VFS implementation using Maps
   - `src/lib/contexts/file-system-context.tsx`: React context wrapping VFS
   - Files stored in memory only (no disk writes during generation)
   - Serialized to database for persistence (authenticated users)

3. **AI Tool System**
   - `src/lib/tools/str-replace.ts`: File creation, viewing, and editing
   - `src/lib/tools/file-manager.ts`: File/folder rename and deletion
   - Tools execute against VFS, return results to AI
   - AI can make multiple sequential tool calls to build components

4. **Authentication & Projects**
   - JWT-based auth via cookies (`src/lib/auth.ts`)
   - Anonymous users can work without saving
   - Authenticated users can save/load projects
   - Projects store chat history + VFS state as JSON in SQLite

### Component Architecture

- **Chat Components** (`src/components/chat/`): ChatInterface, MessageList, MessageInput
- **Editor Components** (`src/components/editor/`): CodeEditor (Monaco), FileTree
- **Preview Components** (`src/components/preview/`): PreviewFrame (Babel transform + iframe)
- **Contexts**: FileSystemContext, ChatContext manage state
- **Actions** (`src/actions/`): Server actions for database operations

### Preview System

The preview system in `src/components/preview/PreviewFrame.tsx` transforms user-generated code in real-time:

1. Collect all files from VFS
2. Use `@babel/standalone` to transform JSX → JS (`src/lib/transform/jsx-transformer.ts`)
3. Replace `@/` imports with blob URLs
4. Create blob URLs for each transformed module
5. Generate HTML with script tags and render in iframe
6. Hot-reload iframe when files change

### Database Schema

Located in `prisma/schema.prisma`:
- **User**: id, email, password (bcrypt), timestamps
- **Project**: id, name, userId (nullable), messages (JSON), data (JSON serialized VFS), timestamps
- Generated Prisma client outputs to `src/generated/prisma/`

### AI Provider System

`src/lib/provider.ts` implements a mock provider for development without API key:
- If `ANTHROPIC_API_KEY` is set: uses Claude Haiku 4.5
- If missing: uses `MockLanguageModel` that generates static counter/form/card components
- Mock provider simulates streaming and multi-step tool calls

### AI Prompt Structure

System prompt in `src/lib/prompts/generation.tsx`:
- Must create `/App.jsx` as entrypoint (default export)
- Use TailwindCSS for styling (not hardcoded styles)
- Use `@/` import alias for local files
- Operating on root route `/` of virtual filesystem
- No HTML files needed

## File Organization

```
src/
├── app/                    # Next.js App Router pages
│   ├── api/chat/          # AI streaming endpoint
│   ├── [projectId]/       # Dynamic project pages
│   └── page.tsx           # Home page
├── components/
│   ├── auth/              # Sign in/up forms, AuthDialog
│   ├── chat/              # Chat UI components
│   ├── editor/            # Code editor & file tree
│   ├── preview/           # Live preview iframe
│   └── ui/                # shadcn/ui components
├── lib/
│   ├── contexts/          # React contexts (chat, file-system)
│   ├── tools/             # AI tool implementations
│   ├── transform/         # JSX → JS transformation
│   ├── prompts/           # AI system prompts
│   ├── auth.ts            # JWT auth utilities
│   ├── file-system.ts     # Virtual file system core
│   ├── provider.ts        # AI provider (Anthropic or mock)
│   └── prisma.ts          # Prisma client singleton
├── actions/               # Next.js server actions
├── hooks/                 # Custom React hooks
└── generated/prisma/      # Generated Prisma client (gitignored)
```

## Development Workflow

### Adding New AI Tools

1. Create tool in `src/lib/tools/`
2. Implement using Vercel AI SDK's `tool()` helper
3. Add to tools object in `src/app/api/chat/route.ts`
4. Handle tool calls in `src/lib/contexts/file-system-context.tsx`'s `handleToolCall()`

### Modifying VFS Behavior

- Core VFS logic: `src/lib/file-system.ts`
- React integration: `src/lib/contexts/file-system-context.tsx`
- Tool interfaces: `src/lib/tools/str-replace.ts` and `file-manager.ts`

### Changing Preview Rendering

- Iframe rendering: `src/components/preview/PreviewFrame.tsx`
- JSX transformation: `src/lib/transform/jsx-transformer.ts`

### Authentication Changes

- Auth logic: `src/lib/auth.ts`
- Middleware: `src/middleware.ts`
- Forms: `src/components/auth/`

## Testing

Tests use Vitest + React Testing Library:
- Run tests: `npm run test`
- Config: `vitest.config.mts`
- Test files: `**/__tests__/*.test.ts(x)`
- UI components use `jsdom` environment
- FileSystemContext and ChatContext have comprehensive test coverage

## Important Notes

- VFS paths must start with `/` (e.g., `/App.jsx`, not `App.jsx`)
- Generated React components must use `@/` alias for local imports
- Preview iframe security: code runs in sandboxed iframe with `sandbox` attribute
- Database migrations are required after Prisma schema changes
- Mock provider limits tool calls to 4 steps (vs 40 for real API) to prevent repetition
- Anonymous work tracking in `src/lib/anon-work-tracker.ts` warns users before navigation
- write comments sparingly. only comment complex code
- when asked questions about the schema of the database, use this file @prisma/schema.prisma