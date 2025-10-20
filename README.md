# AISI Annotate

AISI Annotate is a collaborative document labeling tool built on the Next.js App Router. Documents are loaded on the server, hydrated on the client, and rendered inside a Tiptap-based editor that layers custom annotation marks, Liveblocks-powered collaboration, and contextual metadata around every text selection.【F:src/app/(authed)/document/[documentId]/page.tsx†L1-L45】【F:src/frontend/components/editor/editor-logic.tsx†L1-L122】

## High-level architecture

### Next.js application shell
- App Router routes under `src/app` load data on the server with tRPC before streaming the React tree to the client. The document page fetches the document twice: once through the public router to set SEO metadata and again through the authenticated router to gate editing access before rendering the editor surface.【F:src/app/(authed)/document/[documentId]/page.tsx†L14-L44】
- The `/api/trpc` handler adapts Next.js requests to the tRPC router while attaching a correlation identifier to every request, making it easy to trace logs across the stack.【F:src/app/api/trpc/[trpc]/route.ts†L1-L33】

### Editor experience
- `EditorLogic` composes the Tiptap root, loads all bundled extensions, attaches the Liveblocks collaborative cursor plugin, and hydrates annotation metadata that is fetched via the `document.getAnnotationTypes` query.【F:src/frontend/components/editor/editor-logic.tsx†L1-L119】
- Editor events flow through a Zustand store that holds the active document, selection range, contextual snippets, typing state, and the instantiated editor instance so that UI panels and command palettes can react to text selection changes.【F:src/frontend/stores/editor-store.tsx†L10-L132】

### Annotation engine
- The custom `annotation` mark normalizes all applied highlights: setting a label rewrites the selection to a single contiguous mark, toggling a label removes it from the entire marked region, and clicking within an annotation automatically selects the full range so users do not leave behind partial marks.【F:src/frontend/components/editor/extensions/annotation.tsx†L27-L200】

### Real-time collaboration
- Liveblocks sessions are created server-side. Authenticated users receive named sessions while anonymous visitors fall back to a shared account, and both modes grant full room access before returning the signed authorization response to the client.【F:src/app/api/liveblocks-auth/route.ts†L7-L44】
- The editor registers the Liveblocks Tiptap extension, wiring in awareness state and synchronization status so presence and cursors stay in sync with the shared document.【F:src/frontend/components/editor/editor-logic.tsx†L14-L107】

### Backend API layer
- tRPC routers expose document CRUD, annotation type lookups, and user utilities. Authenticated procedures automatically hydrate the Clerk user into the request context, while public procedures allow anonymous read access where appropriate.【F:src/backend/routers/document.ts†L5-L74】【F:src/backend/routers/public.ts†L5-L26】【F:src/backend/routers/trpc.ts†L1-L95】
- The root router is mounted via the fetch adapter, and `createCaller` lets server components call tRPC procedures without an HTTP round trip.【F:src/backend/routers/root.ts†L1-L13】

### Database schema
- Prisma models describe users and documents, tracking ownership, soft deletion, and timestamps in a PostgreSQL database. The generated client is shared across the API layer so getter and setter modules can compose reusable data access helpers.【F:src/backend/prisma/schema.prisma†L1-L33】【F:src/backend/db/db.ts†L1-L18】

### Configuration
- Runtime configuration is centralized in `src/frontend/config/config.ts` and `src/backend/config.ts`, which derive the base URL, Liveblocks keys, and build metadata from environment variables such as `NEXT_PUBLIC_APP_URL`, `NEXT_PUBLIC_LIVEBLOCKS_PUBLIC_API_KEY`, and `LIVEBLOCKS_SECRET`.【F:src/frontend/config/config.ts†L1-L20】【F:src/backend/config.ts†L1-L7】

