# OAuth Provider — "Continue with Oasis"

## Overview

OasisBio is an OAuth 2.0 Authorization Server. Third-party applications can integrate "Continue with Oasis" to let users sign in with their Oasis identity and access their character data with explicit consent.

**Standards:** OAuth 2.0 (RFC 6749) · PKCE (RFC 7636) · Token Revocation (RFC 7009) · OpenID Connect Core 1.0

---

## Brand Assets

### Oasis OAuth Logo

The official logo for the "Continue with Oasis" button is available at:

```
https://oasisbio.com/assets/oasis_logo.svg
```

Direct download: [`/assets/oasis_logo.svg`](/assets/oasis_logo.svg)

**Specifications:** SVG · 61×51px native size · works at any scale

### Button Guidelines

- Use the exact text **"Continue with Oasis"** — do not alter the wording
- Always place the Oasis logo to the left of the button text
- Minimum button height: 40px · Minimum logo display size: 16×14px
- Recommended style: black background `#000`, white text `#fff`, 8px border radius
- Do not recolor, distort, rotate, or modify the logo in any way
- Do not use the logo in a way that implies endorsement beyond OAuth authentication

### Button HTML

```html
<!-- Logo: https://oasisbio.com/assets/oasis_logo.svg -->
<a href="https://oasisbio.com/oauth/authorize?..."
   style="display:inline-flex;align-items:center;gap:10px;padding:10px 20px;
          background:#000;color:#fff;border-radius:8px;text-decoration:none;
          font-family:sans-serif;font-size:15px;font-weight:500;">
  <img src="https://oasisbio.com/assets/oasis_logo.svg" width="20" height="17" alt="Oasis" />
  Continue with Oasis
</a>
```

---

## Quick Start

### 1. Register your app

Go to [/developer/apps/new](/developer/apps/new) and create an OAuth app. You'll receive:
- `client_id` — public identifier, safe to include in frontend code
- `client_secret` — shown once, store securely server-side

### 2. Add the button

```html
<a href="https://oasisbio.com/oauth/authorize?client_id=YOUR_CLIENT_ID
         &redirect_uri=YOUR_REDIRECT_URI&response_type=code
         &scope=profile+email&state=RANDOM_STATE
         &code_challenge=YOUR_CODE_CHALLENGE&code_challenge_method=S256"
   style="display:inline-flex;align-items:center;gap:10px;padding:10px 20px;
          background:#000;color:#fff;border-radius:8px;text-decoration:none;
          font-family:sans-serif;font-size:15px;font-weight:500;">
  Continue with Oasis
</a>
```

### 3. Handle the callback

After the user approves, they're redirected to your `redirect_uri` with `?code=<code>&state=<state>`.

Exchange the code for tokens:

```javascript
const res = await fetch('https://oasisbio.com/api/oauth/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'authorization_code',
    client_id: YOUR_CLIENT_ID,
    client_secret: YOUR_CLIENT_SECRET,
    code: CODE_FROM_URL,
    redirect_uri: YOUR_REDIRECT_URI,
    code_verifier: STORED_CODE_VERIFIER,
  }),
});
const { access_token, refresh_token, expires_in } = await res.json();
```

### 4. Access user data

```javascript
// User profile (requires 'profile' scope)
const profile = await fetch('https://oasisbio.com/api/oauth/userinfo', {
  headers: { Authorization: `Bearer ${access_token}` },
}).then(r => r.json());
// → { sub, username, display_name, avatar_url, email }

// Character list (requires 'oasisbios:read' scope)
const characters = await fetch('https://oasisbio.com/api/oauth/resources/oasisbios', {
  headers: { Authorization: `Bearer ${access_token}` },
}).then(r => r.json());
```

---

## PKCE Implementation

PKCE is **required** for all authorization requests. There is no non-PKCE flow.

```javascript
// Generate and store code verifier
function generateRandomString(length) {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~';
  return Array.from(crypto.getRandomValues(new Uint8Array(length)))
    .map(b => chars[b % chars.length]).join('');
}

async function sha256(plain) {
  return crypto.subtle.digest('SHA-256', new TextEncoder().encode(plain));
}

function base64url(buffer) {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

// Before redirect:
const verifier = generateRandomString(64);
sessionStorage.setItem('pkce_verifier', verifier);
const challenge = base64url(await sha256(verifier));

// In token exchange:
const verifier = sessionStorage.getItem('pkce_verifier');
```

---

## Available Scopes

