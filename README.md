# Bookmarks

A community-curated learning resource hub where colleagues can share, discover, and upvote high-quality learning material — videos, articles, courses, and internal documents.

## Why this exists

The company's official learning platform surfaces low-quality content. This app lets teams collaboratively curate the resources that actually helped them, so the best material rises to the top. It also serves as a living directory for new starters to find essential links (org charts, key documents, onboarding guides).

## Features

### MVP (v1)

**Browsing (no login required)**
- Browse all shared bookmarks
- Full-text search across titles and descriptions
- Filter by tags, link type, and audience level
- Sort by most upvoted, newest, or relevance

**Curating (login required via company SSO)**
- Submit a new bookmark by pasting a URL
  - Title, description, and thumbnail auto-fetched from Open Graph / page metadata
  - Title and description are editable after auto-fetch
- Categorise with:
  - **Tags** — freeform, community-driven (e.g. "react", "onboarding", "architecture") with autocomplete suggestions from existing tags
  - **Link type** — Video, Article, Course, Document
  - **Audience level** — Beginner, Intermediate, Advanced
- Duplicate URL detection — submitting an existing URL surfaces the existing bookmark instead of creating a duplicate
- Upvote bookmarks that helped you (one upvote per user per bookmark)
- Edit bookmarks you submitted

**Auto-populated metadata**
- Submitter name (from SSO profile)
- Date added
- Thumbnail / favicon

### Future (v2+)

These are explicitly **out of scope for v1** but worth designing for:

- **Admin role** — designated admins who can delete any bookmark, manage content
- **Tag management** — admin ability to merge duplicate/similar tags (e.g. "reactjs" into "react")
- Comments / discussion threads on bookmarks
- Duration / word count metadata
- Collections (curated lists, e.g. "New Starter Essentials", "Frontend Guild Picks")
- Link health checking (detect dead links)
- Admin moderation tools (pin, archive, remove)
- Notification when new content is added to tags you follow

## Tech stack

| Layer | Technology | Rationale |
|---|---|---|
| Framework | Next.js (App Router) + TypeScript | Full-stack React framework, SSR/SSG support, API routes built in |
| Database | SQLite + better-sqlite3 | Single-file DB, FTS5 for full-text search, trivial to back up |
| Auth | NextAuth.js (Auth.js) with Azure AD provider | Handles OIDC flow, session management, and token refresh out of the box |
| Deployment | Azure App Service | Single deployable unit, Node.js native support |

## Project structure

```
bookmarks/
  src/
    app/                      # Next.js App Router
      layout.tsx              # Root layout (nav, auth provider)
      page.tsx                # Home — bookmark list with search/filters
      bookmarks/
        new/page.tsx          # Submit a new bookmark (auth required)
        [id]/page.tsx         # Single bookmark detail view
        [id]/edit/page.tsx    # Edit a bookmark (auth required, owner only)
      api/
        bookmarks/
          route.ts            # GET (list/search), POST (create)
          [id]/
            route.ts          # GET (single), PUT (update), DELETE (delete)
            upvote/route.ts   # POST (toggle upvote)
        tags/route.ts         # GET (list with counts)
        metadata/route.ts     # POST (fetch OG metadata for a URL)
    components/               # Reusable UI components
    lib/
      db/
        index.ts              # Repository interfaces (BookmarkRepo, TagRepo, etc.)
        sqlite.ts             # SQLite implementation of repositories
        connection.ts         # SQLite connection setup
        migrations.ts         # Database schema migrations
      metadata.ts             # OG metadata fetching logic
    types/                    # Shared TypeScript types
  auth.ts                     # NextAuth.js config (Azure AD provider)
  middleware.ts               # Next.js middleware (auth protection for write routes)
  data/                       # SQLite database file (local dev, git-ignored)
```

### Data access layer

All database access goes through repository interfaces defined in `src/lib/db/index.ts` (e.g. `BookmarkRepo`, `TagRepo`, `UpvoteRepo`). The SQLite implementation lives in `sqlite.ts`. To swap to a different data store later (e.g. PostgreSQL, Azure Cosmos DB), add a new implementation of the same interfaces without changing any route handlers or business logic.

## Data model

### Bookmarks

| Field             | Type     | Notes                                          |
|-------------------|----------|------------------------------------------------|
| id                | integer  | Primary key                                    |
| url               | text     | Unique, required                               |
| title             | text     | Auto-fetched from OG tags, editable            |
| description       | text     | Auto-fetched from OG tags, editable            |
| link_type         | text     | `video` \| `article` \| `course` \| `document` |
| audience_level    | text     | `beginner` \| `intermediate` \| `advanced`     |
| thumbnail_url     | text     | Auto-fetched from OG tags                      |
| submitted_by      | text     | User ID from Entra ID                          |
| submitted_by_name | text     | Display name from Entra ID                     |
| created_at        | datetime | Auto-set                                       |
| updated_at        | datetime | Auto-set                                       |

