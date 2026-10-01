# MCP server for AIDoc Studio: protocol, auth, security, SDK and ops research

- **Date:** 2026-10-01
- **Scope:** Adding an MCP server to AIDoc Studio (FastAPI backend `src/aidoc/server/`, React admin `web/`). Phase 1: local/LAN clients with Bearer PATs. Phase 2: remote clients (claude.ai connectors, ChatGPT) via OAuth 2.1, without a rewrite. Plus an admin dashboard.
- **Method:** Primary sources only: the spec markdown in `github.com/modelcontextprotocol/modelcontextprotocol` (main branch), the Python SDK docs and the `mcp==2.2.0` wheel source, and Anthropic, OpenAI, VS Code and Cursor docs. Every claim has a URL. Where I could not verify something, I say so.

> **Headline:** The protocol changed a lot this summer. The **current revision is `2026-07-28`**, released 2026-07-28. It makes MCP **stateless**: there is no `initialize` handshake, no `Mcp-Session-Id` and no GET stream. Many clients, and Anthropic's hosted connectors for auth, still speak `2025-11-25`. So the server has to serve **both eras**. The official Python SDK v2 (`mcp` 2.2.0) does this automatically on one endpoint.

---

## 1. Spec revisions and key protocol features

### 1.1 Which revision is current

| Revision | Status (as of 2026-10-01) | Headline |
|---|---|---|
| **2026-07-28** | **Current** | Stateless core, no sessions, `server/discover`, MRTR, `subscriptions/listen`, tasks moved to an extension, cacheable lists, DCR deprecated in favour of CIMD |
| 2025-11-25 | Final | CIMD, icons, experimental tasks, URL elicitation, tool-name guidance, incremental scope consent |
| 2025-06-18 | Final | Structured tool output, resource links, elicitation, OAuth resource-server role (PRM), RFC 8707, `MCP-Protocol-Version` header, JSON-RPC batching removed |
| 2025-03-26 | Final | Streamable HTTP replaced HTTP+SSE; tool annotations |

