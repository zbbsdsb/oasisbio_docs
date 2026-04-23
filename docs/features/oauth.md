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

To integrate "Continue with Oasis" into your application, you first need to register your app in the OasisBio Developer Portal:

1. **Log in** to your OasisBio account
2. Go to [/developer/apps/new](/developer/apps/new)
3. Fill in the required information:
   - **App Name**: The name of your application (will be shown to users during authorization)
   - **Description**: A brief description of your application (optional)
   - **Homepage URL**: The main URL of your application
   - **Redirect URIs**: One or more URLs where users will be redirected after authorization (must use HTTPS in production)
   - **Logo URL**: A URL to your application's logo (optional)
4. Click "Create App"
5. You'll receive:
   - `client_id` — public identifier, safe to include in frontend code
   - `client_secret` — shown only once, store securely server-side

**Important**: The client secret is only shown once. Make sure to save it securely before leaving the page.

### 2. Add the "Continue with Oasis" button

Add the following button to your login page. Replace `YOUR_CLIENT_ID` and `YOUR_REDIRECT_URI` with your actual values:

```html
<a href="https://oasisbio.com/oauth/authorize?client_id=YOUR_CLIENT_ID
         &redirect_uri=YOUR_REDIRECT_URI&response_type=code
         &scope=profile+email&state=RANDOM_STATE
         &code_challenge=YOUR_CODE_CHALLENGE&code_challenge_method=S256"
   style="display:inline-flex;align-items:center;gap:10px;padding:10px 20px;
          background:#000;color:#fff;border-radius:8px;text-decoration:none;
          font-family:sans-serif;font-size:15px;font-weight:500;">
  <img src="https://oasisbio.com/assets/oasis_logo.svg" width="20" height="17" alt="Oasis" />
  Continue with Oasis
</a>
```

### 3. Implement PKCE (Required)

PKCE (Proof Key for Code Exchange) is required for all authorization requests. Here's how to implement it:

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

// Include challenge in authorization URL
const url = new URL('https://oasisbio.com/oauth/authorize');
url.searchParams.set('client_id', 'YOUR_CLIENT_ID');
url.searchParams.set('redirect_uri', 'YOUR_REDIRECT_URI');
url.searchParams.set('response_type', 'code');
url.searchParams.set('scope', 'profile email');
url.searchParams.set('state', generateRandomString(16)); // CSRF protection
url.searchParams.set('code_challenge', challenge);
url.searchParams.set('code_challenge_method', 'S256');
window.location.href = url.toString();
```

### 4. Handle the callback

After the user approves, they're redirected to your `redirect_uri` with `?code=<code>&state=<state>`.

1. **Validate the state** to prevent CSRF attacks
2. **Exchange the code for tokens**:

```javascript
// Get code from URL
const urlParams = new URLSearchParams(window.location.search);
const code = urlParams.get('code');
const state = urlParams.get('state');

// Validate state (compare with stored state)
if (state !== storedState) {
  // Handle CSRF attack
  return;
}

// Get stored code verifier
const verifier = sessionStorage.getItem('pkce_verifier');
if (!verifier) {
  // Handle error
  return;
}

// Exchange code for tokens
const res = await fetch('https://oasisbio.com/api/oauth/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'authorization_code',
    client_id: 'YOUR_CLIENT_ID',
    client_secret: 'YOUR_CLIENT_SECRET', // Server-side only
    code: code,
    redirect_uri: 'YOUR_REDIRECT_URI',
    code_verifier: verifier,
  }),
});

if (!res.ok) {
  // Handle error
  const error = await res.json();
  console.error('Token exchange error:', error);
  return;
}

const { access_token, refresh_token, expires_in } = await res.json();

// Store tokens securely (e.g., in HttpOnly cookie for server-side apps)
```

### 5. Access user data

Once you have an access token, you can access the user's data:

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
// → { data: [{ id, title, slug, tagline, coverImageUrl, visibility, status, publishedAt, createdAt, updatedAt }], count: number }

// Full character data (requires 'oasisbios:full' scope)
const characterDetail = await fetch('https://oasisbio.com/api/oauth/resources/oasisbios/{id}', {
  headers: { Authorization: `Bearer ${access_token}` },
}).then(r => r.json());
// → { id, title, slug, tagline, coverImageUrl, visibility, status, publishedAt, createdAt, updatedAt, abilities: [...], eras: [...], worlds: [...], references: [...], models: [...] }

// DCOS documents (requires 'dcos:read' scope)
const dcosDocuments = await fetch('https://oasisbio.com/api/oauth/resources/oasisbios/{id}/dcos', {
  headers: { Authorization: `Bearer ${access_token}` },
}).then(r => r.json());
// → { data: [{ id, title, slug, content, folderPath, status, version, createdAt, updatedAt }], count: number }
```

