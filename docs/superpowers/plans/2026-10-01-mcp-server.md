# Doc4AI Studio MCP Server — Implementation Plan (index)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement the phase plans task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Read THIS file before any phase file: it holds the binding decisions, the contract amendments and the coverage table. Then read the spec and the research document named below.

**Goal:** Expose Doc4AI Studio to other AIs over MCP (Streamable HTTP at `/mcp`, Bearer personal access tokens, Phase 1 tool set) and give the admin web UI an "MCP" page to issue/revoke tokens and watch connections and calls.

**Architecture:** The official Python SDK `mcp` 2.2 (`MCPServer`) is mounted inside the existing FastAPI app behind a small ASGI gate of our own (`McpGate`) that does Bearer-PAT authentication, rate limiting and failure logging; an SDK middleware records every MCP message, filters `tools/list` by scope and publishes the per-request `Principal` to tools through a `ContextVar`. Tools read documents through the existing `Store`/output layout; a new FTS5 `page_index` powers `search_library`. The admin API lives under `/api/mcp/*` (existing server-token/loopback auth only) and streams `mcp.*` events over the existing SSE bus to a new React page.

**Tech Stack:** Python 3.10+ (the main env currently runs 3.12.13 under `uv`; code must stay 3.10-compatible), FastAPI/Starlette, `mcp>=2.2,<2.3` (+ `mcp-types`, `httpx2`, `pywin32` on win32), SQLite FTS5 (trigram), tiktoken (`cl100k_base`, cached under `data/tiktoken`), React 19 + TanStack Query + shadcn/ui (Vite 8, Vitest 5, Playwright).

**Spec:** `docs/superpowers/specs/2026-10-01-mcp-server-design.md` (APPROVED; Traditional Chinese). Research: `docs/superpowers/research/2026-10-01-mcp-best-practices.md`. Existing binding contracts: `docs/superpowers/plans/2026-09-24-index.md` (index §0, §3, §4, §7, §8) — amended, additively only, in §4 of this file.

**Phase files (execute in this order):**

| Part | File | Tasks | Delivers |
|---|---|---|---|
| F — Foundation | `docs/superpowers/plans/2026-10-01-mcp-p1-foundation.md` | 1–10 | SDK spike + decisions, `mcp` dependency, `[mcp]` config, token format/secret, schema v4 (`api_tokens`, `mcp_clients`, `mcp_calls`, `page_index`), page indexing + backfill, `Principal`/`PatVerifier`, rate limiter, budget/cursor helpers, call-log retention |
| S — Server | `docs/superpowers/plans/2026-10-01-mcp-p2-server.md` | 11–24 | schemas, call recorder, ASGI gate, `MCPServer` + middleware + mount, all Phase 1 tools, resources, protocol tests for both eras, security suite, SDK acceptance script |
| A — Admin | `docs/superpowers/plans/2026-10-01-mcp-p3-admin.md` | 25–35 | `/api/mcp/*`, SSE `mcp.*`, config snippets (+README), web "MCP" page (總覽 / Tokens / 連線 / 呼叫紀錄), Settings `[mcp]` section, Vitest, Playwright e2e, real-client acceptance run, deviations section |

Each phase file ends with a **Phase acceptance** checklist; the project is done when §9 of this file passes.

---

## 0. Global constraints (copied from the spec; every task's requirements include these)

