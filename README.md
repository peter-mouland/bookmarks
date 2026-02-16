# Bookmarks

A personal learning resource hub for saving, organising, and surfacing high-quality learning material — videos, articles, courses, and useful documents.

## Why this exists

There's plenty of learning content out there, but most of it is low quality. This app lets you curate the resources that actually helped you, so the best material is always easy to find. It also works as a personal directory for useful links (docs, tools, reference material).

## Features

### MVP (v1)

- Browse all saved bookmarks
- Full-text search across titles and descriptions
- Filter by tags, link type, and audience level
- Sort by most upvoted, newest, or relevance
- Submit a new bookmark by pasting a URL
  - Title, description, and thumbnail auto-fetched from Open Graph / page metadata
  - Title and description are editable after auto-fetch
- Categorise with:
  - **Tags** — freeform (e.g. "react", "onboarding", "architecture") with autocomplete suggestions from existing tags
  - **Link type** — Video, Article, Course, Document
  - **Audience level** — Beginner, Intermediate, Advanced
- Duplicate URL detection — submitting an existing URL surfaces the existing bookmark instead of creating a duplicate
- Upvote bookmarks (personal ranking to surface your best finds)
- Edit and delete bookmarks
- **Chrome extension** — click to save the current page as a bookmark
  - Scrapes title, description, thumbnail, and favicon from the page's Open Graph / meta tags
  - Opens the "add new bookmark" page with scraped data pre-filled via query params
  - Pre-filled fields are editable before submitting

**Auto-populated metadata**
- Date added
- Thumbnail / favicon

All actions are performed as a single hardcoded user (`peter.mouland@gmail.com`) for MVP. The data model supports multiple users to make adding auth straightforward in v2.

### Future (v2+)

These are explicitly **out of scope for v1** but worth designing for:

- **Authentication** — Firebase Auth or Microsoft Entra ID for multi-user support
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
| Framework | React Router v7 (framework mode) + Vite + TypeScript | Full-stack React framework with loaders/actions, file-based routing, built on Vite |
| Database | Cloud Firestore | Managed NoSQL DB, no server to run, scales automatically, realtime capable |
| Search | Fuse.js (client-side) | Lightweight fuzzy search over bookmarks loaded in the browser |
| Chrome extension | Manifest V3 + TypeScript | Scrapes page metadata, opens pre-filled "add bookmark" page |
| Deployment | Firebase Hosting + Cloud Functions | Static assets on CDN, SSR via Cloud Functions |

React Router v7 in framework mode (formerly Remix) provides a built-in server with loaders and actions. Data fetching happens in `loader` functions (server-side via Cloud Functions), mutations happen in `action` functions, and Vite handles the build.

## Project structure

```
bookmarks/
  app/
    root.tsx                    # Root layout (nav)
    routes/
      _index.tsx                # Home — bookmark list with search/filters
      bookmarks.new.tsx         # Submit a new bookmark
      bookmarks.$id.tsx         # Single bookmark detail view
      bookmarks.$id.edit.tsx    # Edit a bookmark
      api.bookmarks.tsx         # Resource route: GET (list/search), POST (create)
      api.bookmarks.$id.tsx     # Resource route: GET (single), PUT (update), DELETE
      api.bookmarks.$id.upvote.tsx  # Resource route: POST (toggle upvote)
      api.tags.tsx              # Resource route: GET (list with counts)
      api.metadata.tsx          # Resource route: POST (fetch OG metadata for a URL)
    components/                 # Reusable UI components
    lib/
      db/
        index.ts               # Repository interfaces (BookmarkRepo, TagRepo, etc.)
        firestore.ts           # Firestore implementation of repositories
      metadata.server.ts       # OG metadata fetching logic (server-only)
      user.server.ts           # Hardcoded user config (server-only, swap for auth in v2)
      search.ts                # Fuse.js search index config
    types/                     # Shared TypeScript types
  extension/                   # Chrome extension (Manifest V3)
    manifest.json              # Extension config (permissions, icons, action)
    content.ts                 # Content script — scrapes OG/meta tags from active page
    popup.html                 # Minimal popup with "Save Bookmark" button
    popup.ts                   # Reads scraped data, opens app with query params
  firebase.json                # Firebase Hosting + Functions config
  .firebaserc                  # Firebase project alias
  vite.config.ts
  react-router.config.ts       # React Router framework config
```

### Chrome extension

The extension is intentionally simple — no API calls, no auth, no background service worker. It:

1. Injects a content script that reads OG/meta tags from the current page
2. Shows a popup with a "Save Bookmark" button
3. Opens `/bookmarks/new?url=...&title=...&description=...&thumbnail=...` in a new tab
4. The `bookmarks.new.tsx` route reads these query params and pre-fills the form

The extension lives in `extension/` and is loaded as an unpacked extension during development. It needs rebuilding separately from the main app (`pnpm run build:extension`).

### Data access layer

All database access goes through repository interfaces defined in `app/lib/db/index.ts` (e.g. `BookmarkRepo`, `TagRepo`, `UpvoteRepo`). The Firestore implementation lives in `firestore.ts`. To swap to a different data store later, add a new implementation of the same interfaces without changing any loaders, actions, or business logic.

### Server-only modules

Files suffixed with `.server.ts` are excluded from the client bundle by React Router. Database access, metadata fetching, and user config only ever run on the server.

### Hardcoded user

