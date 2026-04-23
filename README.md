# OasisBio Documentation

## Documents

### Technical Documentation
| Document | Description |
|----------|-------------|
| [technical.md](technical/technical.md) | Full technical reference — architecture, auth, database schema, API, world builder, OAuth provider, testing, deployment |
| [edge-functions.md](technical/edge-functions.md) | Edge Functions spec, current Next.js API routes, and migration guide to Supabase Edge Functions |

### Design Documentation
| Document | Description |
|----------|-------------|
| [design.md](design/design.md) | Design system — color palette, typography, components, layout |
| [world-design-spec.md](design/world-design-spec.md) | Worldbuilding standard — 6-module structure, field definitions |

### Feature Documentation
| Document | Description |
|----------|-------------|
| [oauth.md](features/oauth.md) | OAuth Provider — "Continue with Oasis" integration guide for third-party developers |
| [oasisbio.md](features/oasisbio.md) | OasisBio identity container — modes, eras, structure |
| [worlds.md](features/worlds.md) | Worldbuilding system — creation, components, world-identity binding |
| [abilities.md](features/abilities.md) | Ability Pool system — categories, levels, era/world binding |
| [repositories.md](features/repositories.md) | Repositories — DCOS, References |
| [models.md](features/models.md) | 3D Model system |

### Strategic Documentation
| Document | Description |
|----------|-------------|
| [OasisBio Strategic Plan.md](strategy/OasisBio Strategic Plan.md) | Strategic plan for OasisBio project |

## Database Setup Scripts

Run in Supabase SQL Editor **in this order** after `npx prisma db push`:

| # | Script | What it does |
|---|--------|-------------|
| 1 | `scripts/db/01_enable_rls.sql` | Enables Row Level Security on all tables + access policies |
| 2 | `scripts/db/02_add_indexes.sql` | Performance indexes for common query patterns |
| 3 | `scripts/db/03_service_role_bypass.sql` | Documentation only — no SQL to run |
| 4 | `scripts/db/04_storage_policies.sql` | Storage bucket write policies (run after creating buckets) |
| 5 | `scripts/db/05_domain_events_audit_logs.sql` | Creates `domain_events` and `audit_logs` tables |
| 6 | `scripts/db/06_publish_bio_rpc.sql` | Creates `publish_bio`, `unpublish_bio`, `validate_publishable_bio` RPCs |
| 7 | `scripts/db/07_oauth_tables.sql` | Creates `oauth_apps`, `oauth_authorization_codes`, `oauth_tokens` tables |

## Quick Reference

### Auth pattern in Client Components

```typescript
// ✅ Correct — useAuth() is the primary hook
import { useAuth } from '@/lib/auth.client';

const { user, isLoading, supabase } = useAuth();

// Sign out
await supabase.auth.signOut();

// Check auth state
if (isLoading) return <Spinner />;
if (!user) return null; // middleware handles redirect
```

### Auth pattern in API routes

```typescript
// ✅ Correct — requireAuth() returns User directly
const user = await requireAuth();
const userId = user.id;

// ❌ Wrong — old pattern, will crash
const session = await requireAuth();
const userId = session.user.id;
```

### Error response format

```json
{ "error": { "code": "FORBIDDEN", "message": "You do not own this bio" } }
```

### OAuth error format (RFC 6749)

```json
{ "error": "invalid_grant", "error_description": "Authorization code already used" }
```

### Environment variables checklist

| Variable | Required | Notes |
|----------|---------|-------|
| `DATABASE_URL` | ✅ | Pooler connection (port 6543) |
| `DIRECT_URL` | ✅ | Direct connection (port 5432, migrations only) |
| `NEXT_PUBLIC_SUPABASE_URL` | ✅ | |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | ✅ | |
| `SUPABASE_SERVICE_ROLE_KEY` | ✅ | Must be `eyJ...` JWT format |
| `OAUTH_JWT_SECRET` | ✅ | Min 32 chars, never change after setting |
| `CLOUDFLARE_R2_*` | ⚠️ | Required for 3D models and exports |
| `SUPABASE_WEBHOOK_SECRET` | ⚠️ | Optional, skips signature check if missing |

### Toast notifications

```typescript
// In any Client Component wrapped by ToastProvider
import { useToast } from '@/components/Toast';

const { success, error, info } = useToast();
success('Saved!');
error('Something went wrong');
info('Processing...');
```

### Copy button

```tsx
import { CopyButton } from '@/components/CopyButton';

// Icon-only (inline with text)
<CopyButton value={clientId} iconOnly successMessage="Copied!" />

// With label
<CopyButton value={clientId} label="Copy ID" successMessage="Client ID copied!" />
```

### World completion score

```typescript
// 10 tracked fields: name, summary, timeSetting, timeline, physicsRules,
// rules, socialStructure, factions, geography, majorConflict
calculateCompletionScore(world) // → 0–100
```

### Publish a character

```
POST /api/oasisbios/{id}/publish
Body: { "visibility": "public" }

DELETE /api/oasisbios/{id}/publish  (unpublish)
```

### Era management

```
GET    /api/oasisbios/{id}/eras          # list, ordered by sortOrder
POST   /api/oasisbios/{id}/eras          # create (sortOrder auto-assigned)
PUT    /api/eras/{eraId}                 # update
DELETE /api/eras/{eraId}                 # delete (unlinks abilities + DCOS)
```

### Supabase client selection

| Context | Import |
|---------|--------|
| Client Component | `import { createClient } from '@/lib/supabase/client'` |
| Server Component / API Route | `import { createClient } from '@/lib/supabase/server'` |
| Middleware | `import { updateSession } from '@/lib/supabase/middleware'` |
| Storage operations | `import { uploadFile, storagePath } from '@/lib/supabase/storage'` |

### OAuth token validation in API routes

```typescript
import { requireOAuthToken } from '@/lib/oauth/middleware';

export async function GET(request: NextRequest) {
  const result = await requireOAuthToken(request, 'oasisbios:read');
  if ('error' in result) return result.error;
  const { userId, scope } = result.context;
  // ...
}
```

### OAuth scopes

| Scope | Data |
|-------|------|
| `profile` | username, display name, avatar URL |
| `email` | email address |
| `oasisbios:read` | character list (title, slug, cover image) |
| `oasisbios:full` | full character data (abilities, worlds, eras, references) |
| `dcos:read` | DCOS document content |