| Scope | What it grants |
|-------|---------------|
| `profile` | username, display name, avatar URL |
| `email` | email address |
| `oasisbios:read` | character list (title, slug, cover image, identity mode) |
| `oasisbios:full` | full character data including abilities, worlds, eras, references |
| `dcos:read` | DCOS document content for a character |

Request only the scopes you need. Users see a consent screen listing exactly what they're granting.

---

## API Endpoints

### Authorization

```
GET /oauth/authorize
```

Displays the consent page. Required params: `client_id`, `redirect_uri`, `response_type=code`, `scope`, `code_challenge`, `code_challenge_method=S256`. Recommended: `state`.

### Token Exchange

```
POST /api/oauth/token
Content-Type: application/x-www-form-urlencoded
```

Supports `authorization_code` and `refresh_token` grant types.

### User Info

```
GET /api/oauth/userinfo
Authorization: Bearer <access_token>
```

Returns `{ sub, username, display_name, avatar_url, email? }`.

### Character Resources

```
GET /api/oauth/resources/oasisbios                    # list (oasisbios:read)
GET /api/oauth/resources/oasisbios/:id                # full detail (oasisbios:full)
GET /api/oauth/resources/oasisbios/:id/dcos           # DCOS docs (dcos:read)
```

### Token Revocation

```
POST /api/oauth/revoke
Body: token=<token>&token_type_hint=access_token
```

Always returns `200 OK`.

### OIDC Discovery

```
GET /api/oauth/.well-known/openid-configuration
```

Machine-readable configuration for OIDC clients.

---

## Token Lifetimes

| Token | Lifetime |
|-------|---------|
| Authorization code | 10 minutes, single-use |
| Access token | 1 hour |
| Refresh token | 30 days |

Refresh tokens are rotated on each use — the old token is immediately invalidated.

---

## Error Responses

All OAuth errors follow RFC 6749 format:

```json
{
  "error": "invalid_grant",
  "error_description": "Authorization code already used"
}
```

| Error | Cause |
|-------|-------|
| `invalid_client` | Unknown `client_id` or wrong `client_secret` |
| `invalid_grant` | Code expired, already used, or PKCE verifier mismatch |
| `invalid_request` | Missing required parameters |
| `invalid_scope` | Requested scope not recognized |
| `access_denied` | User denied the authorization request |
| `unsupported_grant_type` | Grant type not supported |

---

## Security Checklist

- Store `client_secret` server-side only — never in frontend code or version control
- Always validate the `state` parameter on callback to prevent CSRF
- Store `code_verifier` in `sessionStorage`, not `localStorage`
- Use HTTPS for all redirect URIs in production
- Request only the scopes your app actually needs
- Revoke tokens when the user disconnects your app

---

## React Component Example

```tsx
function ContinueWithOasis({
  clientId,
  redirectUri,
  scope = 'profile email',
}: {
  clientId: string;
  redirectUri: string;
  scope?: string;
}) {
  const handleClick = async () => {
    const verifier = generateRandomString(64);
    const challenge = base64url(await sha256(verifier));
    sessionStorage.setItem('pkce_verifier', verifier);

    const url = new URL('https://oasisbio.com/oauth/authorize');
    url.searchParams.set('client_id', clientId);
    url.searchParams.set('redirect_uri', redirectUri);
    url.searchParams.set('response_type', 'code');
    url.searchParams.set('scope', scope);
    url.searchParams.set('state', generateRandomString(16));
    url.searchParams.set('code_challenge', challenge);
    url.searchParams.set('code_challenge_method', 'S256');
    window.location.href = url.toString();
  };

  return (
    <button
      onClick={handleClick}
      style={{
        display: 'inline-flex', alignItems: 'center', gap: 10,
        padding: '10px 20px', background: '#000', color: '#fff',
        borderRadius: 8, border: 'none', cursor: 'pointer',
        fontSize: 15, fontWeight: 500,
      }}
    >
      <svg width="18" height="18" viewBox="0 0 24 24" fill="none">
        <circle cx="12" cy="12" r="10" fill="white" fillOpacity="0.15" />
        <circle cx="12" cy="12" r="4" fill="white" />
      </svg>
      Continue with Oasis
    </button>
  );
}
```

---

## Further Reading

- Full technical reference: [technical.md — Section 15](../technical.md#15-oauth-provider)
- Integration guide (in-app): [/developer/docs](/developer/docs)
- OIDC discovery: [/api/oauth/.well-known/openid-configuration](/api/oauth/.well-known/openid-configuration)