`app/lib/user.server.ts` exports a `getCurrentUser()` function that returns the hardcoded user. When auth is added in v2, this single file is replaced with real session lookup — no other code needs to change.

### Search strategy

Firestore doesn't have built-in full-text search. For a personal app with hundreds of bookmarks, client-side search is fast and simple:

- **Full-text search** — Fuse.js runs in the browser over all loaded bookmarks, providing fuzzy matching across title and description
- **Filtering** (tags, type, level) — Firestore `where` queries handle structured filters server-side
- **Sorting** (upvotes, newest) — Firestore `orderBy` queries

The home page loader fetches bookmarks from Firestore with any active filters/sort applied. The search box then filters those results client-side with Fuse.js for instant feedback.

## Data model

Firestore collections and document structure:

### `bookmarks` collection

Each document ID is auto-generated.

| Field             | Type            | Notes                                          |
|-------------------|-----------------|------------------------------------------------|
| url               | string          | Required                                       |
| title             | string          | Auto-fetched from OG tags, editable            |
| description       | string          | Auto-fetched from OG tags, editable            |
| linkType          | string          | `video` \| `article` \| `course` \| `document` |
| audienceLevel     | string          | `beginner` \| `intermediate` \| `advanced`     |
| thumbnailUrl      | string          | Auto-fetched from OG tags                      |
| tags              | array\<string\> | Lowercase tag names, stored directly on the document for querying |
| submittedBy       | string          | User ID (hardcoded to `peter.mouland@gmail.com` for MVP) |
| submittedByName   | string          | Display name (hardcoded to `Peter Mouland` for MVP) |
| upvotedBy         | array\<string\> | User IDs who upvoted (enables one-upvote-per-user and count in one read) |
| createdAt         | timestamp       | Auto-set |
| updatedAt         | timestamp       | Auto-set |

Tags and upvotes are stored as arrays directly on the bookmark document. This avoids join queries (which Firestore doesn't support) and keeps reads to a single document. For a personal app this is the right trade-off — the arrays won't grow large enough to cause issues.

### Duplicate URL detection

A Firestore query with `where("url", "==", submittedUrl)` checks for duplicates before creating a new bookmark.

### Firestore indexes

Composite indexes needed for filtered + sorted queries:

| Collection | Fields | Notes |
|---|---|---|
| bookmarks | `linkType` + `createdAt` | Filter by type, sort by newest |
| bookmarks | `audienceLevel` + `createdAt` | Filter by level, sort by newest |
| bookmarks | `tags` (array-contains) + `createdAt` | Filter by tag, sort by newest |

These are defined in `firestore.indexes.json` and deployed with `firebase deploy`.

## API routes

Resource routes in `app/routes/api.*` expose a JSON API. Page routes use loaders/actions directly for data.

```
GET    /api/bookmarks              # List bookmarks (supports filter, sort, pagination)
GET    /api/bookmarks/:id          # Get a single bookmark
POST   /api/bookmarks              # Create a bookmark
PUT    /api/bookmarks/:id          # Update a bookmark
DELETE /api/bookmarks/:id          # Delete a bookmark

POST   /api/bookmarks/:id/upvote   # Toggle upvote

GET    /api/tags                    # List all unique tags (derived from bookmarks)

POST   /api/metadata               # Fetch OG metadata for a URL
```

Page routes also load data directly via loaders — the API resource routes exist for client-side fetches (e.g. upvoting without a full page reload).

### Query parameters for `GET /api/bookmarks`

| Param | Example | Notes |
|---|---|---|
| `q` | `?q=react hooks` | Client-side full-text search (Fuse.js) |
| `tags` | `?tags=react,typescript` | Firestore `array-contains-any` (OR logic per tag) |
| `type` | `?type=video` | Firestore `where` filter |
| `level` | `?level=beginner` | Firestore `where` filter |
| `sort` | `?sort=upvotes` | `upvotes` \| `newest` (default) |
| `page` | `?page=2` | Firestore cursor-based pagination |

## Running locally

### Prerequisites

- Node.js >= 20
- Firebase CLI (`pnpm add -g firebase-tools`)
- A Firebase project — see [Firebase setup](#firebase-setup)

### Setup

```bash
# Install dependencies
pnpm install

# Log in to Firebase
firebase login

# Start dev server (app + Firebase emulators for Firestore)
pnpm run dev
```

The app runs on `http://localhost:5173`. The Firestore emulator runs alongside it so no production data is touched during development.

### Firebase setup

1. Create a project at [Firebase Console](https://console.firebase.google.com/)
2. Enable **Cloud Firestore** (start in test mode for dev)
3. Run `firebase init` and select Hosting + Functions + Firestore
4. Add your Firebase config to `.env`:

```env
FIREBASE_PROJECT_ID=<your-project-id>
```

The Firebase Admin SDK uses Application Default Credentials locally (via `firebase login`) and auto-detects credentials in production.

## Deploying to Firebase

```bash
# Build
pnpm run build

# Deploy
firebase deploy
```

This deploys:
- Static client assets to **Firebase Hosting** (served from CDN)
- Server-side rendering via **Cloud Functions**
- Firestore indexes and security rules

### Environment variables

| Variable | Description |
|---|---|
| `FIREBASE_PROJECT_ID` | Firebase project ID |

## Backups

Firestore supports automatic daily backups via [scheduled exports](https://firebase.google.com/docs/firestore/manage-data/export-import). For a personal app, manual exports via the Firebase Console are also fine.
