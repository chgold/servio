# Servio Protocol

**Version 1.0.0 — Stable**

Servio is an open standard for exposing structured tools from web platforms to AI agents. It defines a REST-based discovery and invocation API secured by OAuth 2.0 with PKCE.

Any platform that implements Servio becomes immediately accessible to AI agents — Claude, ChatGPT, Gemini, Grok, Copilot, and others — without custom integrations per agent.

---

## At a Glance

| Capability | Specification |
|---|---|
| Discovery | `GET /{prefix}/manifest` → JSON document |
| Authentication | OAuth 2.0 Authorization Code + PKCE (S256, mandatory) |
| Tool invocation | `POST /{prefix}/v1/tools/{platform}.{toolName}` |
| Transport | HTTPS (required in production) |
| Schema format | JSON Schema draft 2020-12 |
| Current version | `1.0` |

---

## Quick Start — Implementing Servio

### Step 1 — Expose the manifest endpoint

```
GET /{prefix}/manifest   (public, no auth)
```

Return a JSON document matching [schemas/manifest.schema.json](schemas/manifest.schema.json). The manifest describes your plugin identity, OAuth endpoints, and available tools.

### Step 2 — Implement OAuth 2.0 + PKCE

Expose three OAuth endpoints:

```
GET  /{prefix}/oauth/authorize    — consent screen
POST /{prefix}/oauth/token        — code exchange + refresh
POST /{prefix}/oauth/revoke       — token revocation
```

PKCE with S256 is **mandatory**. Reject any authorization request without `code_challenge` and `code_challenge_method=S256`.

### Step 3 — Register tools

Each tool has a `name`, `description`, and `input_schema`. Names follow `{platform}.{toolName}` camelCase.

### Step 4 — Expose the tools endpoint

```
POST /{prefix}/v1/tools/{platform}.{toolName}
Authorization: Bearer {access_token}
Content-Type: application/json

{"param": "value", ...}
```

See [SPEC.md](SPEC.md) for the full specification.

---

## Repository Contents

| File | Description |
|---|---|
| [SPEC.md](SPEC.md) | Full protocol specification |
| [CHANGELOG.md](CHANGELOG.md) | Version history |
| [schemas/manifest.schema.json](schemas/manifest.schema.json) | JSON Schema for the manifest document |
| [schemas/tool.schema.json](schemas/tool.schema.json) | JSON Schema for a single tool definition |
| [schemas/oauth-client.schema.json](schemas/oauth-client.schema.json) | JSON Schema for OAuth client registration |
| [schemas/error-response.schema.json](schemas/error-response.schema.json) | JSON Schema for error response envelopes |
| [examples/manifest-minimal.json](examples/manifest-minimal.json) | Minimal valid manifest |
| [examples/manifest-wordpress.json](examples/manifest-wordpress.json) | WordPress implementation example |
| [examples/manifest-xenforo.json](examples/manifest-xenforo.json) | XenForo implementation example |
| [examples/tool-readPost.json](examples/tool-readPost.json) | Example tool definition |
| [examples/oauth-flow.md](examples/oauth-flow.md) | OAuth 2.0 + PKCE flow walkthrough |

---

## Reference Implementations

The **Goldnat** plugin family implements Servio Protocol v1.0 in production across four platforms:

| Plugin | Platform | Repository |
|---|---|---|
| Goldnat for WordPress | WordPress 5.0+ | [chgold/goldt-wp-webmcp-bridge](https://github.com/chgold/goldt-wp-webmcp-bridge) |
| Goldnat for XenForo | XenForo 2.2+ | [chgold/xf-ai-connect](https://github.com/chgold/xf-ai-connect) |
| Goldnat for Drupal | Drupal 10+ | [chgold/drupal-ai-connect](https://github.com/chgold/drupal-ai-connect) |
| Goldnat for Shopify | Shopify | [chgold/shopify-goldt-webmcp-bridge](https://github.com/chgold/shopify-goldt-webmcp-bridge) |

---

## Status

Servio v1.0 is a **production protocol** actively used across 4 platforms with 88+ registered tools and 8 pre-registered AI agent clients.

---

## License

[Apache License 2.0](LICENSE)