## Request & editing lifecycle
1. **Route load** – The `/document/[documentId]` server component resolves the document via the authenticated router. Missing documents trigger a typed `PageError` while successful requests stream the `PageEditor` client component.【F:src/app/(authed)/document/[documentId]/page.tsx†L27-L45】
2. **Editor initialization** – `PageEditor` stores the document in the global editor store and renders `EditorLogic`, which bootstraps Tiptap, loads Liveblocks collaboration, and fetches annotation labels.【F:src/frontend/components/pages/page-editor.tsx†L1-L29】【F:src/frontend/components/editor/editor-logic.tsx†L1-L119】
3. **Title synchronization** – As the user types in the title node, a debounced mutation updates the document title through the `document.updateTitle` tRPC procedure to keep the database in sync.【F:src/frontend/components/editor/editor-logic.tsx†L45-L118】【F:src/backend/routers/document.ts†L55-L69】
4. **Selection enrichment** – Selection updates push the selection range, contextual snippets, and highlighted text into the editor store so inspector panels can display block-level and surrounding context metadata.【F:src/frontend/components/editor/editor-logic.tsx†L67-L95】【F:src/frontend/stores/editor-store.tsx†L10-L97】
5. **Annotation updates** – Commands provided by the custom annotation mark apply, toggle, or remove labeled spans; they ensure that a single click targets the entire annotation and prevent fragmented marks in overlapping regions.【F:src/frontend/components/editor/extensions/annotation.tsx†L65-L200】

## Key directories
- `src/app` – Next.js App Router routes, layouts, and API handlers for tRPC and Liveblocks auth.【F:src/app/(authed)/document/[documentId]/page.tsx†L1-L45】【F:src/app/api/trpc/[trpc]/route.ts†L1-L33】【F:src/app/api/liveblocks-auth/route.ts†L7-L44】
- `src/frontend` – Client-only components, hooks, stores, and shared configuration for the editor experience.【F:src/frontend/components/editor/editor-logic.tsx†L1-L122】【F:src/frontend/stores/editor-store.tsx†L10-L132】【F:src/frontend/config/config.ts†L1-L20】
- `src/backend` – tRPC routers, Prisma schema, data access helpers, and runtime configuration consumed by both server components and API routes.【F:src/backend/routers/document.ts†L5-L74】【F:src/backend/prisma/schema.prisma†L1-L33】【F:src/backend/config.ts†L1-L7】
- `src/shared` – Utilities and cross-cutting types (e.g., ID helpers used to generate correlation IDs for each request).【F:src/app/api/trpc/[trpc]/route.ts†L4-L18】

## Running locally
1. Install Bun (or use a Node.js runtime that can execute the Bun scripts).
2. Install dependencies and generate the Prisma client: `bun install` (runs the `postinstall` hook).
3. Prepare your environment variables (see below) and ensure `DATABASE_URL` points at a PostgreSQL instance.
4. Push the schema: `bun run dbpush` (or `bun run dbreset` for a clean slate).
5. Start the development server: `bun run dev`.
6. Visit `http://localhost:3000` to sign in with Clerk and start annotating.【F:package.json†L5-L16】

### Required environment variables
- `DATABASE_URL` – Prisma database connection string (PostgreSQL).【F:src/backend/prisma/schema.prisma†L6-L9】
- `LIVEBLOCKS_SECRET` and `NEXT_PUBLIC_LIVEBLOCKS_PUBLIC_API_KEY` – Server and client keys for Liveblocks real-time collaboration.【F:src/app/api/liveblocks-auth/route.ts†L7-L43】【F:src/frontend/config/config.ts†L11-L14】【F:src/backend/config.ts†L4-L7】
- `NEXT_PUBLIC_APP_URL` / `NEXT_PUBLIC_VERCEL_PROJECT_PRODUCTION_URL` – Used to construct absolute URLs for API calls in different environments.【F:src/frontend/config/config.ts†L4-L13】
- Clerk environment variables (not shown in the repo) must also be configured so the authentication middleware can resolve the active user before executing private tRPC procedures.【F:src/backend/routers/trpc.ts†L1-L78】

## Useful scripts
- `bun run dev` – Launch the Next.js development server.
- `bun run build` / `bun run start` – Build and run the production bundle.
- `bun run lint` – Execute the Next.js ESLint configuration.
- `bun run dbpush` / `bun run dbgen` / `bun run dbreset` – Manage the Prisma schema and database state.
- `bun run test:open` / `bun run test:a11y` – Run Cypress end-to-end and accessibility test suites.【F:package.json†L5-L16】

## Deployment notes
- The project is optimized for Vercel, and environment-aware configuration automatically derives the public base URL and exposes the current Git SHA for observability when the Vercel-specific variables are present.【F:src/frontend/config/config.ts†L4-L18】【F:src/backend/config.ts†L1-L6】
- Because all tRPC procedures enforce correlation IDs and propagate Clerk context, server-side logging and tracing remain consistent across server components, API routes, and background calls made with `createCaller`.【F:src/app/api/trpc/[trpc]/route.ts†L4-L28】【F:src/backend/routers/trpc.ts†L8-L95】【F:src/backend/routers/root.ts†L1-L13】
