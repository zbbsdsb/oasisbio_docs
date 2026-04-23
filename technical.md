# OasisBio — Technical Documentation

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Technology Stack](#2-technology-stack)
3. [Project Structure](#3-project-structure)
4. [Authentication System](#4-authentication-system)
5. [Database](#5-database)
6. [Storage](#6-storage)
7. [API Reference](#7-api-reference)
8. [World Builder](#8-world-builder)
9. [Domain Events & Audit Logs](#9-domain-events--audit-logs)
10. [Error Handling](#10-error-handling)
11. [Testing](#11-testing)
12. [Deployment](#12-deployment)
13. [Environment Variables](#13-environment-variables)
14. [Planned: Edge Functions](#14-planned-edge-functions)
15. [OAuth Provider](#15-oauth-provider)

---

## 1. Architecture Overview

OasisBio is a Next.js 16 full-stack application deployed on Cloudflare Pages. All data lives in Supabase (PostgreSQL). Authentication is fully passwordless via Supabase OTP. Large binary files (3D models, export ZIPs) are stored in Cloudflare R2.

```
┌─────────────────────────────────────────────────────────┐
│                    Browser / Client                      │
│  React Client Components + @supabase/ssr browser client │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS
┌────────────────────────▼────────────────────────────────┐
│              Cloudflare Pages (Edge)                     │
│  Next.js Middleware — session refresh via updateSession  │
│  Next.js App Router — SSR + API Routes (Node runtime)   │
└──────┬──────────────────────────────────────┬───────────┘
       │ Prisma (pooler:6543)                  │ @supabase/ssr (server)
┌──────▼──────────┐                  ┌─────────▼──────────┐
│  PostgreSQL      │                  │  Supabase Auth     │
│  (Supabase)      │                  │  (JWT + cookies)   │
│  RLS enabled     │                  └────────────────────┘
└─────────────────┘
       │
┌──────▼──────────┐
│  Cloudflare R2  │
│  (large files)  │
└─────────────────┘
```

**Key design decisions:**

- `@supabase/ssr` replaces the deprecated `@supabase/auth-helpers-nextjs`. Sessions are stored in cookies, not localStorage, so server components can read them.
- Prisma connects via the **pooler** (port 6543, pgbouncer) for all runtime queries. The **direct** connection (port 5432) is only used for `prisma db push` / migrations.
- All API routes use `requireAuth()` which returns a Supabase `User` object directly — not a session wrapper.
- Row Level Security (RLS) is enabled on all tables. Prisma bypasses RLS because it connects as the `postgres` superuser via the pooler.

---

## 2. Technology Stack

| Layer | Technology | Version | Notes |
|-------|-----------|---------|-------|
| Framework | Next.js | 16.2.2 | App Router, SSR + API Routes |
| Language | TypeScript | 5.4.3 | Strict mode |
| Styling | Tailwind CSS | 3.4.3 | Utility-first |
| Database | PostgreSQL | — | Hosted on Supabase |
| ORM | Prisma | 6.19.1 | Custom output path `src/generated/prisma` |
| Auth | Supabase Auth | — | OTP (passwordless) |
| Auth SSR | @supabase/ssr | latest | Cookie-based sessions |
| Storage (images) | Supabase Storage | — | Public buckets |
| Storage (large) | Cloudflare R2 | — | Private, signed URLs |
| 3D Rendering | Three.js | 0.183.2 | GLB via GLTFLoader |
| Property Testing | fast-check | latest | 200 iterations/property |
| Test Runner | Jest | latest | jsdom + node environments |
| Deployment | Cloudflare Pages | — | Build: `npm run build` |

---

## 3. Project Structure

```
oasisbio/
├── prisma/
│   ├── schema.prisma          # All table definitions + relations
│   └── seed.ts                # Optional seed data
│
├── scripts/
│   └── db/
│       ├── 01_enable_rls.sql          # RLS policies for all tables
│       ├── 02_add_indexes.sql         # Performance indexes
│       ├── 03_service_role_bypass.sql # Documentation only
│       ├── 04_storage_policies.sql    # Storage bucket policies
│       ├── 05_domain_events_audit_logs.sql  # Event/audit tables
│       ├── 06_publish_bio_rpc.sql     # Publish RPC functions
│       └── 07_oauth_tables.sql        # OAuth apps, codes, tokens + RLS
│
├── src/
│   ├── app/
│   │   ├── api/
│   │   │   ├── abilities/[id]/route.ts
│   │   │   ├── abilities/route.ts
│   │   │   ├── auth/
│   │   │   │   ├── register/route.ts      # Deprecated (returns 410)
│   │   │   │   └── supabase-webhook/route.ts
│   │   │   ├── dashboard/route.ts
│   │   │   ├── dcos/[id]/route.ts
│   │   │   ├── dcos/route.ts
│   │   │   ├── developer/
│   │   │   │   └── apps/
│   │   │   │       ├── [id]/
│   │   │   │       │   ├── route.ts       # GET/PUT/DELETE app
│   │   │   │       │   └── secret/route.ts # POST rotate secret
│   │   │   │       └── route.ts           # GET list / POST create
│   │   │   ├── eras/[id]/route.ts         # PUT/DELETE era by id
│   │   │   ├── export/route.ts
│   │   │   ├── import/route.ts
│   │   │   ├── models/[id]/route.ts
│   │   │   ├── models/route.ts
│   │   │   ├── oauth/
│   │   │   │   ├── .well-known/openid-configuration/route.ts  # OIDC discovery
│   │   │   │   ├── resources/
│   │   │   │   │   └── oasisbios/
│   │   │   │   │       ├── [id]/
│   │   │   │   │       │   ├── dcos/route.ts   # dcos:read scope
│   │   │   │   │       │   └── route.ts        # oasisbios:full scope
│   │   │   │   │       └── route.ts            # oasisbios:read scope
│   │   │   │   ├── revoke/route.ts             # POST revoke token
│   │   │   │   ├── token/route.ts              # POST exchange code / refresh
│   │   │   │   └── userinfo/route.ts           # GET profile + email
│   │   │   ├── oasisbios/
│   │   │   │   ├── [id]/
│   │   │   │   │   ├── abilities/route.ts  # GET list / POST create
│   │   │   │   │   ├── dcos/route.ts       # GET list / POST create
│   │   │   │   │   ├── eras/route.ts       # GET list / POST create (auto sortOrder)
│   │   │   │   │   ├── publish/route.ts    # POST=publish, DELETE=unpublish
│   │   │   │   │   ├── references/route.ts # GET list / POST create
│   │   │   │   │   ├── route.ts            # GET/PUT/DELETE (summary field included)
│   │   │   │   │   └── worlds/route.ts
│   │   │   │   ├── public/route.ts
│   │   │   │   └── route.ts               # GET list / POST create (slug auto-dedup)
│   │   │   ├── profile/route.ts
│   │   │   ├── references/[id]/route.ts
│   │   │   ├── references/route.ts
│   │   │   ├── settings/route.ts
│   │   │   ├── worlddocuments/[id]/route.ts
│   │   │   └── worlds/
│   │   │       ├── [id]/
│   │   │       │   ├── documents/route.ts
│   │   │       │   └── route.ts           # GET/PUT/DELETE (all fields)
│   │   │       └── route.ts
│   │   ├── auth/
│   │   │   ├── login/page.tsx             # OTP login
│   │   │   └── register/page.tsx          # OTP register + displayName
│   │   ├── bio/
│   │   │   └── [slug]/page.tsx            # Public character page (visibility=public only, SEO metadata)
│   │   ├── dashboard/
│   │   │   └── oasisbios/[id]/
│   │   │       ├── page.tsx               # Identity editor + publish/unpublish
│   │   │       ├── eras/page.tsx          # Era timeline management
│   │   │       ├── abilities/page.tsx     # Ability pool management
│   │   │       ├── worlds/
│   │   │       │   ├── page.tsx           # World list (card grid)
│   │   │       │   ├── new/page.tsx       # Step wizard
│   │   │       │   └── [wid]/page.tsx     # World detail editor
│   │   │       ├── dcos/page.tsx          # DCOS file management
│   │   │       └── references/page.tsx    # References library
│   │   └── layout.tsx
│   │
│   ├── components/
│   │   ├── auth/
│   │   │   ├── AuthButton.tsx
│   │   │   ├── AuthForm.tsx
│   │   │   ├── AuthInput.tsx
│   │   │   └── OAuthButtons.tsx
│   │   └── world/
│   │       ├── CharacterSection.tsx       # Linked characters display
│   │       ├── CreateWorldCard.tsx        # "+" card in grid
│   │       ├── ModuleSection.tsx          # Collapsible edit section
│   │       ├── StepWizard.tsx             # 6-step creation wizard
│   │       ├── WizardProgress.tsx         # Step progress bar
│   │       ├── WizardStep.tsx             # Single step renderer
│   │       └── WorldCard.tsx              # World card in grid
│   │
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts      # createBrowserClient (Client Components)
│   │   │   ├── middleware.ts  # updateSession proxy (Middleware)
│   │   │   ├── server.ts      # createServerClient (Server/API)
│   │   │   └── storage.ts     # Upload/delete/URL helpers
│   │   ├── oauth/
│   │   │   ├── crypto.ts      # generateSecret, hashClientSecret, verifyPKCE, signAccessToken
│   │   │   ├── middleware.ts  # requireOAuthToken (Bearer token validation)
│   │   │   ├── scopes.ts      # SCOPES constant, parseScopes, hasScope
│   │   │   └── validate.ts    # validateRedirectUri, validateAuthorizationParams
│   │   ├── auth.ts            # getServerUser, getServerUserWithProfile
│   │   ├── auth-utils.ts      # requireAuth, ownership checks, handleApiError
│   │   ├── auth.client.ts     # SessionProvider, useAuth, useSession (compat shim)
│   │   ├── prisma.ts          # Prisma client singleton
│   │   ├── storage.ts         # Storage abstraction (Supabase + R2)
│   │   ├── user-sync.ts       # syncUserToPrisma, generateUniqueUsername
│   │   └── world-utils.ts     # calculateCompletionScore, validateWorldForm, etc.
│   │
│   ├── types/
│   │   └── world.ts           # WorldFormData, WorldStepConfig, WORLD_STEPS
│   │
│   └── middleware.ts          # Route protection + session refresh
│
└── docs/
    ├── README.md
    ├── technical.md           # This file
    ├── design.md
    └── world-design-spec.md
```

---

## 4. Authentication System

### Overview

Authentication is fully passwordless. Users receive a 6-digit OTP via email. No passwords are stored anywhere.

### Supabase Client Hierarchy

#### `src/lib/supabase/client.ts` — Browser Client

Used in all Client Components (`'use client'`).

```typescript
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

- Stores sessions in **cookies** (not localStorage)
- Automatically refreshes tokens
- Safe to call multiple times — each call creates a new instance

#### `src/lib/supabase/server.ts` — Server Client

Used in Server Components, Server Actions, and API Route Handlers.

```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(url, anonKey, {
    cookies: {
      getAll() { return cookieStore.getAll(); },
      setAll(cookiesToSet) {
        try { cookiesToSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options)); }
        catch { /* Server Component — middleware handles refresh */ }
      },
    },
  });
}
```

- Reads session from request cookies
- `setAll` errors are silently ignored in Server Components (middleware handles token refresh)

#### `src/lib/supabase/middleware.ts` — Session Proxy

Used exclusively in `src/middleware.ts`.

```typescript
export async function updateSession(request: NextRequest) {
  // Creates server client that reads/writes request+response cookies
  // CRITICAL: getClaims() must be called immediately after createServerClient
  const { data } = await supabase.auth.getClaims();
  const user = data?.claims ?? null;
  return { supabaseResponse, user };
}
```

**Critical rule:** No code between `createServerClient` and `getClaims()`. Violating this causes random logouts.

### Auth Flow — Login

```
1. User visits /auth/login
2. Enters email → supabase.auth.signInWithOtp({ email })
3. Supabase sends 6-digit code to email
4. User enters code → supabase.auth.verifyOtp({ email, token, type: 'email' })
5. On success → router.replace(callbackUrl || '/dashboard')
6. Session stored in cookies automatically by @supabase/ssr
```

### Auth Flow — Registration

```
1. User visits /auth/register
2. Enters displayName + email
3. supabase.auth.signInWithOtp({ email, options: { data: { display_name } } })
4. User enters OTP code → verifyOtp
5. On success → router.replace('/dashboard')
6. syncUserToPrisma runs on first API request (fallback sync)
```

### Server-Side Auth Utilities

#### `src/lib/auth.ts`

```typescript
// Returns Supabase User or null — never throws
getServerUser(): Promise<SupabaseUser | null>

// Returns user + triggers Prisma sync (non-fatal if sync fails)
getServerUserWithProfile(): Promise<{ supabaseUser, userId, profileId, username, isNewUser } | null>
```

#### `src/lib/auth-utils.ts`

```typescript
// Throws AuthError(401) if not authenticated. Returns User directly.
requireAuth(): Promise<SupabaseUser>

// Throws AuthError(404) if not found, AuthError(403) if not owner
requireOasisBioOwnership(oasisBioId, userId): Promise<{ id, userId }>
requireDcosFileOwnership(dcosFileId, userId): Promise<DcosFile>
requireAbilityOwnership(abilityId, userId): Promise<Ability>
requireWorldOwnership(worldId, userId): Promise<WorldItem>
requireReferenceOwnership(referenceId, userId): Promise<ReferenceItem>
requireWorldDocumentOwnership(documentId, userId): Promise<WorldDocument>

// Converts AuthError or unknown errors to structured NextResponse
handleApiError(error: unknown): NextResponse
```

**AuthError class:**
```typescript
class AuthError extends Error {
  statusCode: number;  // 401, 403, 404
  code: string;        // 'UNAUTHORIZED', 'FORBIDDEN', 'NOT_FOUND'
}
```

### User Sync — `src/lib/user-sync.ts`

Syncs Supabase Auth users to the Prisma `users` + `profiles` tables.

```typescript
interface SyncResult {
  userId: string;
  profileId: string;
  username: string;
  isNewUser: boolean;
}

syncUserToPrisma(supabaseUser: SupabaseUser): Promise<SyncResult>
generateUniqueUsername(base: string): Promise<string>
```

**Sync rules:**
1. `User` record: upsert by `id`. Only updates `name` if currently null.
2. `Profile` record: create if missing. If exists, only fills empty fields — never overwrites `displayName`, `bio`, `website`.
3. Username generation: lowercase + alphanumeric only. Appends `1`, `2`, `3`... on collision.
4. Fallback: called by `getServerUserWithProfile()` on every authenticated request. Sync failure is non-fatal (logged, user still proceeds).

### Client-Side Auth Context — `src/lib/auth.client.ts`

```typescript
// Provider — wrap app root (already in SessionProviderWrapper)
<SessionProvider>

// Primary hook — use this in all new code
useAuth(): { user: User | null, session: Session | null, isLoading: boolean, supabase: SupabaseClient }

// Compatibility shim — mirrors NextAuth shape, kept for gradual migration
useSession(): { data: { user, session } | null, status: 'loading' | 'authenticated' | 'unauthenticated' }

// Standalone sign-out helper
signOut(): Promise<{ error }>
```

**Usage pattern in Client Components:**

```typescript
import { useAuth } from '@/lib/auth.client';

const { user, isLoading, supabase } = useAuth();

// Route protection
if (isLoading) return <Spinner />;
if (!user) return null; // middleware handles redirect

// Sign out
await supabase.auth.signOut();
```

`SessionProvider` uses `supabase.auth.onAuthStateChange` to keep context in sync with all auth events (sign-in, sign-out, token refresh).

> **Note:** `useSession` is a compatibility shim. All pages have been migrated to `useAuth`. New code should always use `useAuth`.

### Middleware — `src/middleware.ts`

Runs on every request matching the `config.matcher` pattern (all paths except static assets).

**Protected prefixes** (unauthenticated → redirect to `/auth/login?callbackUrl=<path>`):
- `/dashboard`
- `/api/oasisbios`
- `/api/worlds`

**Auth prefixes** (authenticated → redirect to `/dashboard`):
- `/auth/login`
- `/auth/register`

The middleware always returns `supabaseResponse` (never a new `NextResponse`) to preserve refreshed session cookies.

### Webhook — `src/app/api/auth/supabase-webhook/route.ts`

Receives Supabase Auth events and syncs to Prisma.

- Verifies `x-webhook-signature` header using HMAC-SHA256 + `SUPABASE_WEBHOOK_SECRET`
- `user.created` / `user.updated` → calls `syncUserToPrisma`
- `user.deleted` → deletes `User` record (cascades to all related data)
- Missing `SUPABASE_WEBHOOK_SECRET` → logs warning, skips verification (dev mode)

---

## 5. Database

### Connection

| Variable | Port | Purpose |
|----------|------|---------|
| `DATABASE_URL` | 6543 | Prisma runtime queries (pgbouncer pooler) |
| `DIRECT_URL` | 5432 | `prisma db push` / migrations only |

Prisma config in `prisma.config.ts` loads these from environment.

### Schema Summary

All tables use `cuid()` for IDs except `users` (uses Supabase Auth UUID) and the new event tables (use `gen_random_uuid()`).

#### `users`
Mirror of Supabase Auth users. Created/updated by `syncUserToPrisma`.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (UUID) | Matches Supabase Auth `user.id` |
| `name` | String? | Display name |
| `email` | String (unique) | |
| `email_verified` | DateTime? | |
| `image` | String? | Avatar URL |
| `created_at` | DateTime | |
| `updated_at` | DateTime | Auto-updated |

Indexes: `email`, `created_at`

#### `profiles`
Public-facing user profile. One per user.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `user_id` | String | FK → users |
| `username` | String (unique) | Lowercase alphanumeric |
| `display_name` | String | User-editable |
| `avatar_url` | String? | |
| `bio` | String? | User-editable |
| `website` | String? | User-editable |
| `locale` | String | Default: `en` |
| `default_language` | String | Default: `en` |
| `created_at` | DateTime | |
| `updated_at` | DateTime | |

Indexes: `user_id`, `username`, `created_at`, `lower(username)` (case-insensitive search)

#### `oasis_bios`
Core character profile entity.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `user_id` | String | FK → users |
| `title` | String | Character name |
| `slug` | String (unique) | URL-safe identifier |
| `tagline` | String? | One-line description |
| `summary` | String? | |
| `identity_mode` | String | `real` \| `fictional` \| `hybrid` \| `future` \| `alternate` |
| `birth_date` | DateTime? | |
| `gender` | String? | |
| `pronouns` | String? | |
| `origin_place` | String? | |
| `current_era` | String? | |
| `species` | String? | |
| `status` | String | `draft` \| `active` |
| `description` | String? | |
| `cover_image_url` | String? | |
| `default_language` | String | Default: `en` |
| `visibility` | String | `private` \| `public` |
| `featured` | Boolean | Default: false |
| `published_at` | DateTime? | Set by `publish_bio()` RPC |
| `created_at` | DateTime | |
| `updated_at` | DateTime | |

Indexes: `user_id`, `slug`, `visibility`, `featured`, `created_at`, `updated_at`, `published_at`

#### `era_identities`
Time-period versions of a character.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `oasis_bio_id` | String | FK → oasis_bios |
| `name` | String | Era name |
| `era_type` | String | e.g. `past`, `present`, `future` |
| `start_year` | Int? | |
| `end_year` | Int? | |
| `description` | String? | |
| `sort_order` | Int | Default: 0 |

Indexes: `oasis_bio_id`, `sort_order`

#### `abilities`
Skills with category, level, and optional era/world binding.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `oasis_bio_id` | String | FK → oasis_bios |
| `name` | String | |
| `category` | String | |
| `source_type` | String | `custom` \| `official` |
| `level` | Int | 1–5, default: 1 |
| `description` | String? | |
| `related_world_id` | String? | FK → world_items |
| `related_era_id` | String? | FK → era_identities |

Indexes: `oasis_bio_id`, `category`, `level`, `related_world_id`, `related_era_id`

#### `dcos_files`
Character narrative documents (Dynamic Core Operating Script).

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `oasis_bio_id` | String | FK → oasis_bios |
| `title` | String | |
| `slug` | String (unique) | |
| `content` | String | Markdown content |
| `folder_path` | String | Virtual folder, e.g. `/chapter-1/` |
| `status` | String | `draft` \| `published` |
| `version` | Int | Default: 1 |
| `era_id` | String? | FK → era_identities |
| `created_at` | DateTime | |
| `updated_at` | DateTime | |

Indexes: `oasis_bio_id`, `slug`, `status`, `era_id`, `created_at`, `updated_at`, `(oasis_bio_id, status)`

#### `reference_items`
External links and resources.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `oasis_bio_id` | String | FK → oasis_bios |
| `url` | String | |
| `title` | String | |
| `description` | String? | |
| `source_type` | String | `article` \| `video` \| `website` \| etc. |
| `provider` | String? | e.g. `YouTube`, `Wikipedia` |
| `cover_image` | String? | |
| `metadata` | String? | JSON string |
| `era_id` | String? | FK → era_identities |
| `world_id` | String? | FK → world_items |
| `tags` | String | Comma-separated |

Indexes: `oasis_bio_id`, `source_type`, `era_id`, `world_id`, `tags`

#### `world_items`
Fictional world settings. Belongs to one OasisBio.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `oasis_bio_id` | String | FK → oasis_bios |
| `name` | String | Required |
| `summary` | String | Required |
| `time_setting` | String? | Era name / time period |
| `geography` | String? | Landscape, cities, landmarks |
| `physics_rules` | String? | Special laws of the world |
| `social_structure` | String? | Governance + society |
| `aesthetic_keywords` | String? | JSON: `{ genre, tone }` |
| `major_conflict` | String? | Conflict + themes + story hooks |
| `visibility` | String | `private` \| `public` |
| `timeline` | String? | Key historical events |
| `rules` | String? | Power system + limitations |
| `factions` | String? | Major groups |

Indexes: `oasis_bio_id`, `visibility`, `(oasis_bio_id, updated_at DESC)`

**Note:** `aesthetic_keywords` stores genre and tone as JSON: `{"genre":"sci-fi","tone":"dark"}`. Use `serializeGenreTone` / `deserializeGenreTone` from `src/lib/world-utils.ts`.

#### `world_documents`
Detailed content documents within a world.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `world_id` | String | FK → world_items |
| `title` | String | |
| `doc_type` | String | e.g. `lore`, `map`, `history` |
| `slug` | String | |
| `content` | String | Markdown |
| `folder_path` | String | Default: `/` |
| `sort_order` | Int | Default: 0 |
| `created_at` | DateTime | |
| `updated_at` | DateTime | |

Indexes: `world_id`, `doc_type`, `slug`, `folder_path`, `sort_order`

#### `model_items`
3D model references (files stored in Cloudflare R2).

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `oasis_bio_id` | String | FK → oasis_bios |
| `model_name` | String | |
| `file_path` | String | R2 key |
| `model_format` | String | Default: `glb` |
| `preview_image` | String? | Supabase Storage URL |
| `related_world_id` | String? | |
| `related_era_id` | String? | |
| `is_primary` | Boolean | Default: false |
| `version` | Int | Default: 1 |
| `created_at` | DateTime | |
| `updated_at` | DateTime | |

Indexes: `oasis_bio_id`, `is_primary`, `model_format`, `created_at`

#### `oasisbio_publications`
Published character metadata. One-to-one with `oasis_bios`.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `oasis_bio_id` | String (unique) | FK → oasis_bios |
| `public_slug` | String (unique) | URL slug for public page |
| `seo_title` | String? | |
| `seo_description` | String? | |
| `social_image_url` | String? | OG image |
| `custom_domain` | String? | Future use |
| `is_searchable` | Boolean | Default: true |
| `published_at` | DateTime? | |
| `updated_at` | DateTime | |

#### `character_relationships`
Character-to-character links.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `character_a_id` | String | FK → oasis_bios |
| `character_b_id` | String | FK → oasis_bios |
| `relation_type` | String | e.g. `ally`, `rival`, `family` |
| `description` | String? | |
| `created_at` | DateTime | |

#### `export_history`
Tracks recent export operations.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `user_id` | String | FK → users |
| `file_name` | String | |
| `file_size` | Int | Bytes |
| `character_count` | Int | |
| `created_at` | DateTime | |

Indexes: `user_id`, `created_at`, `(user_id, created_at DESC)`

#### `domain_events`
Async event queue for decoupled side effects.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | `gen_random_uuid()` |
| `event_type` | text | `bio.published`, `bio.unpublished`, `asset.uploaded` |
| `aggregate_type` | text | `oasis_bio`, `world_item` |
| `aggregate_id` | text | Entity ID |
| `payload` | jsonb | Event-specific data |
| `status` | text | `pending` \| `processing` \| `done` \| `failed` |
| `retry_count` | int | Default: 0 |
| `created_at` | timestamptz | |
| `processed_at` | timestamptz? | |

Indexes: `(status, created_at) WHERE status IN ('pending','failed')`, `(aggregate_type, aggregate_id, created_at DESC)`

#### `audit_logs`
Immutable operation audit trail.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | `gen_random_uuid()` |
| `actor_user_id` | text? | User who performed the action |
| `action` | text | `bio.publish`, `bio.unpublish`, `asset.token.request` |
| `target_type` | text | `oasis_bio`, `world_item` |
| `target_id` | text | |
| `request_id` | text? | For request tracing |
| `metadata` | jsonb | Action-specific details |
| `created_at` | timestamptz | |

Indexes: `(actor_user_id, created_at DESC)`, `(target_type, target_id, created_at DESC)`, `(action, created_at DESC)`

### Row Level Security (RLS)

All tables have RLS enabled. Prisma bypasses RLS because it connects as the `postgres` superuser via the pooler connection string.

RLS policies protect direct Supabase JS client access (e.g., future Edge Functions, client-side queries).

| Table | Policy |
|-------|--------|
| `profiles` | Owner full access + public SELECT |
| `oasis_bios` | Owner full access + public SELECT where `visibility='public'` |
| `era_identities` | Owner full access (via oasis_bio join) |
| `abilities` | Owner full access (via oasis_bio join) |
| `dcos_files` | Owner full access (via oasis_bio join) |
| `reference_items` | Owner full access (via oasis_bio join) |
| `world_items` | Owner full access + public SELECT where `visibility='public'` |
| `world_documents` | Owner full access (via world → oasis_bio join) |
| `model_items` | Owner full access (via oasis_bio join) |
| `oasisbio_publications` | Owner full access + public SELECT where `published_at IS NOT NULL` |
| `character_relationships` | Owner full access (via character_a join) |
| `export_history` | Owner full access |
| `ability_categories` | Public SELECT only |
| `ability_presets` | Public SELECT only |
| `domain_events` | No access (service role only) |
| `audit_logs` | Owner SELECT only; INSERT blocked for all (service role only) |

### Database RPC Functions

Defined in `scripts/db/06_publish_bio_rpc.sql`. All use `SECURITY DEFINER` to bypass RLS.

#### `validate_publishable_bio(p_bio_id text, p_actor_id text) → jsonb`

Read-only validation. Returns:
```json
{ "ok": true, "errors": [] }
// or
{ "ok": false, "errors": ["Title is required", "Summary is required"] }
```

Checks:
1. Bio exists
2. `actor_id` matches `user_id`
3. `title` is non-empty
4. `summary` is non-empty

#### `publish_bio(p_bio_id text, p_actor_id text, p_request_id text, p_visibility text) → jsonb`

Atomic publish transaction. Returns:
```json
{ "ok": true, "slug": "my-character", "published_at": "2026-04-10T...", "visibility": "public" }
// or
{ "ok": false, "errors": ["You do not own this bio"] }
```

Steps (all in one transaction):
1. Calls `validate_publishable_bio` — aborts if invalid
2. Updates `oasis_bios`: sets `visibility`, `status='active'`, `published_at`
3. Upserts `oasisbio_publications`
4. Inserts into `audit_logs`
5. Inserts `bio.published` event into `domain_events`

#### `unpublish_bio(p_bio_id text, p_actor_id text, p_request_id text) → jsonb`

Reverses publish:
1. Sets `oasis_bios.visibility='private'`, `status='draft'`
2. Sets `oasisbio_publications.is_searchable=false`
3. Inserts into `audit_logs`
4. Inserts `bio.unpublished` event into `domain_events`

---

## 6. Storage

### Supabase Storage (images)

All image buckets are **public** — files are accessible via public URL without authentication. Write access is restricted by RLS policies to the path owner.

#### Bucket: `avatars`

- **Path format:** `{userId}/avatar.{ext}`
- **Allowed types:** `image/webp`, `image/png`, `image/jpeg`
- **Max size:** 512 KB
- **Get URL:** `storagePath.avatar.getUrl(userId)`
- **Get path:** `storagePath.avatar.getPath(userId, ext?)`

#### Bucket: `character-covers`

- **Path format:** `{userId}/{characterId}/cover.{ext}`
- **Allowed types:** `image/webp`, `image/png`, `image/jpeg`
- **Max size:** 800 KB
- **Get URL:** `storagePath.characterCover.getUrl(userId, characterId)`

#### Bucket: `model-previews`

- **Path format:** `{userId}/{characterId}/preview.{ext}`
- **Allowed types:** `image/webp`, `image/png`, `image/jpeg`
- **Max size:** 600 KB
- **Get URL:** `storagePath.modelPreview.getUrl(userId, characterId)`

#### Storage Utilities (`src/lib/supabase/storage.ts`)

```typescript
// Upload a file to a bucket
uploadFile(bucket, path, file, options?): Promise<{ path }>

// Delete a file from a bucket
deleteStorageFile(bucket, path): Promise<true>

// Validate file before upload
validateFile(file, { maxSize, allowedTypes }): { valid, error? }

// Constants
STORAGE_BUCKETS = { AVATARS, CHARACTER_COVERS, MODEL_PREVIEWS }
```

### Cloudflare R2 (large files)

R2 stores files that are too large for Supabase Storage or require private access.

| Path Pattern | Content | Max Size |
|-------------|---------|---------|
| `models/{userId}/{charId}/model.glb` | 3D models | 12 MB |
| `exports/{userId}/{timestamp}/{name}.zip` | Export archives | 20 MB |
| `textures/{userId}/{charId}/{name}.{ext}` | Textures | 5 MB |

Access is via **signed URLs** (1 hour expiry by default). Files are never publicly accessible without a valid signed URL.

R2 client: `src/lib/cloudflare-r2.ts`

### Storage Abstraction Layer (`src/lib/storage.ts`)

Provides a unified interface that routes to Supabase Storage or R2 based on file type:

```typescript
storage.upload(file, { type, userId, characterId?, ... })
// type: 'avatar' | 'character-cover' | 'model-preview' → Supabase
// type: 'model' | 'export' | 'texture' → R2

storage.delete(options)
storage.getUrl(options): Promise<string>
```

---

## 7. API Reference

### Authentication & Error Format

All authenticated endpoints call `requireAuth()` which returns a Supabase `User` object directly.

**Error response format (all endpoints):**
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable description"
  }
}
```

**Error codes:**
| Code | HTTP Status | Meaning |
|------|------------|---------|
| `UNAUTHORIZED` | 401 | No valid session |
| `FORBIDDEN` | 403 | Valid session but not owner |
| `NOT_FOUND` | 404 | Resource doesn't exist |
| `VALIDATION_ERROR` | 400 | Invalid input |
| `INTERNAL_ERROR` | 500 | Unexpected server error |
| `DEPRECATED` | 410 | Endpoint no longer available |

### Auth Endpoints

#### `POST /api/auth/supabase-webhook`
Receives Supabase Auth events. Requires `x-webhook-signature` header.

**Events handled:**
- `user.created` → `syncUserToPrisma`
- `user.updated` → `syncUserToPrisma`
- `user.deleted` → delete User (cascades all data)

**Response:** `{ received: true, requestId: "uuid" }`

#### `POST /api/auth/register` — DEPRECATED
Returns `410 Gone`. Registration is handled client-side via OTP.

### User Endpoints

#### `GET /api/profile`
Returns current user's profile.

**Response:**
```json
{
  "user": { "id", "name", "email" },
  "profile": { "id", "username", "displayName", "avatarUrl", "bio", "website", "locale", "defaultLanguage" }
}
```

#### `PUT /api/profile`
Updates profile fields.

**Body:** `{ username?, displayName?, avatarUrl?, bio?, website?, locale?, defaultLanguage? }`

**Validation:** If `username` changes, checks uniqueness.

#### `GET /api/settings`
Returns account settings + stats.

**Response:**
```json
{
  "user": { "id", "name", "email", "createdAt" },
  "profile": { ... },
  "stats": { "totalOasisBios", "publicOasisBios" },
  "plan": { "name": "Free", "storageLimit": 128, "storageUsed": 0 }
}
```

#### `PUT /api/settings`
Updates settings by section.

**Body:** `{ section: 'account' | 'profile' | 'security', data: { ... } }`

Note: `section: 'security'` returns 400 — password changes must go through Supabase Auth.

#### `GET /api/dashboard`
Returns dashboard stats and recent activity.

**Response:**
```json
{
  "stats": { "oasisBios", "abilities", "worlds", "models", "references", "dcosFiles", "eras" },
  "recentActivities": [{ "id", "type", "title", "slug", "timestamp", "stats" }]
}
```

### OasisBio Endpoints

#### `GET /api/oasisbios`
Returns all OasisBios for the authenticated user, ordered by `createdAt DESC`.

#### `POST /api/oasisbios`
Creates a new OasisBio.

**Required body fields:** `title`

**Optional:** `tagline`, `identityMode`, `birthDate`, `gender`, `pronouns`, `placeOfOrigin`, `currentEra`, `species`, `status`, `description`

**Auto-generated:** `slug` (from title, lowercase + hyphens, auto-deduped with numeric suffix on collision), `visibility: 'private'`

#### `GET /api/oasisbios/[id]`
Returns OasisBio with all relations: `abilities`, `eras`, `dcosFiles`, `references`, `worlds`, `models`.

#### `PUT /api/oasisbios/[id]`
Updates OasisBio. Only provided fields are updated.

**Updatable:** `title`, `tagline`, `summary`, `identityMode`, `birthDate`, `gender`, `pronouns`, `placeOfOrigin`, `currentEra`, `species`, `status`, `description`, `visibility`

Empty strings are coerced to `null` for optional fields.

#### `DELETE /api/oasisbios/[id]`
Deletes OasisBio and all cascaded data (abilities, eras, dcos, references, worlds, models, publication).

#### `POST /api/oasisbios/[id]/publish`
Publishes a character via the `publish_bio` database RPC.

**Body:** `{ visibility?: 'public' | 'private', requestId?: string }`

**Response:**
```json
{ "ok": true, "slug": "my-character", "publishedAt": "2026-04-10T...", "visibility": "public" }
```

**Errors:** `PUBLISH_FAILED` (400) if validation fails, `RPC_ERROR` (500) if database error.

#### `DELETE /api/oasisbios/[id]/publish`
Unpublishes a character via the `unpublish_bio` RPC.

**Response:** `{ "ok": true }`

#### `GET /api/oasisbios/public`
Returns all public OasisBios (no auth required).

**Response:** Array of `{ id, title, slug, tagline, identityMode, currentEra, coverImageUrl, _count }`

### Ability Endpoints

#### `GET /api/oasisbios/[id]/abilities`
Returns all abilities for an OasisBio, ordered by `id DESC`.

#### `POST /api/oasisbios/[id]/abilities`
Creates an ability.

**Required:** `name`, `category`
**Optional:** `level` (1–5, default 1), `description`, `relatedWorldId`, `relatedEraId`

#### `PUT /api/abilities/[id]`
Updates an ability. Only provided fields are updated.

#### `DELETE /api/abilities/[id]`
Deletes an ability.

### DCOS Endpoints

#### `GET /api/oasisbios/[id]/dcos`
Returns all DCOS files, ordered by `createdAt DESC`.

#### `POST /api/oasisbios/[id]/dcos`
Creates a DCOS file.

**Required:** `title`, `content`
**Optional:** `slug` (auto-generated from title), `folderPath` (default `/`), `status` (default `draft`), `eraId`

#### `PUT /api/dcos/[id]`
Updates a DCOS file. Note: the update handler uses `name`/`content`/`type` field names (legacy — maps to `title`/`content`).

#### `DELETE /api/dcos/[id]`
Deletes a DCOS file.

### Reference Endpoints

#### `GET /api/oasisbios/[id]/references`
Returns all references, ordered by `title ASC`.

#### `POST /api/oasisbios/[id]/references`
Creates a reference.

**Required:** `url`, `title`
**Optional:** `description`, `sourceType` (default `article`), `provider`, `coverImage`, `metadata`, `eraId`, `worldId`, `tags`

#### `PUT /api/references/[id]`
Updates a reference.

#### `DELETE /api/references/[id]`
Deletes a reference.

### Era Endpoints

#### `GET /api/oasisbios/[id]/eras`
Returns all eras for an OasisBio, ordered by `sortOrder ASC`.

#### `POST /api/oasisbios/[id]/eras`
Creates an era. `sortOrder` is auto-assigned (current max + 1).

**Required:** `name`, `eraType` (`past` | `present` | `future` | `alternate` | `worldbound`)
**Optional:** `description`, `startYear`, `endYear`

#### `PUT /api/eras/[id]`
Updates an era. Inline ownership check via `oasisBio.userId`.

**Optional fields:** `name`, `eraType`, `description`, `startYear`, `endYear`

#### `DELETE /api/eras/[id]`
Deletes an era. Cascades: unlinks abilities and DCOS files that referenced this era.

### World Endpoints

#### `GET /api/oasisbios/[id]/worlds`
Returns all worlds for an OasisBio, ordered by `name ASC`.

#### `POST /api/oasisbios/[id]/worlds`
Creates a world.

**Required:** `name`, `summary`
**Optional:** All other WorldItem fields. `genre` + `tone` are serialized into `aestheticKeywords` as JSON.

#### `GET /api/worlds/[id]`
Returns a world with its linked OasisBio (character info).

**Response includes:** `oasisBio: { id, userId, title, coverImageUrl, slug }`

#### `PUT /api/worlds/[id]`
Updates a world. Supports all fields including `genre` and `tone` (merged into `aestheticKeywords`).

#### `DELETE /api/worlds/[id]`
Deletes a world and all its documents.

#### `GET /api/worlds/[id]/documents`
Returns all world documents, ordered by `sortOrder ASC`.

#### `POST /api/worlds/[id]/documents`
Creates a world document.

**Required:** `title`, `content`, `docType`
**Optional:** `slug` (auto-generated), `folderPath` (default `/`), `sortOrder` (default 0)

#### `PUT /api/worlddocuments/[id]`
Updates a world document.

#### `DELETE /api/worlddocuments/[id]`
Deletes a world document.

### Model Endpoints

#### `GET /api/models`
Returns all models for the authenticated user (across all characters).

#### `POST /api/models`
Creates a model record.

**Required:** `name`, `filePath`
**Optional:** `previewImage`, `oasisBioId` (auto-selects first if not provided), `relatedWorldId`, `relatedEraId`, `modelFormat` (default `glb`)

#### `GET /api/models/[id]`
Returns a model with its OasisBio.

#### `PUT /api/models/[id]`
Updates a model.

#### `DELETE /api/models/[id]`
Deletes a model record (does not delete the R2 file).

### Import / Export Endpoints

#### `POST /api/export`
Exports characters as a ZIP file.

**Body:**
```json
{
  "type": "single" | "batch",
  "characterIds": ["id1", "id2"],
  "include": {
    "character": true,
    "dcos": true,
    "references": true,
    "world": true,
    "model": true,
    "cover": true,
    "preview": true
  }
}
```

**Ownership check:** All `characterIds` must belong to the authenticated user.

#### `GET /api/export`
Returns export history for the authenticated user.

#### `POST /api/import`
Imports characters from a ZIP file.

**Body:** `multipart/form-data` with `file` field (must be `application/zip`).

All imported records are scoped to the authenticated user's ID.

---

## 8. World Builder

The world builder is a complete rewrite of the original flat-form world creation UI. It lives at `src/app/dashboard/oasisbios/[id]/worlds/`.

### Routes

| Route | Component | Description |
|-------|-----------|-------------|
| `/worlds` | `WorldListPage` | Card grid of all worlds |
| `/worlds/new` | `StepWizard` | 6-step guided creation |
| `/worlds/[wid]` | `WorldDetailPage` | Detail view + inline editing |

### World List Page

Displays worlds as a card grid (3 columns desktop / 2 tablet / 1 mobile).

- First card is always `CreateWorldCard` (links to `/worlds/new`)
- Empty state shows illustration + "Create Your First World" CTA
- Each `WorldCard` shows: name, genre/tone tags, summary excerpt (≤80 chars), completion progress bar, last updated date

### Step Wizard (`StepWizard.tsx`)

6-step guided creation flow. State managed locally in `useState`.

**State shape:**
```typescript
{
  currentStep: number;  // 1–6
  data: WorldFormData;  // all form fields
  isSubmitting: boolean;
  submitError: string;
}
```

**Navigation:**
- `Next` → validates step 1 (name + summary required), advances step
- `Back` → returns to previous step, data preserved
- `Skip for now` → advances without validation (steps 2–5)
- `Create World` (step 6) → validates, POSTs to `/api/oasisbios/[id]/worlds`, redirects to detail page
- `Cancel` → discards all data, returns to world list

**Step configuration** (`src/types/world.ts` → `WORLD_STEPS`):

Each step has: `step` (1–6), `module` name, `title`, `description`, and `fields[]`.

Each field has: `key` (maps to `WorldFormData`), `label`, `placeholder`, `hint`, `examples[]`, `type` (`text` | `textarea` | `select`), `options?`, `required?`.

**The 6 steps:**

| Step | Module | Required Fields | Optional Fields |
|------|--------|----------------|----------------|
| 1 | Core Identity | `name`, `summary` | `tagline`, `genre`, `tone` |
| 2 | Time Structure | — | `timeSetting`, `timeline` |
| 3 | World Rules | — | `physicsRules`, `rules` |
| 4 | Civilization | — | `socialStructure`, `factions` |
| 5 | Environment | — | `geography` |
| 6 | Narrative Context | — | `majorConflict` |

### World Detail Page (`WorldDetailPage`)

Single-page layout with:
1. **Header** — world name, genre/tone tags, completion score (large %), progress bar
2. **6 ModuleSections** — one per step, collapsible, with inline editing
3. **CharacterSection** — shows linked OasisBio character
4. **Danger Zone** — delete with confirmation dialog

**ModuleSection states:**
- **View mode** — read-only display of all fields
- **Edit mode** — inline form, triggered by "Edit" button
- **Saving** — optimistic update: UI updates immediately, rolls back on API error

**Optimistic update flow:**
```
1. User clicks Save
2. setWorld({ ...world, ...updates }) — immediate UI update
3. PUT /api/worlds/[id] — API call
4. On success: update with server response (includes updated_at)
5. On failure: re-fetch world from API (rollback), show error
```

### World Utilities (`src/lib/world-utils.ts`)

```typescript
// Completion score: Math.round((filledCount / 10) * 100)
// Tracked fields: name, summary, timeSetting, timeline, physicsRules,
//                 rules, socialStructure, factions, geography, majorConflict
calculateCompletionScore(world: Partial<WorldFormData>): number

// Per-module completion count
getModuleCompletion(world, fieldKeys): { filled: number, total: number }

// Truncate summary to maxLength (default 80), append "…"
truncateSummary(summary, maxLength?): string

// Validate form — only name and summary are required
validateWorldForm(data): WorldFormErrors  // { name?, summary? }
hasValidationErrors(errors): boolean

// Serialize genre + tone into aestheticKeywords JSON column
serializeGenreTone(genre, tone): string  // '{"genre":"sci-fi","tone":"dark"}'

// Deserialize aestheticKeywords back to { genre, tone }
// Handles legacy plain-text values gracefully
deserializeGenreTone(aestheticKeywords): { genre?, tone? }
```

### WorldFormData Type

```typescript
interface WorldFormData {
  // Step 1
  name: string;        // required
  summary: string;     // required
  tagline: string;
  genre: string;       // stored in aestheticKeywords JSON
  tone: string;        // stored in aestheticKeywords JSON
  // Step 2
  timeSetting: string;
  timeline: string;
  // Step 3
  physicsRules: string;
  rules: string;
  // Step 4
  socialStructure: string;
  factions: string;
  // Step 5
  geography: string;
  // Step 6
  majorConflict: string;
  // Meta
  visibility: 'private' | 'public';
}
```

**Completion fields** (10 total, used for score calculation):
`name`, `summary`, `timeSetting`, `timeline`, `physicsRules`, `rules`, `socialStructure`, `factions`, `geography`, `majorConflict`

---

## 9. Domain Events & Audit Logs

### Domain Events

The `domain_events` table is an async event queue. Events are written atomically inside database transactions (e.g., `publish_bio` RPC). Future consumers will poll `status = 'pending'` events to trigger side effects.

**Current event types:**
- `bio.published` — written by `publish_bio()` RPC
- `bio.unpublished` — written by `unpublish_bio()` RPC

**Planned consumers (not yet implemented):**
- OG image generation
- Search index update
- Page revalidation
- Email notifications

**Status lifecycle:** `pending` → `processing` → `done` | `failed`

**Retry:** `retry_count` increments on failure. Max retries TBD by consumer implementation.

### Audit Logs

The `audit_logs` table records all significant operations for compliance and debugging.

**Current actions logged:**
- `bio.publish` — who published which bio, with visibility and slug
- `bio.unpublish` — who unpublished which bio

**Planned actions:**
- `asset.token.request` — who requested a signed upload/download URL
- `visibility.change` — who changed a bio's visibility

**Access:** Users can SELECT their own audit logs. INSERT is blocked for all roles (only the `publish_bio` / `unpublish_bio` RPCs write to this table via `SECURITY DEFINER`).

---

## 10. Error Handling

### API Error Pattern

All API routes use `handleApiError` from `src/lib/auth-utils.ts`:

```typescript
export function handleApiError(error: unknown): NextResponse {
  if (error instanceof AuthError) {
    return NextResponse.json(
      { error: { code: error.code, message: error.message } },
      { status: error.statusCode }
    );
  }
  return NextResponse.json(
    { error: { code: 'INTERNAL_ERROR', message: 'Internal server error' } },
    { status: 500 }
  );
}
```

### Auth Error Codes

| Scenario | Code | Status |
|----------|------|--------|
| No session | `UNAUTHORIZED` | 401 |
| Not owner | `FORBIDDEN` | 403 |
| Resource missing | `NOT_FOUND` | 404 |

### Client-Side Error Handling

Login/register pages:
- OTP send failure → display Supabase error message
- OTP verify failure → display "Invalid or expired verification code" + resend option
- Network error → display "Network error, please try again"
- Input change → clear error state

World builder:
- API save failure → display inline error, preserve unsaved changes
- World creation failure → display error toast, keep wizard data
- Delete failure → display error, cancel delete

### Sync Error Handling

`syncUserToPrisma` failures are non-fatal:
```typescript
try {
  const syncResult = await syncUserToPrisma(user);
  return { supabaseUser: user, ...syncResult };
} catch (err) {
  console.error('[auth] syncUserToPrisma failed (non-fatal):', err);
  return { supabaseUser: user, userId: user.id, profileId: null, username: null, isNewUser: false };
}
```

The user can still log in and use the app even if Prisma sync fails.

---

## 11. Testing

### Test Stack

- **Jest** — test runner (configured in `jest.config.js`)
- **fast-check** — property-based testing library
- **@testing-library/react** — React component testing
- **@testing-library/jest-dom** — DOM matchers

### Test Files

| File | Tests | Properties |
|------|-------|-----------|
| `src/lib/user-sync.test.ts` | 12 | Property 2 (username uniqueness), Property 3 (field preservation), Property 4 (idempotency) |
| `src/middleware.test.ts` | 11 | Property 1 (route protection) |
| `src/lib/auth.client.test.ts` | 8 | Property 5 (auth state machine) |

### Property-Based Tests

All properties use `fast-check` with 200 iterations minimum.

**Property 1: Protected routes always redirect unauthenticated users**
- For any path under `/dashboard`, `/api/oasisbios`, `/api/worlds`
- Without a session → redirect to `/auth/login?callbackUrl=<path>`
- Validates: Requirements 3.2, 3.5

**Property 2: Username uniqueness invariant**
- For any set of existing usernames + any input string
- `generateUsernameCandidate` returns a username not in the existing set
- Validates: Requirements 5.5

**Property 3: syncUserToPrisma preserves user-edited fields**
- For any existing Profile with non-null `displayName`, `bio`, `website`
- Calling sync again never overwrites those fields
- Validates: Requirements 5.3

**Property 4: syncUserToPrisma is idempotent**
- Calling `mergeProfileFields` twice with same data = calling once
- Validates: Requirements 5.2

**Property 5: Auth state change propagates to context**
- For any auth event (SIGNED_IN, SIGNED_OUT, TOKEN_REFRESHED, INITIAL_SESSION)
- The resulting state correctly reflects the event
- Validates: Requirements 2.5

### Running Tests

```bash
# Run all tests
npm test

# Run specific test file
npm test -- "src/lib/user-sync.test.ts" --no-coverage --testEnvironment node

# Run with coverage
npm run test:coverage
```

### Jest Configuration

```javascript
// jest.config.js
{
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' },
  transform: { '^.+\\.(js|jsx|ts|tsx)$': ['babel-jest', { presets: ['next/babel'] }] }
}
```

Note: Tests that use Node.js APIs (no DOM) should pass `--testEnvironment node` flag.

---

## 12. Deployment

### Cloudflare Pages

1. Connect GitHub repository to Cloudflare Pages
2. **Build command:** `npm run build` (runs `prisma generate` then `next build`)
3. **Output directory:** `.next`
4. Add all environment variables in Cloudflare dashboard → Settings → Environment variables
5. Both `NEXT_PUBLIC_*` and private variables must be added

### Database Setup (one-time, run in Supabase SQL Editor)

Execute in this exact order:

```
1. scripts/db/01_enable_rls.sql          — RLS policies
2. scripts/db/02_add_indexes.sql         — Performance indexes
3. scripts/db/04_storage_policies.sql    — Storage bucket policies
4. scripts/db/05_domain_events_audit_logs.sql  — Event/audit tables
5. scripts/db/06_publish_bio_rpc.sql     — Publish RPC functions
```

`03_service_role_bypass.sql` is documentation only — no SQL to execute.

### Supabase Auth Configuration

In Supabase dashboard → Authentication → URL Configuration:
- **Site URL:** `https://your-domain.com`
- **Redirect URLs:** Add `https://your-domain.com/auth/callback`

### Storage Buckets (create manually in Supabase Storage)

Create these buckets before running `04_storage_policies.sql`:
- `avatars` — set to **Public**
- `character-covers` — set to **Public**
- `model-previews` — set to **Public**

### Build Notes

The `postbuild` script clears `.next/cache` to reduce Cloudflare Pages deployment size:
```json
"postbuild": "node -e \"const fs = require('fs'); const path = '.next/cache'; if (fs.existsSync(path)) { fs.rmSync(path, { recursive: true, force: true }); }\""
```

---

## 13. Environment Variables

| Variable | Required | Description |
|----------|---------|-------------|
| `DATABASE_URL` | ✅ | Prisma pooler connection (port 6543, pgbouncer=true) |
| `DIRECT_URL` | ✅ | Prisma direct connection (port 5432, for migrations only) |
| `NEXT_PUBLIC_SUPABASE_URL` | ✅ | Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | ✅ | Supabase anon key (public, safe for browser) |
| `SUPABASE_SERVICE_ROLE_KEY` | ✅ | Supabase service role key (server-only, never expose to browser) |
| `SUPABASE_WEBHOOK_SECRET` | ⚠️ | HMAC secret for webhook signature verification. If missing, verification is skipped (dev mode). |
| `CLOUDFLARE_R2_ACCESS_KEY_ID` | ✅ | R2 access key |
| `CLOUDFLARE_R2_SECRET_ACCESS_KEY` | ✅ | R2 secret key |
| `CLOUDFLARE_R2_ENDPOINT` | ✅ | `https://{account-id}.r2.cloudflarestorage.com` |
| `CLOUDFLARE_R2_BUCKET_NAME` | ✅ | R2 bucket name |
| `CLOUDFLARE_R2_ACCOUNT_ID` | ✅ | Cloudflare account ID |

**Deprecated (no longer used):**
- `NEXT_PUBLIC_SUPABASE_COOKIE_NAME` — `@supabase/ssr` handles cookie names automatically
- `NEXTAUTH_URL` / `NEXTAUTH_SECRET` — project no longer uses NextAuth

**Security notes:**
- `SUPABASE_SERVICE_ROLE_KEY` must never appear in client-side code or be prefixed with `NEXT_PUBLIC_`
- `DATABASE_URL` contains the database password — never commit `.env` to version control
- Rotate `SUPABASE_WEBHOOK_SECRET` if compromised

---

## 14. Planned: Edge Functions

The following Supabase Edge Functions are designed but not yet deployed. They require Supabase CLI + Docker for local development.

See `prepare_home/supabase edge function规划.md` for full specification.

### `asset-token`
Generates signed upload/download URLs for Supabase Storage and Cloudflare R2.

**Input:**
```json
{
  "action": "upload" | "download",
  "resourceType": "character_cover" | "avatar" | "model_preview",
  "resourceId": "bio_xxx",
  "filename": "cover.webp",
  "contentType": "image/webp",
  "size": 245123
}
```

**Replaces:** Direct client-side uploads (which bypass ownership checks)

### `auth-profile-sync`
Syncs user profile on first login. Equivalent to `syncUserToPrisma` but runs at the Edge.

**Input:** `{ displayName, avatarUrl?, locale? }`

**Output:** `{ userId, profileId, username }`

### `publish-bio`
Command entry point for publishing. Calls `publish_bio()` RPC.

**Input:** `{ bioId, visibility, requestId }`

**Note:** Does NOT handle OG image generation, search indexing, or notifications — those are async consumers of `domain_events`.

### `reference-enrich`
Fetches URL metadata (title, description, cover image) for reference items.

**Input:** `{ url }`

**Output:** `{ title, description, coverImage, provider, sourceType, metadata }`

### Shared Utilities (`supabase/functions/_shared/`)

All Edge Functions share:
- `cors.ts` — CORS headers
- `clients.ts` — `createUserClient(req)` + `createAdminClient()`
- `auth.ts` — JWT verification
- `response.ts` — structured response helpers
- `logger.ts` — structured logging with `request_id`, `user_id`, `duration_ms`

### Edge Function Rules

1. Every request gets a `request_id` (generated if not provided)
2. All inputs validated with Zod schemas
3. All errors return `{ error: { code, message } }`
4. No Prisma-heavy logic in Edge Functions
5. Logs include: `request_id`, `user_id`, `function`, `resource_id`, `duration_ms`

---

## 15. OAuth Provider

OasisBio implements a full OAuth 2.0 Authorization Server with PKCE support and OpenID Connect discovery. Third-party applications can use "Continue with Oasis" to authenticate users and access their character data with explicit consent.

### Overview

```
Third-party App                OasisBio                    User
      │                            │                          │
      │── GET /oauth/authorize ──► │                          │
      │                            │── Show consent page ──► │
      │                            │ ◄── User approves ───── │
      │ ◄── redirect + code ────── │                          │
      │── POST /api/oauth/token ─► │                          │
      │ ◄── access_token ───────── │                          │
      │── GET /api/oauth/userinfo ►│                          │
      │ ◄── user profile ───────── │                          │
```

**Standards implemented:**
- OAuth 2.0 Authorization Code flow (RFC 6749)
- PKCE — Proof Key for Code Exchange (RFC 7636) — **required**
- OpenID Connect Core 1.0 (discovery endpoint)
- Token revocation (RFC 7009)

### Developer Portal

| Route | Description |
|-------|-------------|
| `/developer` | Landing page — overview, button preview, code snippets |
| `/developer/apps` | List of registered OAuth apps (requires login) |
| `/developer/apps/new` | Register a new app (requires login) |
| `/developer/apps/[id]` | Manage app — view credentials, rotate secret |
| `/developer/docs` | Integration guide |

### Database Tables

Defined in `scripts/db/07_oauth_tables.sql`.

#### `oauth_apps`

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `user_id` | String | FK → users (app owner) |
| `name` | String | App display name |
| `description` | String? | |
| `homepage_url` | String | |
| `redirect_uris` | String | JSON array of allowed redirect URIs |
| `client_id` | String (unique) | Public identifier, auto-generated |
| `client_secret_hash` | String | bcrypt hash of the secret |
| `is_active` | Boolean | Default: true |
| `created_at` | DateTime | |
| `updated_at` | DateTime | |

#### `oauth_authorization_codes`

Short-lived codes (10 minutes) exchanged for tokens.

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `code` | String (unique) | Random 32-byte hex |
| `client_id` | String | FK → oauth_apps.client_id |
| `user_id` | String | FK → users |
| `redirect_uri` | String | Must match the one used in authorization |
| `scope` | String | Space-separated scope string |
| `code_challenge` | String | PKCE S256 challenge |
| `code_challenge_method` | String | Always `S256` |
| `expires_at` | DateTime | 10 minutes from creation |
| `used` | Boolean | Default: false — codes are single-use |
| `created_at` | DateTime | |

#### `oauth_tokens`

| Column | Type | Notes |
|--------|------|-------|
| `id` | String (cuid) | |
| `access_token` | String (unique) | JWT signed with `OAUTH_JWT_SECRET` |
| `refresh_token` | String (unique) | Random 32-byte hex |
| `client_id` | String | FK → oauth_apps.client_id |
| `user_id` | String | FK → users |
| `scope` | String | Granted scope string |
| `access_token_expires_at` | DateTime | 1 hour from creation |
| `refresh_token_expires_at` | DateTime | 30 days from creation |
| `revoked` | Boolean | Default: false |
| `created_at` | DateTime | |

### Scopes

Defined in `src/lib/oauth/scopes.ts`.

| Scope | Data Accessible |
|-------|----------------|
| `profile` | username, display name, avatar URL |
| `email` | email address |
| `oasisbios:read` | character list (title, slug, cover image) |
| `oasisbios:full` | full character data (abilities, worlds, eras, references) |
| `dcos:read` | DCOS document content |

### OAuth API Endpoints

All OAuth endpoints follow RFC 6749 error format:
```json
{ "error": "invalid_grant", "error_description": "Authorization code already used" }
```

#### `GET /oauth/authorize`

Displays the consent page. Validates all parameters before showing UI.

**Required query params:**
- `client_id` — registered app client ID
- `redirect_uri` — must match a registered redirect URI
- `response_type` — must be `code`
- `scope` — space-separated list of requested scopes
- `state` — CSRF protection token (recommended)
- `code_challenge` — PKCE S256 challenge
- `code_challenge_method` — must be `S256`

**Validation errors** redirect back to `redirect_uri` with `error=invalid_request`.

**On approval:** redirects to `redirect_uri?code=<code>&state=<state>`

**On denial:** redirects to `redirect_uri?error=access_denied`

#### `POST /api/oauth/authorize`

Handles the consent form submission (approve/deny).

**Body:** `{ clientId, redirectUri, scope, state, codeChallenge, codeChallengeMethod, action: 'approve' | 'deny' }`

Requires the user to be authenticated (Supabase session cookie).

#### `POST /api/oauth/token`

Exchanges an authorization code for tokens, or refreshes an access token.

**Grant type: `authorization_code`**

```
POST /api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&code=CODE_FROM_REDIRECT
&redirect_uri=YOUR_REDIRECT_URI
&code_verifier=STORED_CODE_VERIFIER
```

**Response:**
```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "abc123...",
  "scope": "profile email"
}
```

**Grant type: `refresh_token`**

```
POST /api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&refresh_token=STORED_REFRESH_TOKEN
```

Returns a new `access_token` and `refresh_token`. The old refresh token is immediately invalidated.

**Error responses:**

| Error | Cause |
|-------|-------|
| `invalid_client` | Unknown client_id or wrong client_secret |
| `invalid_grant` | Code expired, already used, or verifier mismatch |
| `invalid_request` | Missing required parameters |
| `unsupported_grant_type` | Grant type not `authorization_code` or `refresh_token` |

#### `GET /api/oauth/userinfo`

Returns the authenticated user's profile. Requires `profile` scope.

**Request:**
```
GET /api/oauth/userinfo
Authorization: Bearer <access_token>
```

**Response:**
```json
{
  "sub": "user_uuid",
  "username": "johndoe",
  "display_name": "John Doe",
  "avatar_url": "https://...",
  "email": "john@example.com"
}
```

Note: `email` is only included if the token has `email` scope.

#### `GET /api/oauth/resources/oasisbios`

Returns the user's character list. Requires `oasisbios:read` scope.

**Response:**
```json
[
  {
    "id": "bio_xxx",
    "title": "Elara Stormrider",
    "slug": "elara-stormrider",
    "tagline": "Intergalactic Explorer",
    "cover_image_url": "https://...",
    "identity_mode": "fictional",
    "visibility": "public"
  }
]
```

#### `GET /api/oauth/resources/oasisbios/[id]`

Returns full character data. Requires `oasisbios:full` scope.

**Response:** Full OasisBio object including abilities, eras, worlds, and references.

#### `GET /api/oauth/resources/oasisbios/[id]/dcos`

Returns DCOS documents for a character. Requires `dcos:read` scope.

**Response:** Array of `{ id, title, slug, content, folderPath, status, eraId }`

#### `POST /api/oauth/revoke`

Revokes an access or refresh token (RFC 7009).

**Body:** `{ token, token_type_hint?: 'access_token' | 'refresh_token' }`

Always returns `200 OK` regardless of whether the token existed.

#### `GET /api/oauth/.well-known/openid-configuration`

OIDC discovery document. No authentication required.

**Response:**
```json
{
  "issuer": "https://oasisbio.com",
  "authorization_endpoint": "https://oasisbio.com/oauth/authorize",
  "token_endpoint": "https://oasisbio.com/api/oauth/token",
  "userinfo_endpoint": "https://oasisbio.com/api/oauth/userinfo",
  "revocation_endpoint": "https://oasisbio.com/api/oauth/revoke",
  "scopes_supported": ["profile", "email", "oasisbios:read", "oasisbios:full", "dcos:read"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["client_secret_post"]
}
```

### Developer API Endpoints

These endpoints manage OAuth apps and require the user to be authenticated via Supabase session (not OAuth token).

#### `GET /api/developer/apps`
Returns all OAuth apps owned by the authenticated user.

#### `POST /api/developer/apps`
Creates a new OAuth app.

**Required body:** `{ name, homepageUrl, redirectUris: string[] }`
**Optional:** `description`

**Response:** App object including `clientId` and `clientSecret` (secret shown only once).

#### `GET /api/developer/apps/[id]`
Returns app details. `clientSecret` is never returned after creation.

#### `PUT /api/developer/apps/[id]`
Updates app metadata (`name`, `description`, `homepageUrl`, `redirectUris`, `isActive`).

#### `DELETE /api/developer/apps/[id]`
Deletes the app and revokes all associated tokens.

#### `POST /api/developer/apps/[id]/secret`
Rotates the client secret. Returns the new secret (shown only once). Invalidates all existing tokens for this app.

### OAuth Middleware

`src/lib/oauth/middleware.ts` — use in any API route that requires an OAuth token:

```typescript
import { requireOAuthToken } from '@/lib/oauth/middleware';

export async function GET(request: NextRequest) {
  const result = await requireOAuthToken(request, 'oasisbios:read');
  if ('error' in result) return result.error;  // 401 with RFC 6749 error body

  const { userId, scope, clientId } = result.context;
  // proceed with authorized request
}
```

`requireOAuthToken(request, requiredScope)` validates:
1. `Authorization: Bearer <token>` header is present
2. JWT signature is valid (`OAUTH_JWT_SECRET`)
3. Token is not expired
4. Token is not revoked
5. Token scope includes `requiredScope`

Returns `{ context: { userId, scope, clientId } }` on success, or `{ error: NextResponse }` on failure.

### Crypto Utilities

`src/lib/oauth/crypto.ts`

```typescript
// Generate a cryptographically random hex string
generateSecureToken(bytes?: number): string  // default: 32 bytes

// Hash a client secret for storage
hashClientSecret(secret: string): Promise<string>

// Verify a client secret against its hash
verifyClientSecret(secret: string, hash: string): Promise<boolean>

// Sign an access token JWT
signAccessToken(payload: { sub, clientId, scope, jti }): string

// Verify and decode an access token JWT
verifyAccessToken(token: string): { sub, clientId, scope, jti, iat, exp } | null
```

### PKCE Flow (Client Implementation)

```javascript
// 1. Generate code verifier (store in sessionStorage)
const verifier = generateRandomString(64);
sessionStorage.setItem('pkce_verifier', verifier);

// 2. Compute code challenge
async function sha256(plain) {
  const data = new TextEncoder().encode(plain);
  return crypto.subtle.digest('SHA-256', data);
}
function base64url(buffer) {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}
const challenge = base64url(await sha256(verifier));

// 3. Redirect to authorization endpoint
const url = new URL('https://oasisbio.com/oauth/authorize');
url.searchParams.set('client_id', YOUR_CLIENT_ID);
url.searchParams.set('redirect_uri', YOUR_REDIRECT_URI);
url.searchParams.set('response_type', 'code');
url.searchParams.set('scope', 'profile email oasisbios:read');
url.searchParams.set('state', generateRandomString(16));
url.searchParams.set('code_challenge', challenge);
url.searchParams.set('code_challenge_method', 'S256');
window.location.href = url.toString();

// 4. On callback, exchange code for tokens
const verifier = sessionStorage.getItem('pkce_verifier');
const res = await fetch('https://oasisbio.com/api/oauth/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'authorization_code',
    client_id: YOUR_CLIENT_ID,
    client_secret: YOUR_CLIENT_SECRET,
    code: CODE_FROM_URL,
    redirect_uri: YOUR_REDIRECT_URI,
    code_verifier: verifier,
  }),
});
const { access_token, refresh_token } = await res.json();
```

### Security Notes

- **PKCE is mandatory.** Authorization requests without `code_challenge` are rejected.
- **`OAUTH_JWT_SECRET`** must be at least 32 characters. Never change it after tokens have been issued — all existing tokens will become invalid.
- **Client secrets** are stored as bcrypt hashes. The plaintext secret is shown only once at creation and once after rotation.
- **Authorization codes** are single-use and expire after 10 minutes.
- **Access tokens** expire after 1 hour. **Refresh tokens** expire after 30 days.
- **Token revocation** is immediate — revoked tokens are rejected on the next API call.
- The `/developer/apps` routes are protected by Supabase session middleware. The `/developer` landing page is public (no login required).