### Tags

| Field | Type | Notes |
|---|---|---|
| id | integer | Primary key |
| name | text | Unique, lowercase |

### Bookmark_Tags (join table)

| Field | Type | Notes |
|---|---|---|
| bookmark_id | integer | FK to bookmarks |
| tag_id | integer | FK to tags |

### Upvotes

| Field | Type | Notes |
|---|---|---|
| bookmark_id | integer | FK to bookmarks |
| user_id | text | From Entra ID |
| created_at | datetime | Auto-set |

Unique constraint on `(bookmark_id, user_id)` — one upvote per user per bookmark.

### Full-text search

A virtual FTS5 table indexes `title`, `description`, and tag names to power the search bar.

## API routes

Next.js Route Handlers in `src/app/api/`:

```
GET    /api/bookmarks              # List bookmarks (supports search, filter, sort, pagination)
GET    /api/bookmarks/:id          # Get a single bookmark with its tags and upvote count
POST   /api/bookmarks              # Create a bookmark (auth required)
PUT    /api/bookmarks/:id          # Update a bookmark you submitted (auth required)
DELETE /api/bookmarks/:id          # Delete a bookmark you submitted (auth required)

POST   /api/bookmarks/:id/upvote   # Toggle upvote (auth required)

GET    /api/tags                    # List all tags (with usage counts)

POST   /api/metadata               # Fetch OG metadata for a URL (auth required)
```

Auth endpoints (`/api/auth/*`) are handled automatically by NextAuth.js.

### Query parameters for `GET /api/bookmarks`

| Param | Example | Notes |
|---|---|---|
| `q` | `?q=react hooks` | Full-text search |
| `tags` | `?tags=react,typescript` | Comma-separated, AND logic |
| `type` | `?type=video` | Filter by link type |
| `level` | `?level=beginner` | Filter by audience level |
| `sort` | `?sort=upvotes` | `upvotes` \| `newest` \| `relevance` (default when searching) |
| `page` | `?page=2` | Pagination |

## Authentication flow

1. User clicks "Sign in"
2. NextAuth.js redirects to Microsoft Entra ID login
3. User authenticates with company credentials (SSO)
4. NextAuth.js receives tokens and creates a server-side session
5. Session is available in both Server Components (`auth()`) and Client Components (`useSession()`)
6. API route handlers check the session — unauthenticated requests to write endpoints return 401
7. Unauthenticated users can still browse all content freely

## Running locally

### Prerequisites

- Node.js >= 20
- An Azure App Registration (for OIDC) — see [Auth setup](#auth-setup)

### Setup

```bash
# Install dependencies
pnpm install

# Set up environment variables
cp .env.example .env.local
# Edit .env.local with your Azure App Registration details

# Run database migrations
pnpm run db:migrate

# Start dev server
pnpm run dev
```

The app runs on `http://localhost:3000`.

### Auth setup

1. Register an app in [Microsoft Entra ID](https://entra.microsoft.com/)
2. Add a **client secret** under "Certificates & secrets"
3. Set the redirect URI to `http://localhost:3000/api/auth/callback/azure-ad`
4. Note the **Application (client) ID**, **Directory (tenant) ID**, and **Client secret**
5. Add these to your `.env.local` file:

```env
AZURE_AD_CLIENT_ID=<your-client-id>
AZURE_AD_CLIENT_SECRET=<your-client-secret>
AZURE_AD_TENANT_ID=<your-tenant-id>
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=<random-string>  # Generate with: openssl rand -base64 32
```

## Deploying to Azure App Service

```bash
# Build
pnpm run build

# Deploy (using Azure CLI)
az webapp up --name bookmarks-app --resource-group <your-rg> --runtime "NODE:20-lts"
```

Next.js builds into `.next/` and runs as a standalone Node.js server in production.

### Environment variables to set in Azure

| Variable | Description |
|---|---|
| `AZURE_AD_CLIENT_ID` | Entra ID App Registration client ID |
| `AZURE_AD_CLIENT_SECRET` | Entra ID App Registration client secret |
| `AZURE_AD_TENANT_ID` | Entra ID tenant ID |
| `NEXTAUTH_URL` | Production URL (e.g. `https://bookmarks-app.azurewebsites.net`) |
| `NEXTAUTH_SECRET` | Random secret for session encryption |
| `DATABASE_PATH` | Path to SQLite file (default: `./data/bookmarks.db`) |

### Next.js standalone output

Add `output: "standalone"` to `next.config.ts` for a minimal production build that includes only the files needed to run. This keeps the deployment small and fast.

### Database persistence

The `DATABASE_PATH` env var controls where the SQLite file lives:

| Environment | Path | Notes |
|---|---|---|
| Local dev | `./data/bookmarks.db` (default) | Within the project, git-ignored |
| Azure production | `/home/data/bookmarks.db` | Persistent storage mount, survives restarts |


