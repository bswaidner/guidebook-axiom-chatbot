# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an AI-powered chatbot application built on Vercel's Chat SDK template with custom features for Guidebook/Axiom. The app supports multi-turn conversations with AI models, rich artifact generation (code, text, images, spreadsheets), document management, and collaborative editing.

**Tech Stack**: Next.js 15 (App Router), React 19 RC, TypeScript, Drizzle ORM (PostgreSQL), NextAuth, AI SDK with xAI Gateway, shadcn/ui, Tailwind CSS 4, ProseMirror, CodeMirror

## Development Commands

### Core Commands
```bash
pnpm install          # Install dependencies
pnpm dev              # Start dev server with Turbo (localhost:3000)
pnpm build            # Run migrations + build for production
pnpm start            # Start production server
pnpm test             # Run Playwright E2E tests
```

### Code Quality
```bash
pnpm lint             # Check code with Ultracite/Biome
pnpm format           # Auto-fix code with Ultracite/Biome
```

### Database Operations
```bash
pnpm db:generate      # Generate migration files from schema changes
pnpm db:migrate       # Run pending migrations
pnpm db:studio        # Open Drizzle Studio (database GUI)
pnpm db:push          # Push schema changes directly (dev only)
pnpm db:pull          # Pull schema from database
pnpm db:check         # Check for schema issues
```

## Code Architecture

### Route Groups & Structure
- `app/(auth)/` - Authentication routes (login, register, guest flow)
- `app/(chat)/` - Main chat interface and API routes
- `app/(chat)/api/` - API endpoints for chat, documents, files, history, votes, suggestions

### Key Directories
- `lib/ai/` - AI model configuration, prompts, entitlements, and provider setup
- `lib/db/` - Database schema (Drizzle ORM), queries, migrations, utilities
- `lib/editor/` - Text editor utilities for ProseMirror
- `artifacts/` - Artifact renderers (code, text, image, sheet) as client components
- `components/ui/` - shadcn/ui components (22 reusable UI primitives)
- `components/elements/` - Custom chat-specific components
- `hooks/` - Custom React hooks (use-artifact, use-chat-visibility, use-auto-resume, etc.)

### Database Schema
All tables use Drizzle ORM. Important schemas:
- `user` - User accounts (email/password, guests have `guest-{timestamp}` emails)
- `chat` - Chat sessions with visibility (public/private) and lastContext
- `message_v2` - Chat messages with parts and attachments (replaces deprecated `Message`)
- `vote_v2` - Message votes (upvote/downvote) (replaces deprecated `Vote`)
- `document` - Artifacts with versioning (text, code, image, sheet kinds)
- `suggestion` - Collaborative editing suggestions for documents
- `stream` - Real-time stream tracking for chat responses

**⚠️ IMPORTANT**: `messageDeprecated` and `voteDeprecated` tables still exist in schema but are deprecated. Always use `message` (Message_v2) and `vote` (Vote_v2) tables. Never query or insert into deprecated tables.

### AI Model Configuration
Located in `lib/ai/providers.ts`:
- Uses AI SDK Gateway to route through Vercel AI Gateway
- Default models: xAI Grok (grok-2-vision-1212 for chat, grok-3-mini for reasoning)
- Provider IDs: `chat-model`, `chat-model-reasoning`, `title-model`, `artifact-model`
- Mock providers available for testing (see `models.mock.ts`)
- Test environment automatically uses mocks when `PLAYWRIGHT=True`

### Authentication Flow
- Uses NextAuth v5 (beta) with custom guest user system
- Middleware (`middleware.ts`) handles auth checks and guest creation
- Guest users have rate limiting (20 msgs/day vs 100 for registered users)
- Guest email format: `guest-{timestamp}` (regex in `lib/constants.ts`)

### Artifacts System
Four artifact types in `artifacts/` directory:
- **Text**: ProseMirror-based rich text editor with markdown support
- **Code**: CodeMirror with syntax highlighting (JS, Python, etc.)
- **Image**: Vercel Blob storage integration for image generation/display
- **Sheet**: react-data-grid for spreadsheet functionality

Artifacts are stored in `document` table and support versioning (composite PK: id + createdAt).

