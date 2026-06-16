# OAuth 2.0 + PKCE Flow Walkthrough

This document shows the complete OAuth 2.0 Authorization Code + PKCE flow for connecting an AI agent to a Servio-compliant plugin.

---

## Prerequisites

- You have obtained the manifest from `GET /{prefix}/manifest`
- You know the `authorization_url`, `token_url`, and your `client_id` from `auth.registered_clients`
- Your `client_id` is `claude` (substitute with your agent's canonical client ID)

---

## Step 1 — Discover Auth Endpoints

```http
GET https://example.com/api/ai-connect/manifest
```

**Response (excerpt)**:
```json
{
  "auth": {
    "type": "oauth2",
    "flow": "authorization_code",
    "authorization_url": "https://example.com/oauth/authorize",
    "token_url": "https://example.com/api/ai-connect/v1/oauth/token",
    "revoke_url": "https://example.com/api/ai-connect/v1/oauth/revoke",
    "pkce_required": true,
    "code_challenge_method": "S256",
    "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
    "scopes": {
      "read": "Read content and data",
      "write": "Create and modify content"
    },
    "registered_clients": {
      "claude": "Claude (Anthropic)",
      "chatgpt": "ChatGPT (OpenAI)"
    }
  }
}
```

---

## Step 2 — Generate PKCE Parameters

The agent generates these values client-side (never sent over the network in full):

```
code_verifier  = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"
                 (43-128 chars, URL-safe base64: A-Z a-z 0-9 - . _ ~)

code_challenge = BASE64URL(SHA256(ASCII(code_verifier)))
               = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

state          = "xK9Pq2rL8mN3vT5w"
                 (random, for CSRF protection)
```

**Python snippet**:
```python
import hashlib, base64, secrets, string

alphabet = string.ascii_letters + string.digits + '-._~'
code_verifier = ''.join(secrets.choice(alphabet) for _ in range(64))

digest = hashlib.sha256(code_verifier.encode('ascii')).digest()
code_challenge = base64.urlsafe_b64encode(digest).rstrip(b'=').decode('ascii')

state = secrets.token_urlsafe(16)
```

---

## Step 3 — Authorization Request

Direct the user to the authorization URL:

```
GET https://example.com/oauth/authorize
  ?response_type=code
  &client_id=claude
  &redirect_uri=urn%3Aietf%3Awg%3Aoauth%3A2.0%3Aoob
  &scope=read+write
  &state=xK9Pq2rL8mN3vT5w
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
```

The server presents a login + consent screen. The user:
1. Logs in with their platform credentials
2. Reviews requested scopes (`read`, `write`)
3. Clicks **Authorize**

For the OOB redirect, the server displays the authorization code on screen:

```
Authorization code: abc123def456ghi789jkl012
```

The agent captures this code (either by prompting the user to paste it, or via deep link on mobile).

---

## Step 4 — Token Exchange

```http
POST https://example.com/api/ai-connect/v1/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=abc123def456ghi789jkl012
&redirect_uri=urn%3Aietf%3Awg%3Aoauth%3A2.0%3Aoob
&client_id=claude
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

**Success response (`200 OK`)**:
```json
{
  "access_token": "dpc_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dpr_x9y8z7w6v5u4x9y8z7w6v5u4x9y8z7w6v5u4x9y8z7w6v5u4x9y8z7w6v5u4x9",
  "refresh_token_expires_in": 2592000,
  "scope": "read write"
}
```

**Error response (`400 Bad Request`)**:
```json
{
  "error": "invalid_grant",
  "error_description": "Authorization code expired"
}
```

**Store securely**:
- `access_token` — use in `Authorization: Bearer` header for all tool calls
- `refresh_token` — use to get a new access token when it expires
- Note expiry: `time.now() + expires_in` seconds

---

## Step 5 — Invoke a Tool

```http
POST https://example.com/api/ai-connect/v1/tools/wordpress.searchPosts
Authorization: Bearer dpc_a1b2c3d4e5f6...
Content-Type: application/json

{
  "query": "release notes",
  "limit": 5
}
```

**Success response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "posts": [
      {
        "id": 142,
        "title": "Release Notes v2.0",
        "excerpt": "We are excited to announce...",
        "url": "https://example.com/release-notes-v2",
        "date": "2026-06-01T10:00:00Z"
      }
    ],
    "total": 1
  }
}
```

**Unauthorized response (`401`)**:
```json
{
  "success": false,
  "code": "unauthorized",
  "message": "No authentication token provided. Use OAuth 2.0 flow."
}
```

**Rate limited response (`429`)**:
```json
{
  "success": false,
  "code": "rate_limited",
  "message": "Rate limit exceeded: 50 requests per minute",
  "data": {
    "retry_after": 47,
    "limit": 50,
    "current": 51
  }
}
```

---

## Step 6 — Refresh an Expired Token

When the access token expires (after `expires_in` seconds), use the refresh token:

```http
POST https://example.com/api/ai-connect/v1/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=dpr_x9y8z7w6v5u4...
&client_id=claude
```

**Response**: Same format as Step 4 success (new `access_token` + new `refresh_token`).

> **Note**: The old refresh token is invalidated on successful rotation. Store the new one.

---

## Step 7 — Revoke Tokens (Logout)

To end the session:

```http
POST https://example.com/api/ai-connect/v1/oauth/revoke
Content-Type: application/x-www-form-urlencoded

token=dpc_a1b2c3d4e5f6...
&client_id=claude
```

**Response (`200 OK`)**:
```json
{"revoked": true}
```

Both the access token and its associated refresh token are invalidated.

---

## Token Lifetime Summary

| Token | Lifetime | Action when expired |
|---|---|---|
| Authorization code | 10 minutes | Restart from Step 3 |
| Access token | 1 hour (3600 s) | Use refresh token (Step 6) |
| Refresh token | 30 days (2592000 s) | Restart full flow from Step 3 |

---

## Security Notes

- Never log or expose `code_verifier`, `access_token`, or `refresh_token`
- Always verify the `state` parameter matches what you sent in Step 3
- Store tokens encrypted at rest
- Use the `revoke` endpoint when the user explicitly disconnects
