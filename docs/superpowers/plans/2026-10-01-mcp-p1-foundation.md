# MCP Server — Part F: Foundation (Tasks 1–10)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Read `docs/superpowers/plans/2026-10-01-mcp-server.md` (index) first: §0 constraints, §3 decisions, §4 contracts, §5 ambiguities bind every task here.

**Goal:** Everything the MCP server and admin need underneath them: the SDK facts (spike), the dependency and `[mcp]` config, the token format and server secret, schema v4 with the four new tables, page indexing (write path + startup backfill + search primitives), the `Principal`/`PatVerifier`, the rate limiter, the response-budget and cursor helpers, and call-log retention.

**Architecture:** Pure modules first (`tokens.py`, `budget.py`, `ratelimit.py`, `pageindex.py`) with unit tests; `Store` grows additive methods; the only existing code touched is `config.py`, `settings.py`, `store.py`, `pipeline.py` (one call), `app.py`/`cli.py` (backfill thread), `maintenance.py` (one call).

**Tech Stack:** Python 3.10-compatible, sqlite3 FTS5, `hmac`/`secrets`/`zlib`, tiktoken via `aidoc.chunk.count_tokens`, pytest.

**Spec:** `docs/superpowers/specs/2026-10-01-mcp-server-design.md` §2, §3, §10, §11 (unit list).

## Conventions for every task in this file

- Tests live in `tests/mcp/` (create `tests/mcp/__init__.py` empty in Task 3). Shared fixtures `tmp_root`, `ctx`, `client`, `token_ctx`, `live_server` come from `tests/conftest.py`.
- Run tests with `uv run pytest <path> -v` from `D:\projects\ai-friendly-doc`. Lint: `uv run ruff check src tests`.
- Commit after each task with the trailer from the index §0.

---

### Task 1: SPIKE — verify the `mcp` 2.2 SDK against this codebase and record decisions

**Files:**
- Create: `docs/superpowers/spikes/2026-10-mcp-sdk.md`
- Create (temporary, deleted at the end of the task): `tests/mcp/_spike/*.py` scratch scripts (never committed)

**Interfaces:**
- Produces: the spike file with one entry per question S1–S14: *finding* (what the SDK does, with the function/class name and a 1–5 line code excerpt or output), *decision* (which branch of the decision rule below applies), *consequence for tasks* (which later task changes, if any). Later tasks say "per spike S<n>" where they depend on it.

Pre-known facts (from an ephemeral probe of `mcp==2.2.0` on 2026-10-01; re-verify, do not trust blindly): `from mcp.server import MCPServer`; `MCPServer.__init__(name, title, description, instructions, …, token_verifier=, auth=, lifespan=, cache_hints=, middleware=)`; `MCPServer.streamable_http_app(*, streamable_http_path="/mcp", json_response=False, stateless_http=False, max_request_body_size=4194304, session_idle_timeout=1800, max_sessions=10000, transport_security=None, host="127.0.0.1") -> Starlette`; `mcp.session_manager` exists only after that call and `session_manager.run()` is an async context manager; `mcp.server.transport_security.TransportSecuritySettings(enable_dns_rebinding_protection, allowed_hosts, allowed_origins)` (Host mismatch → 421, Origin mismatch → 403, POST without `application/json` → 400); `MCPServer.tool(name, title, description, annotations: ToolAnnotations, icons, meta, structured_output)`; `mcp.types.ToolAnnotations(title, read_only_hint, destructive_hint, idempotent_hint, open_world_hint)`; `mcp.server.context.ServerRequestContext(session, lifespan_context, protocol_version, method, params, request_id, meta, request, …)`; middleware = `async def mw(ctx: ServerRequestContext, call_next)` passed as `MCPServer(middleware=[mw])`, runs for every inbound message, `ctx.request_id is None` for notifications, a failing request reaches it as a raised `MCPError`; tool `Context` has `.headers`, `.connection` (`Connection.client_params.client_info`, `.protocol_version`, `.session_id`), `.report_progress(progress, total, message)` (no-op without a progress token), `.meta`; `mcp.server.mcpserver.exceptions.ToolError` → `isError`; `mcp.server.caching.CacheHint(ttl_ms, scope)`; `mcp.types.LATEST_PROTOCOL_VERSION == "2026-07-28"`; client: `from mcp.client.client import Client` with `Client(url_or_transport_or_server, mode="auto"|"legacy", client_info=Implementation(name, version))` as an async context manager exposing `list_tools()`, `call_tool(name, args, progress_callback=)`, `discover…`, `protocol_version`; `mcp.client.streamable_http.streamable_http_client(url, http_client=httpx2.AsyncClient(...))` yields transport streams (pass headers through the `httpx2.AsyncClient`); `httpx2` 2.13 has `ASGITransport`; `ClientSession.validate_tool_result` raises when `structured_content is None` for a tool with an output schema; `MCPServer.list_tools()` is the instance method `_handle_list_tools` calls (no per-request filter built in).

- [ ] **Step 1: Install the SDK into the main env and record versions**

Run: `uv add "mcp>=2.2,<2.3"` then `uv run python -c "import importlib.metadata as m, sqlite3, sys; print(sys.version); print('mcp', m.version('mcp'), 'mcp-types', m.version('mcp-types'), 'httpx2', m.version('httpx2')); print('sqlite', sqlite3.sqlite_version)"`
Expected: Python 3.1x, `mcp 2.2.x`, sqlite ≥ 3.34. Record all four in the spike file header. Then run `uv run pytest -m "not slow" -q` — Expected: the existing suite is still green (the SDK pulls `pywin32`, `pyjwt[crypto]`, `jsonschema`, `opentelemetry-api`, `sse-starlette`; none may break imports). If `uv add` fails to resolve, record the conflict and stop: the plan cannot proceed without the pin.

- [ ] **Step 2: Answer S1–S14 with scratch scripts under `tests/mcp/_spike/` (not committed)**

For each question write the finding + decision into `docs/superpowers/spikes/2026-10-mcp-sdk.md` using this template per entry:

```markdown
### S<n> <title>
**Finding.** <what the SDK does; names; excerpt/output>
**Decision.** <branch taken from the decision rule>
**Affects.** <task numbers>
```

Questions and decision rules:

- **S1 Names.** Confirm every name in the pre-known list above exists with those signatures (`inspect.signature`). Decision rule: any mismatch → write the real name next to the plan's name in the spike; later tasks use the real name.
- **S2 Mounting without `Mount`.** Build `MCPServer("spike")` with one tool, `mcp_app = mcp.streamable_http_app(streamable_http_path="/mcp", stateless_http=True)`, a FastAPI app with `app.add_route("/mcp", mcp_app, methods=["GET","POST","DELETE"])` registered before a catch-all `/{p:path}` GET route, lifespan entering `mcp.session_manager.run()`; run it under uvicorn (port 0) and connect with `Client("http://127.0.0.1:<port>/mcp")` → `list_tools()`. Also confirm `GET /anything` still reaches the catch-all and `POST /mcp/` (trailing slash) is **not** redirected into the MCP app. Decision rule: works → D2 stands. Fails → fallback: `mcp.streamable_http_app(streamable_http_path="/")` + `app.mount("/mcp", …)` and set `app.router.redirect_slashes = False`; verify the same checks; record which.
- **S3 Both eras in tests.** With the S2 server, run `Client(url, mode="auto")` and `Client(url, mode="legacy")`; assert `client.protocol_version` is `"2026-07-28"` and `"2025-11-25"` respectively and both `list_tools()`. Also confirm how to pass `Authorization` (an `httpx2.AsyncClient(headers=…)` handed to `streamable_http_client`, then `Client(transport)` or `ClientSession`). Record the exact helper code that works; Task 14 copies it into `tests/mcp/conftest.py` as `mcp_client(base_url, token, mode)`.
- **S4 Request object in middleware (both eras).** Middleware that records `type(ctx.request)`, `ctx.request.scope.get("state")`, `ctx.request.client`, `ctx.request.headers.get("user-agent")` for `tools/list` and `tools/call` under both modes. An outer ASGI wrapper sets `scope.setdefault("state", {})["doc4ai"] = "marker"` before calling the SDK app. Decision rule: `ctx.request` is a Starlette `Request` and `scope["state"]["doc4ai"] == "marker"` in both eras → D1 stands. Otherwise: set a `ContextVar` in the ASGI wrapper and check it is visible inside the tool under both modes (stateless per-request task groups inherit context); record which path tools/middleware must use.
- **S5 Client identity and protocol version inside middleware and tools.** From the middleware: `ctx.protocol_version`, and where `client_info` lives (`ctx.session.client_params.client_info`? `ctx.session.connection.client_params`?). From a tool: `tool_ctx.connection.client_params.client_info`, `tool_ctx.connection.protocol_version`. Test with `Client(..., client_info=Implementation(name="spike-client", version="9.9"))` in both modes and without `client_info` (expect `None`). Record the exact attribute paths; Task 12/14 use them.
- **S6 Error results with structuredContent.** In a tool with `structured_output=True` returning a pydantic model, test (a) `return CallToolResult(is_error=True, content=[TextContent(type="text", text="…")], structured_content={"code": "x", "message": "m", "hint": "h"})` — does the SDK pass it through unchanged (check `result.is_error` and `result.structured_content` on the client)? (b) `raise ToolError(json.dumps({...}))` — what does the client receive? Also (c) return a `list[ContentBlock]` including a `ResourceLink` plus a model — is that possible, or does `structured_output` require a single model? Decision rule: (a) works → `registry.doc4ai_tool` returns `CallToolResult` for failures and may append `ResourceLink` blocks for hits; (a) fails → failures use (b) and the text is the JSON, `structured_content` absent; links go only into `structuredContent.uri`.
- **S7 Client-side output validation on error results.** With the S6 tool, does `Client.call_tool` raise/validate when `is_error=True` and `structured_content` does not match the output schema (or is absent)? Decision rule: no validation on errors → output models stay success-only. Validation happens → every output model gets `error: ErrorInfo | None = None` and all other fields become optional; failures return `{"error": {...}}` (Task 11 adopts this shape).
- **S8 Progress.** Tool calling `await ctx.report_progress(1, 3, "one")` ×3; client `call_tool(..., progress_callback=cb)` in both modes. Decision rule: notifications received in auto mode with `json_response=False` → Task 20 uses `report_progress`. Not received in legacy stateless mode → record that legacy clients get no progress (spec accepts: notifications dropped on the stateless legacy leg) and Task 20's test asserts only the modern path.
- **S9 `resources/list` pagination and templates.** Subclass `MCPServer` overriding `async def list_resources(self)`; check `_handle_list_resources` signature (does it pass `params.cursor`?) and that `list_resource_templates()` returns templates declared with `@mcp.resource("doc4ai://documents/{doc_id}")`. Check how a resource function returns binary (bytes → `BlobResourceContents`?) and text with a `mime_type`. Decision rule per index A-M9.
- **S10 `cache_hints` keys.** `MCPServer(cache_hints={"tools/list": CacheHint(300000, "private"), "resources/list": CacheHint(60000, "private"), "server/discover": CacheHint(300000, "private")})` — confirm the `CacheableMethod` literal values and that `ListToolsResult.ttl_ms` arrives at the client. Decision rule: keys differ → use the real ones.
- **S11 `Mcp-Method` header and body shape for rate limiting.** Capture one modern `tools/call` POST and one legacy `tools/call` POST (log headers + body in the S2 ASGI wrapper). Confirm the modern request carries `Mcp-Method: tools/call` and `Mcp-Name`, the legacy one does not, and the body is one JSON object with `"method"`. Decision rule: as expected → D5 stands (header first, body fallback). Otherwise record the real headers.
- **S12 Response for a rejected Host/Origin and for a non-JSON POST** (transport security): status codes and bodies (421/403/400), to be mapped by the gate's fallback logging (Task 13). Also what the SDK answers to `GET /mcp` and `DELETE /mcp` in stateless mode (405?). Record them.
- **S13 401 behaviour of real clients** (spec §4.3, §9) — **manual, do it now if Claude Code is installed, else carry to Task 35**: `claude mcp add --transport http spike http://127.0.0.1:<port>/mcp --header "Authorization: Bearer nope"` against a server whose gate answers 401 with `WWW-Authenticate: Bearer realm="doc4ai", error="invalid_token"` (no `resource_metadata`). Does Claude Code show a plain auth error, or try OAuth discovery (`GET /.well-known/oauth-protected-resource…`)? Log every request path the client makes. Decision rule: plain error → Phase 1 header is final. OAuth attempts → add `error_description="Create a token in Doc4AI Studio → MCP"` to the header and record the client's exact behaviour for Task 35.
- **S14 SQLite trigram tokenizer.** In the **main env interpreter** (`uv run python`): create `fts5(doc_id UNINDEXED, page UNINDEXED, text, tokenize='trigram')`, insert `機器學習 machine learning 第三章`, assert `MATCH '"學習"'` returns **no** row (2 chars), `MATCH '"機器學習"'` and `MATCH '"learn"'` return the row, and `bm25()` works. Record `sqlite3.sqlite_version`. Decision rule: trigram works → D6 primary; `OperationalError: unrecognized tokenizer` → D6 fallback (`unicode61`) becomes the default on this host and Task 5's test for the trigram-only behaviour is marked `skipif`.