### Chat Streaming
- Main streaming endpoint: `app/(chat)/api/chat/[id]/stream/route.ts`
- Uses AI SDK's `streamText` with custom data streaming protocol
- Streams saved to `stream` table for resume/replay capability
- DataStreamHandler component (`components/data-stream-handler.tsx`) processes real-time updates

## Important Patterns

### Server vs Client Components
- Follow Next.js App Router conventions: Server Components by default
- Mark client components with `"use client"` directive
- Database queries in `lib/db/queries.ts` are marked with `"server-only"`
- Never import server-only code into client components

### Error Handling
- Custom `ChatSDKError` class in `lib/errors.ts` with structured error codes
- Format: `"category:subcategory"` (e.g., `"bad_request:database"`)
- All database queries wrap try-catch and throw ChatSDKError
- Example pattern:
```typescript
try {
  return await db.select().from(chat).where(eq(chat.id, id));
} catch (_error) {
  throw new ChatSDKError("bad_request:database", "Failed to get chat by id");
}
```

### Environment Variables
Required variables (see `.env.example`):
- `AUTH_SECRET` - NextAuth secret (generate with `openssl rand -base64 32`)
- `AI_GATEWAY_API_KEY` - Required for non-Vercel deployments (OIDC used on Vercel)
- `BLOB_READ_WRITE_TOKEN` - Vercel Blob storage
- `NEON_DB_STORAGE_POSTGRES_URL` - PostgreSQL connection string
- `UPSTASH_REDIS_REDIS_URL` - Redis for caching/sessions

### Linting Philosophy
Project uses Ultracite (Biome-based) with strict rules. Key principles from `.cursor/rules/ultracite.mdc`:
- **No `any` types** - noExplicitAny rule (currently disabled but should be enabled)
- **No console statements** - noConsole rule (currently disabled for debugging)
- **No non-null assertions (`!`)** - Use proper null checks or optional chaining
- **Arrow functions over function expressions**
- **Use `export type` and `import type` for types**
- **Accessibility first** - All interactive elements need proper ARIA attributes
- **React hooks best practices** - Proper dependency arrays, top-level calls only

## Testing

### E2E Tests (Playwright)
- Test files in `tests/` directory
- Run with `pnpm test` (sets `PLAYWRIGHT=True` env var)
- Uses mock models when `PLAYWRIGHT=True` (see `lib/ai/providers.ts`)
- Health check endpoint: `/ping` (returns "pong" 200 status)

### Running Single Tests
```bash
export PLAYWRIGHT=True && pnpm exec playwright test <test-file>
```

## Migration & Deployment

### Database Migrations
1. Modify schema in `lib/db/schema.ts`
2. Generate migration: `pnpm db:generate`
3. Review generated SQL in `drizzle/` directory
4. Run migration: `pnpm db:migrate`
5. Migrations auto-run on build via `tsx lib/db/migrate` in build script

### Vercel Deployment
- Automatic environment variable injection (Blob, Postgres, Redis tokens)
- AI Gateway uses OIDC tokens automatically (no API key needed)
- Build command: `pnpm build` (includes migrations)

## Known Technical Debt

1. **Deprecated schemas** - `messageDeprecated` and `voteDeprecated` tables need removal after data migration
2. **Pre-release dependencies** - Next.js 15 canary, React 19 RC, next-auth beta should be stabilized
3. **Disabled linter rules** - `noExplicitAny`, `noConsole`, `noNonNullAssertion` should be enabled
4. **Incomplete features** - Premium tier (paid membership) in `lib/ai/entitlements.ts` not implemented

## Adding New Features

### Adding a New AI Model
1. Update `lib/ai/models.ts` - Add model to `chatModels` array
2. Update `lib/ai/providers.ts` - Add model mapping in `customProvider`
3. Add corresponding mock in `lib/ai/models.mock.ts` for testing

### Adding a New Artifact Type
1. Create new directory in `artifacts/` (e.g., `artifacts/video/`)
2. Implement `client.tsx` with artifact renderer
3. Update `lib/db/schema.ts` - Add to `kind` enum in `document` table
4. Generate and run migration
5. Update artifact routing logic in chat components

### Adding API Endpoints
- Place in `app/(chat)/api/` or `app/(auth)/api/` based on functionality
- Use Next.js App Router conventions (`route.ts` files)
- Follow existing error handling patterns with ChatSDKError
- Validate requests with Zod schemas
- Check authentication via NextAuth's `auth()` function
