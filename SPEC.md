# Servio Protocol Specification

**Version:** 1.2.0 | **Status:** Stable | **Date:** 2026-11-20

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Terminology](#2-terminology)
3. [Transport](#3-transport)
4. [Discovery](#4-discovery)
5. [Authentication](#5-authentication)
6. [Tools](#6-tools)
7. [Error Handling](#7-error-handling)
8. [Versioning & Compatibility](#8-versioning--compatibility)
9. [Security Considerations](#9-security-considerations)
10. [Reference Implementations](#10-reference-implementations)

---

## 1. Introduction

The **Servio Protocol** is a server-side, REST-based open standard for exposing structured tools from web platforms to AI agents. It defines how any web platform publishes a machine-readable manifest of its capabilities, authenticates AI agents via OAuth 2.0 with PKCE, and executes tool calls over standard HTTPS.

### 1.1 Design Goals

- **Universal** — Any platform can implement Servio using only standard HTTP. No framework-specific runtime dependencies.
- **Self-describing** — The manifest endpoint fully describes capabilities, authentication endpoints, and all available tools. AI agents need only the manifest URL to begin.
- **Secure by default** — OAuth 2.0 Authorization Code flow with mandatory PKCE S256, token expiry, and revocation.
- **AI-agent-first** — Tool names, descriptions, and instructions are written for AI agent consumption, not human developers.
- **Production-ready** — Designed for shared hosting environments where browser runtimes and local server processes are unavailable.

### 1.2 Relation to Other Protocols

Servio occupies a distinct position in the AI tooling landscape:

| Protocol | Transport | Auth | Deployment context |
|---|---|---|---|
| **Servio** | REST / HTTPS | OAuth 2.0 + PKCE (mandatory) | Server-to-server, production hosting |
| Anthropic MCP | JSON-RPC (STDIO / SSE) | Optional OAuth 2.0 | Local developer tooling |
| Google WebMCP | JavaScript browser API | Browser session | Chrome browser only |

Servio is the right choice when:
- The platform is a web application (CMS, forum, e-commerce, etc.)
- AI agents need persistent, authenticated access
- Multiple AI agents from different providers need to connect
- No browser or local process is available

### 1.3 Pre-registered AI Agents

Servio implementations SHOULD pre-register the following AI agents as OAuth clients, enabling them to initiate the OAuth flow without manual registration by the site administrator:

| Agent | Provider | Canonical `client_id` |
|---|---|---|
| Claude | Anthropic | `claude` |
| ChatGPT | OpenAI | `chatgpt` |
| Gemini | Google | `gemini` |
| Grok | xAI | `grok` |
| Copilot | Microsoft | `copilot` |
| Perplexity | Perplexity AI | `perplexity` |
| Meta AI | Meta | `meta-ai` |
| DeepSeek | DeepSeek | `deepseek` |

Implementations SHOULD also accept variant `client_id` values (e.g. `claude_client`, `claude-ai`) by normalizing to the canonical base ID.

---

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

| Term | Definition |
|---|---|
| **Servio Protocol** | This specification. The open standard for AI agent integration with web platforms. |
| **Plugin** | A Servio-compliant software component that implements this protocol for a specific platform. Also called a "Servio bridge." |
| **Platform** | The underlying web application (WordPress, XenForo, Drupal, Shopify, etc.) on which the plugin is installed. |
| **AI Agent** | Any software client that calls the Servio API to discover and invoke tools on behalf of a user. |
| **Manifest** | The JSON document returned by `GET /{prefix}/manifest`. Describes plugin identity, OAuth endpoints, and available tools. |
| **Tool** | A named, schema-defined operation that an AI agent can invoke via the tools endpoint. |
| **Bearer Token** | An OAuth 2.0 access token transmitted in the `Authorization: Bearer` HTTP header. |
| **PKCE** | Proof Key for Code Exchange (RFC 7636). A mandatory extension to the Authorization Code flow. |
| **Scope** | A permission category (`read`, `write`, `delete`, `admin`) that restricts tool access. |
| **URL prefix** | The platform-specific base path shared by all Servio endpoints (e.g. `/api/ai-connect`). |

---

## 3. Transport

### 3.1 Protocol

- All production endpoints MUST be served over HTTPS (TLS 1.2 or higher).
- HTTP (plaintext) MAY be used only for local development.
- All request and response bodies MUST use `Content-Type: application/json` unless the endpoint specifies `application/x-www-form-urlencoded` (OAuth token endpoints).
- Servers MUST respond with appropriate HTTP status codes as specified in this document.

### 3.2 URL Prefix

Each platform implementation defines its own URL prefix. The prefix is stable and forms the common base path for all Servio endpoints on that platform.

**Recommended prefix**: `/api/ai-connect`

**Known platform prefixes**:

| Platform | Prefix |
|---|---|
| WordPress | `/wp-json/goldt-webmcp-bridge` |
| XenForo | `/api/ai-connect` |
| Drupal | `/api/ai-connect` |
| Shopify | `/api/ai-connect` |

### 3.3 Required Endpoints

Every compliant plugin MUST implement all of the following endpoints:

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/{prefix}/manifest` | GET | None | Discovery manifest |
| `/{prefix}/oauth/authorize` | GET | None | OAuth consent screen |
| `/{prefix}/oauth/token` | POST | None | Token exchange and refresh |
| `/{prefix}/oauth/revoke` | POST | None | Token revocation (RFC 7009) — declared as `auth.revocation_url` |
| `/{prefix}/v1/tools/{tool_name}` | POST | Bearer | Tool execution |

---

## 4. Discovery

### 4.1 Manifest Endpoint

```
GET /{prefix}/manifest
```

- **Authentication**: None. MUST be publicly accessible without any token.
- **Content-Type**: `application/json`
- **Caching**: Implementations MAY cache the manifest. Fields that vary per authenticated session (e.g. `auth.registered_clients`, `instructions`) SHOULD be generated per-request.

### 4.2 Manifest Document

The manifest is a JSON object that fully describes the plugin. It is the sole configuration source for AI agents connecting to the platform.

**Required fields**: `schema_version`, `name`, `version`, `api_version`, `capabilities`, `server`, `auth`, `tools`

```json
{
  "schema_version": "1.0",
  "name": "my-platform-ai-connect",
  "version": "1.0.0",
  "description": "Servio bridge for My Platform",
  "api_version": "v1",
  "capabilities": {
    "tools": true,
    "resources": false,
    "prompts": false
  },
  "server": {
    "url": "https://example.com/api/ai-connect/v1",
    "description": "My Platform AI Connect API"
  },
  "auth": {
    "type": "oauth2",
    "flow": "authorization_code",
    "authorization_url": "https://example.com/oauth/authorize",
    "token_url": "https://example.com/api/ai-connect/v1/oauth/token",
    "revocation_url": "https://example.com/api/ai-connect/v1/oauth/revoke",
    "pkce_required": true,
    "code_challenge_method": "S256",
    "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
    "scopes": {
      "read": "Read content and data",
      "write": "Create and modify content",
      "delete": "Delete content",
      "admin": "Administrative access"
    },
    "registered_clients": {
      "claude": "Claude (Anthropic)",
      "chatgpt": "ChatGPT (OpenAI)",
      "gemini": "Gemini (Google)"
    }
  },
  "tools": [
    {
      "name": "myplatform.getContent",
      "description": "Retrieve content by ID.",
      "input_schema": {
        "type": "object",
        "properties": {
          "id": { "type": "integer", "description": "Content ID" }
        },
        "required": ["id"]
      }
    }
  ],
  "instructions": "You have access to My Platform via Servio. Use the available tools to read and manage content.",
  "usage": {
    "tools_endpoint": "https://example.com/api/ai-connect/v1/tools/{tool_name}",
    "method": "POST",
    "headers": {
      "Authorization": "Bearer {access_token}",
      "Content-Type": "application/json"
    }
  }
}
```

### 4.3 Manifest Field Reference

#### Top-level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `schema_version` | string | ✅ | Protocol schema version. MUST be `"1.0"`. |
| `name` | string | ✅ | Plugin identifier slug (e.g. `drupal-ai-connect`). |
| `version` | string | ✅ | Plugin version in semver format (e.g. `1.5.0`). |
| `description` | string | — | Human-readable description of the plugin. |
| `api_version` | string | ✅ | API version. MUST be `"v1"`. |
| `capabilities` | object | ✅ | Declared capability flags (see below). |
| `server` | object | ✅ | Base URL for tool execution. |
| `auth` | object | ✅ | OAuth 2.0 configuration (see Section 5). |
| `tools` | array | ✅ | Array of tool definitions (see Section 6). MUST NOT be empty. |
| `instructions` | string | — | Plain-text instructions for the AI agent. May be personalized per request. |
| `usage` | object | — | Informational hints for tool invocation. |

#### `capabilities` Object

| Field | Type | Required | Constraint |
|---|---|---|---|
| `capabilities.tools` | boolean | ✅ | MUST be `true` when tools are defined. |
| `capabilities.resources` | boolean | ✅ | MUST be `false` unless resource endpoints are fully implemented. |
| `capabilities.prompts` | boolean | ✅ | MUST be `false` unless prompt endpoints are fully implemented. |

#### `server` Object

| Field | Type | Required | Description |
|---|---|---|---|
| `server.url` | string (URI) | ✅ | Base URL for all tool execution endpoints. |
| `server.description` | string | — | Human-readable description of the API. |

#### `auth` Object

| Field | Type | Required | Constraint |
|---|---|---|---|
| `auth.type` | string | ✅ | MUST be `"oauth2"`. |
| `auth.flow` | string | ✅ | MUST be `"authorization_code"`. |
| `auth.authorization_url` | string (URI) | ✅ | URL of the OAuth consent screen. |
| `auth.token_url` | string (URI) | ✅ | URL for token exchange and refresh. |
| `auth.revocation_url` | string (URI) | ✅ (since 1.1) | Absolute **https** URL of the RFC 7009 revocation endpoint (Section 5.2, Step 5). MUST be reachable without redirects. Platforms that hold tokens on a user's behalf call it when access is cut on their side. |
| `auth.revoke_url` | string (URI) | — | **Deprecated** alias of `revocation_url` (1.0). Clients SHOULD read `revocation_url` first, then `revocation_endpoint`, then `revoke_url`. |
| `auth.pkce_required` | boolean | ✅ | MUST be `true`. |
| `auth.code_challenge_method` | string | ✅ | MUST be `"S256"`. |
| `auth.redirect_uri` | string | — | Default redirect URI. SHOULD include `"urn:ietf:wg:oauth:2.0:oob"`. |
| `auth.scopes` | object | ✅ | Map of scope name → human-readable description. |
| `auth.registered_clients` | object | — | Map of `client_id` → `client_name`. Exposed so agents can discover valid client IDs. |

### 4.4 Capability Declaration Rules

- `capabilities.resources` MUST NOT be `true` unless resource list and read endpoints are registered and functional.
- `capabilities.prompts` MUST NOT be `true` unless prompt list and get endpoints are registered and functional.
- Manifests MUST NOT claim capabilities that are not implemented.

### 4.5 The `instructions` Field

The `instructions` field provides free-text guidance for the AI agent. It is generated server-side and MAY be personalized based on the site's content, configuration, or the authenticated user.

**Recommended content**:
- List of available tool names
- Search tips (empty search = latest content, date filter syntax)
- Platform-specific constraints
- Translation capability notice (if applicable)

---

## 5. Authentication

### 5.1 Overview

Servio uses the OAuth 2.0 Authorization Code flow with PKCE as defined in [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749) and [RFC 7636](https://www.rfc-editor.org/rfc/rfc7636). All tool execution endpoints MUST require a valid Bearer token. The manifest endpoint is always public.

### 5.2 PKCE Flow — Step by Step

#### Step 1 — Generate PKCE Parameters (Client Side)

The agent generates a cryptographically random `code_verifier` (43–128 characters, URL-safe base64 alphabet: `[A-Z] [a-z] [0-9] [-._~]`) and derives the `code_challenge`:

```
code_challenge = BASE64URL(SHA256(ASCII(code_verifier)))
```

Where `BASE64URL` uses URL-safe base64 without padding (`-` instead of `+`, `_` instead of `/`, no `=`).

#### Step 2 — Authorization Request

The agent directs the user to the authorization URL:

```
GET /{prefix}/oauth/authorize
  ?response_type=code
  &client_id={client_id}
  &redirect_uri=urn:ietf:wg:oauth:2.0:oob
  &scope=read+write
  &state={random_state_value}
  &code_challenge={code_challenge}
  &code_challenge_method=S256
```

The server presents a consent screen. The user authenticates and approves. On approval, the server returns an `authorization_code`. For the OOB redirect (`urn:ietf:wg:oauth:2.0:oob`), the code is displayed on screen for the agent to copy.

**Server MUST**:
- Reject requests without `code_challenge` and `code_challenge_method=S256`
- Store the `code_challenge` for verification in Step 3
- Bind the code to the `client_id` and `redirect_uri`
- Expire the authorization code within 10 minutes
- Validate `client_id` and `redirect_uri` (exact match, §5.7) **before** showing the consent screen and again before acting on the user's approve **or** deny answer
- Return `state` unchanged with the code (or with the error) when the redirect URI is valid

**Server MUST NOT**:
- Redirect the browser to a `redirect_uri` that failed validation — not on error, not on deny. Show an error page instead (§5.7.3)
- Add the requested `redirect_uri` to a client's registration as a side effect of an authorization request

#### Step 3 — Token Exchange

```http
POST /{prefix}/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code={authorization_code}
&redirect_uri=urn:ietf:wg:oauth:2.0:oob
&client_id={client_id}
&code_verifier={code_verifier}
```

The server MUST:
- Verify `code_verifier` against stored `code_challenge`: `BASE64URL(SHA256(code_verifier)) == code_challenge`
- Verify that `client_id` equals the client the code was issued to, and that `redirect_uri` is **identical** (string comparison) to the value bound to the code. A code issued for the OOB URI MAY be exchanged without `redirect_uri`; for any other URI the parameter is REQUIRED. Any mismatch → `400 invalid_grant`
- Mark the authorization code as used (single-use)
- Return access and refresh tokens on success
- Answer every error with a 4xx status per §7.3 — never `200 OK` with an `error` body

**Success response (`200 OK`)**:

```json
{
  "access_token": "dpc_a1b2c3...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dpr_x9y8z7...",
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

#### Step 4 — Token Refresh

```http
POST /{prefix}/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token={refresh_token}
&client_id={client_id}
```

The server MUST:
- Revoke the old refresh token on successful exchange (rotation)
- Issue a new access token and a new refresh token

**Response**: Same format as Step 3 success response.

#### Step 5 — Token Revocation (RFC 7009)

```http
POST {auth.revocation_url}
Content-Type: application/x-www-form-urlencoded

token={access_token_or_refresh_token}
&token_type_hint={access_token|refresh_token}   (optional)
&client_id={client_id}                          (optional)
```

Requirements (MUST unless stated):

| # | Requirement |
|---|---|
| 1 | A dedicated `POST` route, declared in the manifest as `auth.revocation_url` (absolute https URL, no redirects). MUST accept `application/x-www-form-urlencoded`; SHOULD also accept `application/json`. |
| 2 | **No authentication.** Possession of the token is the authorisation (public PKCE clients). The route MUST NOT require a Bearer token, session cookie, API key or CSRF token. |
| 3 | `token` is required. `token_type_hint` is optional and only changes which token type is looked up first. |
| 4 | `token` may be the access token **or** the refresh token of a pair. Either one revokes the **whole pair** — neither half may keep working. |
| 5 | Implementations storing token hashes MUST look up by hash first and MAY fall back to plaintext for legacy rows. |
| 6 | Response `200 OK` for any well-formed request — including unknown or already-revoked tokens — so the endpoint cannot be used to probe token validity. `400` with error `invalid_request` only when `token` is missing. |
| 7 | Revocation takes effect **immediately**: the next tool call with the access token MUST get `401`, and a refresh with the refresh token MUST fail (`invalid_grant`). Secondary token registries/caches MUST be updated in the same request. |
| 8 | If the plugin keeps an audit log, the revocation SHOULD be recorded (never log the token value). |

Callers (e.g. agent platforms) typically send two requests per connection: the refresh token first, then the access token; any `2xx` is treated as success.

For `auth.type = "mcp_token_via_host_oauth"` plugins the endpoint revokes the plugin-issued token only — never the host platform's own credentials.

**Success response (`200 OK`)** — body is informational:

```json
{"revoked": true}
```

**Conformance test** (each step MUST hold):
1. manifest → `auth.revocation_url` present (https).
2. obtain a token pair; a tool call succeeds.
3. revoke the refresh token (`token_type_hint=refresh_token`) → `200`.
4. tool call with the access token of the same pair → `401`.
5. refresh with the revoked refresh token → fails.
6. `token=unknown` → `200`.
7. no `token` → `400 invalid_request`.
8. new pair, revoke by access token without hint → refresh also fails.

### 5.3 Bearer Token Usage

All tool requests MUST include the Authorization header:

```
Authorization: Bearer {access_token}
```

Missing or invalid tokens MUST result in `401 Unauthorized`.

### 5.4 Token Lifetimes

| Token | Minimum | Maximum | Recommended |
|---|---|---|---|
| Authorization code | — | — | 10 minutes |
| Access token | 1 hour | 24 hours | 1 hour (3600 s) |
| Refresh token | 7 days | 90 days | 30 days (2592000 s) |

### 5.5 Scope System

Each tool declares the minimum scope required for invocation. The agent's access token MUST include that scope.

| Scope | Typical operations |
|---|---|
| `read` | Get, search, list, retrieve |
| `write` | Create, update, publish, modify |
| `delete` | Delete, trash, purge |
| `admin` | Manage users, configure settings, system operations |

If the token lacks the required scope, the server MUST return `403 Forbidden` with error code `forbidden` or `insufficient_scope`.

### 5.6 Client Registration

AI agents identify themselves with a `client_id` during the authorization request. Reference implementations pre-register the canonical agent client IDs listed in Section 1.3.

The `auth.registered_clients` field in the manifest exposes the registered client IDs as a `{client_id: client_name}` map. AI agents SHOULD use this to discover the correct `client_id` for their provider.

Implementations MAY accept variant `client_id` values (e.g. `claude_client`, `gemini_ai`) by normalizing them to the canonical base ID. Accepted variants SHOULD be auto-registered as aliases.

### 5.7 Redirect URI Validation (since 1.2.0)

This section is normative and follows [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700) §2.1 / §4.1 (OAuth 2.0 Security Best Current Practice), OAuth 2.1 §7.5 and [RFC 8252](https://www.rfc-editor.org/rfc/rfc8252) §7.3. It replaces the looser wording of §9.8 in 1.0/1.1.

#### 5.7.1 Exact match

- Every client has a list of **registered** redirect URIs. The server MUST compare the `redirect_uri` of an authorization request with that list by **exact string comparison** (no prefix, substring, wildcard, case-folding or path-normalisation match).
- The only exception: a registered **loopback** URI of a **native** client (`http://127.0.0.1/...`, `http://[::1]/...`, `http://localhost/...`) matches a request URI that differs from it **only in the port** (RFC 8252 §7.3). The scheme, host and path MUST still match exactly. This exception MUST NOT apply to any non-loopback host.
- `urn:ietf:wg:oauth:2.0:oob` is accepted only when it is registered for the client (all pre-registered agents of §1.3 register it). An absent `redirect_uri` means OOB and is accepted only when OOB is registered.
- The same check runs at the authorization request, when the user's approve / deny answer is processed, and at the token endpoint (Step 3: identical to the value bound to the code).

#### 5.7.2 Registration

- Redirect URIs are registered by the site administrator or by a dedicated registration endpoint. An authorization request MUST NOT register, add or change a redirect URI.
- Implementations that auto-register unknown or variant `client_id` values (§5.6) MUST give the new client **only** the OOB URI (or copy the URIs of the canonical client it aliases) — never the URI from the request.

#### 5.7.3 Errors

- If `client_id` is unknown or `redirect_uri` does not match, the server MUST NOT redirect. It MUST answer with an error page (`400`) shown to the user (RFC 6749 §4.1.2.1).
- Only after the redirect URI is validated MAY the server report other errors (`invalid_request`, `access_denied`, `unsupported_response_type`, `invalid_scope`) by redirecting to it, with `error`, `error_description` and the unchanged `state`.
- A deny / cancel button on the consent screen MUST follow the same rule: no redirect to an unvalidated URI.
- The server MUST NOT accept any other request parameter (e.g. `return_to`, `next`, `redirect`) as a redirect target after login or consent unless it is a same-host relative path; anything else is an open redirect.

#### 5.7.4 PKCE

`code_challenge_method` MUST be `S256`. A request without it, or with any other value (including `plain`), MUST be rejected with `invalid_request` (redirected only if the redirect URI is valid). Servers MUST read the method value without lossy filtering (e.g. a filter that strips digits turns `S256` into `S`).

#### 5.7.5 Conformance test

Each step MUST hold (a registered client `C` with registered URI `R`):
1. authorize with `client_id=C&redirect_uri=https://attacker.example/cb` → `400` error page, no `Location` header to that host, no consent screen.
2. same with the deny answer submitted → no redirect to the unregistered URI.
3. authorize with `redirect_uri=R` + a random `state` → after approval, the code is delivered to `R` with the same `state`.
4. token exchange with that code and a different `redirect_uri` (or another `client_id`) → `400 invalid_grant`; the code is not consumed by the failed attempt or is invalid for every later exchange.
5. `code_challenge_method=plain` → rejected.
6. after steps 1–2, the client's registered URIs are unchanged.
7. token endpoint with `grant_type=authorization_code&code=garbage` → `400` + `{"error":"invalid_grant"}`; no client → `400`/`401` `invalid_client` / `invalid_request` — never `200`.

---

## 6. Tools

### 6.1 Tool Definition

Each entry in the `tools` array is a tool definition object:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | `{platform}.{toolName}` format. Must be unique within the manifest. |
| `description` | string | ✅ | Written for AI agent consumption. Explains purpose, return value, and when to call. |
| `input_schema` | object | ✅ | JSON Schema object with `type: "object"` and `properties`. |
| `input_schema.type` | string | ✅ | MUST be `"object"`. |
| `input_schema.properties` | object | ✅ | Parameter definitions. Each property has `type` and `description`. |
| `input_schema.required` | array | — | Names of required parameters. |

### 6.2 Tool Naming Convention

Tool names MUST follow the pattern `{platform}.{toolName}` where:

- `{platform}` is the lowercase platform identifier (e.g. `wordpress`, `xenforo`, `drupal`, `shopify`)
- `{toolName}` is camelCase, starting with an action verb

Tool names MUST be unique within a manifest. Two tools on the same platform MUST NOT share a name.

**Valid examples**:

| Tool name | Platform | Action |
|---|---|---|
| `wordpress.searchPosts` | WordPress | Search published posts by keyword |
| `wordpress.createPost` | WordPress | Create a new WordPress post |
| `wordpress.getPost` | WordPress | Retrieve a single post by ID |
| `xenforo.searchThreads` | XenForo | Search forum threads |
| `xenforo.getThread` | XenForo | Retrieve a thread by ID |
| `xenforo.getCurrentUser` | XenForo | Get the authenticated user's profile |
| `drupal.getNode` | Drupal | Retrieve a content node by ID |
| `drupal.searchContent` | Drupal | Search Drupal content |
| `shopify.getProduct` | Shopify | Retrieve a Shopify product |
| `shopify.fulfillOrder` | Shopify | Fulfill a pending Shopify order |
| `translation.translate` | Translation module | Translate text between languages |
| `translation.getSupportedLanguages` | Translation module | List available language pairs |

### 6.3 Tool Invocation

```http
POST /{prefix}/v1/tools/{platform}.{toolName}
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "param_name": "value"
}
```

The full tool name appears in the URL path. The request body contains only the tool's input parameters as a flat JSON object matching `input_schema`.

### 6.4 Success Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "success": true,
  "data": {
    "id": 42,
    "title": "My Post",
    "content": "..."
  }
}
```

### 6.5 Pending Confirmation Response

Destructive tools (delete, bulk operations) MAY implement a two-step confirmation flow. On the first invocation, the server returns `202 Accepted` instead of executing:

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "status": "pending_confirmation",
  "confirmation_id": "conf_a1b2c3",
  "message": "This will permanently delete 12 posts. Confirm to proceed.",
  "expires_at": "2026-06-16T13:00:00Z"
}
```

To confirm, the agent re-submits the original request with `confirmation_id` added to the request body:

```json
{
  "param_name": "value",
  "confirmation_id": "conf_a1b2c3"
}
```

#### 6.5.1 User confirmation (`user-confirm-v1`, since 1.2.0)

For **destructive and financial** tools (refunds, deletions, cancelling orders, risky site
administration) a server MUST require the **user's own confirmation**, enforced in code — the
agent can never confirm by itself (an agent-supplied `confirm: true` or similar flag is NOT a
confirmation). Servers:

- Mark the tool in the manifest with `"confirmation": "required"` (or `"conditional"` when only some
  argument values need it, e.g. `status=cancelled`) and `"risk": "financial" | "destructive"`, and
  describe the flow in a top-level `confirmation` object:
  `{ "flow": "user-confirm-v1", "pending_status_code": 202, "error_code": "confirmation_required",
  "status_url_template": "…/{confirmation_id}", "confirmed_by": "user", "tools": [ … ] }`.
- Answer the first call with `202` and this body (the 6.5 fields plus `code`, `preview`,
  `confirm_url`, `status_url`; the same keys repeated under `data`):

```json
{
  "success": false,
  "status": "pending_confirmation",
  "code": "confirmation_required",
  "message": "Not executed. Show the user the preview and confirm_url …",
  "confirmation_id": "conf_a1b2c3…",
  "confirm_url": "https://site.example/wp-admin/users.php?page=goldtwmcp-ai-actions&confirmation_id=conf_a1b2c3…",
  "status_url": "https://site.example/wp-json/goldt-webmcp-bridge/v1/confirmations/conf_a1b2c3…",
  "expires_at": "2026-11-15T09:00:00Z",
  "tool": "edd.refundOrder",
  "risk": "financial",
  "preview": { "order_id": 42, "total": 49.0, "currency": "USD", "gateway": "stripe" }
}
```

- MUST put `confirm_url` and `status_url` on the **same host** as the manifest (never a third-party
  or agent-supplied host).
- MUST run the action **only** after the user confirms on `confirm_url` in their own signed-in session:
  the approve / deny form MUST require the platform's session cookie **and** a CSRF token bound to
  that session (WordPress nonce, XenForo `_xfToken`, Drupal form token, Moodle `sesskey`, …), MUST use
  `POST`, and MUST refuse requests carrying a Bearer token or API key. A `GET` of `confirm_url` only
  shows the preview; it never approves.
- MUST bind the pending action to user + tool + arguments hash + client; expire it (15 min
  RECOMMENDED, at most 24 h); execute it at most once. The confirming user MUST be the user the
  pending action belongs to. Re-submitting with `confirmation_id` (as in 6.5) or `GET status_url` MUST
  NOT execute anything — it reports: still pending → `202`; executed → the stored result;
  rejected / expired → `410 confirmation_rejected` / `confirmation_expired`; other tool or other
  arguments → `409 confirmation_mismatch`.
- Ignore any agent-supplied `confirm: true` argument for these tools.
- MAY offer a standing **autonomy grant**: the user lets one client run one tool without asking (per
  user, per tool, per client, revocable at any time by the user, audited). Then the call executes
  directly and answers normally. A grant MUST be created by the user in their session (same CSRF
  rule) — never by the agent.

**Agents MUST:** show the user the `preview` and the `confirm_url`, say that nothing has happened
yet, and wait. Never re-call the tool on your own, never loop or poll aggressively, never claim it
was done. When the user says they confirmed, call the same tool with the same arguments plus
`confirmation_id` (or `GET status_url`) to get the real result. A rejected or expired confirmation is
final — ask the user before starting again.

### 6.6 Writing Tool Descriptions for AI Agents

Tool descriptions are the primary signal AI agents use to decide which tool to call. Write them for agents, not humans.

**Structure**:
1. What the tool does (one sentence, present tense)
2. What it returns
3. When to use it (trigger phrases)
4. Constraints (limits, required preconditions)

**Example**:
```json
{
  "name": "wordpress.searchPosts",
  "description": "Search published posts by keyword. Returns up to 20 results with title, excerpt, URL, and publication date. Use this when the user asks to find, look up, or search for content on the site. Empty search returns the most recent posts.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search keywords. Leave empty to retrieve latest posts."
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of results to return. Defaults to 10.",
        "minimum": 1,
        "maximum": 20
      }
    },
    "required": ["query"]
  }
}
```

**Best practices**:
- Accept raw user input. Do not require the agent to transform values before calling.
- Validate strictly in implementation, loosely in schema. Return descriptive `validation_error` responses so the agent can self-correct.
- Use specific verbs: `searchPosts` not `getPostSearch`; `createPost` not `postCreation`.

---

## 7. Error Handling

### 7.1 Tool Error Envelope

All errors from tool endpoints use a standard JSON envelope:

```json
{
  "success": false,
  "code": "error_code",
  "message": "Human-readable description suitable for AI agent self-correction"
}
```

For errors with supplemental data:

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

### 7.2 Standard Tool Error Codes

| Code | HTTP Status | Meaning |
|---|---|---|
| `unauthorized` | 401 | Missing or invalid Bearer token |
| `forbidden` | 403 | Token lacks the required scope |
| `insufficient_scope` | 403 | Token lacks scope for this specific tool |
| `not_found` | 404 | Tool does not exist in this plugin |
| `rate_limited` | 429 | Per-user rate limit exceeded |
| `user_blocked` | 403 | The authenticated user has been blocked |
| `validation_error` | 422 | Input parameters failed schema validation |
| `internal_error` | 500 | Server-side error |
| `confirmation_required` | 202 | Not executed — waiting for the user's confirmation (6.5.1) |
| `confirmation_mismatch` | 409 | `confirmation_id` used with another tool or other arguments |
| `confirmation_rejected` / `confirmation_expired` | 410 | The user rejected it / it expired; final |

### 7.3 OAuth Error Codes

OAuth token endpoint errors follow [RFC 6749 Section 5.2](https://www.rfc-editor.org/rfc/rfc6749#section-5.2) error codes. The HTTP status is normative (since 1.2.0): an error MUST NOT be answered with `200 OK`, and the body MUST be JSON with `Content-Type: application/json` and `Cache-Control: no-store`.

| Code | HTTP status | Meaning |
|---|---|---|
| `invalid_request` | 400 | Missing / duplicated / malformed parameter, or `code_challenge_method` not `S256` |
| `invalid_client` | 401 (400 when the client authenticated by no credential and is simply unknown) | Client ID mismatch or unknown client. With `401`, include `WWW-Authenticate` |
| `invalid_grant` | 400 | Authorization code / refresh token invalid, expired, revoked, already used, issued to another client, `redirect_uri` not identical, or PKCE verification failed |
| `unauthorized_client` | 400 | Client not allowed to use this grant type |
| `unsupported_grant_type` | 400 | Unknown `grant_type` |
| `invalid_scope` | 400 | Requested scope invalid or exceeds the grant |
| `server_error` | 500 | Server-side failure during token issuance |

**OAuth error response format** (RFC 6749):
```json
{
  "error": "invalid_grant",
  "error_description": "Authorization code expired"
}
```

### 7.4 Rate Limiting

Implementations MUST enforce per-user rate limits on all authenticated tool endpoints:

- **Per minute**: 50 requests maximum
- **Per hour**: 1000 requests maximum

The rate-limiting identifier MUST be the authenticated user ID, not the IP address.

When limits are exceeded, return `429 Too Many Requests` with:
- `code: "rate_limited"`
- `data.retry_after`: seconds until the limit window resets
- `data.limit`: the applicable limit
- `data.current`: the current request count

---

## 8. Versioning & Compatibility

### 8.1 Schema Version

The current protocol version is `1.0`. All manifests MUST declare:

```json
"schema_version": "1.0"
```

### 8.2 Versioning Policy

| Change type | Version increment | Example |
|---|---|---|
| Breaking change to manifest structure or tool invocation | Major (`2.0`) | Remove required field, change URL structure |
| New optional manifest field or endpoint | Minor (`1.1`) | Add optional field, new informational endpoint |
| Documentation clarification, example fix | Patch (`1.0.1`) | No implementation change required |

### 8.3 API Path Versioning

The tool execution path includes `/v1/`. Future breaking changes to the execution API will introduce a `/v2/` path. Implementations MAY support multiple API versions simultaneously during transition periods.

### 8.4 Backward Compatibility Commitment

A plugin declaring `schema_version: "1.0"` MUST:
- Accept any tool invocation valid under schema 1.0 for the lifetime of 1.x
- Return all required manifest fields defined in Section 4.3
- Not remove a published tool without a deprecation period

### 8.5 Known Limitations (v1.0)

The following limitations are acknowledged and tracked for future versions:

| Limitation | Status |
|---|---|
| No `/.well-known/` endpoint for automatic plugin discovery | Planned for v1.1 |
| `capabilities.resources` and `capabilities.prompts` not implemented in any reference plugin | Infrastructure declared; implementations pending |
| `output_schema` not yet defined for tool definitions | Planned for v1.1 |

---

## 9. Security Considerations

### 9.1 PKCE is Mandatory

PKCE with S256 is non-negotiable. Implementations MUST reject any authorization request missing `code_challenge` or with `code_challenge_method` other than `S256`. Plain `code_challenge_method=plain` is forbidden. This prevents authorization code interception attacks.

### 9.2 HTTPS in Production

All production deployments MUST use HTTPS with a valid TLS certificate. Transmitting Bearer tokens or authorization codes over HTTP exposes credentials to network attackers.

### 9.3 Token Expiry

- Access tokens MUST expire within 24 hours. Short-lived tokens limit the blast radius of token theft.
- Refresh tokens MUST expire within 90 days. Long-lived refresh tokens MUST be revocable individually.

### 9.4 Token Revocation

Implementations MUST provide the `/revoke` endpoint. Users MUST be able to revoke individual tokens from a management UI visible to the authenticated user. When a refresh token is revoked, its associated access token MUST also be revoked (cascade revocation).

### 9.5 User Blocking

Implementations SHOULD support per-user blocking. A blocked user's requests to any tool endpoint MUST return `403 Forbidden` with `code: "user_blocked"`, regardless of token validity.

### 9.6 Rate Limiting

Per-user rate limiting (Section 7.4) is REQUIRED. The identifier used for rate limiting MUST be the authenticated user ID, not the client IP. Using IP addresses would affect multiple users sharing NAT.

### 9.7 Scope Minimization

AI agents SHOULD request only the scopes necessary for the current session. Implementations MUST check the required scope on every tool invocation, not just at token issuance.

### 9.8 Redirect URI Validation

See §5.7 (normative since 1.2.0): exact string match against the client's registered URIs at the authorization request, at the approve / deny step and at the token endpoint; the port-only loopback exception applies only to registered loopback URIs of native clients; never redirect to an unvalidated URI (also not on error or deny); an authorization request never registers a URI.

### 9.9 Authorization Code Single Use

Authorization codes MUST be single-use. The server MUST mark a code as used immediately upon successful token exchange and reject any subsequent use of the same code.

### 9.10 State Parameter

AI agents SHOULD include a random `state` parameter in authorization requests to prevent CSRF attacks and MUST verify it on callback. Servers MUST return `state` unchanged — with the code and with every error redirect (§5.7.3).

### 9.11 Agent Tokens Only on Servio Routes (since 1.2.0)

An access token issued by a Servio plugin MUST authenticate **only** that plugin's own agent endpoints (manifest-declared tool / resource / revocation routes). The platform's native API, admin / AJAX routes, pages and the plugin's own browser-only screens (consent, confirmation, token management) MUST refuse it (`401`/`403`, or treat the request as anonymous). Otherwise a token scoped to a few tools becomes a full login for the user.

### 9.12 User Confirmation Forms (since 1.2.0)

The consent screen (§5.2 Step 2) and the confirmation screen (§6.5.1) change state only on `POST` with the platform's session cookie **and** a CSRF token bound to that session. They MUST refuse Bearer / API-key requests and MUST NOT approve on `GET`.

---

## 10. Reference Implementations

The following **Goldnat** plugins implement Servio Protocol v1.0 in production:

| Plugin | Platform | Version | Repository |
|---|---|---|---|
| Goldnat for WordPress | WordPress 5.0+ | 2.x | [chgold/goldt-wp-webmcp-bridge](https://github.com/chgold/goldt-wp-webmcp-bridge) |
| Goldnat for XenForo | XenForo 2.2+ | 1.2.x | [chgold/xf-ai-connect](https://github.com/chgold/xf-ai-connect) |
| Goldnat for Drupal | Drupal 10+ | 1.5.x | [chgold/drupal-ai-connect](https://github.com/chgold/drupal-ai-connect) |
| Goldnat for Shopify | Shopify | — | [chgold/shopify-goldt-webmcp-bridge](https://github.com/chgold/shopify-goldt-webmcp-bridge) |

### 10.1 Known Deviations from This Specification

The following deviations exist in production reference implementations. They are tracked and will be corrected in future plugin releases.

| Plugin | Section | Deviation | Impact |
|---|---|---|---|
| Goldnat for XenForo | §6.3 Tool Invocation | Uses `POST /api/aiconnect-tools` with `{"name": "...", "arguments": {...}}` body instead of REST path routing | Agents connecting to XenForo must use the body-based format documented in the manifest's `usage.body_format: "mcp"` field |
| Goldnat for WordPress | §4.4 Capability Rules | Some versions declare `capabilities.resources: true` without implementing resource endpoints | Agents should ignore declared `resources` capability until a tool using it appears in the `tools` array |
| Goldnat for Shopify | §5.1 Authentication | Declares `auth.type: "mcp_token"` instead of `"oauth2"` | Non-compliant; tracked for correction |

### 10.2 Compliance Rules

A plugin is compliant with Servio Protocol v1.0 when it satisfies all CRITICAL and MAJOR rules below:

| Rule | Severity | Description |
|---|---|---|
| RULE-001 | CRITICAL | Manifest endpoint `GET /{prefix}/manifest` MUST be publicly accessible |
| RULE-002 | MAJOR | Manifest MUST include all required fields (§4.3) |
| RULE-003 | MAJOR | Tool names MUST follow `{platform}.{toolName}` camelCase pattern |
| RULE-004 | CRITICAL | Tool execution endpoint MUST use HTTP POST |
| RULE-005 | CRITICAL | Tool execution endpoints MUST require Bearer token authentication |
| RULE-006 | CRITICAL | `auth.type` MUST be `"oauth2"` |
| RULE-007 | MAJOR | `capabilities.resources` MUST NOT be `true` unless resources are implemented |
| RULE-008 | MAJOR | Tool name MUST appear in the URL path, not in the request body |
| RULE-009 | CRITICAL | OAuth MUST use PKCE with S256; `pkce_required` MUST be `true` |
| RULE-010 | MAJOR | Access tokens MUST expire between 1 hour and 24 hours |
| RULE-011 | MAJOR | Refresh tokens MUST expire between 7 days and 90 days |
| RULE-012 | MAJOR | Per-user rate limiting MUST be implemented on all tool endpoints |
| RULE-013 | CRITICAL | `redirect_uri` exact match against registered URIs at authorize, approve/deny and token (§5.7); never redirect to an unvalidated URI |
| RULE-014 | CRITICAL | `code_challenge_method` other than `S256` MUST be rejected (§5.7.4) |
| RULE-015 | MAJOR | Token endpoint errors use RFC 6749 §5.2 codes with `400` / `401` (never `200`) (§7.3) |
| RULE-016 | CRITICAL | Destructive / financial tools run only after the user's own confirmation (§6.5.1): same-host `confirm_url`, session + CSRF, single use, bound to user + tool + arguments + client |
| RULE-017 | CRITICAL | Agent access tokens are refused outside the plugin's own agent routes (§9.11) |

---

*Servio Protocol is an open standard. Contributions and corrections are welcome via pull request to [github.com/chgold/servio](https://github.com/chgold/servio).*

---

## Changelog

- **1.2.0 (2026-11-20)** — new §5.7 redirect URI validation (exact match at authorize, approve/deny and token per RFC 9700 / OAuth 2.1; loopback port exception only for registered native-client loopback URIs; no redirect to an unvalidated URI, also on error/deny; requests never register URIs; PKCE `S256` only; conformance test); §5.2 Step 3 binds `client_id` + identical `redirect_uri`; §7.3 token-endpoint error codes with normative `400`/`401` status (never `200`); `state` returned unchanged; §9.11 agent tokens only on Servio routes; §9.12 CSRF on consent/confirmation forms; RULE-013..017. §6.5.1 user confirmation now MUST for destructive/financial tools, same-host `confirm_url`/`status_url`, session + CSRF + `POST`, user-created revocable autonomy grant. Also: §6.5.1 user confirmation (`user-confirm-v1`): manifest `confirmation` / `risk` per tool + top-level `confirmation` descriptor, 202 `confirmation_required` envelope with `preview` / `confirm_url` / `status_url`, user-only confirm step, single-use + replay-safe re-submission, optional per-user/tool/client autonomy grant, agent handling rules; error codes `confirmation_*`. Reference: GoldT WebMCP Bridge 1.2.13.
- **1.1.0 (2026-11-08)** — `auth.revocation_url` REQUIRED; Section 5.2 Step 5 rewritten as an RFC 7009 contract with a conformance test; `auth.revoke_url` deprecated as an alias.
- **1.0.0 (2026-06-16)** — initial specification.