- [ ] **Step 3: Delete the scratch scripts, keep the spike file, commit**

Run: `git rm -r --cached tests/mcp/_spike 2>/dev/null; rm -rf tests/mcp/_spike; git status --short`
Expected: only `pyproject.toml`, `uv.lock` and the spike file are changed/added.

```bash
git add pyproject.toml uv.lock docs/superpowers/spikes/2026-10-mcp-sdk.md
git commit -m "spike(mcp): verify mcp 2.2 SDK mounting, eras, middleware context, errors, progress, FTS5 trigram

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 2: `[mcp]` config section + settings whitelist

**Files:**
- Modify: `src/aidoc/config.py` (add `Mcp` dataclass, register in `_SECTIONS`, field on `AidocConfig`)
- Modify: `aidoc.toml` (append `[mcp]` with defaults and comments)
- Modify: `src/aidoc/server/api/settings.py` (`_validate`: list keys, path rules, ranges)
- Test: `tests/unit/test_config.py` (append), `tests/api/test_system_settings.py` (append)

**Interfaces:**
- Produces: `AidocConfig.mcp: Mcp` with fields `enabled: bool = True`, `allowed_hosts: list[str] = []`, `local_path_roots: list[str] = []`, `default_token_ttl_days: int = 90`, `max_token_ttl_days: int = 365`, `allow_no_expiry: bool = False`, `rate_limit_per_min: int = 60`, `max_concurrent_jobs_per_token: int = 3`, `max_upload_mb: int = 20`, `response_token_budget: int = 8000`, `call_log_retention_days: int = 30`, `call_log_max_rows: int = 200000`. `GET /api/settings` returns the section; `PUT /api/settings {"mcp": {...}}` validates per index §4.

- [ ] **Step 1: Write the failing tests**

Append to `tests/unit/test_config.py`:

```python
def test_mcp_section_defaults_and_load(tmp_path, monkeypatch):
    from aidoc.config import load_config
    p = tmp_path / "aidoc.toml"
    p.write_text('[mcp]\nenabled = false\nallowed_hosts = ["box.tail74077f.ts.net"]\nlocal_path_roots = []\n'
                 'response_token_budget = 4000\n', encoding="utf-8")
    cfg = load_config(p)
    assert cfg.mcp.enabled is False
    assert cfg.mcp.allowed_hosts == ["box.tail74077f.ts.net"]
    assert cfg.mcp.local_path_roots == []
    assert cfg.mcp.response_token_budget == 4000
    assert cfg.mcp.default_token_ttl_days == 90 and cfg.mcp.max_token_ttl_days == 365
    assert cfg.mcp.allow_no_expiry is False and cfg.mcp.rate_limit_per_min == 60
    assert cfg.mcp.max_concurrent_jobs_per_token == 3 and cfg.mcp.max_upload_mb == 20
    assert cfg.mcp.call_log_retention_days == 30 and cfg.mcp.call_log_max_rows == 200000
    assert "mcp" in cfg.to_dict() and cfg.to_dict()["mcp"]["enabled"] is False


def test_committed_aidoc_toml_has_mcp_section():
    from pathlib import Path
    from aidoc.config import load_config
    cfg = load_config(Path(__file__).resolve().parents[2] / "aidoc.toml")
    assert cfg.mcp.enabled is True and cfg.mcp.local_path_roots == []
```

Append to `tests/api/test_system_settings.py`:

```python
def test_settings_mcp_section_roundtrip(client, tmp_root):
    s = client.get("/api/settings").json()["settings"]
    assert s["mcp"]["response_token_budget"] == 8000 and s["mcp"]["local_path_roots"] == []
    roots = tmp_root / "shared"
    roots.mkdir()
    r = client.put("/api/settings", json={"mcp": {"response_token_budget": 6000, "local_path_roots": [str(roots)],
                                                  "allowed_hosts": ["box.tail74077f.ts.net", "100.64.0.9:*"]}})
    assert r.status_code == 200, r.text
    got = r.json()["settings"]["mcp"]
    assert got["response_token_budget"] == 6000 and got["local_path_roots"] == [str(roots)]
    assert got["allowed_hosts"] == ["box.tail74077f.ts.net", "100.64.0.9:*"]


def test_settings_mcp_rejects_bad_values(client, tmp_root):
    bad = [
        {"local_path_roots": ["\\\\nas\\share"]},                    # UNC
        {"local_path_roots": ["relative/dir"]},                     # not absolute
        {"local_path_roots": [str(tmp_root / "nope")]},             # does not exist
        {"local_path_roots": ["\\\\?\\C:\\x"]},                      # \\?\ prefix
        {"local_path_roots": "C:/x"},                                # not a list
        {"allowed_hosts": ["http://box"]},                           # scheme not allowed
        {"allowed_hosts": ["a b"]},                                  # space
        {"rate_limit_per_min": 0},
        {"max_upload_mb": 0},
        {"default_token_ttl_days": 400},                             # > max_token_ttl_days (365)
        {"call_log_retention_days": -1},
        {"enabled": 1},                                              # bool, not int
    ]
    for body in bad:
        r = client.put("/api/settings", json={"mcp": body})
        assert r.status_code == 422 and r.json()["error"] == "invalid_settings", (body, r.text)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/unit/test_config.py tests/api/test_system_settings.py -k "mcp" -v`
Expected: FAIL — `AttributeError: 'AidocConfig' object has no attribute 'mcp'` / `unknown section mcp`.

- [ ] **Step 3: Implement**

`src/aidoc/config.py` — after `class Server`:

```python
@dataclass
class Mcp:
    """[mcp] (spec 2026-10-01 MCP §10)."""
    enabled: bool = True
    allowed_hosts: list[str] = field(default_factory=list)      # extra Host values; the bound interface is added
    local_path_roots: list[str] = field(default_factory=list)   # convert_path whitelist; empty = tool hidden
    default_token_ttl_days: int = 90
    max_token_ttl_days: int = 365
    allow_no_expiry: bool = False
    rate_limit_per_min: int = 60
    max_concurrent_jobs_per_token: int = 3
    max_upload_mb: int = 20
    response_token_budget: int = 8000
    call_log_retention_days: int = 30
    call_log_max_rows: int = 200000


_SECTIONS = {"general": General, "engines": Engines, "limits": Limits, "server": Server, "mcp": Mcp}
```

and on `AidocConfig`: `mcp: Mcp = field(default_factory=Mcp)` (before `root`). `_build` already rejects unknown keys; TOML arrays arrive as lists.

`aidoc.toml` — append:

```toml
[mcp]
enabled = true
allowed_hosts = []                 # extra Host names (e.g. the MagicDNS name); the bound interface is added automatically
local_path_roots = []              # convert_path whitelist; empty = the tool is hidden
default_token_ttl_days = 90
max_token_ttl_days = 365
allow_no_expiry = false
rate_limit_per_min = 60
max_concurrent_jobs_per_token = 3
max_upload_mb = 20
response_token_budget = 8000
call_log_retention_days = 30
call_log_max_rows = 200000
```

`src/aidoc/server/api/settings.py` — add rules. Near the top:

```python
import re
from pathlib import Path

_HOST_RE = re.compile(r"^[A-Za-z0-9.\-\[\]:*]+$")
_MIN.update({("mcp", "default_token_ttl_days"): 1, ("mcp", "max_token_ttl_days"): 1, ("mcp", "rate_limit_per_min"): 1,
             ("mcp", "max_concurrent_jobs_per_token"): 1, ("mcp", "max_upload_mb"): 1,
             ("mcp", "response_token_budget"): 500, ("mcp", "call_log_max_rows"): 1000,
             ("mcp", "call_log_retention_days"): 0})
_MAX.update({("mcp", "max_upload_mb"): 2048, ("mcp", "response_token_budget"): 25000,
             ("mcp", "max_token_ttl_days"): 3650, ("mcp", "default_token_ttl_days"): 3650})


def _validate_mcp_lists(key: str, val) -> None:
    if not isinstance(val, list) or not all(isinstance(v, str) for v in val):
        raise ApiError(422, "invalid_settings", detail=f"mcp.{key} must be a list of strings")
    for v in val:
        if key == "allowed_hosts":
            if not v or not _HOST_RE.match(v):
                raise ApiError(422, "invalid_settings", detail=f"mcp.allowed_hosts entry {v!r} is not a host[:port|:*]")
        else:                                   # local_path_roots
            s = v.replace("\\", "/")
            if s.startswith("//") or v.startswith("\\\\?\\"):
                raise ApiError(422, "invalid_settings", detail=f"mcp.local_path_roots entry {v!r} must be a local path")
            p = Path(v)
            if not p.is_absolute():
                raise ApiError(422, "invalid_settings", detail=f"mcp.local_path_roots entry {v!r} must be absolute")
            if not p.is_dir():
                raise ApiError(422, "invalid_settings", detail=f"mcp.local_path_roots entry {v!r} does not exist")
```

Inside `_validate`'s loop, right after the `token_readonly` block and before the `ok = type(val) is type(cur)` check, add:

```python
            if section == "mcp" and key in ("allowed_hosts", "local_path_roots"):
                _validate_mcp_lists(key, val)
                clean.setdefault(section, {})[key] = val
                continue
