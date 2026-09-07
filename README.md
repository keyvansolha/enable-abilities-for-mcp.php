# Enable Abilities for MCP — with ChatGPT OAuth connectors

A WordPress plugin that exposes ~100 WordPress "abilities" (content, SEO, WooCommerce, FSE,
LMS, menus, CPTs and more) to any [Model Context Protocol](https://modelcontextprotocol.io)
client, so an AI assistant can read and act on the site through a normal, authenticated
WordPress session.

This repository is the upstream plugin **plus a locally-built OAuth layer for ChatGPT and other
RFC 7591 connectors**. Upstream only supported Claude. See
[What was added in this fork](#what-was-added-in-this-fork).

- **Upstream:** [Enable Abilities for MCP](https://mcp.fabiomontenegro.com/) v2.8.0 by Fabio Montenegro
- **This fork:** keyvansolha
- **License:** GPL-2.0-or-later (see [LICENSE](LICENSE))

---

## Table of contents

- [Requirements](#requirements)
- [How the pieces fit together](#how-the-pieces-fit-together)
- [Authentication methods](#authentication-methods)
- [Quick start](#quick-start)
- [The ChatGPT OAuth connector](#the-chatgpt-oauth-connector)
  - [Why it had to be built](#why-it-had-to-be-built)
  - [Design: no vendor patching](#design-no-vendor-patching)
  - [Endpoints](#endpoints)
  - [The callback allowlist](#the-callback-allowlist)
  - [Full authorization flow](#full-authorization-flow)
  - [Registered connectors and manual clients](#registered-connectors-and-manual-clients)
  - [Security model](#security-model)
  - [Known limits](#known-limits)
- [Abilities](#abilities)
- [Admin screens](#admin-screens)
- [Options reference](#options-reference)
- [Repository layout](#repository-layout)
- [Troubleshooting](#troubleshooting)
- [Testing status](#testing-status)

---

## Requirements

| | |
|---|---|
| WordPress | 6.9+ |
| PHP | 8.0+ |
| Companion plugin | [MCP Adapter](https://github.com/WordPress/mcp-adapter) — **required**, it owns the REST transport |
| For OAuth connectors | A public site over **HTTPS** with pretty permalinks enabled |

The MCP Adapter plugin must be active. Without it the abilities register but nothing is
exposed over REST, and the admin screen shows a warning notice.

---

## How the pieces fit together

```
  Claude / ChatGPT / any MCP client
                │
                │  HTTPS + JSON-RPC over MCP
                ▼
  ┌─────────────────────────────────────────────┐
  │  WordPress REST API                         │
  │                                             │
  │  /wp-json/mcp/mcp-adapter-default-server ───┼──► Bearer token
  │                                             │    or Application Password
  │  /wp-json/mcp/mcp-oauth-server ─────────────┼──► OAuth 2.1 JWT
  └─────────────────────────────────────────────┘
                │
                ▼
  ┌─────────────────────────────────────────────┐
  │  MCP Adapter  (separate plugin)             │
  │  meta-tools: discover / get-info / execute  │
  └─────────────────────────────────────────────┘
                │
                ▼
  ┌─────────────────────────────────────────────┐
  │  WordPress Abilities API                    │
  │  98 ewpa/* abilities + 3 core/* + others    │
  └─────────────────────────────────────────────┘
```

Two MCP servers are exposed, and they differ **only** in how the caller proves who they are.
Both land on the same abilities, and both run as a real WordPress user, so an editor connecting
their own account gets editor permissions and nothing more.

---

## Authentication methods

All three can be on at once. Each is toggled independently in **Settings › WP Abilities › Connection**.

| Method | Endpoint | Who it authenticates as | Best for |
|---|---|---|---|
| **OAuth 2.1** | `/wp-json/mcp/mcp-oauth-server` | Each user, via their own login + consent screen | claude.ai and ChatGPT hosted connectors — no local setup |
| **Application Passwords** | `/wp-json/mcp/mcp-adapter-default-server` | The user the password belongs to | Teams; per-person credentials, revocable from the profile screen |
| **Bearer API key** | `/wp-json/mcp/mcp-adapter-default-server` | One fixed admin user | A single-operator setup, local CLI clients |

The Bearer key is stored as a SHA-256 hash and only ever intercepts requests under
`/wp-json/mcp/` ([`includes/auth.php`](includes/auth.php)), so it cannot be used to authenticate
against the rest of the REST API.

---

## Quick start

### 1. Turn on the OAuth server

**Settings › WP Abilities › Connection** → enable **claude.ai OAuth Custom Connector**.

That single switch boots the whole OAuth 2.1 layer: the `/oauth/*` endpoints, the
`.well-known` discovery documents, and the JWT-authenticated MCP server. With it off,
every OAuth surface returns 404.

### 2. Connect Claude

Claude needs no further configuration. In claude.ai → **Settings › Connectors › Add custom
connector**, paste:

```
https://your-site.com/wp-json/mcp/mcp-oauth-server
```

No Client ID, no secret. Claude identifies itself with a metadata URL that the bundled
library already trusts.

### 3. Connect ChatGPT

ChatGPT needs one extra step, because it registers itself rather than publishing a metadata URL.

1. Enable **ChatGPT & Other OAuth Connectors** in the same panel.
2. Check the **Allowed callback URLs** box. It is prefilled with the callbacks ChatGPT commonly
   uses — **confirm the exact one your connector screen shows, and delete the rest.**
3. Save, then add the same MCP server URL in ChatGPT → **Settings › Connectors**.

ChatGPT will read `/.well-known/oauth-authorization-server`, find `registration_endpoint`,
register itself at `/oauth/register`, and start the normal login-and-consent flow.

---

## The ChatGPT OAuth connector

### Why it had to be built

The bundled OAuth library ([`wp-media/mcp-oauth`](https://github.com/wp-media/mcp-oauth)) implements
**CIMD** — Client ID Metadata Documents, the client-registration model in the 2025-11-25 MCP
specification. A client presents an HTTPS URL as its `client_id`; the server fetches that URL and
validates the metadata document it finds there.

ChatGPT *also* uses CIMD — but the library's implementation rejects it on three separate counts.
Here is a real ChatGPT connector document:

```json
{
  "client_id": "https://chatgpt.com/oauth/j0wKlDBsmsUM/client.json",
  "redirect_uris": ["https://chatgpt.com/connector/oauth/j0wKlDBsmsUM"],
  "token_endpoint_auth_method": "private_key_jwt",
  "token_endpoint_auth_methods_supported": ["none", "private_key_jwt"],
  "client_name": "ChatGPT",
  "jwks_uri": "https://chatgpt.com/oauth/jwks.json"
}
```

1. **`token_endpoint_auth_method` is `private_key_jwt`.** `CimdResolver::validate_document()`
   refuses any document whose method is not `none`, so `resolve()` returns null and
   `AuthorizeEndpoint` dies with a bare `Unknown OAuth client.`
2. **`chatgpt.com` is not a trusted host.** The allowlist is hardcoded to `claude.ai`
   ([`ClaudeClientVerifier.php`](vendor/wp-media/mcp-oauth/inc/Auth/ClaudeClientVerifier.php)),
   and that gate also decides which URLs are fetched at all.
3. **The client ID is unique per connector.** `j0wKlDBsmsUM` is minted fresh for each connector
   you create, and `matches_publisher()` pins each publisher to an *exact* client_id URL — so no
   static list can ever match.

Point 3 is why the library's `wpmedia_mcp_oauth_trusted_publishers` filter cannot bridge the gap:
it can add a host, but the exact-URL pin behind it still fails.

Dynamic Client Registration is supported here too (`/oauth/register`), since other MCP clients use
it — but ChatGPT itself never calls it.

### Design: no vendor patching

Everything lives in one new file, [`includes/oauth-connectors.php`](includes/oauth-connectors.php).
**Nothing under `vendor/` is modified**, so the library can still be replaced wholesale.

The trick is ordering. The library registers its request handlers on `template_redirect` at the
default priority 10. This module registers at **priority 9** — one tick earlier — and each handler
begins by asking a single question: *is this request's client one of mine?*

- **No** → return immediately, output nothing. The library's CIMD path runs at priority 10 and
  behaves exactly as it did before. Claude is untouched.
- **Yes** → handle it, then `exit`.

Because a locally-registered client is admitted by writing the *same* `mcp_oauth_state_*` transient
the library writes, everything downstream is still the library's code: the consent form, single-use
auth-code issuance, PKCE verification, Application Password creation, JWT signing, refresh-token
rotation with reuse detection, and per-request transport authentication.

Only four handlers were needed:

| Handler | What it does |
|---|---|
| `/oauth/register` | New endpoint. RFC 7591 dynamic client registration. |
| `/oauth/authorize` | Admits locally-registered clients **and resolves CIMD documents for allowlisted publisher hosts**; writes the library's state transient. |
| `/oauth/authorize-callback` | Renders the consent screen with the opaque `client_id` suppressed. |
| `/oauth/token` | Verifies a client secret when the client has one, then hands off. |
| `.well-known/oauth-authorization-server` | Republished with `registration_endpoint` added. |

The consent-screen override exists for a small but real reason: the library renders `client_id` as
a clickable link, which is right for a CIMD URL. A dynamically-issued id is an opaque string, and
`esc_url()` turns it into `http://ewpa_a1b2c3…` — a broken link on a security screen, which reads
like a phishing artifact. The override reuses the library's own template with the ID line omitted,
so the client is identified by name and site instead.

### Endpoints

Served from `home_url()`, so they follow the Site Address on subdirectory installs.

| Path | Method | Source | Purpose |
|---|---|---|---|
| `/.well-known/oauth-protected-resource` | GET | library | RFC 9728 resource metadata |
| `/.well-known/oauth-authorization-server` | GET | **this fork** | RFC 8414 metadata + `registration_endpoint` |
| `/oauth/register` | POST | **this fork** | RFC 7591 dynamic client registration |
| `/oauth/authorize` | GET | library + fork | Authorization request |
| `/oauth/authorize-callback` | GET | library + fork | Post-login consent screen |
| `/oauth/consent` | POST | library | Allow / Deny, issues the auth code |
| `/oauth/token` | POST | library + fork | Code exchange and refresh |
| `/oauth/revoke` | POST | library | RFC 7009 token revocation |

`/oauth/register` also answers `OPTIONS` for CORS preflight and sends
`Access-Control-Allow-Origin: *`, since it is unauthenticated and carries no credentials — a
browser-based MCP client has to be able to reach it.

### The callback allowlist

This is the security boundary of the whole feature, and the reason the admin section exists.
An OAuth authorization server that lets a client nominate any `redirect_uri` is an open redirect
that hands out access tokens. So a locally-registered client may only ever send a user back to a
URL **you** have listed.

Enter it in **Settings › WP Abilities › Connection › ChatGPT & Other OAuth Connectors**, one URL
per line.

**Matching rules**

| Component | Rule |
|---|---|
| Scheme | Literal. `https` required, except plain `http` on loopback (`127.0.0.1`, `localhost`, `::1`) for native apps per RFC 8252 §8.3 |
| Host | **Literal — never wildcarded.** A pattern with `*` in the host is discarded on save |
| Port | Must match exactly, except on loopback, where the port is assigned per session and is ignored |
| Path / query | `*` matches any run of characters **except `?` and `#`**, so a wildcard can widen a path but can never swallow a query string |
| Fragment, userinfo | Any `redirect_uri` containing `#`, or `user:pass@`, is rejected outright |
| `#` at line start | Treated as a comment, so you can annotate the list |

**Examples**

```sh
# ── exact match ──
https://chatgpt.com/connector_platform_oauth_redirect

# ── one wildcard path segment (custom GPT ids) ──
https://chat.openai.com/aip/*/oauth/callback

# ── native app on loopback; any port matches ──
http://127.0.0.1/callback
```

What these reject, and why it matters:

| Rejected | Reason |
|---|---|
| `https://chatgpt.com.evil.io/connector_platform_oauth_redirect` | Host-suffix spoofing — host is compared literally, not by prefix |
| `https://user:pw@chatgpt.com/…` | Userinfo can mislead about the real destination |
| `https://chatgpt.com/…?next=https://evil.io` | Pattern has no query; wildcards stop at `?` |
| `https://chat.openai.com/aip/x/oauth/callback#frag` | Fragments are refused unconditionally |
| `http://chatgpt.com/…` | Plain HTTP on a non-loopback host |
| `https://chatgpt.com:8443/…` | Port mismatch on a non-loopback host |

**The list is enforced twice** — when a client registers, *and again on every single authorization
request*. Removing a URL immediately blocks clients that registered while it was still allowed;
you do not have to hunt down and revoke them.

It does double duty as the **CIMD publisher allowlist**. A metadata document is only ever fetched
from a host that already appears here, which is deliberate: a host you have trusted to receive
authorization codes is, by construction, safe to fetch a document from — strictly less dangerous
than redirecting a user there with a code in hand. One list, no drift between two settings. And a
publisher does not get to nominate its own redirect targets just because its host is trusted: every
`redirect_uri` in a fetched document is still checked against the list.

### Full authorization flow

```
ChatGPT                     WordPress                          User
   │                            │                                │
   │─ GET /.well-known/oauth-protected-resource ─────────────────►│
   │◄─ resource + authorization_servers ─────────────────────────│
   │                            │                                │
   │─ GET /.well-known/oauth-authorization-server ───────────────►│
   │◄─ metadata incl. registration_endpoint ─────────────────────│
   │                            │                                │
   │─ POST /oauth/register ─────►│                                │
   │      redirect_uris          │  ✓ every URI in the allowlist  │
   │◄─ 201 client_id ───────────│    (else 400 invalid_redirect_uri)
   │                            │                                │
   │─ GET /oauth/authorize ─────►│                                │
   │   client_id, redirect_uri,  │  ✓ client is known             │
   │   PKCE S256, state          │  ✓ redirect_uri registered     │
   │                            │  ✓ redirect_uri still allowed  │
   │                            │  ✓ response_type=code, S256    │
   │                            │  ✓ state present               │
   │                            │                                │
   │                            │─ redirect to wp-login ────────►│
   │                            │◄─ logs in as themselves ───────│
   │                            │─ consent screen ──────────────►│
   │                            │◄─ Allow ───────────────────────│
   │◄─ 302 ?code=… &state=… ────│                                │
   │                            │                                │
   │─ POST /oauth/token ────────►│  ✓ client secret, if any       │
   │   code, code_verifier       │  ✓ client_id binds to the code │
   │                            │  ✓ PKCE verifier matches       │
   │                            │  → creates an Application Password
   │◄─ access + refresh JWT ────│                                │
   │                            │                                │
   │─ MCP calls, Bearer JWT ────►│  → runs as that WordPress user │
```

Each MCP session is anchored to a real WordPress **Application Password**, so it appears under
**Users › Profile › Application Passwords** and can be revoked there. Deleting it kills the
session immediately. Sessions are capped at 5 per user per client; the oldest is evicted.

### Registered connectors and manual clients

The admin panel lists every client that has registered itself, with its callback URLs and a
**Remove** button.

For connectors whose UI insists on a **Client ID** (and optionally a **Secret**) rather than
registering themselves, the panel has a *Create a Client ID and Secret by hand* section. The
callback URLs you type there must already be in the allowlist. The secret is shown **once** —
only a hash is stored, via `wp_hash_password()`.

Client authentication at the token endpoint supports `client_secret_post` and
`client_secret_basic`. A client registered without a secret stays a public PKCE client, which is
what MCP connectors normally want. Credentials are read verbatim rather than passed through
`sanitize_text_field()`, because a secret is compared byte for byte.

### Security model

Everything the library already enforced still applies. On top of that:

- **PKCE S256 is mandatory.** A request without `code_challenge`, or with any other method, is
  refused.
- **`state` is mandatory.** The server never silently generates one — a server-generated `state`
  cannot reach the client in time to be validated, so it would provide no CSRF protection at all.
- **`redirect_uri` is validated before it is ever used as a redirect target**, per OAuth 2.1 §7.5.2.
  Errors that occur before validation are rendered as pages, not redirects.
- **Client-to-grant binding.** If a client names itself at the token endpoint, that name must match
  the client the auth code was issued to. One client cannot redeem another's code.
- **Registration is rate-limited** to 20 attempts per IP per hour, and the client store is capped at
  50 records with oldest-first eviction of self-registered entries. Manually created clients are
  never auto-evicted.
- **Client IDs are opaque and format-checked** (`ewpa_` + 32 hex). A value that does not match the
  shape is never looked up.
- **Hostile input is handled.** A JSON body can nest an array anywhere a string is expected; all
  such values pass through `ewpa_oauth_str()` / `ewpa_oauth_str_list()`, which drop non-scalars
  instead of raising a conversion notice and yielding `"Array"`.
- **Turning the feature off is a hard stop.** With connectors disabled, `/oauth/register` returns
  403, authorize refuses registered clients, and the token endpoint rejects their grants.

### Known limits

- **Custom GPT Actions are not supported.** They use the older `client_id` + `client_secret` flow
  *without* PKCE. The library's token endpoint requires PKCE unconditionally, and supporting
  Actions would mean replacing `TokenEndpoint` outright. ChatGPT's MCP **connectors** — the thing
  most people mean — do use PKCE and work.
- **The prefilled callback URLs are a starting point, not a guarantee.** They are the callbacks
  ChatGPT has been observed to use. Confirm the exact URL on your connector screen.
- **The authorization-server metadata document is republished, not filtered.** The library provides
  no hook for its body, so the shared fields are mirrored. If a future library version adds a field
  there, mirror it in `ewpa_oauth_serve_metadata()` too.
- **Requires pretty permalinks.** Every endpoint is a rewrite rule.

---

## Abilities

98 `ewpa/*` abilities plus 3 core ones, reachable through the adapter's three meta-tools
(`discover-abilities`, `get-ability-info`, `execute-ability`).

| Area | Count |
|---|---|
| Content management | 32 |
| Custom post types | 11 |
| Menus | 8 |
| Tutor LMS | 8 |
| WooCommerce | 7 |
| LearnDash | 6 |
| Site information | 5 |
| The Events Calendar | 4 |
| Multilanguage, JetEngine (query builder / options pages), FSE templates, Elementor | 3 each |
| User management, accessibility | 1 each |

Every ability is individually toggleable in the **Abilities** tab. Abilities registered by *other*
plugins (Fluent Forms and the like) are also listed and can be switched off — a disabled
third-party ability is unregistered late in `wp_abilities_api_init`, so it is never exposed
anywhere ([`includes/thirdparty.php`](includes/thirdparty.php)).

---

## Admin screens

**Settings › WP Abilities**, three tabs:

- **Connection** — the three auth methods, the ChatGPT connector panel, and copy-paste config
  snippets for Claude Desktop / Claude Code and `curl`.
- **Activity Log** — every MCP call, per user, in a custom table.
- **Abilities** — per-ability on/off, grouped by area, including third-party abilities.

---

## Options reference

| Option | Added by | Purpose |
|---|---|---|
| `ewpa_enabled_abilities` | upstream | Which abilities are on |
| `ewpa_api_key` | upstream | Bearer key hash + owning user |
| `ewpa_bearer_enabled` | upstream | Bearer auth on/off |
| `ewpa_thirdparty_disabled` / `_seen` | upstream | Third-party ability denylist and snapshot |
| `ewpa_oauth_enabled` | upstream | Master switch for the OAuth 2.1 layer |
| `mcp_jwt_secret` | library | HMAC secret signing every JWT |
| `ewpa_oauth_connectors_enabled` | **fork** | ChatGPT / DCR connectors on/off |
| `ewpa_oauth_callback_allowlist` | **fork** | Newline-separated allowed callback URLs |
| `ewpa_oauth_clients` | **fork** | Registered client records |
| `ewpa_oauth_connectors_rewrite_version` | **fork** | Rewrite-flush bookkeeping for `/oauth/register` |

Uninstall ([`uninstall.php`](uninstall.php)) removes every option marked **fork** above, plus the
upstream ones, across every site on multisite.

Three are deliberately *not* removed, and it is worth knowing which:

- `mcp_jwt_secret` belongs to the vendored library, which owns its own lifecycle.
- `ewpa_thirdparty_disabled` and `ewpa_thirdparty_seen` are an upstream gap — they survive an
  uninstall. Harmless, but they are orphaned rows if you never reinstall.

---

## Repository layout

```
enable-abilities-for-mcp.php     Plugin bootstrap, OAuth boot, discovery compat shims,
                                 multisite discovery bridge, ability registry (~1,940 lines)
includes/
  abilities.php                  The 98 ability implementations (~10,500 lines)
  admin.php                      Settings screens + AJAX handlers
  auth.php                       Bearer token authentication
  activity-log.php               Per-call logging to a custom table
  thirdparty.php                 Toggling abilities registered by other plugins
  oauth-connectors.php           ★ ChatGPT / RFC 7591 connector layer (this fork)
assets/                          Admin CSS and JS
languages/                       en_US, es_ES translations + .pot
vendor/                          Shipped Composer dependencies:
  wp-media/mcp-oauth               OAuth 2.1 server (CIMD, Claude)
  wordpress/mcp-adapter            MCP protocol adapter
  wordpress/php-mcp-schema         MCP schema DTOs
```

`vendor/` is committed on purpose: there is no `composer.json` at the plugin root, so the
dependencies ship with the plugin and are required for it to run at all.

---

## What was added in this fork

Upstream v2.8.0 supported Claude only. Added here:

- `includes/oauth-connectors.php` — the entire ChatGPT / RFC 7591 connector layer (~1,260 lines),
  written to sit alongside the vendored library without patching it.
- `includes/admin.php` — the **ChatGPT & Other OAuth Connectors** panel: enable toggle, callback
  allowlist editor, registered-connector table with revoke, manual client creation. Four new AJAX
  handlers, all nonce-checked and `manage_options`-gated.
- `enable-abilities-for-mcp.php` — boots the new module from `ewpa_maybe_boot_oauth()`.
- `uninstall.php` — cleans up the new options (and `ewpa_oauth_enabled`, which upstream left behind).

---

## Troubleshooting

**Enable debug logging.** The library logs the whole OAuth flow, and this fork logs into the same
channel. Both require **`WP_DEBUG` and `WP_DEBUG_LOG` to be true** — `WP_DEBUG_LOG` alone is not
enough, because WordPress only redirects `error_log()` to `debug.log` when `WP_DEBUG` is on.

```php
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
```

Then watch `wp-content/debug.log` for `[MCP][AUTHORIZE]`, `[MCP][TOKEN]`, `[MCP][REGISTER]`,
`[MCP][CIMD]` and `[MCP][DISCOVERY]` lines.

| Symptom | Likely cause |
|---|---|
| Every `/oauth/*` URL 404s | OAuth master switch off, or rewrite rules not flushed — re-save **Settings › Permalinks** |
| `400 invalid_redirect_uri` at registration | The callback URL is not in the allowlist. The log line names the rejected URL |
| "redirect_uri does not match an allowed callback URL" | Registered earlier, then the URL was removed from the list. Re-register the connector |
| `403 access_denied` from `/oauth/register` | The ChatGPT connectors toggle is off |
| ChatGPT never calls `/oauth/register` | It did not see `registration_endpoint` — check the connectors toggle, then fetch `/.well-known/oauth-authorization-server` yourself |
| Metadata fetch fails on a subdirectory install | Endpoints follow the **Site Address** (`home_url()`), not the WordPress Address |
| `401 invalid_client` at the token endpoint | Client secret missing or wrong, or a `client_id` that does not match the code's client |

---

## Testing status

Honest accounting of what has and has not been verified.

**Verified**

- All modified files pass `php -l`.
- The callback URL matcher was exercised against 15 cases including host-suffix spoofing,
  userinfo smuggling, port mismatch, fragment injection, query-string smuggling, wildcard-in-host,
  bad schemes and loopback port-agnostic matching — all pass.
- Client creation was tested against hostile metadata (nested arrays where strings are expected,
  non-array `redirect_uris`, disallowed callbacks) with no PHP notices raised.
- No duplicate function names or constant definitions across the plugin.

**Not verified**

- The end-to-end OAuth round trip against a live ChatGPT connector. There is no database in this
  working copy, so the flow could not be exercised against a running WordPress.
- The prefilled callback URLs against ChatGPT's current behaviour.

---

## License

GPL-2.0-or-later, inherited from the upstream plugin. See [LICENSE](LICENSE).