### 6. Refresh tokens

Access tokens expire after 1 hour. Use the refresh token to get a new access token:

```javascript
const res = await fetch('https://oasisbio.com/api/oauth/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'refresh_token',
    client_id: 'YOUR_CLIENT_ID',
    client_secret: 'YOUR_CLIENT_SECRET',
    refresh_token: 'YOUR_REFRESH_TOKEN',
  }),
});

const { access_token: newAccessToken, refresh_token: newRefreshToken } = await res.json();

// Store new tokens, discard old ones
```

### 7. Revoke tokens

When the user logs out or disconnects your app, revoke the tokens:

```javascript
await fetch('https://oasisbio.com/api/oauth/revoke', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    token: 'YOUR_ACCESS_TOKEN',
    token_type_hint: 'access_token',
  }),
});

// Also revoke refresh token if needed
await fetch('https://oasisbio.com/api/oauth/revoke', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    token: 'YOUR_REFRESH_TOKEN',
    token_type_hint: 'refresh_token',
  }),
});
```

---

## PKCE Implementation

PKCE (Proof Key for Code Exchange) is **required** for all authorization requests. There is no non-PKCE flow.

PKCE adds an extra layer of security for public clients (like single-page applications) by preventing authorization code interception attacks. The implementation details are already covered in the [Quick Start](#quick-start) section, which includes:

1. Generating a random code verifier
2. Creating a code challenge from the verifier
3. Storing the verifier securely (in sessionStorage)
4. Including the challenge in the authorization request
5. Using the verifier to exchange the authorization code for tokens

Make sure to follow the implementation provided in the Quick Start section to ensure your integration is secure.

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

**Return formats:**
- `/api/oauth/resources/oasisbios`: `{ data: [...], count: number }`
- `/api/oauth/resources/oasisbios/:id`: Full character object with nested abilities, eras, worlds, references, and models
- `/api/oauth/resources/oasisbios/:id/dcos`: `{ data: [...], count: number }`

**Permission verification:** All resource endpoints verify that the authenticated user is the owner of the requested character. Attempting to access another user's character will return a 403 Forbidden error.

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

Machine-readable configuration for OIDC clients. Includes endpoints, supported scopes, and other OIDC metadata.

**Note:** The discovery document references a JWKS endpoint at `/api/oauth/.well-known/jwks.json`, but this endpoint is not currently implemented. For now, OIDC clients should use the issuer URL for token validation.

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

Here's a complete React component for the "Continue with Oasis" button that includes the full PKCE implementation:

```tsx
import React from 'react';

// Helper functions for PKCE
function generateRandomString(length: number): string {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~';
  return Array.from(crypto.getRandomValues(new Uint8Array(length)))
    .map(b => chars[b % chars.length]).join('');
}

async function sha256(plain: string): Promise<ArrayBuffer> {
  return crypto.subtle.digest('SHA-256', new TextEncoder().encode(plain));
}

function base64url(buffer: ArrayBuffer): string {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

interface ContinueWithOasisProps {
  clientId: string;
  redirectUri: string;
  scope?: string;
  onError?: (error: Error) => void;
}

export function ContinueWithOasis({
  clientId,
  redirectUri,
  scope = 'profile email',
  onError,
}: ContinueWithOasisProps) {
  const handleClick = async () => {
    try {
      // Generate PKCE code verifier and challenge
      const verifier = generateRandomString(64);
      const challenge = base64url(await sha256(verifier));
      
      // Store verifier in sessionStorage
      sessionStorage.setItem('pkce_verifier', verifier);
      
      // Generate state for CSRF protection
      const state = generateRandomString(16);
      sessionStorage.setItem('oauth_state', state);
      
      // Construct authorization URL
      const url = new URL('https://oasisbio.com/oauth/authorize');
      url.searchParams.set('client_id', clientId);
      url.searchParams.set('redirect_uri', redirectUri);
      url.searchParams.set('response_type', 'code');
      url.searchParams.set('scope', scope);
      url.searchParams.set('state', state);
      url.searchParams.set('code_challenge', challenge);
      url.searchParams.set('code_challenge_method', 'S256');
      
      // Redirect to authorization endpoint
      window.location.href = url.toString();
    } catch (error) {
      if (onError) {
        onError(error as Error);
      } else {
        console.error('Error initiating Oasis authorization:', error);
      }
    }
  };

  return (
    <button
      onClick={handleClick}
      style={{
        display: 'inline-flex', alignItems: 'center', gap: 10,
        padding: '10px 20px', background: '#000', color: '#fff',
        borderRadius: 8, border: 'none', cursor: 'pointer',
        fontSize: 15, fontWeight: 500,
        fontFamily: 'sans-serif',
      }}
    >
      <img 
        src="https://oasisbio.com/assets/oasis_logo.svg" 
        width="20" 
        height="17" 
        alt="Oasis" 
      />
      Continue with Oasis
    </button>
  );
}

// Example usage
function LoginPage() {
  return (
    <div>
      <h1>Login</h1>
      <ContinueWithOasis
        clientId="YOUR_CLIENT_ID"
        redirectUri="https://your-app.com/auth/callback"
        scope="profile email oasisbios:read"
        onError={(error) => {
          console.error('Authorization error:', error);
          // Show error message to user
        }}
      />
    </div>
  );
}
```

### Handling the Callback in React

Here's an example of how to handle the OAuth callback in a React component:

```tsx
import React, { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

function AuthCallback() {
  const navigate = useNavigate();

  useEffect(() => {
    const handleCallback = async () => {
      try {
        // Get code and state from URL
        const urlParams = new URLSearchParams(window.location.search);
        const code = urlParams.get('code');
        const state = urlParams.get('state');
        const error = urlParams.get('error');

        // Handle error response
        if (error) {
          const errorDescription = urlParams.get('error_description');
          console.error('Authorization error:', error, errorDescription);
          navigate('/login?error=' + error);
          return;
        }

        // Validate state
        const storedState = sessionStorage.getItem('oauth_state');
        if (state !== storedState) {
          console.error('Invalid state parameter');
          navigate('/login?error=invalid_state');
          return;
        }

        // Get stored code verifier
        const verifier = sessionStorage.getItem('pkce_verifier');
        if (!verifier || !code) {
          console.error('Missing code or verifier');
          navigate('/login?error=missing_parameters');
          return;
        }

        // Exchange code for tokens
        const response = await fetch('https://oasisbio.com/api/oauth/token', {
          method: 'POST',
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
          body: new URLSearchParams({
            grant_type: 'authorization_code',
            client_id: 'YOUR_CLIENT_ID',
            client_secret: 'YOUR_CLIENT_SECRET', // Server-side only
            code: code,
            redirect_uri: 'https://your-app.com/auth/callback',
            code_verifier: verifier,
          }),
        });

        if (!response.ok) {
          const errorData = await response.json();
          console.error('Token exchange error:', errorData);
          navigate('/login?error=token_exchange_failed');
          return;
        }

        const { access_token, refresh_token } = await response.json();

        // Store tokens securely
        // For server-side apps: send to your backend and store in HttpOnly cookie
        // For client-side apps: use a secure storage solution

        // Clean up sessionStorage
        sessionStorage.removeItem('pkce_verifier');
        sessionStorage.removeItem('oauth_state');

        // Redirect to protected area
        navigate('/dashboard');
      } catch (error) {
        console.error('Callback handling error:', error);
        navigate('/login?error=callback_failed');
      }
    };

    handleCallback();
  }, [navigate]);

  return (
    <div className="flex items-center justify-center min-h-screen">
      <div className="text-center">
        <h1>Processing login...</h1>
        <p>Please wait while we complete your authentication.</p>
      </div>
    </div>
  );
}
```

---

## Further Reading

- Full technical reference: [technical.md — Section 15](../technical/technical.md#15-oauth-provider)
- Integration guide (in-app): [/developer/docs](/developer/docs)
- OIDC discovery: [/api/oauth/.well-known/openid-configuration](/api/oauth/.well-known/openid-configuration)