```

After the loop (before `return clean`) add the cross-field rule:

```python
    if "mcp" in clean:
        m = clean["mcp"]
        default_ttl = m.get("default_token_ttl_days", cfg.mcp.default_token_ttl_days)
        max_ttl = m.get("max_token_ttl_days", cfg.mcp.max_token_ttl_days)
        if default_ttl > max_ttl:
            raise ApiError(422, "invalid_settings", detail="mcp.default_token_ttl_days must be <= mcp.max_token_ttl_days")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/unit/test_config.py tests/api/test_system_settings.py -v`
Expected: all PASS (including the pre-existing settings tests).

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/config.py aidoc.toml src/aidoc/server/api/settings.py tests/unit/test_config.py tests/api/test_system_settings.py
git commit -m "feat(config): [mcp] section with settings validation

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 3: Token format, checksum, hash and server secret (`aidoc/mcp/tokens.py`)

**Files:**
- Create: `src/aidoc/mcp/__init__.py` (empty), `src/aidoc/mcp/tokens.py`, `tests/mcp/__init__.py` (empty)
- Test: `tests/mcp/test_tokens.py`

**Interfaces:**
- Produces:
  ```python
  PREFIX = "doc4ai_pat_"; BODY_LEN = 43; CRC_LEN = 6; TOKEN_LEN = 61
  ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
  def b62encode(n: int, width: int) -> str
  def crc_b62(body: str) -> str                        # zlib.crc32(body.encode()) → 6 base62 chars
  def generate_token() -> str                          # PREFIX + body(43) + "_" + crc(6)
  def is_well_formed(token: str) -> bool               # prefix, lengths, alphabet, CRC
  def display_prefix(token: str) -> str                # "doc4ai_pat_" + body[:4]
  def prefix_seen(raw: str | None) -> str | None       # first 15 chars when it starts with PREFIX, else None (for logs)
  def token_hash(secret: bytes, token: str) -> str     # HMAC-SHA256 hex
  def load_or_create_secret(data_dir: Path) -> bytes   # data/secret.key, 32 bytes, 0600 best effort
  ```

- [ ] **Step 1: Write the failing tests**

`tests/mcp/test_tokens.py`:

```python
import os
import stat
import sys

import pytest

from aidoc.mcp import tokens as T


def test_generate_token_shape_and_checksum():
    t = T.generate_token()
    assert t.startswith(T.PREFIX) and len(t) == T.TOKEN_LEN == 61
    body, crc = t[len(T.PREFIX):].split("_")
    assert len(body) == 43 and len(crc) == 6
    assert set(body) <= set(T.ALPHABET) and set(crc) <= set(T.ALPHABET)
    assert T.crc_b62(body) == crc
    assert T.is_well_formed(t)
    assert T.display_prefix(t) == T.PREFIX + body[:4]


def test_tokens_are_unique_and_high_entropy():
    seen = {T.generate_token() for _ in range(500)}
    assert len(seen) == 500


@pytest.mark.parametrize("mutate", [
    lambda t: t[:-1],                      # truncated
    lambda t: t[:-1] + ("A" if t[-1] != "A" else "B"),   # bad crc
    lambda t: t.replace(T.PREFIX, "doc4ai_pxt_"),
    lambda t: t[: len(T.PREFIX) + 10] + "!" + t[len(T.PREFIX) + 11:],   # illegal char
    lambda t: t.replace("_", "-", 2),
    lambda t: "",
    lambda t: "Bearer " + t,
])
def test_malformed_tokens_are_rejected(mutate):
    assert not T.is_well_formed(mutate(T.generate_token()))


def test_b62encode_width_and_roundtrip_range():
    assert T.b62encode(0, 6) == "000000"
    assert T.b62encode(61, 2) == "0z"
    assert T.b62encode(62, 2) == "10"
    assert len(T.b62encode(2 ** 256 - 1, 43)) == 43      # every 256-bit value fits in 43 chars


def test_prefix_seen_never_returns_the_secret_part():
    t = T.generate_token()
    assert T.prefix_seen(t) == t[:15] and len(T.prefix_seen(t)) == 15
    assert T.prefix_seen("garbage") is None and T.prefix_seen(None) is None


def test_hash_is_keyed_and_stable():
    t = T.generate_token()
    h1 = T.token_hash(b"k" * 32, t)
    assert h1 == T.token_hash(b"k" * 32, t) and len(h1) == 64
    assert h1 != T.token_hash(b"j" * 32, t)
    assert h1 != T.token_hash(b"k" * 32, t[:-1] + "0")


def test_secret_created_once_and_private(tmp_path):
    s1 = T.load_or_create_secret(tmp_path)
    s2 = T.load_or_create_secret(tmp_path)
    assert s1 == s2 and len(s1) == 32
    p = tmp_path / "secret.key"
    assert p.read_bytes() == s1
    if sys.platform != "win32":
        assert stat.S_IMODE(os.stat(p).st_mode) == 0o600
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/mcp/test_tokens.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'aidoc.mcp'`.

- [ ] **Step 3: Implement**

`src/aidoc/mcp/tokens.py`:

```python
"""Personal access token format (spec MCP §2.3): `doc4ai_pat_<43 base62>_<6 base62 CRC32>`.

The prefix lets secret scanners recognise a leaked token; the checksum lets the server drop malformed strings
before any database lookup. Only HMAC-SHA256(server_secret, token) is ever stored."""
from __future__ import annotations

import hmac
import hashlib
import os
import secrets
import zlib
from pathlib import Path

PREFIX = "doc4ai_pat_"
BODY_LEN = 43                       # 62**43 > 2**256: 32 random bytes always fit
CRC_LEN = 6                         # 62**6 > 2**32
TOKEN_LEN = len(PREFIX) + BODY_LEN + 1 + CRC_LEN      # 61
ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
_ALPHA = set(ALPHABET)
SECRET_FILE = "secret.key"


def b62encode(n: int, width: int) -> str:
    out = []
    while n:
        n, r = divmod(n, 62)
        out.append(ALPHABET[r])
    s = "".join(reversed(out)) or "0"
    return s.rjust(width, "0")


def crc_b62(body: str) -> str:
    return b62encode(zlib.crc32(body.encode("ascii")) & 0xFFFFFFFF, CRC_LEN)


def generate_token() -> str:
    body = b62encode(int.from_bytes(secrets.token_bytes(32), "big"), BODY_LEN)
    return f"{PREFIX}{body}_{crc_b62(body)}"


def is_well_formed(token: str) -> bool:
    if not isinstance(token, str) or len(token) != TOKEN_LEN or not token.startswith(PREFIX):
        return False
    rest = token[len(PREFIX):]
    if rest[BODY_LEN] != "_":
        return False
    body, crc = rest[:BODY_LEN], rest[BODY_LEN + 1:]
    if not (set(body) <= _ALPHA and set(crc) <= _ALPHA):
        return False
    return hmac.compare_digest(crc_b62(body), crc)


def display_prefix(token: str) -> str:
    return PREFIX + token[len(PREFIX):len(PREFIX) + 4]


def prefix_seen(raw: str | None) -> str | None:
    """What a failed attempt gets logged as: the prefix plus 4 chars, never more (spec §3 `token_prefix_seen`)."""
    if isinstance(raw, str) and raw.startswith(PREFIX):
        return raw[: len(PREFIX) + 4]
    return None


def token_hash(secret: bytes, token: str) -> str:
    return hmac.new(secret, token.encode("utf-8"), hashlib.sha256).hexdigest()


