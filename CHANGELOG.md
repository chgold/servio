# Changelog

All notable changes to the Servio Protocol specification are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Version numbers follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.2.0] — 2026-11-20

### Added
- §5.7 Redirect URI validation (normative, RFC 9700 / OAuth 2.1 / RFC 8252 §7.3): exact string match against the client's registered URIs at the authorization request, at the approve/deny step and at the token endpoint; port-only loopback exception for registered native-client loopback URIs; never redirect to an unvalidated URI (error page instead, also for deny); an authorization request never registers or adds a URI; auto-registered client aliases get OOB only; PKCE `S256` only; 7-step conformance test.
- §9.11 Agent access tokens are accepted only on the plugin's own agent routes.
- §9.12 Consent and confirmation forms: `POST` + session + CSRF token, Bearer refused.
- Compliance rules RULE-013 … RULE-017.

### Changed
- §5.2 Step 3: token exchange MUST verify `client_id` and an identical `redirect_uri` (OOB codes may omit it); errors never `200`.
- §7.3: OAuth error codes now carry normative HTTP status (`400`, `401` for `invalid_client`, `500` for `server_error`) plus `unauthorized_client`, `unsupported_grant_type`, `invalid_scope`.
- §9.8 points to §5.7 (the old "loopback MAY be accepted" wording is removed); §9.10 `state` returned unchanged.
- §6.5.1 user confirmation is MUST for destructive/financial tools; `confirm_url`/`status_url` on the manifest's host; confirm form needs session + CSRF + `POST`; the confirming user must own the pending action; autonomy grants are created by the user only.

### Added (user confirmation)
- §6.5.1 User confirmation (`user-confirm-v1`) for destructive and financial tools: manifest `confirmation` (`required` / `conditional`) + `risk` per tool, top-level `confirmation` descriptor, 202 `confirmation_required` response with `preview`, `confirm_url`, `status_url`, user-only confirm step, single use, replay-safe re-submission, optional autonomy grant, and the rules agents must follow.
- Error codes `confirmation_required` (202), `confirmation_mismatch` (409), `confirmation_rejected` / `confirmation_expired` (410).

## [1.1.0] — 2026-11-08

### Added
- `auth.revocation_url` REQUIRED; §5.2 Step 5 rewritten as an RFC 7009 token revocation contract with a conformance test. `auth.revoke_url` deprecated as an alias.

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
