# Edge Functions — Implementation & Migration Guide

## Current Status

The four planned Edge Functions are currently implemented as **Next.js API routes**. They are functionally identical to the Edge Function specification. Migration to Supabase Edge Functions can be done at any time with minimal code changes.

## Why Next.js API Routes (Current)

| Concern | Next.js API Route | Supabase Edge Function |
|---------|------------------|----------------------|
| Cold start | ~100–300ms (Cloudflare Pages) | ~50ms (global edge nodes) |
| Deployment | Automatic with main app | Requires Supabase CLI + Docker |
| Code location | Single repository | Separate `supabase/functions/` directory |
| Prisma support | Full support | Limited (no connection pooling) |
| Maintenance | One deploy pipeline | Two deploy pipelines |
| When to prefer | Current stage (low traffic) | High traffic, latency-sensitive |

**Decision:** Use Next.js API routes until traffic justifies the operational overhead of Edge Functions.

---

## Current API Routes (Next.js)

### `POST /api/asset-token`

Generates signed upload/download URLs for Supabase Storage and Cloudflare R2.

**Request:**
```json
{
  "action": "upload" | "download",
  "resourceType": "avatar" | "character_cover" | "model_preview" | "model",
  "resourceId": "bio_xxx",
  "filename": "cover.webp",
  "contentType": "image/webp",
  "size": 245123
}
```

**Response (upload, Supabase):**
```json
{
  "provider": "supabase",
  "method": "signed-upload",
  "bucket": "character-covers",
  "path": "user_123/bio_456/cover.webp",
  "token": "xxx",
  "signedUrl": "https://...",
  "expiresIn": 300,
  "requestId": "uuid"
}
```

**Response (download, R2):**
```json
{
  "provider": "r2",
  "method": "signed-download",
  "url": "https://...",
  "expiresIn": 3600,
  "requestId": "uuid"
}
```

**Ownership check:** Verifies `resourceId` belongs to the authenticated user before issuing any token.

**Resource config:**

| resourceType | Provider | Bucket | Allowed MIME | Max Size |
|-------------|---------|--------|-------------|---------|
| `avatar` | Supabase | `avatars` | webp, png, jpeg | 512 KB |
| `character_cover` | Supabase | `character-covers` | webp, png, jpeg | 800 KB |
| `model_preview` | Supabase | `model-previews` | webp, png, jpeg | 600 KB |
| `model` | R2 | R2 bucket | gltf-binary, octet-stream | 12 MB |

---

### `POST /api/auth/profile-sync`

Syncs the authenticated user's profile to Prisma. Creates `User` + `Profile` records if they don't exist. Only fills empty fields — never overwrites user-edited data.

**Request:**
```json
{
  "displayName": "Stephen",
  "avatarUrl": "https://...",
  "locale": "zh-CN"
}
```

All fields are optional. If omitted, uses data from the Supabase session.

**Response:**
```json
{
  "userId": "user_xxx",
  "profileId": "profile_xxx",
  "username": "stephen",
  "isNewUser": true,
  "requestId": "uuid"
}
```

**When to call:** After first login, or when the user updates their display name/avatar in a third-party OAuth flow.

---

### `POST /api/reference-enrich`

Fetches URL metadata for reference items. Extracts Open Graph tags, Twitter Card tags, and standard meta tags.

**Request:**
```json
{
  "url": "https://example.com/article"
}
```

**Response:**
```json
{
  "url": "https://example.com/article",
  "title": "Example Article",
  "description": "A brief description...",
  "coverImage": "https://example.com/og-image.jpg",
  "provider": "example.com",
  "sourceType": "article",
  "metadata": {
    "siteName": "Example Site",
    "author": "John Doe"
  },
  "requestId": "uuid"
}
```

**Provider detection:** Automatically identifies YouTube, GitHub, Wikipedia, Reddit, Bilibili, Pixiv, ArtStation, DeviantArt, Medium, Substack, and X/Twitter.

**Fetch behavior:**
- 5-second timeout
- Reads max 100 KB of HTML (prevents memory issues with large pages)
- Returns minimal metadata (hostname as title) if fetch fails — never errors on unreachable URLs

---

### `POST /api/oasisbios/[id]/publish` (already implemented)

Publishes a character via the `publish_bio` database RPC. See [technical.md](technical.md#publish-a-character) for details.

---

## Migration to Supabase Edge Functions

When you're ready to migrate, follow these steps.

### Prerequisites

```bash
# Install Supabase CLI
npm install -g supabase

# Install Docker Desktop (required for local testing)
# https://www.docker.com/products/docker-desktop/

# Login to Supabase
supabase login

# Link to your project
supabase link --project-ref dhkgfdllgtmbkwcbubqt
```

### Directory Structure

```
supabase/
  functions/
    _shared/
      cors.ts          # CORS headers
      auth.ts          # JWT verification
      clients.ts       # createUserClient + createAdminClient
      response.ts      # Structured response helpers
      logger.ts        # Structured logging
    asset-token/
      index.ts
    auth-profile-sync/
      index.ts
    reference-enrich/
      index.ts
    publish-bio/
      index.ts
```

### `_shared/cors.ts`

```typescript
export const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type, x-request-id',
};
```

### `_shared/clients.ts`

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

// User client — uses the request's Authorization header
export function createUserClient(req: Request) {
  return createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
  );
}

// Admin client — bypasses RLS
export function createAdminClient() {
  return createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );
}
```

### Example: `asset-token/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { corsHeaders } from '../_shared/cors.ts';
import { createUserClient, createAdminClient } from '../_shared/clients.ts';

serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    const userClient = createUserClient(req);
    const { data: { user }, error } = await userClient.auth.getUser();
    if (error || !user) {
      return new Response(
        JSON.stringify({ error: { code: 'UNAUTHORIZED', message: 'Unauthorized' } }),
        { status: 401, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    const body = await req.json();
    // ... same logic as the Next.js route ...

    return new Response(JSON.stringify(result), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    });
  } catch (err) {
    return new Response(
      JSON.stringify({ error: { code: 'INTERNAL_ERROR', message: err.message } }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});
```

### Deploy

```bash
# Deploy a single function
supabase functions deploy asset-token

# Deploy all functions
supabase functions deploy

# Test locally (requires Docker)
supabase start
supabase functions serve asset-token --env-file .env
```

### Update Frontend Calls

After deploying, update the frontend to call Edge Functions instead of Next.js routes:

```typescript
// Before (Next.js API route)
const res = await fetch('/api/asset-token', { method: 'POST', body: JSON.stringify(payload) });

// After (Supabase Edge Function)
const { data, error } = await supabase.functions.invoke('asset-token', { body: payload });
```

### Rollback

If Edge Functions cause issues, simply revert the frontend calls back to `/api/*` routes. The Next.js routes remain in the codebase and can be re-enabled instantly.

---

## Edge Function Rules (from spec)

All Edge Functions must follow these 5 rules:

1. **Mandatory `request_id`** — generate if not provided, always include in response
2. **Schema validation** — validate all inputs (use Zod or manual checks)
3. **Structured errors** — always `{ error: { code, message } }`
4. **Structured logs** — include `request_id`, `user_id`, `function`, `resource_id`, `duration_ms`
5. **No Prisma** — Edge Functions must not use Prisma or heavy Node.js services

---

## What Stays in Next.js (Never Migrates to Edge)

Per the architecture spec, these always stay in Node.js:

- ZIP import/export (`/api/import`, `/api/export`)
- Large file aggregation
- 3D model preprocessing
- Complex Prisma transactions
- Webhook handlers (`/api/auth/supabase-webhook`)