- SDK: `mcp>=2.2,<2.3` (research §5.6 #9: the middleware API is provisional; pin the minor). `MCPServer(name="Doc4AI Studio", instructions=…)`. **Never** the deprecated HTTP+SSE transport (`sse_app`), sampling, roots, logging capability, tasks extension.
- Transport: Streamable HTTP at **`/mcp`**, same FastAPI app and port, registered **before** the SPA catch-all; `stateless_http=True`, `json_response=False`; the FastAPI lifespan enters `mcp.session_manager.run()`; `EnvelopeMiddleware` must pass `/mcp` and `/.well-known/` through untouched; the (concurrently added) compression middleware keeps `/mcp` excluded and never buffers `text/event-stream`.
- Auth: `/mcp` **always** needs a token, even from loopback. `Authorization: Bearer <pat>` only; `?token=` on `/mcp` is rejected (400 `token_in_query`, logged as `auth_error`). 401 carries `WWW-Authenticate: Bearer realm="doc4ai", error="invalid_token"` **without** `resource_metadata` (Phase 1 uses our own ASGI gate, not SDK `AuthSettings`). PATs are **never** valid on `/api/*`.
- Token format: `doc4ai_pat_` + 43 base62 chars (256 random bits) + `_` + 6 base62 chars (CRC32 of the 43-char body) — e.g. `doc4ai_pat_3kX…9Qa_4fZ1bC` (61 chars). DB stores `HMAC-SHA256(server_secret, token)` hex; `server_secret` = `data/secret.key` (32 random bytes, created on first use, mode 0600 best effort). Display prefix = `doc4ai_pat_` + first 4 body chars. Expiry mandatory: default 90 days, max 365 (config), "never" only with `allow_no_expiry = true`. Every token has `doc4ai:read`.
- Scopes: `doc4ai:read`, `doc4ai:convert`, `doc4ai:convert:local`, `doc4ai:manage`. Enforced per tool; `tools/list` filtered by scope; a hard call without scope → `isError: true`, `code = "forbidden_scope"`, logged as `forbidden_scope`.
- Rate limit: per-token token bucket, default 60 `tools/call` per minute (`api_tokens.rate_limit_per_min` overrides; `tools/list`, `server/discover`, `initialize`, `resources/*` not counted); over → HTTP 429 + `Retry-After`, logged `rate_limited`.
- Logging: one `mcp_calls` row per MCP HTTP request (auth failures and 429 included); never the raw token, never full arguments (`args_summary` redacted + truncated to 500 chars; base64 content → length + sha256[:8]); `token_prefix_seen` on failures only.
- Responses: `response_token_budget` default 8,000 tokens (tiktoken estimate); over budget → truncate at a page/paragraph boundary, `truncated: true` + `next`. List limits: `limit ≤ 20` (`list_documents` ≤ 25). Opaque base64-JSON cursors. Every tool: `title`, annotations, `inputSchema`, `outputSchema`; `structuredContent` + a text rendering. Tool names are stable; extensions only add optional params or new tools.
- Both protocol eras must work: 2026-07-28 (stateless, `server/discover`) and 2025-11-25 (`initialize`). Progress notifications flow on the request's SSE stream when the client sent `progressToken`.
- Data: schema **v4**; only new tables (`api_tokens`, `mcp_clients`, `mcp_calls`, `page_index`, `oauth_clients` reserved); `mcp_calls` retention `call_log_retention_days` = 30 / `call_log_max_rows` = 200,000 via the existing `Maintenance`.
- Admin: `/api/mcp/*` accepts only the existing server token or loopback (existing `auth.require_token`); SSE `/api/events` gains `mcp.call`, `mcp.client`, `mcp.token`.
- Web: new sidebar entry **MCP** directly above Settings; tabs 總覽 / Tokens / 連線 / 呼叫紀錄; token shown exactly once (creation/rotation response) with copy + snippets; page titles use the nav word, other copy in Traditional Chinese; existing design system (Panel, StatusBadge, Tabs, Dialog from `web/src/components/ui`), light + dark, usable at 400 px.
- Platform: Windows 11 host (`uv`, Node 24, pnpm 10); paths may be UNC/`\\?\`/junctions — refuse them where the spec says. Tests: `uv run pytest -m "not slow"` must stay green; `pnpm --dir web test`, `pnpm --dir web build`, `pnpm --dir web e2e` must stay green.
- Commit trailer (both lines at the end of every commit message made while executing this plan):
  ```
  Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba
  ```
- Concurrency note: another agent is editing `src/aidoc/server/app.py` (HTTP compression + static caching with an exclusion list that already contains `/mcp`, and pdf.js range tuning). Before every edit of `app.py`: `git pull --rebase`, re-read the file, keep your edits to the three MCP hooks (lifespan, `/mcp` route before `_mount_web`, Envelope passthrough), never touch the compression exclusion list except to confirm `/mcp` is in it. Task 14 has a test that `/mcp` responses carry no `Content-Encoding`.

## 1. Review Focus

Input classes the spec implies but no spec example exercises; each line's test is added to the task named.

1. **A single page bigger than the budget** (a 30k-token table page or a page-less `.docx` read as one chunk): `read_document` must return a non-empty slice cut at a paragraph boundary with `truncated: true` and `next.offset`, never an empty `markdown` and never a response over the budget. → Task 17 `test_single_oversized_page_is_sliced_not_dropped`.
2. **Queries that are FTS5 syntax or too short**: `"`, `*`, `(`, `-`, `NEAR`, `AND`, a lone CJK character or two (`學習` is only two trigram characters and matches nothing in trigram FTS) must never raise an SQLite error; they return hits (via the LIKE fallback) or an empty list. → Task 5 `test_short_and_syntax_queries_never_error`.
3. **Copy-paste damage to the token**: `bearer` in any case, surrounding whitespace, a trailing newline, a token truncated by one character (CRC fails) or wrapped in quotes. Scheme is case-insensitive and the value is stripped; anything that fails the format check is a 401 logged with `error_code = bad_format|bad_checksum` and the prefix seen. → Task 13 `test_token_copy_paste_variants_accepted` / `_rejected`.
4. **A document whose output is gone** (orphaned row, `.md` deleted, output dir renamed by hand): every read tool answers `isError` `output_missing` (or `document_not_found` for an unknown id), never a 500 / `UnexpectedToolError`. → Task 16 `test_output_missing_is_a_tool_error`.
5. **`content_base64` that is not clean base64**: a `data:application/pdf;base64,` prefix, embedded newlines/spaces, URL-safe alphabet, or a non-base64 string. The prefix is stripped, whitespace ignored, URL-safe accepted, garbage → `invalid_base64` with a hint; the declared size check happens **before** decoding. → Task 20 `test_base64_variants`.

## 2. File structure (new and modified; one responsibility per file)

```
pyproject.toml                         F-T2  + "mcp>=2.2,<2.3"
aidoc.toml                             F-T2  + [mcp] section (defaults)
src/aidoc/config.py                    F-T2  + Mcp dataclass, _SECTIONS["mcp"]
src/aidoc/server/api/settings.py       F-T2  + [mcp] validation (lists, paths, ranges)
src/aidoc/mcp/__init__.py              F-T3  package marker (empty)
src/aidoc/mcp/tokens.py                F-T3  format, CRC, base62, HMAC hash, server secret
src/aidoc/store.py                     F-T4  schema v4 + api_tokens/mcp_clients/mcp_calls methods; F-T5 page_index methods
src/aidoc/pageindex.py                 F-T5  pages_for_index(), index_document(), fts_query(), search_pages(), make_snippet(), backfill
src/aidoc/pipeline.py                  F-T6  index after upsert_document (one call)
src/aidoc/server/app.py                F-T6  start_page_index_backfill(); S-T14 lifespan, /mcp route, Envelope passthrough
src/aidoc/cli.py                       F-T6  backfill thread in cmd_serve; S-T14 extras["bind_port"]
src/aidoc/mcp/principal.py             F-T7  SCOPES, Principal, AuthFailure, current_principal, PatVerifier
src/aidoc/mcp/ratelimit.py             F-T8  RateLimiter (token bucket)
src/aidoc/mcp/budget.py                F-T9  estimate_tokens(), cut_to_budget(), encode_cursor()/decode_cursor()
src/aidoc/server/maintenance.py        F-T10 prune_mcp_calls in run_once
src/aidoc/mcp/errors.py                S-T11 ToolFailure, ERROR_CODES
src/aidoc/mcp/schemas.py               S-T11 pydantic input models + output models for every tool
src/aidoc/mcp/calllog.py               S-T12 CallRecorder (summarize_args, record, note_client, touch_token, idle watcher)
src/aidoc/mcp/gate.py                  S-T13 CallState, McpGate (ASGI)
src/aidoc/mcp/registry.py              S-T14 doc4ai_tool() registration wrapper (scope check, ToolFailure → result)
src/aidoc/mcp/middleware.py            S-T14 TOOL_SCOPES, make_middleware(), visible_tools()
src/aidoc/mcp/server.py                S-T14 INSTRUCTIONS, transport_security_for(), endpoint_urls(), build_mcp()
src/aidoc/mcp/docs.py                  S-T15 DocView cache, outline(), token_ranges(), page_warnings(), stale_job(), rewrite_assets()
src/aidoc/mcp/tools_read.py            S-T15..18 search_library, list_documents, get_document_info, read_document, get_chunks, get_job
src/aidoc/mcp/resources.py             S-T19 resource templates + list_resources override
src/aidoc/mcp/tools_convert.py         S-T20/21 convert_document, convert_path, check_local_path()
src/aidoc/server/api/documents.py      S-T22 start_reconvert() extracted from the endpoint
src/aidoc/mcp/tools_manage.py          S-T22 cancel_job, reconvert_document
tests/mcp/sdk_call.py                  S-T24 SDK-driven acceptance script (search → info → read ×3, token counts)
src/aidoc/mcp/snippets.py              A-T27 render_snippets() — single source for API, UI and README
src/aidoc/server/api/mcp.py            A-T25/26/27 /api/mcp/* router
README.md                              A-T27 "MCP" section (snippets from render_snippets)
web/src/api/types.ts                   A-T28 Mcp* types, "mcp.*" event kinds, origin "mcp", Settings.mcp
web/src/api/client.ts                  A-T28 + api.patch
web/src/api/mcp.ts                     A-T28 queries/mutations for /api/mcp/*
web/src/events/applyEvent.ts           A-T28 mcp.call / mcp.client / mcp.token merging
web/src/components/layout/nav.ts       A-T28 + MCP entry (Plug icon) above Settings
web/src/routes.tsx                     A-T28 + /mcp
web/src/pages/McpPage.tsx              A-T28 tabs shell (?tab=overview|tokens|clients|calls)
web/src/features/mcp/tokenStatus.ts    A-T29 pure: status + expiry warning
web/src/features/mcp/ScopeBadges.tsx   A-T29
web/src/features/mcp/CreateTokenDialog.tsx  A-T29
web/src/features/mcp/TokenRevealDialog.tsx  A-T29 one-time reveal + snippets
web/src/features/mcp/TokensTab.tsx     A-T29
web/src/features/mcp/ConfigSnippets.tsx A-T30 tabs per client + copy
web/src/features/mcp/Sparkline.tsx     A-T30
web/src/features/mcp/OverviewTab.tsx   A-T30
web/src/features/mcp/clientState.ts    A-T31 pure: active/idle from last_seen
web/src/features/mcp/ClientsTab.tsx    A-T31
web/src/features/mcp/mergeCalls.ts     A-T28 pure: prepend/dedupe/cap live rows (used by applyEvent)
web/src/features/mcp/callFilters.ts    A-T32 pure: URL ↔ CallFilters
web/src/features/mcp/CallsTab.tsx      A-T32
web/src/pages/SettingsPage.tsx         A-T33 [mcp] section
web/src/features/settings/diffSettings.ts  A-T33 lists compared by content
web/e2e/mcp.spec.ts                    A-T34
tests/mcp/                             F/S/A tests (conftest.py with mcp fixtures)
docs/superpowers/spikes/2026-10-mcp-sdk.md   F-T1 spike record
```

## 3. Binding decisions (resolved here; see also §5 ambiguities)

- **D1 Auth placement.** `McpGate` (ASGI, ours) → SDK `TransportSecurityMiddleware` (Host/Origin, 421/403) → SDK Streamable HTTP. The gate stores `CallState` in `scope["state"]["doc4ai"]`; the SDK middleware (Task 14) reads it from `ctx.request.scope["state"]`, sets `current_principal` for the handler and records the call; the gate records whatever the SDK answered without the middleware having recorded (421, 400 header mismatch, 405, 406, 415) as `protocol_error` with `http_status`. Spike S4 confirms `ctx.request` is the Starlette `Request` on both eras.
- **D2 Route composition.** `mcp_app = mcp.streamable_http_app(streamable_http_path="/mcp", stateless_http=True, json_response=False, transport_security=…, max_request_body_size=…)`; FastAPI gets `app.add_route("/mcp", McpGate(ctx, mcp_app, …), methods=["GET", "POST", "DELETE"])` **before** `_mount_web`. Starlette treats a non-function endpoint as an ASGI app and passes the scope with `path="/mcp"`, which the SDK's own `Route("/mcp")` matches — no `Mount`, no slash redirect. Spike S2 verifies; fallback in the spike.
- **D3 Scope filtering** happens in the SDK middleware after `call_next` for `tools/list` (`result.tools` filtered by `TOOL_SCOPES`). `convert_path` is always registered but hidden (and refused with `tool_disabled`) while `mcp.local_path_roots` is empty, so a settings change applies without a restart (spec says "不註冊"; externally identical).
- **D4 Tool errors** are raised as `ToolFailure(code, message, hint, **extra)`; `registry.doc4ai_tool` converts them to `isError: true` results with `structuredContent = {code, message, hint, …}` and a text rendering — exact mechanism per spike S6 (return a `CallToolResult` from the tool if the SDK passes it through, else `ToolError` with the JSON text). Output schemas per S7.
- **D5 Request-body peek.** The gate buffers the POST body (bounded by `max_request_body_size`; over → 413) to read `method` when the `Mcp-Method` header is absent (2025-era clients), replays it to the SDK via a receive wrapper, and rate-limits only `tools/call`.
- **D6 page_index.** FTS5 `tokenize='trigram'` when SQLite ≥ 3.34 (verified 3.50.4 here), else `unicode61` with `meta.page_index_tokenizer` recording the choice (search still works, substring matching degrades to word matching). Queries with every term shorter than 3 characters use `LIKE '%term%'` (case-insensitive via `lower()`) over `page_index` with a 2,000-row scan cap; terms are quoted for FTS (`"…"` with `"` doubled) so user text is never parsed as FTS syntax.
- **D7 Identity of docs.** `title` = output stem (`Path(output_dir).name`), `source_name` = `Path(source_path).name`; `updated_at` = `documents.created_at` (the pipeline rewrites it on every conversion).
- **D8 Jobs from MCP.** `jobs.origin = "mcp"` (new value; API `JobIn.origin` still validates only `web|cli` for `POST /jobs`), the token is attributed through `mcp_calls.job_id`; "jobs in flight per token" = distinct `job_id` in `mcp_calls` for that token whose job status is `queued|running`.
- **D9 Clients.** `mcp_clients` row keyed by `(token_id, client_name, client_version)`; `client_name = "unknown"` when `clientInfo` is absent. "Active" = `last_seen` within 300 s; `CallRecorder` emits `mcp.client` with `state: "new"|"active"|"idle"` (idle from a 30 s watcher thread).
- **D10 Cursors.** base64url(JSON): `list_documents` `{"k": <sort key>, "id": <doc_id>}`, `search_library` `{"skip": n}`, `get_chunks` `{"i": n}`, admin `/api/mcp/calls` `cursor=<last id>`. Invalid cursor → tool error `invalid_cursor` (tools) / 422 `invalid_cursor` (admin).
- **D11 Token estimate** = `aidoc.chunk.count_tokens` (tiktoken cl100k_base); when the tokenizer is unavailable, `len(text.encode()) // 3`, and `get_document_info.token_estimate.method = "tiktoken"|"bytes"`.
- **D12 Rotation** has no grace window in Phase 1 (spec default 0): the old token is revoked with `revoked_reason = "rotated"` the moment the new one is created; the new row copies name/scopes/note/rate limit and keeps the old TTL length (or the default when the old had none). `grace_seconds` is a Phase 1.5 hook.

## 4. Contract amendments to `2026-09-24-index.md` (additive only)

**§4 schema — v4 (`SCHEMA_VERSION = 4`, see A-M1):** adds exactly the spec §3 tables (`api_tokens`, `mcp_clients`, `mcp_calls` + 2 indexes, `page_index` FTS5, `oauth_clients` reserved). Row dicts: `token["scopes"]` (parsed from `scopes_json`); `_JSON_COLS` gains `scopes_json`. `delete_documents(status)` also deletes the `page_index` rows of those documents. New `Store` methods (names fixed; signatures in F-T4/F-T5):

```python
# api_tokens
create_api_token(*, name, prefix, token_hash, scopes: list[str], expires_at: float | None, note=None,
                 rate_limit_per_min=None, rotated_from=None) -> str
get_api_token(token_id) -> dict | None;  get_api_token_by_hash(token_hash) -> dict | None
list_api_tokens() -> list[dict]           # created_at DESC
update_api_token(token_id, **fields);     revoke_api_token(token_id, reason: str | None, at: float) -> bool
# mcp_clients
upsert_mcp_client(*, token_id, client_name, client_version, protocol_version, user_agent, ip, now) -> tuple[str, bool]
list_mcp_clients(token_id=None, active_since: float | None = None) -> list[dict];  get_mcp_client(client_id)
# mcp_calls
insert_mcp_call(**fields) -> int
list_mcp_calls(*, token_id=None, client_id=None, tool=None, status=None, since=None, until=None,
               before_id=None, limit=100) -> list[dict]          # id DESC
count_mcp_calls(since: float, token_id=None, errors_only=False) -> int
mcp_call_rows_since(since: float) -> list[dict]                   # for stats (tool_name, status, duration_ms, response_tokens_est, token_id, ts)
prune_mcp_calls(*, older_than_ts: float, max_rows: int) -> int
# page_index
replace_page_index(doc_id, pages: list[tuple[int | None, str]]) -> None;  delete_page_index(doc_id)
search_page_index(match: str, *, doc_ids: list[str] | None, limit: int) -> list[dict]   # {doc_id, page, text, rank}
like_page_index(needle: str, *, doc_ids, limit, scan_cap=2000) -> list[dict]
page_index_doc_ids() -> set[str];  page_index_tokenizer() -> str
```

**§7 config:** new section `[mcp]` exactly as spec §10 (`enabled`, `allowed_hosts`, `local_path_roots`, `default_token_ttl_days`, `max_token_ttl_days`, `allow_no_expiry`, `rate_limit_per_min`, `max_concurrent_jobs_per_token`, `max_upload_mb`, `response_token_budget`, `call_log_retention_days`, `call_log_max_rows`). `PUT /settings` whitelist: all keys; `local_path_roots` entries must be absolute, existing directories, not UNC/`\\?\` (422 `invalid_settings`); `allowed_hosts` entries `[A-Za-z0-9.\-\[\]:*]+` (422 otherwise); ints ≥ 1 except `call_log_retention_days` ≥ 0; `max_token_ttl_days ≥ default_token_ttl_days`.

**§8 API — `/api/mcp/*`** (all under the existing `api` router, so `require_token` applies; every body carries `workspace`):

| Method + path | Request | Response |
|---|---|---|
| `GET /mcp/status` | — | `{enabled, endpoint_urls: [str], bind: {host, port}, allowed_hosts: [str], protocol_versions: ["2026-07-28","2025-11-25","2025-06-18","2025-03-26"], sdk_version, active_clients, calls_24h, errors_24h, tokens_expiring_soon: n, local_path_roots: [str], plaintext_http: bool, tokenizer: str}` |
| `GET /mcp/tokens` | — | `{tokens: [Token]}` where `Token = {id, name, prefix, scopes, note, created_at, expires_at, revoked_at, revoked_reason, rotated_from, rate_limit_per_min, last_used_at, last_used_ip, last_client, status: "active"\|"expired"\|"revoked", calls_24h, errors_24h}` |
| `POST /mcp/tokens` | `{name (1–80), scopes: [str] ⊇ ["doc4ai:read"], expires_in_days?: int (1..max) \| 0 (= never; needs allow_no_expiry), note? (≤ 500), rate_limit_per_min? (1..100000)}` | `201 {token: "<plaintext, once>", record: Token, snippets: [Snippet]}`; 422 `validation_error` / `invalid_scopes` / `ttl_too_long {max}` / `no_expiry_disabled` |
| `PATCH /mcp/tokens/{id}` | `{name?, note?, rate_limit_per_min?: int \| null}` | `{token: Token}`; 404; 422 `scopes_immutable` when `scopes` is present |
| `POST /mcp/tokens/{id}/revoke` | `{reason?}` | `{token: Token}`; 404; 409 `already_revoked` |
| `POST /mcp/tokens/{id}/rotate` | — | `201 {token, record, snippets, revoked: Token}`; 404; 409 `already_revoked` |
| `GET /mcp/clients` | `?active=1&token_id=` | `{clients: [Client]}`, `Client = {id, token_id, token_name, client_name, client_version, protocol_version, user_agent, first_seen, last_seen, last_ip, request_count, active: bool}` |
| `GET /mcp/calls` | `?token_id=&client_id=&tool=&status=&since=&until=&cursor=&limit=` (limit 1..100, default 50) | `{calls: [Call], next_cursor: str \| null}`; `Call` = `mcp_calls` row + `token_name`, `client_name`; 422 `invalid_cursor` |
| `GET /mcp/stats` | `?window=24h\|7d` | `{window, since, tools: [{tool, calls, errors, error_rate, p50_ms, p95_ms, tokens_median}], tokens: [{token_id, name, calls, errors}], series: [{ts, calls, errors}]}` (24 hourly buckets / 28 six-hour buckets) |
| `GET /mcp/config-snippets` | `?token_id=&endpoint=` | `{endpoint_url, snippets: [Snippet]}`, `Snippet = {client: "claude-code"\|"cursor"\|"vscode"\|"claude-desktop", title, language, text}`; token placeholder `<YOUR_TOKEN>`; 404 unknown token; 422 `unknown_endpoint` |

Jobs: `Job.origin` may now be `"mcp"`. `/mcp` itself is **not** under `/api` and is **not** enveloped; its error bodies are `{"error": "<code>", "error_description": "<text>"}` (OAuth style, spec §4.3).

**SSE kinds added:** `mcp.call` = `{id, ts, token_id, token_name, client_id, client_name, method, tool_name, status, error_code, http_status, duration_ms, response_tokens_est, job_id}`; `mcp.client` = `Client + {state: "new"|"active"|"idle"}`; `mcp.token` = `{id, name, prefix, action: "created"|"updated"|"revoked"|"rotated"}`. The raw token never appears in any event.

### 4.1 Amendments made while implementing Parts F and S (binding for Part A)

Recorded by the F/S implementer (2026-10-02); details and the tests that pin them are in the "Implementation notes / deviations" sections of the F and S phase files.

- **`ChunkOut.chunk_id`** (tool `get_chunks`, resource `doc4ai://chunks/{doc_id}/{chunk_id}`) is `c0007` for the default 800-token chunking and `c0007m<max_tokens>` otherwise (helpers `aidoc.mcp.docs.chunk_uri_id()` / `chunk_index()`). `aidoc chunk` ids (`<stem>#0007`) cannot be used: `#` ends a URI.
- **`/mcp` gate answers** (all `{"error", "error_description"}` bodies, never enveloped): `401 invalid_token` (+ `WWW-Authenticate`), `400 token_in_query`, `404 mcp_disabled`, `413 payload_too_large`, `429 rate_limited` (+ `Retry-After`), **new:** `400 method_mismatch` when an `Mcp-Method` header disagrees with the JSON-RPC body (the body decides the method), `?access_token=` (any case) is refused like `?token=`, **new:** `403 forbidden_origin` when an `Origin` header is present and is not the request's own `Host` (same-origin rule, logged `protocol_error`/`http_403`), **new:** `405 method_not_allowed` (`Allow: POST`) for an authenticated `GET /mcp` (stateless server: no standalone stream; logged `http_405`).
- **`tools/list`** is sorted by tool name after the scope filter.
- **Tool result text**: every result carries `structuredContent` plus one compact text block — `read_document`: `<!-- doc4ai read_document: doc_id=…; pages a-b; truncated=…; next=… -->` + the Markdown; `get_chunks`: one `<!-- chunk … -->` header per chunk + text; everything else: compact JSON. Search hits also come as `resource_link` blocks. Read/chunk budgets reserve 100 tokens for these headers (`registry.TEXT_RESERVE_TOKENS`).
- **`endpoint_urls(cfg, bind_host, port)`** also lists the MagicDNS FQDN when `bind_host` is a tailnet IP (100.64.0.0/10), found with `tailscale status --json` (3 s timeout) or a `*.ts.net` reverse lookup (`server.tailnet_names()`); FQDN and short name are allowed Hosts automatically. Configured `mcp.allowed_hosts` are listed for non-loopback binds. `/api/mcp/status.endpoint_urls` should use this as is.
- **`CallRecorder`** gains `remember_identity(token_id, user_agent, name, version)` / `recall_identity(token_id, user_agent)` (1 h): 2025-era stateless clients send `clientInfo` only on `initialize`; later messages of the same token + User-Agent are attributed to it. The middleware (not the gate) touches `api_tokens.last_*` so `last_client` is `name/version` (else the User-Agent).
- **`Store.active_mcp_jobs(token_id)`** (Task 20) is part of the Store surface.
- **Test helpers** (`tests/mcp/conftest.py`): `mcp_client()` is built on `mcp.client.client.Client(streamable_http_client(url, http_client=httpx2.AsyncClient(headers=…)), mode=…, cache=None)` (spike S3) and unwraps single-exception groups; `mcp_call()` lists tools before calling (otherwise the SDK client lists *after* the call to validate output, and the call is not the newest `mcp_calls` row). `tests/mcp/sdk_call.py` steps carry `tokens_structured` next to `tokens_text`.

## 5. Spec ambiguities resolved in this plan (binding)

| # | Ambiguity | Resolution |
|---|---|---|
| A-M1 | Spec §3 titles the schema "v4" but says "current + 1"; the DB is at v2 (per-page work did not bump it). | `SCHEMA_VERSION = 4` (matches the spec title and the brief); 3 is skipped and said so in `store.py`. The migration is idempotent (`CREATE … IF NOT EXISTS`), so a v2 or a hypothetical v3 DB upgrades the same way. |
| A-M2 | Token example shows `_` between body and CRC; the prose lists them without a separator. | Follow the example: `doc4ai_pat_<43>_<6>`; the `_` makes parsing unambiguous (base62 has no `_`). |
| A-M3 | `read_document` has no way to continue inside one oversized page; page-less documents need a chunk position. | Add optional `chunk: int` (0-based chunk index, page-less documents only) and `offset: int` (0-based character offset inside the first unit — page or chunk — when that unit had to be cut). `next` is `{pages?, chunk?, offset?}`: `{pages: "16-19"}` after whole pages, `{pages: "16-19", offset: n}` after a cut page, `{chunk: i}` / `{chunk: i, offset: n}` for page-less documents. Allowed by spec §5.1 ("擴充只加選填參數"). |
| A-M4 | `page_warnings: [{page, reason}]` vs `flagged_pages: [{page, reason, repaired}]`. | Both use `{page, reason, repaired, reasons}`: `reason` = first reason code, `reasons` = all, `repaired = bool(repaired_by)`. |
| A-M5 | `convert_path` "不註冊" when the whitelist is empty vs hot-applied settings. | D3: always registered, hidden from `tools/list` and refused with `tool_disabled` while the list is empty. |
| A-M6 | Where to record a job's token (spec: "記錄 token_id"). | D8: through `mcp_calls.job_id`; no new column on `jobs`. |
| A-M7 | `server/discover` `instructions` language. | English (model-facing), content as spec §4.1: search → info → read; mentions `<!-- page: N -->` markers and the 8k budget. Tool descriptions likewise English. |
| A-M8 | `mcp_clients` idle transition needs a timer; the spec only names the event. | D9: 30 s watcher in `CallRecorder`; the web also derives active/idle from `last_seen` so a missed event is harmless. |
| A-M9 | `resources/list` pagination: the SDK's list handler may ignore the cursor. | Spike S9 decides: SDK pagination if `_handle_list_resources` forwards `params`; otherwise the 100 most recent documents are listed and `ttlMs = 60000`; templates always listed. Recorded in the spike file. |
| A-M10 | `convert_document` magic-byte check scope. | PDF (`%PDF`), zip-based Office (`PK\x03\x04`), PNG, JPEG, GIF, WebP (`RIFF…WEBP`), TIFF; other allowed extensions (`.html`, `.md`, `.txt`, `.csv`, `.json`, `.xml`, legacy `.doc/.xls/.ppt` with `\xD0\xCF\x11\xE0`) check only that the bytes are not one of the above with a mismatching extension. Mismatch → `unsupported_file`. |
| A-M11 | Spec §7 `GET /api/mcp/tokens` "每筆附 24 小時呼叫數、錯誤數、最後 client". | `calls_24h`, `errors_24h` computed from `mcp_calls`; `last_client` from `api_tokens.last_client`. |
| A-M12 | `delete_document` is Phase 1.5; nothing in Phase 1 removes `page_index` rows for deleted docs except the existing `DELETE /documents/orphaned`. | `Store.delete_documents` deletes matching `page_index` rows; Phase 1.5 `delete_document` must call `delete_page_index` too (hook noted). |
| A-M13 | Rate limit "預設 60 次 tools/call 每分鐘" — burst? | Token bucket with capacity = per-minute limit and refill = limit/60 per second (a full minute of burst, then steady state). `Retry-After` = ceil(seconds until one token). |
| A-M14 | `/api/mcp/config-snippets?token_id=` — what does the id change? | Only validation (404 for an unknown id) and the snippet's server alias stays `doc4ai`. The real token appears only in the create/rotate response. `?endpoint=` picks one of `endpoint_urls` for multi-interface binds. |
| A-M15 | `GET /mcp/status.protocol_versions` — which list? | The SDK's known versions: `MODERN_PROTOCOL_VERSIONS + HANDSHAKE_PROTOCOL_VERSIONS` minus `2024-11-05` (we never serve HTTP+SSE), i.e. `["2026-07-28","2025-11-25","2025-06-18","2025-03-26"]`. |

## 6. Phase 1.5 / Phase 2 — out of scope, hooks to leave in place

- **1.5 `delete_document(doc_id, confirm_title)`** (scope `doc4ai:manage`, `destructiveHint: true`): register in `tools_manage.py`; must call `store.delete_page_index(doc_id)` and `fsops.remove_tree(output_dir)`; `TOOL_SCOPES["delete_document"] = "doc4ai:manage"` already listed as a comment in Task 14.
- **1.5 prompts** `summarize_document(doc_id, pages?)`, `ask_document(doc_id, question)`: names reserved in `server.py` as a comment; `MCPServer.prompt` decorator.
- **1.5 rotation grace** (`grace_seconds`, default 0): `revoke_api_token(..., at=now + grace)` already takes `at`; the verifier treats `revoked_at > now` as still valid (Task 7 tests this edge so the hook is real).
- **2 OAuth**: `Principal.kind = "oauth"`, `client_id`; `PatVerifier.verify` returns the same dataclass; `oauth_clients` table exists (unused); `McpGate` gets a JWT branch and a `resource_metadata` parameter on its 401 (`www_authenticate(resource_metadata: str | None)` helper in Task 13 takes the argument today and is called with `None`); `/.well-known/` is already passed through `EnvelopeMiddleware`.
- **2 public HTTPS / canonical URL**: `endpoint_urls()` in `server.py` is the single place that knows the server's URLs.

## 7. Coverage table (spec section → tasks)

| Spec | Requirement | Task(s) |
|---|---|---|
| §1 分期 Phase 1 table | local/tailnet clients, Bearer PAT, `/mcp` Streamable HTTP, read + convert + convert:local + manage (no delete), admin pages | all; delete excluded (§6) |
| §1 成功標準 1 | Claude Code `claude mcp add --transport http … --header`; search → info → read on the 1192-page book ≤ 8,000 tokens each | S-T24 script, A-T35 checklist rows 1–6 |
| §1 成功標準 2 | both protocol eras | F-T1 (S3), S-T14 `test_both_eras_list_tools` |
| §1 成功標準 3 | revoke → next request 401, admin sees it within 5 s | S-T13, S-T23 `test_revoked_token_401_is_logged`, A-T34 e2e |
| §1 成功標準 4 | PAT never valid on `/api/*`; scope filtering of `tools/list` | S-T23 `test_pat_rejected_on_api`, S-T14 `test_tools_list_filtered_by_scope` |
| §1 成功標準 5 | every request logged, never token or full args | S-T12 `summarize_args`, S-T13 fallback logging, S-T23 `test_token_never_persisted_or_streamed` |
| §2.1 Principal, isolation | dataclass, verifier, `/api` isolation | F-T7, S-T23 |
| §2.2 scopes | table, defaults, decorator, filtering, `forbidden_scope` | F-T7 `SCOPES`, S-T14 registry/middleware, A-T29 dialog defaults + red warnings |
| §2.3 token format, HMAC, secret.key, prefix, expiry rules, rotate | | F-T3, F-T7, A-T25 |
| §3 schema v4 tables, indexes, FTS5 trigram, page_index on finalize/reconvert/startup, retention, active = 5 min | | F-T4, F-T5, F-T6, F-T10, D9 |
| §4.1 SDK pin, `MCPServer`, `/mcp` before SPA, stateless/json_response, lifespan, Envelope exclusions, no `?token=`, no SSE transport, instructions | | F-T1, F-T2, S-T13, S-T14 |
| §4.2 bind 127.0.0.1, allowed_hosts auto + config, allowed_origins = UI, plaintext notice, token even on loopback | | S-T14 `transport_security_for`, S-T13, A-T30 notice |
| §4.3 auth flow steps 1–4, 401 shape without `resource_metadata`, real-client 401 test | | S-T13, F-T7, F-T8, S-T12 `touch_token`, A-T35 rows 7–9 |
| §5.1 common rules (annotations, outputSchema, budget, cursors, errors, compat, doc_id, page_warnings) | | S-T11, S-T14 registry, F-T9, S-T15..18 |
| §5.2 tool table (scopes + annotations) | | S-T14 `TOOL_SCOPES`, S-T15..22 |
| §5.3 `search_library` | FTS bm25, ≤ 3 pages/doc, `more_in_doc`, snippet, resource_link | F-T5, S-T15 |
| §5.3 `list_documents` | sort, filters, cursor, 25 | S-T15 |
| §5.3 `get_document_info` | outline ≤ 200, ranges per 20 pages, flags, page_map, resources | S-T16 |
| §5.3 `read_document` | pages/heading, budget, `next`, chunk mode, assets as `doc4ai://`, errors | S-T17 |
| §5.3 `get_chunks` | params like `aidoc chunk`, cache by (doc_id, max_tokens) | S-T18 |
| §5.3 `get_job` | shape | S-T18 |
| §5.3 `convert_document` | upload area, job origin mcp, token attribution, wait + progress, cache hit, ext/magic, disk, concurrent cap | S-T20 |
| §5.3 `convert_path` | resolve, UNC/`\\?\`, symlink, whitelist, not registered when empty | S-T21 (+ D3) |
| §5.3 `cancel_job`, `reconvert_document` | `{changed}`, `source_missing` | S-T22 |
| §6 resources + templates, `resources/list` documents only with ttlMs, links in tool results | | S-T19, S-T15 |
| §6 prompts (1.5) | reserved names | §6 hooks |
| §7 admin API table | status, tokens CRUD, clients, calls, stats, snippets, settings keys | A-T25, A-T26, A-T27, F-T2 |
| §7 SSE `mcp.call` / `mcp.client` / `mcp.token` | | S-T12, A-T25, A-T28 |
| §8 pages 1–4, design system, dialogs, one-time reveal, live updates | | A-T28..32 |
| §8.1 snippets single source (UI + README) | | A-T27, A-T30 |
| §9 error table (expired/revoked, scope, 429, not found, stale, chunk mode, budget, too large, path, disabled 404, restart) | | S-T13, S-T14, S-T17, S-T20, S-T21; `mcp.enabled=false` → S-T13 `test_disabled_is_404` |
| §9 must-test list (401 behaviour of Claude Code/Cursor/VS Code, protocol versions per client, mcp-remote `--allow-http`) | | A-T35 rows 7–12 (manual) |
| §10 `[mcp]` config + settings whitelist | | F-T2, A-T33 |
| §11 tests (unit / protocol / security / web / real clients) | | F-T3..T9 unit, S-T14 protocol, S-T23 security, A-T29..34 web, A-T35 real |
| §12 Phase 2 reserved | hooks | §6 |
| §13 out of scope | nothing to build | — |

## 8. Real-client acceptance checklist

Lives in `2026-10-01-mcp-p3-admin.md` Task 35 (scripted rows run `tests/mcp/sdk_call.py`; manual rows are marked). Results go into the deviations section of that file.

## 9. Definition of done

1. All three phase acceptance checklists pass on the Windows host.
2. `uv run pytest -m "not slow"` green (including `tests/mcp/`); `pnpm --dir web test`, `pnpm --dir web build`, `pnpm --dir web e2e` green.
3. Spike file `docs/superpowers/spikes/2026-10-mcp-sdk.md` records every S-question's answer and the decision taken.
4. Task 35 checklist rows 1–6 pass against `http://ulove-arrangement.tail74077f.ts.net:3333/mcp` with Claude Code; manual rows recorded (pass/fail/not run) in the deviations section.
5. `README.md` has the MCP section; `aidoc.toml` has `[mcp]`.