def load_or_create_secret(data_dir: Path) -> bytes:
    """`data/secret.key`: 32 random bytes created on first use; readable only by the owner where the OS allows."""
    p = Path(data_dir) / SECRET_FILE
    if p.is_file():
        data = p.read_bytes()
        if len(data) == 32:
            return data
    p.parent.mkdir(parents=True, exist_ok=True)
    data = secrets.token_bytes(32)
    tmp = p.with_name(f".{p.name}.tmp")
    fd = os.open(tmp, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
    try:
        os.write(fd, data)
    finally:
        os.close(fd)
    os.replace(tmp, p)
    try:
        os.chmod(p, 0o600)
    except OSError:
        pass
    return data
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/mcp/test_tokens.py -v`
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/mcp/__init__.py src/aidoc/mcp/tokens.py tests/mcp/__init__.py tests/mcp/test_tokens.py
git commit -m "feat(mcp): PAT format with CRC32, HMAC hashing and server secret

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 4: Schema v4 — `api_tokens`, `mcp_clients`, `mcp_calls`, `oauth_clients` + Store methods

**Files:**
- Modify: `src/aidoc/store.py`
- Test: `tests/mcp/test_store_v4.py`

**Interfaces:**
- Produces: `SCHEMA_VERSION = 4`; the Store methods listed in index §4 for `api_tokens`, `mcp_clients`, `mcp_calls`. Row dicts carry `scopes` (list) for tokens. `insert_mcp_call` accepts exactly the `mcp_calls` columns as keyword arguments (missing ones default to NULL; `ts` defaults to now; `status` is required).

- [ ] **Step 1: Write the failing tests**

`tests/mcp/test_store_v4.py`:

```python
import sqlite3
import time

import pytest

from aidoc.store import SCHEMA_VERSION, Store


@pytest.fixture
def store(tmp_path):
    s = Store(tmp_path / "aidoc.db")
    yield s
    s.close()


def test_schema_v4_tables_and_version(store):
    names = {r[0] for r in store.con.execute("SELECT name FROM sqlite_master WHERE type IN ('table','index')")}
    assert {"api_tokens", "mcp_clients", "mcp_calls", "oauth_clients", "mcp_calls_ts", "mcp_calls_token_ts"} <= names
    assert SCHEMA_VERSION == 4
    assert store._q1("SELECT value FROM meta WHERE key='schema_version'")[0] == "4"


def test_v2_database_upgrades_in_place(tmp_path):
    db = tmp_path / "old.db"
    con = sqlite3.connect(db)
    con.executescript("CREATE TABLE meta(key TEXT PRIMARY KEY, value TEXT); INSERT INTO meta VALUES('schema_version','2');"
                      "CREATE TABLE tasks(id TEXT PRIMARY KEY, job_id TEXT NOT NULL, source_path TEXT NOT NULL, work_path TEXT,"
                      " sha256 TEXT NOT NULL, size INTEGER NOT NULL, mtime REAL NOT NULL, lang TEXT NOT NULL, engine TEXT,"
                      " tried_json TEXT NOT NULL DEFAULT '[]', attempt INTEGER NOT NULL DEFAULT 0, status TEXT NOT NULL,"
                      " error_kind TEXT, error_msg TEXT, quality_json TEXT, output_dir TEXT NOT NULL, pid INTEGER,"
                      " created_at REAL NOT NULL, updated_at REAL NOT NULL, flags_json TEXT NOT NULL DEFAULT '{}');")
    con.commit()
    con.close()
    s = Store(db)
    assert s._q1("SELECT value FROM meta WHERE key='schema_version'")[0] == "4"
    assert s.list_api_tokens() == []
    s.close()


def test_api_token_crud(store):
    tid = store.create_api_token(name="Claude Code @ laptop", prefix="doc4ai_pat_3kX9", token_hash="h" * 64,
                                 scopes=["doc4ai:read", "doc4ai:convert"], expires_at=time.time() + 86400, note="n",
                                 rate_limit_per_min=None)
    row = store.get_api_token(tid)
    assert row["name"] == "Claude Code @ laptop" and row["scopes"] == ["doc4ai:read", "doc4ai:convert"]
    assert row["workspace"] == "default" and row["revoked_at"] is None and row["rotated_from"] is None
    assert store.get_api_token_by_hash("h" * 64)["id"] == tid
    assert store.get_api_token_by_hash("x" * 64) is None
    store.update_api_token(tid, name="renamed", last_used_at=5.0, last_used_ip="100.64.0.9", last_client="claude-code/2.3")
    row = store.get_api_token(tid)
    assert row["name"] == "renamed" and row["last_client"] == "claude-code/2.3"
    assert store.revoke_api_token(tid, "lost laptop", at=123.0) is True
    assert store.revoke_api_token(tid, "again", at=124.0) is False          # already revoked: unchanged
    row = store.get_api_token(tid)
    assert row["revoked_at"] == 123.0 and row["revoked_reason"] == "lost laptop"
    t2 = store.create_api_token(name="second", prefix="doc4ai_pat_AAAA", token_hash="g" * 64, scopes=["doc4ai:read"],
                                expires_at=None, rotated_from=tid)
    ids = [t["id"] for t in store.list_api_tokens()]
    assert ids == [t2, tid]                                                  # newest first


def test_token_hash_is_unique(store):
    store.create_api_token(name="a", prefix="p", token_hash="same", scopes=["doc4ai:read"], expires_at=None)
    with pytest.raises(sqlite3.IntegrityError):
        store.create_api_token(name="b", prefix="p", token_hash="same", scopes=["doc4ai:read"], expires_at=None)


def test_mcp_clients_upsert_and_active_filter(store):
    tid = store.create_api_token(name="a", prefix="p", token_hash="h1", scopes=["doc4ai:read"], expires_at=None)
    cid, created = store.upsert_mcp_client(token_id=tid, client_name="claude-code", client_version="2.3.1",
                                          protocol_version="2026-07-28", user_agent="ua", ip="127.0.0.1", now=1000.0)
    assert created is True
    cid2, created2 = store.upsert_mcp_client(token_id=tid, client_name="claude-code", client_version="2.3.1",
                                            protocol_version="2026-07-28", user_agent="ua2", ip="127.0.0.2", now=1500.0)
    assert cid2 == cid and created2 is False
    row = store.get_mcp_client(cid)
    assert row["request_count"] == 2 and row["first_seen"] == 1000.0 and row["last_seen"] == 1500.0
    assert row["last_ip"] == "127.0.0.2" and row["user_agent"] == "ua2"
    other, _ = store.upsert_mcp_client(token_id=tid, client_name="cursor", client_version=None, protocol_version="2025-11-25",
                                       user_agent=None, ip="127.0.0.1", now=100.0)
    assert {c["id"] for c in store.list_mcp_clients()} == {cid, other}
    assert [c["id"] for c in store.list_mcp_clients(active_since=1200.0)] == [cid]
    assert [c["id"] for c in store.list_mcp_clients(token_id=tid)] == [cid, other]      # last_seen DESC


def test_mcp_calls_insert_list_count_prune(store):
    tid = store.create_api_token(name="a", prefix="p", token_hash="h1", scopes=["doc4ai:read"], expires_at=None)
    ids = []
    for i in range(10):
        ids.append(store.insert_mcp_call(ts=1000.0 + i, token_id=tid if i % 2 else None, method="tools/call",
                                         tool_name="read_document" if i < 7 else "search_library",
                                         status="ok" if i % 3 else "tool_error", duration_ms=10 * i, http_status=200,
                                         args_summary='{"doc_id":"d"}', response_tokens_est=100 + i, ip="127.0.0.1",
                                         protocol_version="2026-07-28"))
    assert ids == sorted(ids)
    rows = store.list_mcp_calls(limit=3)
    assert [r["id"] for r in rows] == ids[-1:-4:-1]                           # newest first
    page2 = store.list_mcp_calls(limit=3, before_id=rows[-1]["id"])
    assert [r["id"] for r in page2] == ids[-4:-7:-1]
    assert len(store.list_mcp_calls(token_id=tid)) == 5
    assert len(store.list_mcp_calls(tool="search_library")) == 3
    assert len(store.list_mcp_calls(status="tool_error")) == 4
    assert len(store.list_mcp_calls(since=1005.0, until=1007.0)) == 3
    assert store.count_mcp_calls(since=0) == 10 and store.count_mcp_calls(since=0, errors_only=True) == 4
    assert store.count_mcp_calls(since=0, token_id=tid) == 5
    assert len(store.mcp_call_rows_since(1008.0)) == 2
    # retention: by age first, then by row cap (oldest go)
    assert store.prune_mcp_calls(older_than_ts=1003.0, max_rows=100) == 3
    assert store.prune_mcp_calls(older_than_ts=0, max_rows=5) == 2
    assert [r["id"] for r in store.list_mcp_calls(limit=100)] == ids[-1:-6:-1]


def test_insert_mcp_call_requires_status_and_defaults_ts(store):
    before = time.time()
    cid = store.insert_mcp_call(status="auth_error", token_prefix_seen="doc4ai_pat_3kX9", http_status=401, ip="::1")
    row = store.list_mcp_calls(limit=1)[0]
    assert row["id"] == cid and row["ts"] >= before and row["token_id"] is None and row["method"] is None
    with pytest.raises(TypeError):
        store.insert_mcp_call(http_status=200)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/mcp/test_store_v4.py -v`
Expected: FAIL — `AttributeError: 'Store' object has no attribute 'create_api_token'`, `assert SCHEMA_VERSION == 4`.

- [ ] **Step 3: Implement**

In `src/aidoc/store.py`:

1. `SCHEMA_VERSION = 4   # v3 was never issued: the MCP work (spec 2026-10-01) goes straight to the spec's "v4"`.
2. `_JSON_COLS = {..., "scopes_json"}`.
3. New DDL constant appended to `_DDL` (same `executescript`; `IF NOT EXISTS` makes it idempotent):

```sql
CREATE TABLE IF NOT EXISTS api_tokens(
  id TEXT PRIMARY KEY, workspace TEXT NOT NULL DEFAULT 'default', name TEXT NOT NULL, prefix TEXT NOT NULL,
  token_hash TEXT NOT NULL UNIQUE, scopes_json TEXT NOT NULL, note TEXT, created_at REAL NOT NULL, expires_at REAL,
  revoked_at REAL, revoked_reason TEXT, rotated_from TEXT, rate_limit_per_min INTEGER, last_used_at REAL,
  last_used_ip TEXT, last_client TEXT);
CREATE TABLE IF NOT EXISTS mcp_clients(
  id TEXT PRIMARY KEY, token_id TEXT, oauth_client_id TEXT, client_name TEXT NOT NULL, client_version TEXT,
  protocol_version TEXT, user_agent TEXT, first_seen REAL NOT NULL, last_seen REAL NOT NULL, last_ip TEXT,
  request_count INTEGER NOT NULL DEFAULT 0, UNIQUE(token_id, client_name, client_version));
CREATE TABLE IF NOT EXISTS mcp_calls(
  id INTEGER PRIMARY KEY AUTOINCREMENT, ts REAL NOT NULL, token_id TEXT, token_prefix_seen TEXT, client_id TEXT,
  method TEXT, tool_name TEXT, resource_uri TEXT, args_summary TEXT, status TEXT NOT NULL, error_code TEXT,
  http_status INTEGER, duration_ms INTEGER, response_bytes INTEGER, response_tokens_est INTEGER, ip TEXT,
  protocol_version TEXT, job_id TEXT);
CREATE INDEX IF NOT EXISTS mcp_calls_ts ON mcp_calls(ts);
CREATE INDEX IF NOT EXISTS mcp_calls_token_ts ON mcp_calls(token_id, ts);
CREATE TABLE IF NOT EXISTS oauth_clients(
  client_id TEXT PRIMARY KEY, metadata_url TEXT, name TEXT, redirect_uris_json TEXT, created_at REAL NOT NULL, last_seen REAL);
```

Note: `UNIQUE(token_id, client_name, client_version)` with `client_version NULL` would allow duplicate rows in SQLite (NULLs are distinct). `upsert_mcp_client` therefore stores `client_version = ""` for None and the row dict converts `""` back to `None` is **not** done — the API shows `""` as unknown; tests above pass `None` and read it back only through `list`/`get` without asserting the version. Keep it simple: store `""`.

4. Methods (append in a `# ---- MCP (schema v4)` section):

```python
    # ---- api_tokens (MCP spec §2.3/§3)
    def create_api_token(self, *, name, prefix, token_hash, scopes: list[str], expires_at, note=None,
                         rate_limit_per_min=None, rotated_from=None, workspace="default") -> str:
        tid = _new_id()
        self.con.execute(
            "INSERT INTO api_tokens(id, workspace, name, prefix, token_hash, scopes_json, note, created_at, expires_at, "
            "rate_limit_per_min, rotated_from) VALUES(?,?,?,?,?,?,?,?,?,?,?)",
            (tid, workspace, name, prefix, token_hash, json.dumps(list(scopes)), note, time.time(), expires_at,
             rate_limit_per_min, rotated_from))
        return tid

    def get_api_token(self, token_id) -> dict | None:
        return _row(self._q1("SELECT * FROM api_tokens WHERE id=?", (token_id,)))

    def get_api_token_by_hash(self, token_hash) -> dict | None:
        return _row(self._q1("SELECT * FROM api_tokens WHERE token_hash=?", (token_hash,)))

    def list_api_tokens(self) -> list[dict]:
        return self._qa("SELECT * FROM api_tokens ORDER BY created_at DESC, rowid DESC")

    def update_api_token(self, token_id, **fields) -> None:
        self._update("api_tokens", "id", token_id, fields)

    def revoke_api_token(self, token_id, reason: str | None, at: float) -> bool:
        """False when the token is already revoked (nothing changes)."""
        cur = self.con.execute("UPDATE api_tokens SET revoked_at=?, revoked_reason=? WHERE id=? AND revoked_at IS NULL",
                               (at, reason, token_id))
        return cur.rowcount == 1

    # ---- mcp_clients (observed clients; display only)
    def upsert_mcp_client(self, *, token_id, client_name, client_version, protocol_version, user_agent, ip,
                          now) -> tuple[str, bool]:
        version = client_version or ""
        with self.con.lock:
            old = self._q1("SELECT id FROM mcp_clients WHERE token_id IS ? AND client_name=? AND client_version=?",
                           (token_id, client_name, version))
            if old is not None:
                self.con.execute("UPDATE mcp_clients SET protocol_version=?, user_agent=?, last_seen=?, last_ip=?, "
                                 "request_count=request_count+1 WHERE id=?",
                                 (protocol_version, user_agent, now, ip, old["id"]))
                return old["id"], False
            cid = _new_id()
            self.con.execute("INSERT INTO mcp_clients(id, token_id, client_name, client_version, protocol_version, "
                             "user_agent, first_seen, last_seen, last_ip, request_count) VALUES(?,?,?,?,?,?,?,?,?,1)",
                             (cid, token_id, client_name, version, protocol_version, user_agent, now, now, ip))
            return cid, True

    def get_mcp_client(self, client_id) -> dict | None:
        return _row(self._q1("SELECT * FROM mcp_clients WHERE id=?", (client_id,)))

    def list_mcp_clients(self, token_id=None, active_since: float | None = None) -> list[dict]:
        sql, args = "SELECT * FROM mcp_clients WHERE 1=1", []
        if token_id is not None:
            sql += " AND token_id=?"
            args.append(token_id)
        if active_since is not None:
            sql += " AND last_seen >= ?"
            args.append(active_since)
        return self._qa(sql + " ORDER BY last_seen DESC, rowid DESC", args)

    # ---- mcp_calls (one row per MCP HTTP request)
    _MCP_CALL_COLS = ("ts", "token_id", "token_prefix_seen", "client_id", "method", "tool_name", "resource_uri",
                      "args_summary", "status", "error_code", "http_status", "duration_ms", "response_bytes",
                      "response_tokens_est", "ip", "protocol_version", "job_id")

    def insert_mcp_call(self, *, status: str, **fields) -> int:
        unknown = set(fields) - set(self._MCP_CALL_COLS)
        if unknown:
            raise TypeError(f"unknown mcp_calls columns: {sorted(unknown)}")
        fields["status"] = status
        fields.setdefault("ts", time.time())
        cols = list(fields)
        cur = self.con.execute(f"INSERT INTO mcp_calls({', '.join(cols)}) VALUES({', '.join('?' * len(cols))})",
                               tuple(_val(fields[c]) for c in cols))
        return cur.lastrowid

    def list_mcp_calls(self, *, token_id=None, client_id=None, tool=None, status=None, since=None, until=None,
                       before_id=None, limit=100) -> list[dict]:
        sql, args = "SELECT * FROM mcp_calls WHERE 1=1", []
        for col, val in (("token_id", token_id), ("client_id", client_id), ("tool_name", tool), ("status", status)):
            if val is not None:
                sql += f" AND {col}=?"
                args.append(val)
        if since is not None:
            sql += " AND ts >= ?"
            args.append(since)
        if until is not None:
            sql += " AND ts <= ?"
            args.append(until)
        if before_id is not None:
            sql += " AND id < ?"
            args.append(before_id)
        return self._qa(sql + " ORDER BY id DESC LIMIT ?", [*args, max(1, min(int(limit), 1000))])

    def count_mcp_calls(self, since: float, token_id=None, errors_only=False) -> int:
        sql, args = "SELECT COUNT(*) FROM mcp_calls WHERE ts >= ?", [since]
        if token_id is not None:
            sql += " AND token_id=?"
            args.append(token_id)
        if errors_only:
            sql += " AND status != 'ok'"
        return int(self._q1(sql, args)[0])

    def mcp_call_rows_since(self, since: float) -> list[dict]:
        return self._qa("SELECT id, ts, token_id, client_id, tool_name, method, status, duration_ms, response_tokens_est "
                        "FROM mcp_calls WHERE ts >= ? ORDER BY ts", (since,))

    def prune_mcp_calls(self, *, older_than_ts: float, max_rows: int) -> int:
        n = self.con.execute("DELETE FROM mcp_calls WHERE ts < ?", (older_than_ts,)).rowcount
        r = self._q1("SELECT COUNT(*), MAX(id) FROM mcp_calls")
        total, max_id = (int(r[0]), r[1]) if r else (0, None)
        if max_id is not None and total > max_rows:
            cut = self._q1("SELECT id FROM mcp_calls ORDER BY id DESC LIMIT 1 OFFSET ?", (max_rows - 1,))
            if cut is not None:
                n += self.con.execute("DELETE FROM mcp_calls WHERE id < ?", (cut[0],)).rowcount
        return n
```

`_fields_to_cols` already maps `scopes=` to `scopes_json`; `insert_mcp_call` bypasses it (no JSON columns in `mcp_calls`).

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/mcp/test_store_v4.py tests/unit/test_store.py -v`
Expected: all PASS (the existing store tests must still pass; `test_schema_version`-style assertions in `tests/unit/test_store.py` that hard-code `"2"` must be updated to `"4"` — check with `grep -n "schema_version" tests/`).

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/store.py tests/mcp/test_store_v4.py tests/unit/test_store.py
git commit -m "feat(store): schema v4 with api_tokens, mcp_clients, mcp_calls (+retention) and oauth_clients

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 5: `page_index` FTS5 table + `aidoc/pageindex.py` (split, index, query, snippet)

**Files:**
- Modify: `src/aidoc/store.py` (FTS5 creation with tokenizer fallback; `replace_page_index`, `delete_page_index`, `search_page_index`, `like_page_index`, `page_index_doc_ids`, `page_index_tokenizer`; `delete_documents` cleanup)
- Create: `src/aidoc/pageindex.py`
- Test: `tests/mcp/test_pageindex.py`

**Interfaces:**
- Produces (`aidoc/pageindex.py`):
  ```python
  MIN_TRIGRAM_CHARS = 3
  def pages_for_index(markdown: str) -> list[tuple[int | None, str]]   # [(page, text)]; no markers → [(None, whole)]; preamble before page 1 joins page 1
  def index_document(store, doc: dict) -> int                           # rows written (0 when the .md is missing); deletes old rows first
  def fts_query(q: str) -> str | None       # '"term1" "term2"' with quotes doubled; None when no term has ≥ 3 chars
  def search_pages(store, query: str, *, doc_ids: list[str] | None = None, limit: int = 50) -> list[dict]  # [{doc_id, page, text, rank}] best first
  def make_snippet(text: str, query: str, width: int = 300) -> str      # window around the first term hit; "…" at cut ends
  def backfill_page_index(store, log=print) -> int                      # docs indexed
  ```
- Store: `search_page_index(match, *, doc_ids, limit)` runs `SELECT doc_id, page, text, bm25(page_index) AS rank FROM page_index WHERE page_index MATCH ? [AND doc_id IN (...)] ORDER BY rank LIMIT ?`; `like_page_index(needle, *, doc_ids, limit, scan_cap=2000)` runs `SELECT doc_id, page, text, 0.0 AS rank FROM page_index WHERE instr(lower(text), lower(?)) > 0 [AND doc_id IN (...)] LIMIT ?` (scan cap through `LIMIT`).

- [ ] **Step 1: Write the failing tests**

`tests/mcp/test_pageindex.py`:

```python
import sqlite3

import pytest

from aidoc import pageindex as PI
from aidoc.store import Store

MD = ("Preamble line\n<!-- page: 1 -->\n# 第一章 機器學習\n\n機器學習 machine learning 概論。\n"
      "<!-- page: 2 -->\n## 2.1 Gradient descent\n\n梯度下降 gradient descent 的步驟如下。\n"
      "<!-- page: 3 -->\n\n參考文獻 references\n")


@pytest.fixture
def store(tmp_path):
    s = Store(tmp_path / "aidoc.db")
    yield s
    s.close()


def _doc(store, tmp_path, name, md, pages=3):
    out = tmp_path / "out" / name
    out.mkdir(parents=True)
    (out / f"{name}.md").write_text(md, encoding="utf-8")
    (out / f"{name}.json").write_text("{}", encoding="utf-8")
    did = store.upsert_document(sha256="s" + name, source_path=f"{name}.pdf", output_dir=str(out), engine="mineru",
                                quality={"score": 1, "level": "ok", "reasons": []}, pages=pages, lang="cht",
                                aidoc_version="0.1.0", status="ok")
    return store.get_document(did)


def test_pages_for_index_splits_on_markers_and_joins_preamble():
    pages = PI.pages_for_index(MD)
    assert [p for p, _ in pages] == [1, 2, 3]
    assert pages[0][1].startswith("Preamble line") and "機器學習" in pages[0][1]
    assert "gradient descent" in pages[1][1] and "references" in pages[2][1]
    assert PI.pages_for_index("no markers at all") == [(None, "no markers at all")]
    assert PI.pages_for_index("") == []


def test_tokenizer_is_trigram_when_supported(store):
    tok = store.page_index_tokenizer()
    assert tok in ("trigram", "unicode61")
    if sqlite3.sqlite_version_info >= (3, 34, 0):
        assert tok == "trigram"


def test_index_search_cjk_and_english(store, tmp_path):
    doc = _doc(store, tmp_path, "ml", MD)
    assert PI.index_document(store, doc) == 3
    assert store.page_index_doc_ids() == {doc["id"]}
    hits = PI.search_pages(store, "機器學習")
    assert hits and hits[0]["doc_id"] == doc["id"] and hits[0]["page"] == 1
    hits = PI.search_pages(store, "gradient")
    assert [h["page"] for h in hits] == [2]
    if store.page_index_tokenizer() == "trigram":
        assert [h["page"] for h in PI.search_pages(store, "escent")] == [2]        # substring match
    assert PI.search_pages(store, "nothing-here-xyz") == []


def test_reindex_replaces_rows(store, tmp_path):
    doc = _doc(store, tmp_path, "ml", MD)
    PI.index_document(store, doc)
    out = tmp_path / "out" / "ml"
    (out / "ml.md").write_text("<!-- page: 1 -->\nonly one page now\n", encoding="utf-8")
    assert PI.index_document(store, store.get_document(doc["id"])) == 1
    assert store._q1("SELECT COUNT(*) FROM page_index WHERE doc_id=?", (doc["id"],))[0] == 1
    assert PI.search_pages(store, "gradient") == []


def test_missing_markdown_indexes_nothing_and_clears(store, tmp_path):
    doc = _doc(store, tmp_path, "ml", MD)
    PI.index_document(store, doc)
    (tmp_path / "out" / "ml" / "ml.md").unlink()
    assert PI.index_document(store, store.get_document(doc["id"])) == 0
    assert store.page_index_doc_ids() == set()


@pytest.mark.parametrize("q", ['"', "*", "(", "-x", "NEAR", "AND", "a", "學習", "學", '"機器" OR', "機器 OR 學習", "   "])
def test_short_and_syntax_queries_never_error(store, tmp_path, q):
    doc = _doc(store, tmp_path, "ml", MD)
    PI.index_document(store, doc)
    hits = PI.search_pages(store, q)          # must not raise
    assert isinstance(hits, list)
    if q == "學習":                            # 2 chars: LIKE fallback still finds page 1
        assert hits and hits[0]["page"] == 1


def test_fts_query_quotes_terms():
    assert PI.fts_query('機器學習 "gradient" x') == '"機器學習" """gradient"""'
    assert PI.fts_query("ab") is None and PI.fts_query("") is None
    assert PI.fts_query("ab cde") == '"cde"'


def test_doc_ids_filter(store, tmp_path):
    a = _doc(store, tmp_path, "a", MD)
    b = _doc(store, tmp_path, "b", MD)
    PI.index_document(store, a)
    PI.index_document(store, b)
    hits = PI.search_pages(store, "gradient", doc_ids=[b["id"]])
    assert {h["doc_id"] for h in hits} == {b["id"]}


def test_snippet_window_and_ellipses():
    text = "x" * 400 + " gradient descent here " + "y" * 400
    s = PI.make_snippet(text, "gradient", width=100)
    assert "gradient" in s and len(s) <= 104 and s.startswith("…") and s.endswith("…")
    assert PI.make_snippet("short text", "zzz", width=50) == "short text"
    assert PI.make_snippet("機器學習 概論", "學習", width=50) == "機器學習 概論"


def test_backfill_indexes_only_unindexed(store, tmp_path, capsys):
    a = _doc(store, tmp_path, "a", MD)
    b = _doc(store, tmp_path, "b", MD)
    PI.index_document(store, a)
    assert PI.backfill_page_index(store, log=lambda s: None) == 1
    assert store.page_index_doc_ids() == {a["id"], b["id"]}
    assert PI.backfill_page_index(store, log=lambda s: None) == 0


def test_delete_documents_drops_index_rows(store, tmp_path):
    a = _doc(store, tmp_path, "a", MD)
    PI.index_document(store, a)
    store.set_document_status(a["id"], "orphaned")
    assert store.delete_documents("orphaned") == 1
    assert store.page_index_doc_ids() == set()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/mcp/test_pageindex.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'aidoc.pageindex'`.

- [ ] **Step 3: Implement**

`src/aidoc/store.py` — in `__init__` after the v4 DDL:

```python
        self._page_index_tokenizer = self._ensure_page_index()
```

and methods:

```python
    # ---- page_index (FTS5; MCP spec §3)
    def _ensure_page_index(self) -> str:
        exists = self._q1("SELECT sql FROM sqlite_master WHERE type='table' AND name='page_index'")
        if exists is not None:
            return "trigram" if "trigram" in (exists[0] or "") else "unicode61"
        for tok in ("trigram", "unicode61"):          # D6: trigram needs SQLite >= 3.34
            try:
                self.con.execute("CREATE VIRTUAL TABLE page_index USING fts5(doc_id UNINDEXED, page UNINDEXED, text, "
                                 f"tokenize = '{tok}')")
            except sqlite3.OperationalError:
                continue
            self.con.execute("INSERT INTO meta(key, value) VALUES('page_index_tokenizer', ?) "
                             "ON CONFLICT(key) DO UPDATE SET value=excluded.value", (tok,))
            return tok
        raise RuntimeError("SQLite build has no FTS5")

    def page_index_tokenizer(self) -> str:
        return self._page_index_tokenizer

    def replace_page_index(self, doc_id, pages: list[tuple[int | None, str]]) -> None:
        with self.con.lock:
            self.con.execute("DELETE FROM page_index WHERE doc_id=?", (doc_id,))
            for page, text in pages:
                self.con.execute("INSERT INTO page_index(doc_id, page, text) VALUES(?,?,?)", (doc_id, page, text))

    def delete_page_index(self, doc_id) -> None:
        self.con.execute("DELETE FROM page_index WHERE doc_id=?", (doc_id,))

    def page_index_doc_ids(self) -> set[str]:
        return {r[0] for r in self.con.execute("SELECT DISTINCT doc_id FROM page_index")}

    def _doc_filter(self, doc_ids) -> tuple[str, list]:
        if not doc_ids:
            return "", []
        return f" AND doc_id IN ({', '.join('?' * len(doc_ids))})", list(doc_ids)

    def search_page_index(self, match: str, *, doc_ids=None, limit: int) -> list[dict]:
        f, args = self._doc_filter(doc_ids)
        return self._qa("SELECT doc_id, page, text, bm25(page_index) AS rank FROM page_index WHERE page_index MATCH ?"
                        f"{f} ORDER BY rank LIMIT ?", [match, *args, int(limit)])

    def like_page_index(self, needle: str, *, doc_ids=None, limit: int, scan_cap: int = 2000) -> list[dict]:
        f, args = self._doc_filter(doc_ids)
        rows = self._qa("SELECT doc_id, page, text, 0.0 AS rank FROM page_index WHERE "
                        f"instr(lower(text), lower(?)) > 0{f} LIMIT ?", [needle, *args, int(min(limit, scan_cap))])
        return rows
```

Change `delete_documents`:

```python
    def delete_documents(self, status) -> int:
        with self.con.lock:
            self.con.execute("DELETE FROM page_index WHERE doc_id IN (SELECT id FROM documents WHERE status=?)",
                             (_val(status),))
            return self.con.execute("DELETE FROM documents WHERE status=?", (_val(status),)).rowcount
```

`src/aidoc/pageindex.py`:

```python
"""Per-page full-text index for `search_library` (MCP spec §3 `page_index`). Pages come from `<!-- page: N -->`
markers (aidoc.pagemap); documents without markers are one row with page NULL."""
from __future__ import annotations

import re
import threading
from collections.abc import Callable
from pathlib import Path

from aidoc.pagemap import split_pages

MIN_TRIGRAM_CHARS = 3
_WS = re.compile(r"\s+")


def pages_for_index(markdown: str) -> list[tuple[int | None, str]]:
    if not markdown.strip():
        return []
    pre, secs = split_pages(markdown)
    if not secs:
        return [(None, markdown)]
    out = []
    for page in sorted(secs):
        text = secs[page]
        if page == min(secs) and pre.strip():
            text = pre + text
        out.append((page, text))
    return out


def _markdown_path(doc: dict) -> Path:
    out = Path(doc["output_dir"])
    return out / f"{out.name}.md"


def index_document(store, doc: dict) -> int:
    """Rewrite the index rows of one document from its Markdown; 0 rows when the output is gone."""
    md = _markdown_path(doc)
    try:
        text = md.read_text(encoding="utf-8")
    except OSError:
        store.delete_page_index(doc["id"])
        return 0
    pages = pages_for_index(text)
    store.replace_page_index(doc["id"], pages)
    return len(pages)


def _terms(q: str) -> list[str]:
    return [t for t in _WS.split(q.strip()) if t]


def fts_query(q: str) -> str | None:
    """Every whitespace-separated term as a quoted FTS5 string (user text is never FTS syntax); terms shorter than
    three characters are dropped because the trigram tokenizer cannot match them (D6)."""
    terms = [t for t in _terms(q) if len(t) >= MIN_TRIGRAM_CHARS]
    if not terms:
        return None
    return " ".join('"' + t.replace('"', '""') + '"' for t in terms)


def search_pages(store, query: str, *, doc_ids: list[str] | None = None, limit: int = 50) -> list[dict]:
    terms = _terms(query)
    if not terms:
        return []
    match = fts_query(query)
    if match is not None:
        return store.search_page_index(match, doc_ids=doc_ids, limit=limit)
    # D6 fallback: only short terms (e.g. 兩個字) — substring scan, longest term first
    needle = max(terms, key=len)
    return store.like_page_index(needle, doc_ids=doc_ids, limit=limit)


def make_snippet(text: str, query: str, width: int = 300) -> str:
    flat = _WS.sub(" ", text).strip()
    if len(flat) <= width:
        return flat
    low = flat.lower()
    pos = -1
    for t in sorted(_terms(query), key=len, reverse=True):
        pos = low.find(t.lower())
        if pos >= 0:
            break
    if pos < 0:
        return flat[:width].rstrip() + "…"
    start = max(0, pos - width // 2)
    end = min(len(flat), start + width)
    start = max(0, end - width)
    out = flat[start:end].strip()
    return ("…" if start > 0 else "") + out + ("…" if end < len(flat) else "")


def backfill_page_index(store, log: Callable[[str], None] = print) -> int:
    done = store.page_index_doc_ids()
    n = 0
    for doc in store.list_documents():
        if doc["id"] in done or doc["status"] not in ("ok", "warn", "low"):
            continue
        rows = index_document(store, doc)
        if rows:
            n += 1
            log(f"indexed {Path(doc['output_dir']).name}: {rows} pages")
    return n


def start_page_index_backfill(store, log: Callable[[str], None] | None = None) -> threading.Thread:
    """Background backfill at server start (spec §3): never blocks startup; errors are logged, not raised."""
    import sys

    def emit(line: str) -> None:
        print(f"page_index: {line}", file=sys.stderr, flush=True)

    def run() -> None:
        try:
            backfill_page_index(store, log=log or emit)
        except Exception as e:  # noqa: BLE001  a background helper must never take the server down
            (log or emit)(f"failed: {type(e).__name__}: {e}")
    th = threading.Thread(target=run, name="aidoc-page-index", daemon=True)
    th.start()
    return th
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/mcp/test_pageindex.py tests/mcp/test_store_v4.py -v`
Expected: all PASS. (If `test_tokenizer_is_trigram_when_supported` fails with `unicode61` on SQLite ≥ 3.34, the FTS5 build lacks trigram: record it in the spike S14 entry and mark the test `skipif` with the reason.)

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/store.py src/aidoc/pageindex.py tests/mcp/test_pageindex.py
git commit -m "feat(search): FTS5 page_index (trigram with unicode61 fallback), per-page indexing and query helpers

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 6: Index on finalize + startup backfill

**Files:**
- Modify: `src/aidoc/pipeline.py:392-396` (`_write_output`: index after `upsert_document`)
- Modify: `src/aidoc/server/app.py` (`build_context(start_workers=True)` starts the backfill), `src/aidoc/cli.py` (`cmd_serve` starts it after recovery, next to `start_reassessment`)
- Test: `tests/mcp/test_pageindex_hooks.py`

**Interfaces:**
- Consumes: `aidoc.pageindex.index_document`, `start_page_index_backfill` (Task 5).
- Produces: every conversion (CLI or server, first run or reconvert) leaves `page_index` rows for the document; a server start indexes documents that have none.

- [ ] **Step 1: Write the failing tests**

`tests/mcp/test_pageindex_hooks.py`:

```python
import time

from aidoc.batch import register_source
from aidoc.config import load_config
from aidoc.engines.registry import get_engines
from aidoc.models import ConvertOptions
from aidoc.pipeline import run_task
from aidoc.store import Store
from tests.fakes.scenario import fake_env, write_scenario


def test_conversion_writes_page_index(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json", pages_per_doc=3))
    cfg = load_config()
    store = Store(tmp_root / "data" / "aidoc.db")
    opts = ConvertOptions(output_dir=tmp_root / "out")
    job = store.create_job(opts, "cli")
    tid, _ = register_source(store, job, fixtures / "text.pdf", opts)
    assert run_task(store, tid, get_engines(cfg), cfg).value in ("done", "low")
    doc = next(iter(store.list_documents()))
    rows = store._qa("SELECT page FROM page_index WHERE doc_id=? ORDER BY page", (doc["id"],))
    assert [r["page"] for r in rows] == [1, 2, 3]
    store.close()


def test_server_start_backfills_missing_index(tmp_root, monkeypatch):
    from aidoc.server.app import build_context
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json"))
    store = Store(tmp_root / "data" / "aidoc.db")
    out = tmp_root / "out" / "old"
    out.mkdir(parents=True)
    (out / "old.md").write_text("<!-- page: 1 -->\nold text\n<!-- page: 2 -->\nmore\n", encoding="utf-8")
    (out / "old.json").write_text("{}", encoding="utf-8")
    store.upsert_document(sha256="s", source_path="old.pdf", output_dir=str(out), engine="mineru",
                          quality={"score": 1, "level": "ok", "reasons": []}, pages=2, lang="cht", aidoc_version="0.1.0",
                          status="ok")
    store.close()
    ctx = build_context(load_config(), token=None, start_workers=True)
    try:
        deadline = time.time() + 5
        while time.time() < deadline and not ctx.store.page_index_doc_ids():
            time.sleep(0.05)
        assert len(ctx.store.page_index_doc_ids()) == 1
    finally:
        ctx.close()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/mcp/test_pageindex_hooks.py -v`
Expected: FAIL — `assert [] == [1, 2, 3]` and `assert 0 == 1`.

- [ ] **Step 3: Implement**

`src/aidoc/pipeline.py` — directly after the `store.upsert_document(...)` call in `_write_output` (line ~392):

```python
    from aidoc.pageindex import index_document
    doc_row = store.get_document_by_output(str(output_dir))
    if doc_row is not None:
        try:
            index_document(store, doc_row)
        except Exception as e:  # noqa: BLE001  the index is a convenience; never fail a finished conversion over it
            ctx.log(f"page index skipped: {type(e).__name__}: {e}") if hasattr(ctx, "log") else None
```

(Check `PipelineContext` for its log callable name — `grep -n "def log\|emit" src/aidoc/pipeline.py`; use the existing one, else drop the message.)

`src/aidoc/server/app.py` — in `build_context`, inside `if start_workers:` after `start_reassessment(ctx)`:

```python
        from aidoc.pageindex import start_page_index_backfill
        start_page_index_backfill(ctx.store)
```

`src/aidoc/cli.py` — in `cmd_serve`, right after `start_reassessment(ctx)`:

```python
        from aidoc.pageindex import start_page_index_backfill
        start_page_index_backfill(ctx.store)                 # MCP search index for documents converted before v4
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/mcp/test_pageindex_hooks.py tests/unit/test_pipeline.py tests/api/test_serve.py -v`
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/pipeline.py src/aidoc/server/app.py src/aidoc/cli.py tests/mcp/test_pageindex_hooks.py
git commit -m "feat(search): index pages at finalize and backfill at server start

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 7: `Principal`, `AuthFailure`, `PatVerifier`, `current_principal`

**Files:**
- Create: `src/aidoc/mcp/principal.py`
- Test: `tests/mcp/test_principal.py`

**Interfaces:**
- Produces:
  ```python
  SCOPE_READ = "doc4ai:read"; SCOPE_CONVERT = "doc4ai:convert"; SCOPE_CONVERT_LOCAL = "doc4ai:convert:local"; SCOPE_MANAGE = "doc4ai:manage"
  SCOPES = (SCOPE_READ, SCOPE_CONVERT, SCOPE_CONVERT_LOCAL, SCOPE_MANAGE)
  @dataclass(frozen=True) class Principal: kind: Literal["pat","oauth","admin"]; subject: str; token_id: str | None; client_id: str | None; scopes: frozenset[str]; expires_at: float | None; name: str = ""; rate_limit_per_min: int | None = None
      def has(self, scope: str) -> bool
  @dataclass(frozen=True) class AuthFailure: reason: Literal["missing_token","bad_format","bad_checksum","unknown_token","revoked","expired"]; prefix_seen: str | None
  current_principal: ContextVar[Principal | None]
  class PatVerifier:
      def __init__(self, store, secret: bytes)
      def verify(self, raw: str | None, now: float | None = None) -> Principal | AuthFailure
  def parse_bearer(header_value: str | None) -> str | None    # case-insensitive scheme, stripped, quotes removed
  ```

- [ ] **Step 1: Write the failing tests**

`tests/mcp/test_principal.py`:

```python
import time

import pytest

from aidoc.mcp import tokens as T
from aidoc.mcp.principal import SCOPES, AuthFailure, PatVerifier, Principal, current_principal, parse_bearer
from aidoc.store import Store


@pytest.fixture
def store(tmp_path):
    s = Store(tmp_path / "aidoc.db")
    yield s
    s.close()


SECRET = b"s" * 32


def _issue(store, scopes=("doc4ai:read",), expires_in=3600, rate=None):
    raw = T.generate_token()
    tid = store.create_api_token(name="t", prefix=T.display_prefix(raw), token_hash=T.token_hash(SECRET, raw),
                                 scopes=list(scopes), expires_at=None if expires_in is None else time.time() + expires_in,
                                 rate_limit_per_min=rate)
    return raw, tid


def test_scopes_constant():
    assert SCOPES == ("doc4ai:read", "doc4ai:convert", "doc4ai:convert:local", "doc4ai:manage")


def test_verify_good_token(store):
    raw, tid = _issue(store, scopes=("doc4ai:read", "doc4ai:manage"), rate=5)
    p = PatVerifier(store, SECRET).verify(raw)
    assert isinstance(p, Principal) and p.kind == "pat" and p.subject == "owner" and p.token_id == tid
    assert p.scopes == frozenset({"doc4ai:read", "doc4ai:manage"}) and p.has("doc4ai:manage") and not p.has("doc4ai:convert")
    assert p.rate_limit_per_min == 5 and p.name == "t" and p.expires_at is not None


@pytest.mark.parametrize("raw,reason", [
    (None, "missing_token"), ("", "missing_token"), ("abc", "bad_format"), ("doc4ai_pat_" + "A" * 43 + "_000000", "bad_checksum"),
])
def test_verify_format_failures(store, raw, reason):
    f = PatVerifier(store, SECRET).verify(raw)
    assert isinstance(f, AuthFailure) and f.reason == reason
    assert f.prefix_seen == (raw[:15] if raw and raw.startswith("doc4ai_pat_") else None)


def test_unknown_revoked_expired(store):
    v = PatVerifier(store, SECRET)
    unknown = T.generate_token()
    f = v.verify(unknown)
    assert f.reason == "unknown_token" and f.prefix_seen == unknown[:15]
    raw, tid = _issue(store)
    store.revoke_api_token(tid, "bye", at=time.time())
    assert v.verify(raw).reason == "revoked"
    raw2, _ = _issue(store, expires_in=-1)
    assert v.verify(raw2).reason == "expired"
    raw3, _ = _issue(store, expires_in=None)                      # never expires
    assert isinstance(v.verify(raw3), Principal)


def test_future_revocation_is_a_grace_window(store):
    raw, tid = _issue(store)
    store.revoke_api_token(tid, "rotated", at=time.time() + 60)   # Phase 1.5 grace hook
    v = PatVerifier(store, SECRET)
    assert isinstance(v.verify(raw), Principal)
    assert v.verify(raw, now=time.time() + 61).reason == "revoked"


def test_wrong_secret_never_matches(store):
    raw, _ = _issue(store)
    assert PatVerifier(store, b"x" * 32).verify(raw).reason == "unknown_token"


@pytest.mark.parametrize("header,expect", [
    ("Bearer abc", "abc"), ("bearer abc", "abc"), ("BEARER  abc  ", "abc"), ('Bearer "abc"', "abc"), ("Bearer abc\n", "abc"),
    ("Basic abc", None), ("abc", None), (None, None), ("Bearer", None), ("Bearer ", None),
])
def test_parse_bearer(header, expect):
    assert parse_bearer(header) == expect


def test_contextvar_default_is_none():
    assert current_principal.get() is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/mcp/test_principal.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'aidoc.mcp.principal'`.

- [ ] **Step 3: Implement**

`src/aidoc/mcp/principal.py`:

```python
"""Who is calling (MCP spec §2.1). Every verifier produces a `Principal`; tools, scope checks and logs see only it."""
from __future__ import annotations

import time
from contextvars import ContextVar
from dataclasses import dataclass
from typing import Literal

from aidoc.mcp import tokens as T

SCOPE_READ = "doc4ai:read"
SCOPE_CONVERT = "doc4ai:convert"
SCOPE_CONVERT_LOCAL = "doc4ai:convert:local"
SCOPE_MANAGE = "doc4ai:manage"
SCOPES = (SCOPE_READ, SCOPE_CONVERT, SCOPE_CONVERT_LOCAL, SCOPE_MANAGE)

FailureReason = Literal["missing_token", "bad_format", "bad_checksum", "unknown_token", "revoked", "expired"]


@dataclass(frozen=True)
class Principal:
    kind: Literal["pat", "oauth", "admin"]
    subject: str                    # pat: "owner" (Phase 1); oauth: sub
    token_id: str | None
    client_id: str | None           # oauth client_id (Phase 2)
    scopes: frozenset[str]
    expires_at: float | None
    name: str = ""                  # api_tokens.name, for logs and events
    rate_limit_per_min: int | None = None

    def has(self, scope: str) -> bool:
        return scope in self.scopes


@dataclass(frozen=True)
class AuthFailure:
    reason: FailureReason
    prefix_seen: str | None


current_principal: ContextVar[Principal | None] = ContextVar("doc4ai_principal", default=None)


def parse_bearer(header_value: str | None) -> str | None:
    """The token in `Authorization: Bearer <t>`; scheme case-insensitive, whitespace and quotes stripped."""
    if not header_value:
        return None
    parts = header_value.strip().split(None, 1)
    if len(parts) != 2 or parts[0].lower() != "bearer":
        return None
    tok = parts[1].strip().strip('"').strip()
    return tok or None


class PatVerifier:
    def __init__(self, store, secret: bytes):
        self.store = store
        self.secret = secret

    def verify(self, raw: str | None, now: float | None = None) -> Principal | AuthFailure:
        now = time.time() if now is None else now
        if not raw:
            return AuthFailure("missing_token", None)
        seen = T.prefix_seen(raw)
        if len(raw) != T.TOKEN_LEN or not raw.startswith(T.PREFIX):
            return AuthFailure("bad_format", seen)
        if not T.is_well_formed(raw):
            return AuthFailure("bad_checksum", seen)
        row = self.store.get_api_token_by_hash(T.token_hash(self.secret, raw))
        if row is None:
            return AuthFailure("unknown_token", seen)
        if row["revoked_at"] is not None and row["revoked_at"] <= now:
            return AuthFailure("revoked", seen)
        if row["expires_at"] is not None and row["expires_at"] <= now:
            return AuthFailure("expired", seen)
        return Principal(kind="pat", subject="owner", token_id=row["id"], client_id=None,
                         scopes=frozenset(row["scopes"]), expires_at=row["expires_at"], name=row["name"],
                         rate_limit_per_min=row["rate_limit_per_min"])
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/mcp/test_principal.py -v`
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/mcp/principal.py tests/mcp/test_principal.py
git commit -m "feat(mcp): Principal, PatVerifier and Bearer parsing

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 8: Rate limiter (per-token token bucket)

**Files:**
- Create: `src/aidoc/mcp/ratelimit.py`
- Test: `tests/mcp/test_ratelimit.py`

**Interfaces:**
- Produces:
  ```python
  class RateLimiter:
      def __init__(self, default_per_min: Callable[[], int])     # reads config live (hot-apply)
      def acquire(self, key: str, per_min: int | None = None, now: float | None = None) -> tuple[bool, int]
          # (allowed, retry_after_s); per_min None → default; capacity = per_min, refill = per_min/60 per second
      def forget(self, key: str) -> None
  ```

- [ ] **Step 1: Write the failing tests**

`tests/mcp/test_ratelimit.py`:

```python
from aidoc.mcp.ratelimit import RateLimiter


def test_burst_then_refill():
    rl = RateLimiter(lambda: 60)
    t = 1000.0
    for _ in range(60):
        assert rl.acquire("k", now=t) == (True, 0)
    ok, retry = rl.acquire("k", now=t)
    assert ok is False and retry == 1                       # 1 token per second at 60/min
    assert rl.acquire("k", now=t + 1.0)[0] is True
    assert rl.acquire("k", now=t + 1.0)[0] is False
    assert rl.acquire("k", now=t + 61.0) == (True, 0)        # fully refilled after a minute


def test_per_token_override_and_isolation():
    rl = RateLimiter(lambda: 60)
    assert rl.acquire("slow", per_min=2, now=0.0)[0] and rl.acquire("slow", per_min=2, now=0.0)[0]
    ok, retry = rl.acquire("slow", per_min=2, now=0.0)
    assert ok is False and retry == 30                      # 2/min → one token every 30 s
    assert rl.acquire("other", now=0.0)[0] is True          # another key unaffected


def test_default_is_read_live():
    limit = {"v": 1}
    rl = RateLimiter(lambda: limit["v"])
    assert rl.acquire("k", now=0.0)[0] and not rl.acquire("k", now=0.0)[0]
    limit["v"] = 5
    rl.forget("k")
    assert all(rl.acquire("k", now=0.0)[0] for _ in range(5))


def test_retry_after_is_at_least_one_second():
    rl = RateLimiter(lambda: 100000)
    for _ in range(100000):
        rl.acquire("k", now=0.0)
    assert rl.acquire("k", now=0.0)[1] >= 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/mcp/test_ratelimit.py -v`
Expected: FAIL — `ModuleNotFoundError`.

- [ ] **Step 3: Implement**

`src/aidoc/mcp/ratelimit.py`:

```python
"""Per-token token bucket for tools/call (MCP spec §4.3 step 3, index A-M13)."""
from __future__ import annotations

import math
import threading
import time
from collections.abc import Callable


class RateLimiter:
    def __init__(self, default_per_min: Callable[[], int]):
        self._default = default_per_min
        self._buckets: dict[str, tuple[float, float]] = {}      # key -> (tokens, updated_at)
        self._lock = threading.Lock()

    def acquire(self, key: str, per_min: int | None = None, now: float | None = None) -> tuple[bool, int]:
        now = time.time() if now is None else now
        limit = max(1, int(per_min if per_min is not None else self._default()))
        rate = limit / 60.0
        with self._lock:
            tokens, updated = self._buckets.get(key, (float(limit), now))
            tokens = min(float(limit), tokens + (now - updated) * rate)
            if tokens >= 1.0:
                self._buckets[key] = (tokens - 1.0, now)
                return True, 0
            self._buckets[key] = (tokens, now)
            wait = (1.0 - tokens) / rate
            return False, max(1, math.ceil(wait))

    def forget(self, key: str) -> None:
        with self._lock:
            self._buckets.pop(key, None)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/mcp/test_ratelimit.py -v`
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/mcp/ratelimit.py tests/mcp/test_ratelimit.py
git commit -m "feat(mcp): per-token token-bucket rate limiter

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 9: Response budget and cursor helpers (`aidoc/mcp/budget.py`)

**Files:**
- Create: `src/aidoc/mcp/budget.py`
- Test: `tests/mcp/test_budget.py`

**Interfaces:**
- Produces:
  ```python
  def estimate_tokens(text: str) -> tuple[int, str]        # (tokens, "tiktoken"|"bytes"); bytes fallback = len(utf8)//3
  def cut_to_budget(text: str, max_tokens: int, *, start: int = 0) -> tuple[str, int | None]
      # keeps text[start:] up to max_tokens, cut at the last "\n\n" (else "\n", else a space) before the limit;
      # returns (kept, next_offset) with next_offset None when everything fit; never returns "" for non-empty input
  def encode_cursor(d: dict) -> str                        # base64url(json, compact), no padding
  def decode_cursor(s: str | None) -> dict | None          # None/"" → None; invalid → raises CursorError
  class CursorError(ValueError): ...
  ```

- [ ] **Step 1: Write the failing tests**

`tests/mcp/test_budget.py`:

```python
import pytest

from aidoc.mcp import budget as B


def test_estimate_tokens_methods(monkeypatch):
    n, method = B.estimate_tokens("hello world " * 10)
    assert method in ("tiktoken", "bytes") and n > 0
    import aidoc.chunk as chunk

    def boom(text):
        raise chunk.TokenizerUnavailable("offline")
    monkeypatch.setattr(chunk, "count_tokens", boom)
    n2, m2 = B.estimate_tokens("héllo")                      # 6 utf-8 bytes → 2
    assert (n2, m2) == (2, "bytes")


def test_cut_to_budget_prefers_paragraph_boundary():
    text = "para one " * 30 + "\n\n" + "para two " * 30 + "\n\n" + "para three " * 30
    kept, nxt = B.cut_to_budget(text, max_tokens=120)
    assert kept.endswith("para two") or kept.endswith("para one")
    assert nxt is not None and text[nxt:].lstrip("\n").startswith("para")
    assert B.estimate_tokens(kept)[0] <= 120
    rest, nxt2 = B.cut_to_budget(text, max_tokens=10_000, start=nxt)
    assert nxt2 is None and rest.strip().endswith("para three")


def test_cut_to_budget_never_returns_empty_for_nonempty_input():
    one_line = "x" * 50_000                                   # no boundary anywhere
    kept, nxt = B.cut_to_budget(one_line, max_tokens=100)
    assert kept and nxt is not None and nxt == len(kept)
    assert B.cut_to_budget("", max_tokens=5) == ("", None)


def test_cut_to_budget_fits():
    assert B.cut_to_budget("short\n\ntext", max_tokens=1000) == ("short\n\ntext", None)


def test_cursor_roundtrip_and_errors():
    c = B.encode_cursor({"k": "2026", "id": "abc"})
    assert "=" not in c and B.decode_cursor(c) == {"k": "2026", "id": "abc"}
    assert B.decode_cursor(None) is None and B.decode_cursor("") is None
    for bad in ("!!!", "eyJ4Ijo", B.encode_cursor({"k": 1})[:-2] + "zz", "W10"):   # garbage, truncated, tampered, not an object
        with pytest.raises(B.CursorError):
            B.decode_cursor(bad)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/mcp/test_budget.py -v`
Expected: FAIL — `ModuleNotFoundError`.

- [ ] **Step 3: Implement**

`src/aidoc/mcp/budget.py`:

```python
"""Response budget (MCP spec §5.1: default 8,000 tokens, truncate at a boundary) and opaque cursors (D10)."""
from __future__ import annotations

import base64
import json

from aidoc import chunk as _chunk


class CursorError(ValueError):
    pass


def estimate_tokens(text: str) -> tuple[int, str]:
    try:
        return _chunk.count_tokens(text), "tiktoken"
    except _chunk.TokenizerUnavailable:
        return len(text.encode("utf-8")) // 3, "bytes"


def _fits(text: str, max_tokens: int) -> bool:
    return estimate_tokens(text)[0] <= max_tokens


def cut_to_budget(text: str, max_tokens: int, *, start: int = 0) -> tuple[str, int | None]:
    """text[start:] cut to ~max_tokens at the last paragraph break (then line break, then space) before the limit.
    Binary search on the character length (token counting is monotone enough for this purpose)."""
    body = text[start:]
    if not body:
        return "", None
    if _fits(body, max_tokens):
        return body, None
    lo, hi = 1, len(body)                     # largest prefix length that fits
    while lo < hi:
        mid = (lo + hi + 1) // 2
        if _fits(body[:mid], max_tokens):
            lo = mid
        else:
            hi = mid - 1
    limit = lo
    cut = -1
    for sep in ("\n\n", "\n", " "):
        cut = body.rfind(sep, 0, limit)
        if cut > limit // 2:                  # a boundary in the second half of the window is worth using
            break
        cut = -1
    if cut <= 0:
        cut = limit
    kept = body[:cut].rstrip()
    if not kept:
        kept = body[:limit]
        cut = limit
    return kept, start + cut


def encode_cursor(d: dict) -> str:
    raw = json.dumps(d, separators=(",", ":"), sort_keys=True).encode("utf-8")
    return base64.urlsafe_b64encode(raw).decode("ascii").rstrip("=")


def decode_cursor(s: str | None) -> dict | None:
    if not s:
        return None
    try:
        raw = base64.urlsafe_b64decode(s + "=" * (-len(s) % 4))
        d = json.loads(raw)
    except (ValueError, UnicodeDecodeError) as e:
        raise CursorError("invalid cursor") from e
    if not isinstance(d, dict):
        raise CursorError("invalid cursor")
    return d
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/mcp/test_budget.py -v`
Expected: all PASS. (The tiktoken BPE must be cached under `data/tiktoken`; `estimate_tokens` falls back to bytes if not, and the tests accept either method except where they force the fallback.)

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/mcp/budget.py tests/mcp/test_budget.py
git commit -m "feat(mcp): token budget cutting and opaque cursors

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

### Task 10: Call-log retention in `Maintenance`

**Files:**
- Modify: `src/aidoc/server/maintenance.py` (`run_once`)
- Test: `tests/api/test_serve.py` (append)

**Interfaces:**
- Consumes: `Store.prune_mcp_calls` (Task 4), `cfg.mcp.call_log_retention_days`, `cfg.mcp.call_log_max_rows` (Task 2).
- Produces: `Maintenance.run_once()` result gains `"mcp_calls_pruned": n`.

- [ ] **Step 1: Write the failing test**

Append to `tests/api/test_serve.py`:

```python
def test_maintenance_prunes_mcp_calls(ctx):
    from aidoc.server.maintenance import Maintenance
    import time
    old = time.time() - 40 * 86400
    for i in range(5):
        ctx.store.insert_mcp_call(ts=old + i, status="ok", method="tools/list")
    for i in range(3):
        ctx.store.insert_mcp_call(status="ok", method="tools/list")
    ctx.config.mcp.call_log_retention_days = 30
    ctx.config.mcp.call_log_max_rows = 2
    res = Maintenance(ctx).run_once()
    assert res["mcp_calls_pruned"] == 6                      # 5 by age + 1 by the row cap
    assert ctx.store.count_mcp_calls(since=0) == 2
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/api/test_serve.py::test_maintenance_prunes_mcp_calls -v`
Expected: FAIL — `KeyError: 'mcp_calls_pruned'`.

- [ ] **Step 3: Implement**

In `Maintenance.run_once`, extend `res`:

```python
        mcp = ctx.config.mcp
        res["mcp_calls_pruned"] = ctx.store.prune_mcp_calls(older_than_ts=now - mcp.call_log_retention_days * 86400,
                                                            max_rows=mcp.call_log_max_rows)
```

(before `ctx.store.prune_events(...)`).

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/api/test_serve.py -v`
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/aidoc/server/maintenance.py tests/api/test_serve.py
git commit -m "feat(maintenance): prune mcp_calls by age and row cap

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0175gLq9uam6M5Z9yNNxM4Ba"
```

---

## Phase F acceptance

- [ ] `uv run pytest -m "not slow" -q` green; `uv run ruff check src tests` clean.
- [ ] `docs/superpowers/spikes/2026-10-mcp-sdk.md` has S1–S14 each with Finding / Decision / Affects (S13 may say "deferred to Task 35" if Claude Code was not available).
- [ ] `uv run python -c "from aidoc.store import Store; s=Store('data/aidoc.db'); print(s.page_index_tokenizer(), len(s.page_index_doc_ids()))"` on the real DB prints `trigram 0` **before** the first server start, and after `uv run aidoc serve` has run once (then Ctrl+C) prints `trigram 65` (all documents indexed by the backfill; number = current document count).
- [ ] `aidoc.toml` committed with `[mcp]`; `GET /api/settings` shows it.

## Implementation notes / deviations

Implemented 2026-10-02 (commits `95a5b37`..`4973744` on master).

- **Task 1 — installing the SDK.** The user's running server holds `.venv/Scripts/aidoc.exe`, so `uv add` could not reinstall the editable project. `mcp>=2.2,<2.3` was added to `pyproject.toml` by hand, then `uv lock` + `uv sync --no-install-project --inexact`; tests ran with `uv run --no-sync` until the server restart. The spike found three things the plan did not anticipate (recorded as rulings in the spike file and applied in Part S): S4 the modern era runs an internal `tools/list` through the middleware for every `tools/call` with arguments; S5 legacy stateless clients carry `clientInfo` only on `initialize`; S12 a stateless `GET /mcp` opens an SSE stream that never ends. S13 (Claude Code 401 behaviour) was run: plain error, no OAuth discovery.
- **Task 3.** `load_or_create_secret` opens the file with `O_BINARY` as well (Windows text-mode safety). No behaviour change elsewhere.
- **Task 4.** `tests/api/test_app_basics.py` asserted `schema_version == "2"`; moved to `"4"` as the plan instructs.
- **Task 6.** The pipeline hook is a small helper `pipeline._index_pages()` that logs through `PipelineContext.log` (the plan left the log callable to be checked).
- **Tasks 2/9/10.** Lint-only adjustments to the plan's test code (import order, `from aidoc import chunk`, `endswith((…))`); assertions unchanged.
- **Phase acceptance — real DB.** The `page_index_tokenizer()` check on `data/aidoc.db` was run only around the final server restart (opening the DB with the new code migrates it to v4 while the old server still runs). Results are in the Part S deviations section.
