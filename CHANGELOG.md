# Changelog

All notable changes to the Servio Protocol specification are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Version numbers follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.0.0] — 2026-06-16

Initial public release of the Servio Protocol specification.

### Added

**Specification**
- `SPEC.md` — Full 10-section protocol specification covering Introduction, Terminology, Transport, Discovery, Authentication, Tools, Error Handling, Versioning, Security, and Reference Implementations

**JSON Schemas (draft 2020-12)**
- `schemas/manifest.schema.json` — Validates the discovery manifest document
- `schemas/tool.schema.json` — Validates a single tool definition
- `schemas/oauth-client.schema.json` — Validates an OAuth client registration record
- `schemas/error-response.schema.json` — Validates tool error response envelopes

**Examples**
- `examples/manifest-minimal.json` — Minimal valid manifest (1 tool, no optional fields)
- `examples/manifest-wordpress.json` — Real WordPress implementation manifest (sanitized)
- `examples/manifest-xenforo.json` — Real XenForo implementation manifest (sanitized, with deviation notes)
- `examples/tool-readPost.json` — Standalone tool definition example
- `examples/oauth-flow.md` — OAuth 2.0 + PKCE flow walkthrough with HTTP traces

**CI**
- `.github/workflows/validate-schemas.yml` — Validates all example JSON files against their schemas on every push and pull request

### Protocol v1.0 Feature Set

| Feature | Description |
|---|---|
| Discovery | `GET /{prefix}/manifest` public endpoint; JSON manifest document |
| Tool naming | `{platform}.{toolName}` camelCase convention |
| Tool invocation | `POST /{prefix}/v1/tools/{tool_name}` with Bearer auth |
| Authentication | OAuth 2.0 Authorization Code flow with mandatory PKCE S256 |
| Token lifetimes | Access: 1 h recommended; Refresh: 30 days recommended |
| Token revocation | `POST /{prefix}/oauth/revoke` mandatory endpoint |
| Scopes | `read`, `write`, `delete`, `admin` |
| Rate limiting | 50 req/min, 1000 req/hour per authenticated user |
| Pre-registered agents | Claude, ChatGPT, Gemini, Grok, Copilot, Perplexity, Meta AI, DeepSeek |
| Error envelope | `{success: false, code, message}` standard |
| Confirmation flow | `202 Accepted` with `confirmation_id` for destructive operations |

### Extracted From

This specification was extracted from production implementations of the Goldnat plugin family:
- Goldnat for WordPress (v2.x) — `chgold/goldt-wp-webmcp-bridge`
- Goldnat for XenForo (v1.2.26) — `chgold/xf-ai-connect`
- Goldnat for Drupal (v1.5.0) — `chgold/drupal-ai-connect`
- Goldnat for Shopify — `chgold/shopify-goldt-webmcp-bridge`

---

## [Unreleased]

Planned for v1.1:
- `/.well-known/servio.json` discovery redirect
- `output_schema` field in tool definitions
- Structured `errors[]` array form alongside flat error envelope
- Resource reading (`resources/list`, `resources/read`) capability