- "The **current** protocol version is **2026-07-28**." https://modelcontextprotocol.io/specification/versioning
- Release post, dated July 28, 2026: "the largest revision of the protocol since launch". All Tier 1 SDKs (TypeScript, Python, Go, C#) support it. Rust support is beta. https://blog.modelcontextprotocol.io/posts/2026-07-28/
- Full changelog: https://modelcontextprotocol.io/specification/2026-07-28/changelog
- 2025-11-25 changelog: https://modelcontextprotocol.io/specification/2025-11-25/changelog
- 2025-06-18 changelog: https://modelcontextprotocol.io/specification/2025-06-18/changelog
- New lifecycle policy: deprecated features stay at least 12 months (90 days under the expedited exception) before removal. https://modelcontextprotocol.io/specification/versioning

### 1.2 Transports: Streamable HTTP (2026-07-28 shape) and legacy SSE

Source: https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http

- **One MCP endpoint that accepts POST.** Every client message is its own POST. The server answers each request with either `application/json` or a `text/event-stream` scoped to that request. That stream carries request-scoped notifications such as progress, then the final response.
- **No GET stream, no `DELETE`, no `Last-Event-ID` resumability** in 2026-07-28. A server that speaks only this revision SHOULD answer GET or DELETE with `405` and ignore `Mcp-Session-Id` and `Last-Event-ID`. A broken stream loses the in-flight request, and the client re-issues it under a new id. (Changelog items 1, 4, 9.)
- **Long-lived change notifications** now go through a `subscriptions/listen` POST whose SSE response stays open. Servers are encouraged to send `:` comment keep-alives and SHOULD set `X-Accel-Buffering: no` on SSE.
- **Cancellation on HTTP = closing the response stream.** "Closing the SSE response stream MUST be treated by the server as cancellation of that request." `notifications/cancelled` is used only on stdio. https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/cancellation
- **Required headers on every POST:** `MCP-Protocol-Version`, `Mcp-Method`, and, for `tools/call`, `resources/read` and `prompts/get`, `Mcp-Name`. The server MUST reject header/body mismatches with `400` and `-32020 HeaderMismatch`. Non-ASCII values use `=?base64?...?=`. Optional `x-mcp-header` schema annotations mirror chosen tool params into `Mcp-Param-*` headers. Do not use it for secrets.
- **Security MUSTs on the transport:** validate `Origin` (an invalid, present Origin returns `403`). Bind to 127.0.0.1 when local. Authenticate all connections.
- **HTTP+SSE (2024-11-05)** is formally "Deprecated" under the lifecycle policy (SEP-2596). New implementations SHOULD NOT adopt it. https://modelcontextprotocol.io/specification/2026-07-28/deprecated
- **Backward compatibility.** A dual-era client tries a modern request first. If it gets `400` without a recognized modern JSON-RPC error, it falls back to `initialize` (2025-era). The earlier Streamable HTTP shape (2025-03-26 to 2025-11-25) had `Mcp-Session-Id`, GET SSE streams, server-to-client requests on SSE, and resumability.

### 1.3 Sessions, handshake and protocol version

- **Sessions removed.** "Remove protocol-level sessions and the `Mcp-Session-Id` header… Servers that need cross-call state use explicit, server-minted handles passed as ordinary tool arguments" (SEP-2567). https://modelcontextprotocol.io/specification/2026-07-28/changelog
- **Handshake removed.** Every request carries `_meta["io.modelcontextprotocol/protocolVersion"]`, `.../clientCapabilities`, and SHOULD carry `.../clientInfo`. Servers SHOULD put `.../serverInfo` in each result's `_meta`. A version mismatch returns `UnsupportedProtocolVersionError` (`-32022`, HTTP 400). https://modelcontextprotocol.io/specification/2026-07-28/basic/index
- **`server/discover`:** the changelog and the Discovery page say servers **MUST** implement it. The release blog calls it optional, but that refers to the *client* calling it. https://modelcontextprotocol.io/specification/2026-07-28/server/discover
- **`MCP-Protocol-Version` header:** MUST be on every POST and MUST equal the `_meta` value. A server MAY treat a missing header as `2025-03-26` if it supports pre-2025-06-18 clients. (Streamable HTTP page, as above.)
- **The 2025-era session rules still matter for legacy clients:** "MCP Servers MUST NOT use sessions for authentication"; session IDs must be secure, random and bound to the user. https://modelcontextprotocol.io/docs/2025-11-25/tutorials/security/security_best_practices (Session Hijacking)

### 1.4 Tools: structured output, annotations, names, errors

Source: https://modelcontextprotocol.io/specification/2026-07-28/server/tools

- **`tools/list`** MUST NOT vary per connection. It MAY vary by the authorization on the request, for example returning only the tools the caller's scopes permit. It SHOULD use a deterministic order (helps prompt caching). It is paginated and cacheable.
- **Structured output:** `outputSchema` (JSON Schema 2020-12 by default; any 2020-12 keywords allowed since SEP-2106) plus `structuredContent`, which can be any JSON value. If an `outputSchema` is given, the server MUST conform to it. "A tool that returns structured content SHOULD also return the serialized JSON in a TextContent block." Introduced in 2025-06-18.
- **Annotations** (`ToolAnnotations` in https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/schema/2026-07-28/schema.ts). All are *hints*, untrusted unless the server is trusted:
  - `title`
  - `readOnlyHint`: default false
  - `destructiveHint`: default true, meaningful only when not read-only
  - `idempotentHint`: default false
  - `openWorldHint`: default true
  - `Tool.title` is also a top-level field. `icons` arrived in 2025-11-25.
- **Tool names:** SHOULD be 1–128 characters from `[A-Za-z0-9_.-]`, case-sensitive and unique per server. Aggregating clients may prefix them with the server name.
- **Errors:** two mechanisms.
  - **Protocol errors** (JSON-RPC `error`): unknown tool, malformed request, server error.
  - **Tool execution errors** (`isError: true` in the result): API failures, **input validation errors** (moved here in 2025-11-25, SEP-1303) and business-logic errors. Clients SHOULD feed these to the model so it can self-correct.
- **Stateful tools (non-normative):** return an opaque, high-entropy handle from a creation tool and accept it as an argument later. Check authorization against the handle on every call, state the handle's lifetime in the description, and return an `isError` result on an expired handle.
- **Server MUSTs:** validate inputs, implement access control, **rate-limit tool invocations**, and sanitize outputs.

### 1.5 Resources, templates, resource links

Source: https://modelcontextprotocol.io/specification/2026-07-28/server/resources

- **Resource templates** use RFC 6570 URI templates (`resources/templates/list`). Custom URI schemes are allowed if they follow RFC 3986.
- **Resource-not-found** changed from `-32002` to **`-32602`**. Use `-32603` for internal errors.
- **Annotations** on resources and content blocks: `audience` (`user`/`assistant`), `priority` (0–1) and `lastModified`.
- **Tool results can carry:**
  - `resource_link` (URI, name, description, mimeType). It "is not guaranteed to appear in `resources/list`".
  - Embedded `resource` content (servers using it SHOULD implement the `resources` capability).

  Both arrived in 2025-06-18. (Tools page.)
- **Subscriptions:** `resources/subscribe` was replaced by `subscriptions/listen` in 2026-07-28. Anthropic's hosted clients don't support resource subscriptions anyway: "Claude doesn't yet support these MCP features… Resource subscriptions, Sampling…" https://claude.com/docs/connectors/building/index

### 1.6 Pagination, caching, progress, cancellation

- **Pagination:** opaque cursor and `nextCursor`. The page size is server-chosen. An empty string is a valid cursor. An invalid cursor returns `-32602`. It applies only to `tools/list`, `resources/list`, `resources/templates/list` and `prompts/list`, so it does not apply to `tools/call` results. https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/pagination
- **Caching (new):** results of `server/discover`, `tools/list`, `prompts/list`, `resources/list`, `resources/templates/list` and `resources/read` MUST carry `ttlMs` and `cacheScope` (`public`/`private`). https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/caching
- **Progress:** the client sends `_meta.progressToken`, and the server MAY send `notifications/progress` with `progress`, optional `total` and `message`. Progress must increase monotonically. On HTTP it flows on the request's own SSE stream. https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/progress
- **Client timeouts matter for long tool calls:**
  - claude.ai and Desktop: **240 s per tool call** (https://claude.com/docs/connectors/building/index).
  - Claude Code: the default idle timeout is 5 min for HTTP servers without a response or progress. Calls longer than 2 min are auto-backgrounded. Per-server `timeout` in `.mcp.json`. https://code.claude.com/docs/en/mcp

### 1.7 Long-running work: tasks

- Tasks were "experimental" in core in 2025-11-25. In 2026-07-28 they **moved out of core into the official extension `io.modelcontextprotocol/tasks`**:
  - Poll with `tasks/get`.
  - Send client input with `tasks/update`.
  - `tasks/list` was removed.
  - Servers may return task handles unsolicited (`resultType: "task"`).

  https://modelcontextprotocol.io/specification/2026-07-28/changelog (item 6), https://modelcontextprotocol.io/extensions/tasks/overview, repo https://github.com/modelcontextprotocol/ext-tasks
- The client opts in through the extension capability. I could not confirm that Claude Code, Claude Desktop, Cursor or VS Code implement the tasks extension today. **Treat it as optional**, and expose AIDoc's own job queue as explicit `job_id` handles (the "Stateful Tools" pattern), which works with every client.

### 1.8 Elicitation, sampling, roots, logging, completions

- **MRTR replaces server-to-client requests.**
  1. Server-initiated `elicitation/create`, `sampling/createMessage` and `roots/list` now ride inside an `InputRequiredResult` (`resultType: "input_required"`, `inputRequests`, opaque `requestState`).
  2. The client retries the original request with `inputResponses`.

  All results now carry `resultType`. https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr
- **Deprecated in 2026-07-28 (SEP-2577):** Roots, Sampling and **Logging**. `ping` and `logging/setLevel` were removed. The log level is now per request via `_meta["io.modelcontextprotocol/logLevel"]`. The suggested migration is to "log to `stderr` (stdio) or use OpenTelemetry". https://modelcontextprotocol.io/specification/2026-07-28/deprecated
- **Elicitation** stays; URL mode came in 2025-11-25, and `notifications/elicitation/complete` and `elicitationId` were removed in 2026-07-28.
- **Completions** (`completion/complete`) are not deprecated. https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/completion

### 1.9 `_meta` conventions, icons, batching, tracing

- **`_meta` key format:** an optional reverse-DNS prefix plus a name. Any prefix whose second label is `modelcontextprotocol` or `mcp` is reserved. `traceparent`, `tracestate` and `baggage` are reserved for OpenTelemetry trace propagation (SEP-414). Vendors use their own prefix, for example `anthropic/maxResultSizeChars`. https://modelcontextprotocol.io/specification/2026-07-28/basic/index
- **`clientInfo` and `serverInfo` are self-reported and unverified:** "intended for display, logging, and debugging… SHOULD NOT rely on them for security decisions." (same page)
- **Icons** on tools, resources, templates and prompts arrived in 2025-11-25 (SEP-973).
- **JSON-RPC batching was removed in 2025-06-18** (PR #416). The 2026-07-28 HTTP body "MUST be a single JSON-RPC request or notification."
- **Error-code partition:** `-32000..-32019` are implementation-defined (legacy); `-32020..-32099` are reserved for the MCP spec. https://modelcontextprotocol.io/specification/2026-07-28/basic/index

---

## 2. Authorization (OAuth 2.1 resource server)

Primary: https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization, plus the sub-pages on [AS discovery](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/authorization-server-discovery), [client registration](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration) and [security considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations).

### 2.1 Roles and what the spec requires of the server

- Authorization is OPTIONAL. When it is used over HTTP it SHOULD follow the spec. stdio SHOULD NOT; it uses the environment instead.
- **The MCP server is an OAuth 2.1 resource server (RS).** The authorization server (AS) "may be hosted with the resource server or a separate entity."
- **The server MUST implement Protected Resource Metadata (RFC 9728)**, and the PRM document MUST list at least one `authorization_servers` entry. The server can advertise the PRM URL in one of two ways:
  - a `WWW-Authenticate: Bearer resource_metadata="…"` header on `401`, or
  - a well-known URI, either path-inserted (`/.well-known/oauth-protected-resource/<mcp-path>`) or at the root.

  Clients try the header first, then path-inserted, then root. https://datatracker.ietf.org/doc/html/rfc9728
- **The AS MUST offer RFC 8414 metadata or OIDC Discovery.** Clients MUST try both, in a defined priority order for issuer URLs that contain a path. https://datatracker.ietf.org/doc/html/rfc8414
- **Scopes:** the server SHOULD put `scope="…"` in the `WWW-Authenticate` challenge, and clients treat it as authoritative. `scopes_supported` in the PRM should be the **minimal** set for basic functionality. Do NOT list `offline_access`.
- **Runtime insufficient scope:** `403` with `WWW-Authenticate: Bearer error="insufficient_scope", scope="…", resource_metadata="…"`, listing all scopes the operation needs in one challenge (step-up authorization). The server must honour scope hierarchies.
- **Status codes:** 401 for a missing, invalid or expired token; 403 for insufficient scope; 400 for a malformed request.

### 2.2 Audience binding and token passthrough

- **Clients MUST send the RFC 8707 `resource` parameter** (the canonical server URI, e.g. `https://mcp.example.com/mcp`) in both the authorization and token requests. https://www.rfc-editor.org/rfc/rfc8707.html
- **"MCP servers MUST validate that access tokens were issued specifically for them as the intended audience."** Invalid tokens get 401. "MCP servers MUST only accept tokens that are valid for use with their own resources… MUST NOT accept or transit any other tokens."
- **Tokens MUST be sent in the `Authorization: Bearer` header on every request, and MUST NOT be in the query string.** AIDoc's current `auth._presented_token` accepts `?token=`, which must not apply to `/mcp`.
- **Token passthrough is "explicitly forbidden"** because it breaks audit trails, bypasses controls and creates a confused deputy. https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices#token-passthrough

### 2.3 Client registration: CIMD, pre-registration, DCR

- **Priority order:** pre-registered client, then **Client ID Metadata Documents** (if the AS advertises `client_id_metadata_document_supported`), then DCR, then manual entry.
- **CIMD:** `client_id` is an HTTPS URL to a JSON document with `client_id`, `client_name` and `redirect_uris`. The AS fetches it and validates the redirect URIs. https://datatracker.ietf.org/doc/html/draft-ietf-oauth-client-id-metadata-document-00
- **DCR (RFC 7591) is Deprecated as of 2026-07-28** (PR #2858). It is kept for backward compatibility.
- **Other client-side hardening in 2026-07-28** (relevant if AIDoc builds its own AS):
  - `iss` in the authorization response (RFC 9207), which the AS SHOULD send and clients MUST validate.
  - `application_type` in DCR.
  - Credentials keyed by issuer.
- **PKCE:** OAuth 2.1 makes it mandatory. ChatGPT requires `code_challenge_methods_supported` to include `S256`. https://developers.openai.com/apps-sdk/build/auth

### 2.4 What remote clients actually need (phase 2 targets)

- **claude.ai / Desktop / Cowork custom connectors:**
  - "Claude connects to your remote MCP server from Anthropic's cloud infrastructure… must be reachable over the public internet from Anthropic's IP ranges." https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp
  - Auth choices: OAuth ("Use Claude's published identity" = **CIMD hosted by Anthropic**, "Register automatically" = DCR, or your own client ID), **No sign-in**, or **Request headers (beta, limited orgs)** for a static `Authorization: Bearer …`. https://claude.com/docs/connectors/custom/remote-mcp
  - Claude follows the **2025-03-26, 2025-06-18 and 2025-11-25** authorization specs.
  - Redirect URI is `https://claude.ai/api/mcp/auth_callback`; Claude Code uses a loopback redirect.
  - Limits: about 150,000 characters per tool result and 240 s per tool call. https://claude.com/docs/connectors/building/index
  - Private networks: **MCP tunnels** (Enterprise, research preview, Cloudflare-based, outbound-only). They still require OAuth on the server. https://claude.com/docs/connectors/mcp-tunnels/overview
- **ChatGPT (Apps SDK / developer mode):**
  - OAuth 2.1 authorization code with PKCE S256, PRM at `/.well-known/oauth-protected-resource`, and the `resource` parameter echoed into the token's `aud`. CIMD is preferred and DCR is the alternative.
  - Static API keys are not supported: "nor can it present custom API keys".
  - Per-tool `securitySchemes` (`noauth` / `oauth2` + scopes), with runtime errors carrying `_meta["mcp/www_authenticate"]` to trigger sign-in.

  https://developers.openai.com/apps-sdk/build/auth
- **Claude Code v2 runtime** sends OAuth credentials **only to HTTPS token endpoints or localhost**: "Sign-in fails for a server whose token endpoint is plain `http://` anywhere else, such as a device on your local network." https://code.claude.com/docs/en/mcp. So **OAuth on a LAN requires HTTPS**.

### 2.5 Static bearer tokens today, OAuth later without a rewrite

The Python SDK models the server strictly as an RS. You implement a `TokenVerifier`, and the SDK serves PRM and the 401 challenge for you (details in §5.4). So the migration path is:

1. **Put the PAT check behind the SDK's `TokenVerifier` interface now.** `verify_token(raw) -> AccessToken(client_id, scopes, expires_at, resource, subject, claims)`. Phase 2 adds a second branch for JWTs (verify via JWKS: `iss`, `aud`, `exp`) or introspection (RFC 7662). Tools and the dashboard keep using `get_access_token()`. https://py.sdk.modelcontextprotocol.io/run/authorization/
2. **Use OAuth-shaped scopes from day one** (e.g. `aidoc:read`, `aidoc:convert`, `aidoc:manage`), store them on the PAT, and enforce them per tool. Use the same strings later in `scopes_supported` and the AS consent screen.
3. **Treat a token as bound to this resource.** Record a canonical `resource_server_url`. For PATs the verifier fills `AccessToken.resource` itself. For JWTs it checks `aud`. Never forward client tokens to anything downstream.
4. **Return proper 401/403 with `WWW-Authenticate`** (the SDK does 401; the server adds 403 `insufficient_scope` for per-tool checks).
5. **Model identity as (issuer, subject, client_id).** The SDK's `principal_components()` uses exactly this triple. A PAT maps to `issuer=aidoc-local`, `subject=<admin user>`, `client_id=<token id>`.
6. **Phase 2 AS choice:**
   - Use an external IdP (Keycloak, Auth0, Entra, etc.) and set `issuer_url` to it, or
   - embed a minimal AS with RFC 8414 metadata, PKCE S256, CIMD support (needed for "Claude's published identity" and ChatGPT), optional DCR, `iss` in responses, and refresh tokens.

   The SDK's embedded `auth_server_provider=` is discouraged: "New servers should not reach for it." https://py.sdk.modelcontextprotocol.io/run/authorization/

---

## 3. Security best practices

Primary: https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices (2026-07-28 version) and the transport page.

- **Confused deputy:** applies to MCP *proxy* servers that use a static client ID with a third-party AS. They MUST get per-client consent, validate `state`, and so on. It is not relevant to AIDoc unless it later brokers third-party APIs.
- **Token passthrough:** forbidden. "MCP servers MUST NOT accept any tokens that were not explicitly issued for the MCP server."
- **State-handle hijacking** (the 2026 replacement for "session hijacking"):
  - "MUST NOT treat possession of a state handle as authentication."
  - Use secure random handles.
  - Bind them to the authenticated user, e.g. `<user_id>:<handle>`.

  This applies to AIDoc `job_id`, `document_id` and `upload_id` values returned through MCP: check ownership and scope on every call. Legacy 2025-era rule: "MCP Servers MUST NOT use sessions for authentication."
- **DNS rebinding / Origin:** servers MUST validate `Origin` (a present, invalid Origin gets 403), SHOULD bind to 127.0.0.1 locally, and SHOULD authenticate. https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http#security--endpoint
  - The Python SDK enforces this by default: with no `transport_security=`, it accepts only localhost Host values and returns **`421 Misdirected Request`** for others.
  - **For LAN mode you must allowlist the LAN hostnames and IPs** (`allowed_hosts=["192.168.1.10", "192.168.1.10:*", "aidoc.lan:*"]`).
  - Passing a non-localhost `host=` "does not allowlist that hostname"; it disables the protection.

  https://py.sdk.modelcontextprotocol.io/run/deploy/
- **Local MCP servers:** "Restrict access if using an HTTP transport, such as: require an authorization token; use unix domain sockets or other IPC." Users must explicitly consent to one-click local server configs. (Local MCP Server Compromise section.)
- **SSRF:** the spec's SSRF section targets *clients* fetching OAuth metadata URLs. It recommends HTTPS, blocking private/reserved IP ranges, and validating redirects, citing OWASP. https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html. **For AIDoc it applies to any tool that fetches a URL** (e.g. "convert this URL"), and to any phase-2 AS that fetches CIMD documents. Block RFC 1918, loopback and link-local addresses, re-check after DNS resolution and redirects, and cap size and time.
- **Scope minimization:** a minimal initial scope set with step-up via `WWW-Authenticate scope=`. "Common mistakes" include publishing every scope in `scopes_supported` and using wildcard or omnibus scopes. (Scope Minimization section.)
- **Tool-level obligations:** validate inputs, access control, rate limiting, output sanitization. https://modelcontextprotocol.io/specification/2026-07-28/server/tools#security-considerations
- **Prompt injection via tool descriptions and outputs:** Anthropic rejects tool descriptions that "tell Claude how to behave", call other tools, pull instructions from external sources, and similar. Converted document content is **untrusted text** that reaches the model. Label it as document content and never mix it into tool descriptions. https://claude.com/docs/connectors/building/review-criteria
- **Header-mirroring hazard:** don't put secrets in `x-mcp-header` parameters, because intermediaries see headers. (Tools page.)

---

## 4. Tool design best practices

Sources:
- Anthropic engineering, "Writing effective tools for agents" (Ken Aizawa, 2025-09-11): https://www.anthropic.com/engineering/writing-tools-for-agents
- Anthropic connector review checklist: https://claude.com/docs/connectors/building/review-criteria
- MCP tools spec: https://modelcontextprotocol.io/specification/2026-07-28/server/tools

- **Granularity: consolidate around tasks, not endpoints.** Prefer a workflow tool (Anthropic's example is `schedule_event`) over thin `list_*` wrappers. But **separate read and write tools**: a catch-all with a `method` parameter "is rejected", and write tools are ideally split by create, update and delete. (Review criteria.)
- **Naming:**
  - Spec charset is `[A-Za-z0-9_.-]`, up to 128 characters. Anthropic's directory requires **≤ 64 characters**.
  - Use namespacing prefixes by service or resource (e.g. `asana_projects_search`).
  - Use unambiguous parameter names (`user_id`, not `user`).
- **Descriptions:** state precisely what the tool does and when to use it. Make implicit context explicit (query syntax, terminology, relationships between resources). Small description changes produced large eval gains. (Engineering post.)
- **Annotations:** Anthropic's directory requires a **`title` on every tool, plus `readOnlyHint: true` for read-only tools and `destructiveHint: true` for tools that modify or delete data**. These drive auto-permissions in Claude: "Read-only tools can run without per-call confirmation, and destructive tools always prompt." (Review criteria.)
- **Response size:**
  - Claude Code warns above **10,000 tokens** and truncates at **25,000** by default (`MAX_MCP_OUTPUT_TOKENS`). A tool can raise its own text limit with `_meta["anthropic/maxResultSizeChars"]` on the tool definition, up to 500,000 characters. https://code.claude.com/docs/en/mcp
  - claude.ai and Desktop: about **150,000 characters**. https://claude.com/docs/connectors/building/index
  - The engineering post: "implement some combination of pagination, range selection, filtering, and/or truncation with sensible default parameter values". When truncating, tell the agent how to narrow its request.
- **High-signal output:**
  - Return human-meaningful fields (names, titles, page numbers) rather than bare UUIDs, but include the IDs needed for follow-up calls.
  - Consider a `response_format: "concise" | "detailed"` enum.
  - Structured output: provide `outputSchema` + `structuredContent` **and** a text rendering, per the spec's SHOULD.
- **Resource links vs embedded content:**
  - Return `resource_link`s (`aidoc://documents/{id}`) when the client or model may fetch more later.
  - Embed content only for the slice that was requested.
  - Links don't need to appear in `resources/list`.
  - Claude's hosted surfaces support text and binary resources but **not subscriptions**. Hosts vary in whether the model can follow a `resource_link` on its own, so always pair a link with a tool-callable ID (e.g. `read_document(doc_id, …)`).

  https://claude.com/docs/connectors/building/index
- **Errors:**
  - Validation and business errors become `isError: true` with actionable text, e.g. "page_end 900 exceeds page_count 412; use get_document_info to see page_count".
  - Protocol errors are only for unknown tools and malformed requests.
  - Generic messages such as "Internal Server Error" fail Anthropic's review. In the Python SDK, raising `ToolError` produces `isError`.
- **Idempotency:** mark `idempotentHint` truthfully. Make `convert_document` idempotent by content hash (AIDoc already hashes uploads), returning the existing doc or job instead of duplicating it.
- **Exposing large documents (AIDoc's core use case):**
  - Use **metadata-first discovery**: a `get_document_info` that returns page count, an outline or TOC with page anchors, a chunk count and size estimate, and resource links.
  - **Range reads:** `read_document(doc_id, page_start, page_end)`, capped by tokens or characters. If the cap is hit, return `truncated: true` and a `next_page` or `next_offset` so the agent can continue.
  - **Search-first retrieval:** `search_library(query, filters, limit)` returns ranked hits with doc, page and chunk IDs, short snippets and resource links. `get_chunks(chunk_ids[])` returns full chunk text.
  - Default page and limit values should keep a typical response well under 10k tokens.
- **Long-running conversion:** return a `job_id` immediately and expose `get_job(job_id)`. Optionally send progress notifications if the client sent a `progressToken`. This avoids the 240 s claude.ai tool timeout and the 5-minute Claude Code idle timeout (§1.6).
- **Determinism and caching:** fixed tool order, stable descriptions (prompt cache), and `ttlMs`/`cacheScope` on list results (the SDK handles this with `cache_hints`).

---

## 5. Python SDK (`mcp` on PyPI)

### 5.1 Version and requirements

- **Latest release: `mcp` 2.2.0 (2026-09-07)**. The 2.x line started with 2.0.0 on 2026-07-28; the 1.x line ends at 1.30.0. **Requires-Python ≥ 3.10**, which matches AIDoc's `requires-python = ">=3.10"`. https://pypi.org/project/mcp/
- **Dependencies of 2.2.0** (wheel METADATA):
  - `starlette>=0.27`, `sse-starlette>=3`, `uvicorn>=0.31.1`, `pydantic>=2.12`
  - `pyjwt[crypto]`, `httpx2`, `jsonschema`, `opentelemetry-api`
  - `mcp-types==2.2.0`, and `pywin32` on Windows
- **v2 breaking changes:**
  - **`FastMCP` was renamed `MCPServer`** (`from mcp.server import MCPServer`; v1 used `from mcp.server.fastmcp import FastMCP`). `ctx.fastmcp` became `ctx.mcp_server`.
  - Python attributes are now snake_case.
  - Protocol types moved to the `mcp-types` distribution.

  https://py.sdk.modelcontextprotocol.io/whats-new/ and the migration guide at https://py.sdk.modelcontextprotocol.io/migration/
- **The same `streamable_http_app()` serves 2025-era clients (initialize and sessions) and 2026-era clients (stateless), routed by the `MCP-Protocol-Version` header**, with nothing to configure. https://py.sdk.modelcontextprotocol.io/run/legacy-clients/

### 5.2 Mounting inside FastAPI / Starlette

The relevant docs are https://py.sdk.modelcontextprotocol.io/run/asgi/ and https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/run/asgi.md. A sketch adapted to FastAPI:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from mcp.server import MCPServer
from mcp.server.transport_security import TransportSecuritySettings

mcp = MCPServer("AIDoc Studio", token_verifier=PatVerifier(...), auth=AuthSettings(...))
mcp_app = mcp.streamable_http_app(               # build at module/factory time, BEFORE lifespan runs
    streamable_http_path="/mcp",
    stateless_http=True,                         # legacy leg only; see 5.3
    transport_security=TransportSecuritySettings(allowed_hosts=[...], allowed_origins=[...]),
)

@asynccontextmanager
async def lifespan(app: FastAPI):
    async with mcp.session_manager.run():        # REQUIRED: mounted sub-app lifespans never run
        yield

app = FastAPI(lifespan=lifespan)
app.mount("/", mcp_app)   # or a Mount placed BEFORE the SPA catch-all; see pitfalls
```

- "**a mounted sub-application's lifespan never runs**… Whichever app sits at the top of your ASGI stack must enter `mcp.session_manager.run()` in its own lifespan." `mcp.session_manager` exists only after `streamable_http_app()` is called. (ASGI page.)
- `streamable_http_app()` keyword arguments (from the 2.2.0 source):
  - `streamable_http_path="/mcp"`, `json_response=False`, `stateless_http=False`
  - `event_store`, `retry_interval`, `max_request_body_size`
  - `session_idle_timeout` (default 1800 s), `max_sessions` (default 10,000)
  - `transport_security`, `host="127.0.0.1"`
- **CORS** is only needed for browser clients. If used, allow the `Mcp-*`, `Authorization` and `Mcp-Protocol-Version` headers and expose `Mcp-Session-Id`. (ASGI page.)
- `@mcp.custom_route(...)` routes are "**never authenticated**". (ASGI page.)

### 5.3 Stateless vs stateful

https://py.sdk.modelcontextprotocol.io/run/deploy/, https://py.sdk.modelcontextprotocol.io/run/legacy-clients/

- **2026-07-28 requests are always stateless.** "nothing ties a modern request to a worker."
- **Legacy (≤2025-11-25) clients** get an in-memory `Mcp-Session-Id` by default. That requires sticky routing with more than one worker, and there is no distributed store.
- **`stateless_http=True` affects only the legacy leg.** It makes per-request throwaway sessions. The cost is no server-to-client channel for legacy clients: `ctx.elicit()` raises `NoBackChannelError` and notifications are dropped.
- `json_response=True` removes the per-request SSE stream, so progress notifications are dropped.
- **For AIDoc (single process, tools that don't elicit or sample):** stateless is fine. Keep `json_response=False` so `notifications/progress` can flow for long calls.
- MRTR `requestState` across workers needs shared `RequestStateSecurity(keys=[...])`. This is irrelevant for one process.

### 5.4 Auth hooks

https://py.sdk.modelcontextprotocol.io/run/authorization/ and the source in `mcp/server/auth/settings.py` and `provider.py`.

- `TokenVerifier.verify_token(token) -> AccessToken | None`.
  - `AccessToken` fields: `token`, `client_id`, `scopes`, `expires_at`, `resource`, `subject`, `claims`.
- `AuthSettings` fields:
  - `issuer_url` (**required**) and `resource_server_url`
  - `required_scopes`, `validate_token_resource`
  - `service_documentation_url`, plus AS-only options
- `token_verifier=` and `auth=` must be passed together, or construction raises `ValueError`.
- **`validate_token_resource` defaults to `None`.** With `resource_server_url` set, that warns and behaves as `False`; v3 will default it to `True`. If `True`, the server rejects tokens whose `AccessToken.resource` ≠ `resource_server_url`. That is awkward when the same server is reached as `localhost`, a LAN IP, and a hostname. **For PAT mode set it to `False` and check the audience in the verifier** (or set `resource` yourself).
- The SDK auto-serves **PRM at `/.well-known/oauth-protected-resource/<path>`** and answers 401 with `WWW-Authenticate: Bearer error="invalid_token", …, resource_metadata="…"`.
- In handlers, `get_access_token()` (from `mcp.server.auth.middleware.auth_context`) returns the verifier's `AccessToken`.
- Authorization is HTTP-only; stdio and the in-memory test client bypass it.

### 5.5 Request context for logging

- **Tool-level `Context`** (inject with a `ctx: Context` parameter):
  - `ctx.request_id`
  - `ctx.headers`: the HTTP headers. "client-supplied input… never an identity."
  - `ctx.report_progress(...)`
  - `ctx.request_context.lifespan_context`

  https://py.sdk.modelcontextprotocol.io/handlers/context/
- **Middleware** (provisional API): `MCPServer(middleware=[...])` with `async def mw(ctx, call_next)`. It wraps **every** inbound message (`server/discover`, `initialize`, `tools/call`, …).
  - `ctx` is a `ServerRequestContext` with `method`, `params` (raw, before validation), `request_id`, `meta`, `protocol_version`, `session` and `request` (the Starlette request on HTTP).
  - Raising `MCPError` refuses a single message.
  - On the 2026-07-28 HTTP path, client notification POSTs are 202'd at the transport and never reach middleware.

  https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/advanced/middleware.md
- **Client identity:** `ctx.session.client_params.client_info` (`name`, `version`), for both eras. For modern requests the SDK synthesizes `client_params` from the `_meta` envelope when `clientInfo` is present; `clientInfo` is optional for clients, so it may be `None`. `ctx.connection.session_id` is set only for legacy stateful sessions. (Source: `mcp/server/connection.py` `Connection.from_envelope`, `mcp/server/context.py`, in the 2.2.0 wheel.)
- **OpenTelemetry:** a built-in middleware emits a SERVER span per message with `mcp.method.name` and `mcp.protocol.version`, plus `gen_ai.operation.name="execute_tool"` and `gen_ai.tool.name` on `tools/call`. It is a no-op until an exporter is installed. https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/run/opentelemetry.md

### 5.6 Pitfalls (some specific to AIDoc's current `app.py`)

1. **Lifespan.** AIDoc's `create_app()` currently has no lifespan. Add one that enters `mcp.session_manager.run()`, or the first MCP request fails.
2. **Route order and the SPA catch-all.** `_mount_web()` installs an SPA fallback. The MCP mount (and `/.well-known/oauth-protected-resource…`) must be registered **before** it.
3. **`EnvelopeMiddleware` rewrites every `application/json` body** to add `workspace`. It would mutate JSON-RPC responses and PRM JSON. **Exempt `/mcp` and `/.well-known/`.** SSE bodies already pass through.
4. **Query-string tokens.** `auth._presented_token` accepts `?token=`. The MCP spec forbids tokens in the URI, so the MCP path must read only the header.
5. **421 on LAN.** Without `transport_security` allowlisting the LAN hosts, every non-localhost request gets `421`. The client side shows only a generic failure.
6. **PRM location when mounted under a prefix.** If the MCP app is mounted at `/x`, its PRM route ends up at `/x/.well-known/...`, but RFC 9728 clients look at `/.well-known/oauth-protected-resource/x/mcp` (path inserted at the **origin root**). Mount at `/` with `streamable_http_path="/mcp"`, or serve PRM from the root FastAPI app. (This is my inference from RFC 9728 §3.1 and the SDK route layout; verify with a test.)
7. **Two auth systems.** Keep the existing single admin token (`ctx.token`) for the web UI and API, and use **per-client PATs for MCP**, so usage can be attributed.
8. **Windows.** The SDK pulls in `pywin32` on win32. That is fine, but keep an eye on it for the packaged app.
9. **Middleware is provisional** ("may change in a 2.x minor"). Pin `mcp>=2.2,<2.3`, or isolate the logging middleware behind an adapter.

---

## 6. Client configuration snippets (Bearer, HTTP)

Assume a token `aidoc_pat_…` and the server at `http://127.0.0.1:8765/mcp`, or on the LAN `http://192.168.1.10:8765/mcp`.

**Claude Code** (https://code.claude.com/docs/en/mcp):
```bash
claude mcp add --transport http aidoc http://127.0.0.1:8765/mcp \
  --header "Authorization: Bearer aidoc_pat_XXXX"
# scopes: --scope local (default, ~/.claude.json) | project (.mcp.json, shared) | user
```
`.mcp.json` (project scope, with environment-variable expansion so the secret isn't committed):
```json
{ "mcpServers": { "aidoc": {
    "type": "http",
    "url": "${AIDOC_URL:-http://127.0.0.1:8765}/mcp",
    "headers": { "Authorization": "Bearer ${AIDOC_TOKEN}" },
    "timeout": 600000
} } }
```
- `type` is required for URL entries; `streamable-http` is accepted as an alias.
- `-H` is short for `--header`.
- `headersHelper` can emit headers from a script.
- Output limits: warning at 10k tokens, maximum 25k (`MAX_MCP_OUTPUT_TOKENS`).

**Claude Desktop:**
- **Custom connectors are brokered from Anthropic's cloud**, so they cannot reach `127.0.0.1` or a LAN IP. Static headers are a beta, limited-org feature. https://claude.com/docs/connectors/custom/remote-mcp
- For a local or LAN server, use a **stdio bridge in `claude_desktop_config.json`** with `mcp-remote`. On Windows, avoid spaces inside args, because Desktop on Windows doesn't escape them. https://github.com/geelen/mcp-remote

```json
{ "mcpServers": { "aidoc": {
    "command": "npx",
    "args": ["mcp-remote", "http://192.168.1.10:8765/mcp", "--allow-http",
             "--header", "Authorization:${AUTH_HEADER}"],
    "env": { "AUTH_HEADER": "Bearer aidoc_pat_XXXX" }
} } }
```
`--allow-http` is needed for non-HTTPS URLs and is meant for trusted private networks. The longer-term options are an **MCPB desktop extension** (https://claude.com/docs/connectors/custom/remote-mcp#install-a-local-connector-in-the-desktop-app) or a public HTTPS endpoint with OAuth.

**Cursor** (`~/.cursor/mcp.json` or `.cursor/mcp.json`) (https://cursor.com/docs/context/mcp):
```json
{ "mcpServers": { "aidoc": {
    "url": "http://127.0.0.1:8765/mcp",
    "headers": { "Authorization": "Bearer ${env:AIDOC_TOKEN}" }
} } }
```

**VS Code** (`.vscode/mcp.json`, or "MCP: Open User Configuration") (https://github.com/microsoft/vscode-docs/blob/main/docs/agents/reference/mcp-configuration.md):
```json
{
  "inputs": [ { "type": "promptString", "id": "aidoc-token",
                "description": "AIDoc Studio MCP token", "password": true } ],
  "servers": { "aidoc": {
      "type": "http",
      "url": "http://127.0.0.1:8765/mcp",
      "headers": { "Authorization": "Bearer ${input:aidoc-token}" }
  } }
}
```
VS Code tries Streamable HTTP first and falls back to SSE. The prompted value is stored securely.

**Recommendation:** the admin UI's "Create token" dialog should render all four snippets with the real URL and token filled in. This is the only time the token is shown.

---

## 7. Personal access tokens

- **Format.** Follow GitHub's design: an identifiable prefix plus `_` (not a Base64 character, so it isn't confused with random strings, and a double-click selects the whole token), random body, CRC32 checksum encoded as 6 base62 characters. This allows offline validation and secret-scanning with near-zero false positives. https://github.blog/engineering/platform-security/behind-githubs-new-authentication-token-formats/
  - Proposal: `aidoc_pat_` + 32 random bytes in base62 (about 43 characters, 256 bits) + 6-character CRC32 in base62.
  - Reject tokens with a bad checksum before any DB lookup. That cheaply filters garbage and brute-force noise.
- **Hashing.** NIST SP 800-63B (rev 4, §3.1.2.2, look-up secrets): secrets with ≥112 bits of entropy "SHALL be stored in a hashed form using an approved hashing function". Only lower-entropy secrets need a salted password-hashing KDF. https://pages.nist.gov/800-63-4/sp800-63b.html
  - Store **SHA-256 (or HMAC-SHA-256 with a server-side pepper kept outside the DB)** of the full token, with a unique index, and look up by hash.
  - bcrypt or Argon2 add per-request latency with no security gain for 256-bit random tokens.
  - Also store a short **display prefix** (e.g. the first 4 characters of the body) and the last 4 characters for UI identification.
- **Show once.** Display the plaintext only in the creation response, with copy buttons and the §6 snippets. It can never be retrieved again.
- **Scopes.** Use the same strings as the future OAuth scopes:
  - `aidoc:read`: search, list and read documents and chunks
  - `aidoc:convert`: submit and check conversion jobs
  - optional `aidoc:manage`: delete documents, re-index
  - Default new tokens to `aidoc:read`. Enforce per tool. Filter `tools/list` by scope, which the spec allows, so the model never sees tools it can't call.
- **Expiry.** Require an expiry, defaulting to 90 days (30, 90, 365 or custom). Allow "no expiry" only behind an explicit warning. Have the dashboard warn about tokens that expire soon.
- **Last-used tracking.** Record `last_used_at`, `last_used_ip`, `last_client_name/version` and `last_protocol_version`. Throttle writes, e.g. at most once per minute per token.
- **Revocation and rotation.** Revoke immediately. Since validation is a DB lookup there is no cache, or the cache TTL must be ≤ a few seconds. "Regenerate" creates a new secret on the same token record (keeping name, scopes and history) and invalidates the old one. An optional overlap window is useful for scripted clients.
- **Rate limiting.** The spec says servers MUST rate-limit tool invocations. https://modelcontextprotocol.io/specification/2026-07-28/server/tools#security-considerations
  - Use a per-token token bucket, e.g. 60 tool calls per minute with a burst of 20, and a separate cap on concurrent `convert` jobs.
  - Return HTTP `429` with `Retry-After` on the transport. Return an `isError` result for per-tool quota errors so the model can adapt.
  - Also rate-limit failed authentications per IP.
- **Hygiene.** Never log raw tokens (redact `Authorization`). Compare in constant time where comparing secrets. Bearer only, never the query string. On LAN, plain HTTP exposes tokens to the local network: document this and offer TLS (or a reverse proxy).

---

## 8. Observability and the admin dashboard

### 8.1 What to log per request / tool call

| Field | Source |
|---|---|
| timestamp, duration_ms | middleware `try/finally` (SDK middleware docs) |
| token_id, scopes, subject | `get_access_token()` / verifier |
| client name/version | `ctx.session.client_params.client_info` (both eras; may be absent) |
| User-Agent, remote IP | `ctx.request` (Starlette) or `ctx.headers` |
| protocol_version | `ctx.protocol_version` / `MCP-Protocol-Version` header |
| method, tool/resource name | `ctx.method`, `ctx.params["name"/"uri"]` (also `Mcp-Method`/`Mcp-Name` headers on modern clients) |
| args (redacted, truncated, e.g. 1 KB) | `ctx.params["arguments"]` |
| outcome: ok / isError / JSON-RPC error code / HTTP 401/403/429 | middleware result/exception + ASGI layer |
| result size (bytes, estimated tokens), truncated? | serialized result |
| legacy session id | `ctx.connection.session_id` (legacy stateful only) |
| request id, trace id | `ctx.request_id`, `_meta.traceparent` |

- `clientInfo` and `serverInfo` are for "display, logging, and debugging" only, not for security decisions. https://modelcontextprotocol.io/specification/2026-07-28/basic/index
- User-Agent is equally untrusted. I could not find documented User-Agent strings for Claude Code, Cursor or VS Code. Record whatever arrives and show it as-is.
- Also log auth failures at the ASGI layer (401 or 403 before the MCP layer). The SDK's MCP middleware never sees requests rejected by its auth middleware, so wrap `mcp_app` with a small ASGI logger.
- Keep **OpenTelemetry** as an optional exporter. The SDK already creates spans, and the spec now standardizes `traceparent` in `_meta`.

### 8.2 "Active connections" in a stateless world

- **2026-07-28 clients have no connection or session to count.** Each POST stands alone. The only long-lived thing is an optional `subscriptions/listen` stream. https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http
- **Legacy stateful clients** have an `Mcp-Session-Id` with an idle timeout (30 min default) and usually an open GET stream ("a connected client's session never expires"). With `stateless_http=True` the legacy leg has no sessions either. https://py.sdk.modelcontextprotocol.io/run/legacy-clients/
- **So define dashboard status in terms of activity, not sockets:**
  - **Active client** = a distinct `(token_id, client_name)` pair with a request in the last 5 minutes. **Idle** = last seen within 24 hours. **Inactive** otherwise.
  - Show "currently running tool calls", meaning in-flight requests counted by middleware enter and exit.
  - Optionally show open `subscriptions/listen` streams and legacy sessions as secondary counts.
  - A `server/discover` or `tools/list` call is a good "client connected" heartbeat. Claude Code, for example, re-lists after reconnecting. https://code.claude.com/docs/en/mcp

### 8.3 Dashboard views

- **Server card:** URL(s), protocol versions served, bind address, TLS on/off, host allowlist, and the copy-paste client snippets.
- **Tokens table:** name, scopes, created, expires, last used (time, client, IP), 24h/7d call counts, and revoke/regenerate actions.
- **Tool-call log:** filter by token, client, tool and outcome. Show a detail drawer with redacted args, result size and error text.
- **Per-tool stats:** calls, error rate, p50/p95 latency, average result tokens. The last helps tune default page sizes against the 10k/25k Claude Code thresholds.
- **Retention:** ring-buffer the request log in SQLite, e.g. 30 days or N rows, and roll up daily counts.

---

## 9. Recommendations for AIDoc Studio

**Protocol, SDK and transport**
1. Use **`mcp>=2.2,<2.3`** (`MCPServer`, the official SDK). Serve **Streamable HTTP only** at **`/mcp`** on the existing FastAPI app and port. Don't implement the deprecated HTTP+SSE transport; Claude Code and VS Code fall back to SSE automatically only for servers that need it.
2. Serve both eras (the default). Set `stateless_http=True` and keep `json_response=False` so progress notifications work. No tools use elicitation or sampling: both are deprecated or unsupported in Claude's hosted surfaces, and stateless legacy has no back-channel.
3. In `create_app()`:
   - Add a lifespan that enters `mcp.session_manager.run()`.
   - Mount MCP at the root with `streamable_http_path="/mcp"`, **before** `_mount_web`.
   - Exempt `/mcp` and `/.well-known/` from `EnvelopeMiddleware`.
   - Never accept `?token=` on `/mcp`.
4. Bind to 127.0.0.1 by default. LAN mode is an explicit opt-in that:
   - fills `TransportSecuritySettings.allowed_hosts` with the bound IP and hostnames (with `:*` port variants),
   - sets `allowed_origins` to the web UI origin only (non-browser MCP clients send no Origin),
   - requires a token for every request (no loopback trust on `/mcp`, even locally, because the spec says "require an authorization token" for local HTTP servers),
   - shows a "traffic is unencrypted on your LAN" notice, with optional TLS.

**Auth (phase 1, ready for OAuth)**

5. Implement `PatVerifier(TokenVerifier)` and pass `token_verifier=` + `auth=AuthSettings(issuer_url=<AIDoc base URL>, resource_server_url=<canonical /mcp URL>, required_scopes=["aidoc:read"], validate_token_resource=False)`. The verifier:
   - checks prefix and CRC,
   - looks up by SHA-256 hash,
   - checks expiry and revocation,
   - returns `AccessToken(client_id=token_id, subject=owner, scopes=…, resource=resource_server_url, expires_at=…)`.

   *Uncertain:* the PRM then advertises an `authorization_servers` entry with no AS metadata behind it in phase 1. Header-configured clients only see it after a 401. Before release, test how Claude Code, VS Code and Cursor behave on a 401 for a revoked token: do they try OAuth discovery and show a confusing error? If that's a problem, put a tiny ASGI auth layer in front instead of `AuthSettings` in phase 1, keeping the same verifier interface.
6. **Scopes:** `aidoc:read`, `aidoc:convert`, `aidoc:manage`. Enforce them per tool with a decorator that returns 403 `insufficient_scope` (or an `isError` with clear text), and filter `tools/list` by scope.
7. **Phase 2 (remote):**
   - Expose a **public HTTPS** origin (reverse proxy or tunnel).
   - Pick an AS:
     - **external IdP preferred**: Keycloak or Authentik for self-hosters, or Auth0/Entra; or
     - a small embedded AS supporting RFC 8414, PKCE S256, **CIMD** (Claude's published identity and ChatGPT's preferred path), DCR (still needed for Claude's "Register automatically", the 2025-era clients and the ChatGPT fallback), `iss` per RFC 9207, and refresh tokens.
   - Add a JWT/JWKS branch to the same verifier with an `aud` check (RFC 8707). Then switch `validate_token_resource=True` once the resource URL is canonical.
   - Allow the redirect `https://claude.ai/api/mcp/auth_callback`, plus the ChatGPT redirect URIs.

**Tools (v1 set, all with `title` and annotations)**

| Tool | Hints | Notes |
|---|---|---|
| `search_library(query, filters?, limit=10, cursor?)` | readOnly, idempotent, closed-world | ranked hits: doc title, doc_id, page, chunk_id, ~300-char snippet, `resource_link` |
| `list_documents(cursor?, limit=25, sort?)` | readOnly | titles, ids, page counts, status; paginated |
| `get_document_info(doc_id)` | readOnly | metadata, page_count, outline/TOC with pages, chunk count, est. tokens, resource links |
| `read_document(doc_id, page_start=1, page_end?, max_chars=40000)` | readOnly | Markdown slice + `structuredContent {pages, truncated, next_page}`; actionable `isError` on bad ranges |
| `get_chunks(chunk_ids[])` | readOnly | full chunk text with doc/page provenance |
| `convert_document(source, options?)` | not readOnly, not destructive, idempotent (by content hash), openWorld only if URL sources | returns `job_id` immediately; URL sources need SSRF guards; local-path sources only when the client is on the same host |
| `get_job(job_id)` | readOnly | status/progress/result doc_id; tasks extension optional later |
| `delete_document(doc_id)` *(scope `aidoc:manage`, phase 1.5)* | destructive | separate tool; never combined with reads |

- Resources and templates: `aidoc://documents/{doc_id}`, `aidoc://documents/{doc_id}/pages/{page}` and `aidoc://chunks/{chunk_id}`, with `ttlMs` hints.
- Keep default responses under about 8–10k tokens. Optionally set `_meta["anthropic/maxResultSizeChars"]` on `read_document`.
- Send server `instructions` (in `server/discover`) that briefly explain the search, then info, then read workflow.

**Observability**

8. Add a `mcp_request_log` table (fields from §8.1) and a `mcp_tokens` table. Write the log from an SDK middleware (MCP-level) plus a thin ASGI wrapper (auth failures and 429s). Raw tokens and full arguments are never logged.
9. Dashboard: server card with snippets, tokens table, live activity ("active in the last 5 min" by token and client), tool-call log, and per-tool stats. Define "active" by recent activity, not sockets.

**Open items / uncertainties**
- Which of the target clients currently negotiate 2026-07-28 and which stay on 2025-11-25:
  - Claude Code's v2 runtime negotiates it with HTTP servers (https://code.claude.com/docs/en/mcp).
  - I did not verify this for Cursor, VS Code or Claude Desktop.

  The SDK makes this transparent, but the dashboard should display the protocol version per client.
- Client support for the tasks extension, and for following `resource_link`s, is unverified, so don't depend on either.
- Claude custom-connector static "Request headers" is beta and limited to some orgs. Don't rely on it for claude.ai; plan on OAuth.
- The RFC 9728 path placement of PRM when mounting under a prefix (§5.6 #6) is an inference to verify with a test.
