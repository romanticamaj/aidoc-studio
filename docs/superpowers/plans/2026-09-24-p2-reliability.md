# P2 — Reliability + RAG Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Read `docs/superpowers/plans/2026-09-24-index.md` first; P1 (`2026-09-24-p1-core.md`) must be complete and its acceptance checklist green.

**Goal:** Make conversion resumable and robust — >40-page PDFs are converted in 40-page segments by one long-lived runner, interrupted work resumes from the last finished segment, failures are classified and retried correctly, missing/changed sources are detected, and `aidoc chunk` + the Claude Code skill ship.

**Architecture:** `segment.py` splits PDFs physically with PyMuPDF and merges normalised segment outputs; `pipeline.run_task` grows a segment loop backed by `RunnerSession` (one runner process per engine attempt); `retry.py` decides retry/fallback/fail; `sources.py` stages sources; `recovery.py` runs at every CLI/server start. Everything is exercised with the fake engine in `tests/reliability/`.

**Tech Stack:** as P1 + tiktoken (`cl100k_base`).

**Spec:** §3 (persistent runner), §4 (segment quick check), §5 (chunk), §7, §8.2–§8.6, §10 (reliability), §12 P2, §13.4.

## Global Constraints

Index §0 plus:
- No change to the runner protocol (index §5). Persistent mode only sends more than one line on stdin.
- Segment state lives in the `segments` table and `data/work/<task_id>/seg_<idx>/` (files: `seg_<idx>.pdf`, `raw/`, `normalized.md`, `assets.json`, `edges.json`). A segment is `done` only after all three result files are written.
- Every status change: DB first, then `emit`.

## Review Focus

1. A PDF of exactly 40 pages is one segment; 41 pages gives `[(1,40),(41,41)]` — test in Task 1.
2. Two adjacent segment-edge tables with different column counts, or without bbox evidence, must not be merged — test in Task 3.
3. A segment whose engine gave no page markers still yields `<!-- page: <page_start> -->` at its start in the merged document (segment bounds are known facts) — test in Task 3.
4. Retry backoff sleeps must be interruptible by cancel; a cancel during the 30 s backoff must end the task within 1 s — test in Task 5.
5. Startup recovery must not kill an unrelated process that reused a recorded PID — test in Task 7.
6. `aidoc chunk` on a document without page markers writes `page_start: null` and never crashes on a heading-less document — test in Task 10.

---

## File map for P2

| File | Responsibility |
|---|---|
| `src/aidoc/segment.py` | `plan_segments`, `split_pdf`, `merge_segments`, `merge_edge_tables` |
| `src/aidoc/engines/base.py`, `host.py`, `fake.py`, `markitdown.py`, `docling.py`, `mineru.py` | `RunnerSession` (persistent runner), cancel support, `oom_downgrade` |
| `src/aidoc/retry.py` | `RetryPolicy`, `classify` |
| `src/aidoc/sources.py` | `stage_source`, `purge_expired_work_copies` |
| `src/aidoc/recovery.py` | `recover_on_startup` |
| `src/aidoc/pipeline.py` | segment loop, retries, cancel, resume |
| `src/aidoc/store.py` | `create_task` reuse of `failed|cancelled`, `requeue_task` |
| `src/aidoc/chunk.py`, `cli.py` | `aidoc chunk`, `aidoc cancel` stub (P3 completes) |
| `skill/SKILL.md` | Claude Code skill |
| `tests/unit/test_segment.py`, `test_retry.py`, `test_sources.py`, `test_recovery.py`, `test_chunk.py`, `test_session.py`; `tests/reliability/*` | tests |

---

### Task 1: Segment planning and physical split

**Files:**
- Create: `src/aidoc/segment.py`, `tests/unit/test_segment.py`

**Interfaces:**
- Produces: `plan_segments(pages: int | None) -> list[tuple[int | None, int | None]]`; `split_pdf(src: Path, page_start: int, page_end: int, dst: Path) -> Path` (1-based inclusive; uses `insert_pdf(from_page=..., to_page=...)`); `segment_dir(work_dir: Path, idx: int) -> Path` = `work_dir / f"seg_{idx}"`.

- [x] **Step 1: Failing tests**

```python
# tests/unit/test_segment.py
import fitz
from aidoc.segment import plan_segments, split_pdf, segment_dir

def test_plan():
    assert plan_segments(None) == [(None, None)]
    assert plan_segments(1) == [(1, 1)] and plan_segments(40) == [(1, 40)]
    assert plan_segments(41) == [(1, 40), (41, 41)]
    assert plan_segments(125) == [(1, 40), (41, 80), (81, 120), (121, 125)]

def test_split_pdf(fixtures, tmp_path):
    out = split_pdf(fixtures / "big.pdf", 41, 45, tmp_path / "seg_1.pdf")
    d = fitz.open(out)
    assert d.page_count == 5 and "第 41 頁" in d[0].get_text() and "第 45 頁" in d[4].get_text()

def test_segment_dir(tmp_path):
    assert segment_dir(tmp_path, 3) == tmp_path / "seg_3"
```

- [x] **Step 2: Run** → FAIL. **Step 3: Implement** (`SEGMENT_PAGES` from models). **Step 4: Run** → pass. **Step 5: Commit** `feat: segment planning and PDF splitting`.

---

### Task 2: Persistent runner sessions and cancellation

**Files:**
- Modify: `src/aidoc/engines/base.py`, `src/aidoc/engines/host.py`, `src/aidoc/engines/fake.py`, `src/aidoc/engines/markitdown.py`, `src/aidoc/engines/docling.py`, `src/aidoc/engines/mineru.py`
- Create: `tests/unit/test_session.py`

**Interfaces:**
- Produces: `base.EngineCancelled(Exception)`; `base.RunnerSession` (concrete): `.pid: int | None`, `.engine_opts: dict`, `.convert(src: Path, workdir: Path, pages: tuple[int,int] | None, on_progress, *, timeout_s: float, segment_idx: int = 0, cancel: threading.Event | None = None) -> RawResult`, `.close()`; `RunnerEngine.open_session(opts: ConvertOptions, probe: ProbeResult, *, engine_opts: dict | None = None, full_page_ocr: bool = False) -> RunnerSession` (starts the runner once; `engine_opts` override lets retry pass downgraded options); `RunnerEngine.oom_downgrade(engine_opts: dict) -> dict | None` (default None; Docling halves `page_batch_size` down to 1; MinerU `standard → basic`); `RunnerEngine.convert` (P1 one-shot) is now implemented as open_session → convert → close. `RunnerHost.run(..., cancel: threading.Event | None = None)` polls `cancel` every 0.25 s; when set → `kill_tree`, raise `EngineCancelled`.

- [x] **Step 1: Failing tests**

```python
# tests/unit/test_session.py
import json, threading, time
from pathlib import Path
import pytest
from aidoc.engines.registry import get_engines
from aidoc.engines.base import EngineCancelled
from aidoc.engines.docling import DoclingEngine
from aidoc.engines.mineru import MineruEngine
from aidoc.config import load_config
from aidoc.models import ConvertOptions
from aidoc.probe import probe_file
from aidoc.segment import split_pdf
from tests.fakes.scenario import write_scenario, fake_env

def test_session_reuses_one_process(tmp_root, fixtures, monkeypatch):
    sc = write_scenario(tmp_root / "sc.json"); fake_env(monkeypatch, sc)
    e = get_engines(load_config())["docling"]; pr = probe_file(fixtures / "big.pdf")
    s = e.open_session(ConvertOptions(output_dir=tmp_root / "out"), pr)
    pid = s.pid; assert pid
    a = s.convert(split_pdf(fixtures / "big.pdf", 1, 40, tmp_root / "s0.pdf"), tmp_root / "seg_0", (1, 40), lambda f, l: None, timeout_s=60, segment_idx=0)
    b = s.convert(split_pdf(fixtures / "big.pdf", 41, 45, tmp_root / "s1.pdf"), tmp_root / "seg_1", (41, 45), lambda f, l: None, timeout_s=60, segment_idx=1)
    assert s.pid == pid and a.page_count == 40 and b.page_count == 5
    s.close()
    calls = [json.loads(l) for l in (tmp_root / "calls.jsonl").read_text().splitlines()]
    assert [c["pages"] for c in calls] == [[1, 40], [41, 45]]

def test_cancel_kills_runner(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json", default="slow_ok", slow_s=30))
    e = get_engines(load_config())["mineru"]; pr = probe_file(fixtures / "text.pdf")
    s = e.open_session(ConvertOptions(output_dir=tmp_root / "out"), pr); ev = threading.Event()
    threading.Timer(0.5, ev.set).start(); t0 = time.time()
    with pytest.raises(EngineCancelled):
        s.convert(fixtures / "text.pdf", tmp_root / "seg_0", None, lambda f, l: None, timeout_s=60, cancel=ev)
    assert time.time() - t0 < 5
    import psutil; assert not psutil.pid_exists(s.pid) or psutil.Process(s.pid).status() == psutil.STATUS_ZOMBIE

def test_oom_downgrade_rules():
    assert DoclingEngine().oom_downgrade({"page_batch_size": 16}) == {"page_batch_size": 8}
    assert DoclingEngine().oom_downgrade({"page_batch_size": 1}) is None
    assert MineruEngine().oom_downgrade({"tier": "standard"}) == {"tier": "basic"}
    assert MineruEngine().oom_downgrade({"tier": "basic"}) is None
```

- [x] **Step 2: Run** → FAIL. **Step 3: Implement** (`RunnerSession.__init__(engine_name, host, engine_opts, kind)`; `convert` writes request with `"pages"` and `"segment_idx"`, calls `host.run(request, workdir, timeout_s, on_progress, cancel=cancel)`, builds `RawResult` as in P1). **Step 4: Run** all unit tests (P1 host tests must still pass). **Step 5: Commit** `feat: persistent runner sessions with cancellation`.

---

### Task 3: Merge segments and cross-segment tables

**Files:**
- Modify: `src/aidoc/segment.py`
- Create: `tests/unit/test_merge.py`

**Interfaces:**
- Produces: `@dataclass SegmentPart(idx: int, page_start: int | None, page_end: int | None, markdown: str, assets: list[tuple[Path, str]], has_page_markers: bool, first_table: TableEdge | None, last_table: TableEdge | None, page_count: int | None)`; `merge_segments(parts: list[SegmentPart]) -> NormalizedResult`; `merge_edge_tables(prev_md: str, next_md: str, prev: SegmentPart, nxt: SegmentPart) -> tuple[str, str] | None` (returns rewritten `(prev_md, next_md)` when merged, else None); `split_trailing_gfm_table(md) -> tuple[str, list[str]]`, `split_leading_gfm_table(md) -> tuple[list[str], str]` (leading skips a first `<!-- page: N -->` line and blank lines).

Merge rule (spec §8.3): merge only if `prev.last_table` and `nxt.first_table` exist, `prev.last_table.touches_edge` and `prev.last_table.page == prev.page_count`, `nxt.first_table.touches_edge` and `nxt.first_table.page == 1`, and `n_cols` equal, and both markdown edges actually are GFM tables with that column count. The second table's header row + separator are dropped when identical to the first's header; otherwise the header row is kept as a data row (separator dropped). The `<!-- page: N -->` marker of `nxt` is preserved *before* the continuation rows? No — a marker inside a GFM table breaks it; place the marker **after** the merged table.

- [x] **Step 1: Failing tests**

```python
# tests/unit/test_merge.py
from pathlib import Path
from aidoc.models import TableEdge
from aidoc.segment import SegmentPart, merge_segments, merge_edge_tables

T1 = "<!-- page: 40 -->\ntext\n\n| a | b |\n| --- | --- |\n| 1 | 2 |\n"
T2 = "<!-- page: 41 -->\n| a | b |\n| --- | --- |\n| 3 | 4 |\n\nafter\n"
def part(idx, ps, pe, md, first=None, last=None, markers=True, pc=None):
    return SegmentPart(idx, ps, pe, md, [], markers, first, last, pc or (pe - ps + 1))

def test_merge_concatenates_and_moves_assets(tmp_path):
    a = part(0, 1, 40, "<!-- page: 1 -->\nA\n"); a.assets = [(tmp_path / "x.png", "p1_1.png")]
    b = part(1, 41, 45, "<!-- page: 41 -->\nB\n"); b.assets = [(tmp_path / "y.png", "p41_1.png")]
    r = merge_segments([a, b])
    assert r.markdown == "<!-- page: 1 -->\nA\n\n<!-- page: 41 -->\nB\n" and [n for _, n in r.assets] == ["p1_1.png", "p41_1.png"]

def test_segment_without_markers_gets_start_marker():
    r = merge_segments([part(0, 1, 40, "A\n", markers=False), part(1, 41, 45, "B\n", markers=False)])
    assert r.markdown.startswith("<!-- page: 1 -->\nA") and "<!-- page: 41 -->\nB" in r.markdown

def test_edge_tables_merged_same_header():
    p = part(0, 1, 40, T1, last=TableEdge(page=40, n_cols=2, touches_edge=True))
    n = part(1, 41, 45, T2, first=TableEdge(page=1, n_cols=2, touches_edge=True))
    r = merge_segments([p, n]).markdown
    assert r.count("| a | b |") == 1 and "| 1 | 2 |\n| 3 | 4 |" in r and r.index("| 3 | 4 |") < r.index("<!-- page: 41 -->")

def test_edge_tables_not_merged_when_cols_differ_or_no_bbox():
    p = part(0, 1, 40, T1, last=TableEdge(page=40, n_cols=3, touches_edge=True))
    n = part(1, 41, 45, T2, first=TableEdge(page=1, n_cols=2, touches_edge=True))
    assert merge_segments([p, n]).markdown.count("| a | b |") == 2
    p2 = part(0, 1, 40, T1); n2 = part(1, 41, 45, T2)          # no edge info at all
    assert merge_segments([p2, n2]).markdown.count("| a | b |") == 2
    p3 = part(0, 1, 40, T1, last=TableEdge(page=39, n_cols=2, touches_edge=True))   # table not on last page
    assert merge_segments([p3, n]).markdown.count("| a | b |") == 2

def test_edge_tables_different_header_keeps_row():
    n = part(1, 41, 45, "<!-- page: 41 -->\n| c | d |\n| --- | --- |\n| 3 | 4 |\n", first=TableEdge(page=1, n_cols=2, touches_edge=True))
    p = part(0, 1, 40, T1, last=TableEdge(page=40, n_cols=2, touches_edge=True))
    r = merge_segments([p, n]).markdown
    assert "| 1 | 2 |\n| c | d |\n| 3 | 4 |" in r and r.count("| --- | --- |") == 1
```

- [x] **Step 2: Run** → FAIL. **Step 3: Implement**. **Step 4: Run** → pass. **Step 5: Commit** `feat: segment merge with cross-segment table joining`.

---

### Task 4: Pipeline segment loop with quick checks, fail-fast and resume

**Files:**
- Modify: `src/aidoc/pipeline.py`, `src/aidoc/output.py` (sidecar `segments` = number of segments)
- Create: `tests/unit/test_pipeline_segments.py`

**Interfaces:**
- Produces (pipeline internals, referenced by later tasks): `run_engine_attempt(ctx, engine, engine_opts) -> tuple[NormalizedResult, QualityResult]` (raises `EngineError`/`EngineCancelled`); `load_segment_part(seg_dir, seg_row) -> SegmentPart`; `save_segment_part(seg_dir, part)`; per-segment quick check = `quality.assess(norm.markdown, probe, pages_in_segment)` with `blank_pages` filtered to the segment's range → `low` → raise `EngineError(engine, "segment quick check failed: <reasons>")`.

Flow changes vs P1:
1. After routing: `ranges = plan_segments(probe.pages)`; if the task already has segment rows (resume) keep them, else `store.create_segments`.
2. For each engine attempt: `session = engine.open_session(opts, probe, engine_opts=..., full_page_ocr=...)`; `store.update_task(pid=session.pid, engine=name)`; for each segment row in idx order: if `status == done` and its files exist → load part and continue (resume); else `update_segment(status=converting)`, split PDF (or use whole file when `page_start is None`), `raw = session.convert(...)`, `norm = normalize(raw, page_start - 1 if page_start else 0, idx)`, quick check, `save_segment_part`, `update_segment(status=done, output_path=str(seg_dir))`, emit `segment.updated`, emit progress `task.updated` with `progress = {pages_done, pages_total}`; `session.close()` in `finally`.
3. Engine failure (`EngineError(engine)`) or final quality `low` in the multi-engine loop → `store.reset_segments(task_id)`, delete segment dirs, next engine (spec §4: completed segments are discarded when switching engine).
4. `merge_segments(parts)` → whole-document `assess` → done/low as P1. Sidecar `segments = len(ranges)`.
5. Cancel: `EngineCancelled` or `cancel.is_set()` between segments → `cancelled`; done segments keep their rows/files.

- [x] **Step 1: Failing tests**

```python
# tests/unit/test_pipeline_segments.py
import json, shutil, threading
from pathlib import Path
import pytest
from aidoc.config import load_config
from aidoc.engines.registry import get_engines
from aidoc.models import ConvertOptions, TaskStatus
from aidoc.names import file_sha256
from aidoc.pipeline import run_task
from aidoc.store import Store
from tests.fakes.scenario import write_scenario, fake_env

@pytest.fixture
def env(tmp_root, monkeypatch, fixtures):
    sc = write_scenario(tmp_root / "sc.json"); fake_env(monkeypatch, sc)
    cfg = load_config(); store = Store(tmp_root / "data" / "aidoc.db")
    def make(name="big.pdf", **opt):
        src = tmp_root / name; shutil.copy(fixtures / name, src)
        opts = ConvertOptions(output_dir=tmp_root / "out", **opt); job = store.create_job(opts, "cli")
        tid, _ = store.create_task(job, str(src), file_sha256(src), src.stat().st_size, src.stat().st_mtime, opts.lang, str(tmp_root / "out" / src.stem))
        return tid
    return dict(cfg=cfg, store=store, sc=sc, make=make, root=tmp_root, engines=lambda: get_engines(cfg))

def calls(root): return [json.loads(l) for l in (root / "calls.jsonl").read_text().splitlines()]

def test_big_pdf_two_segments_one_runner(env):
    tid = env["make"]()
    assert run_task(env["store"], tid, env["engines"](), env["cfg"]) == TaskStatus.done
    segs = env["store"].list_segments(tid)
    assert [(s["page_start"], s["page_end"], s["status"]) for s in segs] == [(1, 40, "done"), (41, 45, "done")]
    sc = json.loads((env["root"] / "out" / "big" / "big.json").read_text(encoding="utf-8"))
    assert sc["segments"] == 2 and sc["pages"] == 45
    md = (env["root"] / "out" / "big" / "big.md").read_text(encoding="utf-8")
    assert "<!-- page: 41 -->" in md and "<!-- page: 45 -->" in md and md.count("<!-- page: ") == 45
    assert (env["root"] / "out" / "big" / "assets" / "p45_1.png").exists()
    assert [c["pages"] for c in calls(env["root"])] == [[1, 40], [41, 45]]

def test_quick_check_fail_fast_switches_engine(env):
    write_scenario(env["sc"], rules=[{"match": {"engine": "docling", "segment_idx": 0}, "behavior": "low"}])
    tid = env["make"]()
    assert run_task(env["store"], tid, env["engines"](), env["cfg"]) == TaskStatus.done
    t = env["store"].get_task(tid)
    assert t["engine"] == "mineru" and "quick check" in (t["tried"][0]["error_msg"] or "")
    c = calls(env["root"])
    assert [(x["engine"], x["pages"]) for x in c] == [("docling", [1, 40]), ("mineru", [1, 40]), ("mineru", [41, 45])]

def test_resume_skips_done_segments(env):
    write_scenario(env["sc"], rules=[{"match": {"segment_idx": 1, "attempt": 1}, "behavior": "crash"}])
    tid = env["make"]()
    st = run_task(env["store"], tid, env["engines"](), env["cfg"])      # crash -> transient retry (Task 5) resumes seg 1
    assert st == TaskStatus.done
    c = calls(env["root"])
    assert [x["pages"] for x in c] == [[1, 40], [41, 45], [41, 45]]

def test_cancel_keeps_done_segments(env):
    write_scenario(env["sc"], rules=[{"match": {"segment_idx": 1}, "behavior": "slow_ok"}], slow_s=30)
    tid = env["make"](); ev = threading.Event()
    def watch():
        import time
        while env["store"].list_segments(tid)[0]["status"] != "done": time.sleep(0.05)
        ev.set()
    threading.Thread(target=watch, daemon=True).start()
    assert run_task(env["store"], tid, env["engines"](), env["cfg"], cancel=ev) == TaskStatus.cancelled
    segs = env["store"].list_segments(tid)
    assert segs[0]["status"] == "done" and segs[1]["status"] in ("queued", "converting")
    assert not (env["root"] / "out" / "big").exists()
```

(The third test depends on Task 5's transient retry; write it now, expect it to fail until Task 5, and note that in the Task 4 commit message.)

- [x] **Step 2: Run** → FAIL. **Step 3: Implement**. **Step 4: Run** → tests 1, 2, 4 pass; test 3 fails on retry (expected until Task 5). All P1 pipeline tests still pass. **Step 5: Commit** `feat: segmented conversion with quick checks and resume (retry pending)`.

---

### Task 5: Failure classification and retry policy

**Files:**
- Create: `src/aidoc/retry.py`, `tests/unit/test_retry.py`
- Modify: `src/aidoc/pipeline.py`

**Interfaces:**
- Produces: `@dataclass RetryDecision(action: Literal["retry","fallback","fail"], delay_s: float = 0.0, engine_opts: dict | None = None, note: str = "")`; `class RetryPolicy(max_transient=2, backoff=(5.0, 30.0))` with `.decide(err: EngineError, transient_retries_used: int, engine, engine_opts: dict) -> RetryDecision`:
  - `err.kind == input` → `fail`.
  - `err.kind == engine` → `fallback`.
  - `err.kind == transient and err.oom` → `engine.oom_downgrade(engine_opts)`; None → `fallback` (note `"oom_no_downgrade"`); else `retry` with `delay_s=0`, new opts (does not consume a transient retry).
  - `err.kind == transient` (timeout/crash/kill) → `retry` with `delay_s = backoff[min(used, len-1)]` while `used < max_transient`, else `fallback`.
- Pipeline: wraps each engine attempt: on `retry` → `store.append_attempt(... error_kind="transient")`, wait `delay_s` via `cancel.wait(delay)` if cancel given else `time.sleep`, re-open session with new `engine_opts`, resume from unfinished segments (done segments kept); on `fallback` → reset segments, next engine; `fail` → `failed(input)`. `tasks.attempt` counts every session opened.

- [x] **Step 1: Failing tests**

```python
# tests/unit/test_retry.py
import json, shutil, threading, time
import pytest
from aidoc.engines.base import EngineError
from aidoc.engines.docling import DoclingEngine
from aidoc.engines.mineru import MineruEngine
from aidoc.models import ErrorKind, ConvertOptions, TaskStatus
from aidoc.retry import RetryPolicy
from aidoc.config import load_config
from aidoc.engines.registry import get_engines
from aidoc.names import file_sha256
from aidoc.pipeline import run_task
from aidoc.store import Store
from tests.fakes.scenario import write_scenario, fake_env

P = RetryPolicy()
def test_input_fails(): assert P.decide(EngineError(ErrorKind.input, "x"), 0, DoclingEngine(), {}).action == "fail"
def test_engine_falls_back(): assert P.decide(EngineError(ErrorKind.engine, "x"), 0, DoclingEngine(), {}).action == "fallback"
def test_transient_backoff():
    d0 = P.decide(EngineError(ErrorKind.transient, "timeout"), 0, DoclingEngine(), {"page_batch_size": 16})
    d1 = P.decide(EngineError(ErrorKind.transient, "timeout"), 1, DoclingEngine(), {"page_batch_size": 16})
    d2 = P.decide(EngineError(ErrorKind.transient, "timeout"), 2, DoclingEngine(), {"page_batch_size": 16})
    assert (d0.action, d0.delay_s) == ("retry", 5.0) and (d1.action, d1.delay_s) == ("retry", 30.0) and d2.action == "fallback"
def test_oom_downgrades_then_fallback():
    d = P.decide(EngineError(ErrorKind.transient, "oom", oom=True), 0, MineruEngine(), {"tier": "standard"})
    assert d.action == "retry" and d.engine_opts == {"tier": "basic"} and d.delay_s == 0
    d = P.decide(EngineError(ErrorKind.transient, "oom", oom=True), 0, MineruEngine(), {"tier": "basic"})
    assert d.action == "fallback"
    d = P.decide(EngineError(ErrorKind.transient, "oom", oom=True), 0, DoclingEngine(), {"page_batch_size": 16})
    assert d.engine_opts == {"page_batch_size": 8}

@pytest.fixture
def env(tmp_root, monkeypatch, fixtures):
    sc = write_scenario(tmp_root / "sc.json"); fake_env(monkeypatch, sc)
    cfg = load_config(); store = Store(tmp_root / "data" / "aidoc.db")
    monkeypatch.setattr("aidoc.pipeline.RETRY_POLICY", RetryPolicy(max_transient=2, backoff=(0.05, 0.1)))
    def make(name="text.pdf", **opt):
        src = tmp_root / name; shutil.copy(fixtures / name, src)
        opts = ConvertOptions(output_dir=tmp_root / "out", **opt); job = store.create_job(opts, "cli")
        return store.create_task(job, str(src), file_sha256(src), src.stat().st_size, src.stat().st_mtime, opts.lang, str(tmp_root / "out" / src.stem))[0]
    return dict(cfg=cfg, store=store, sc=sc, make=make, root=tmp_root, engines=lambda: get_engines(cfg))

def test_pipeline_retries_transient_then_succeeds(env):
    write_scenario(env["sc"], rules=[{"match": {"engine": "docling", "attempt": 1}, "behavior": "crash"}])
    tid = env["make"]()
    assert run_task(env["store"], tid, env["engines"](), env["cfg"]) == TaskStatus.done
    t = env["store"].get_task(tid)
    assert t["engine"] == "docling" and t["attempt"] == 2 and t["tried"][0]["error_kind"] == "transient"

def test_pipeline_oom_downgrade_recorded(env):
    write_scenario(env["sc"], rules=[{"match": {"engine": "mineru", "attempt": 1}, "behavior": "oom"}])
    tid = env["make"]("scanned_cht.pdf", mineru_tier="standard")
    assert run_task(env["store"], tid, env["engines"](), env["cfg"]) == TaskStatus.done
    t = env["store"].get_task(tid)
    assert t["engine"] == "mineru" and "oom" in t["tried"][0]["error_msg"].lower() and "basic" in t["tried"][0]["error_msg"]

def test_pipeline_exhausts_retries_then_fallback(env):
    write_scenario(env["sc"], rules=[{"match": {"engine": "docling"}, "behavior": "crash"}])
    tid = env["make"]()
    assert run_task(env["store"], tid, env["engines"](), env["cfg"]) == TaskStatus.done
    t = env["store"].get_task(tid)
    assert [a["engine"] for a in t["tried"]] == ["docling", "docling", "docling", "mineru"] and t["engine"] == "mineru"

def test_timeout_is_transient(env):
    write_scenario(env["sc"], rules=[{"match": {"engine": "docling", "attempt": 1}, "behavior": "timeout"}])
    tid = env["make"](timeout_s=1)
    assert run_task(env["store"], tid, env["engines"](), env["cfg"]) == TaskStatus.done
    assert "timeout" in env["store"].get_task(tid)["tried"][0]["error_msg"]

def test_cancel_during_backoff(env, monkeypatch):
    monkeypatch.setattr("aidoc.pipeline.RETRY_POLICY", RetryPolicy(max_transient=2, backoff=(30.0, 30.0)))
    write_scenario(env["sc"], rules=[{"match": {"engine": "docling"}, "behavior": "crash"}])
    tid = env["make"](); ev = threading.Event(); threading.Timer(0.5, ev.set).start(); t0 = time.time()
    assert run_task(env["store"], tid, env["engines"](), env["cfg"], cancel=ev) == TaskStatus.cancelled
    assert time.time() - t0 < 5
```

- [x] **Step 2: Run** → FAIL. **Step 3: Implement** (`RETRY_POLICY = RetryPolicy()` module constant in `pipeline.py`; the OOM attempt's `error_msg` must mention both `oom` and the downgraded option, e.g. `"oom: retrying with {'tier': 'basic'}"`). **Step 4: Run** `tests/unit` → all pass including `test_resume_skips_done_segments` from Task 4. **Step 5: Commit** `feat: failure classification and retry policy`.

---

### Task 6: Source staging and work-copy retention

**Files:**
- Create: `src/aidoc/sources.py`, `tests/unit/test_sources.py`
- Modify: `src/aidoc/pipeline.py` (replace P1's plain copy with `stage_source`)

**Interfaces:**
- Produces: `class SourceError(Exception)` with `.code in {"source_missing","source_changed"}`; `stage_source(source_path: Path, expected_sha: str, work_dir: Path) -> Path` (missing → `source_missing`; `file_sha256 != expected` → `source_changed`; hardlink via `os.link` when both on the same drive (`Path(...).drive` equal on Windows, `st_dev` equal on POSIX) and `os.link` does not raise, else `shutil.copy2`; returns `work_dir / f"src{ext}"`; idempotent when already staged with the right sha); `purge_expired_work_copies(store, now: float) -> int` (deletes `work_copy_path` dirs of documents whose `work_copy_expires_at < now`, nulls both columns, returns count). Pipeline: `SourceError` → `failed(input: <code>)`; after success `documents.work_copy_path = work_dir`, `work_copy_expires_at = now + retention_days*86400`; for uploads (P3) the file is already in `work_dir` and `stage_source` just verifies.

- [x] **Step 1: Failing tests**

```python
# tests/unit/test_sources.py
import os, shutil, time
import pytest
from aidoc.names import file_sha256
from aidoc.sources import stage_source, SourceError, purge_expired_work_copies
from aidoc.store import Store
from aidoc.config import load_config
from aidoc.engines.registry import get_engines
from aidoc.models import ConvertOptions, TaskStatus
from aidoc.pipeline import run_task
from tests.fakes.scenario import write_scenario, fake_env

def test_stage_hardlink_or_copy(tmp_path, fixtures):
    src = tmp_path / "a.pdf"; shutil.copy(fixtures / "text.pdf", src)
    staged = stage_source(src, file_sha256(src), tmp_path / "work")
    assert staged.name == "src.pdf" and file_sha256(staged) == file_sha256(src)
    assert os.stat(staged).st_nlink == 2 or staged.stat().st_size == src.stat().st_size

def test_missing_and_changed(tmp_path, fixtures):
    with pytest.raises(SourceError) as e: stage_source(tmp_path / "nope.pdf", "x", tmp_path / "w")
    assert e.value.code == "source_missing"
    src = tmp_path / "a.pdf"; shutil.copy(fixtures / "text.pdf", src)
    with pytest.raises(SourceError) as e: stage_source(src, "0" * 64, tmp_path / "w")
    assert e.value.code == "source_changed"

def test_pipeline_source_changed(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json")); cfg = load_config(); store = Store(tmp_root / "data" / "aidoc.db")
    src = tmp_root / "a.pdf"; shutil.copy(fixtures / "text.pdf", src); sha = file_sha256(src)
    job = store.create_job(ConvertOptions(output_dir=tmp_root / "out"), "cli")
    tid, _ = store.create_task(job, str(src), sha, 1, 1.0, "cht", str(tmp_root / "out" / "a"))
    shutil.copy(fixtures / "sample.docx", src)                           # modified after job creation
    assert run_task(store, tid, get_engines(cfg), cfg) == TaskStatus.failed
    assert store.get_task(tid)["error_msg"].startswith("source_changed")
    src.unlink(); tid2, _ = store.create_task(job, str(src), sha, 1, 1.0, "cht", str(tmp_root / "out" / "a2"))
    assert run_task(store, tid2, get_engines(cfg), cfg) == TaskStatus.failed
    assert store.get_task(tid2)["error_msg"].startswith("source_missing")

def test_engine_reads_work_copy_not_original(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json")); cfg = load_config(); store = Store(tmp_root / "data" / "aidoc.db")
    src = tmp_root / "a.pdf"; shutil.copy(fixtures / "text.pdf", src)
    job = store.create_job(ConvertOptions(output_dir=tmp_root / "out"), "cli")
    tid, _ = store.create_task(job, str(src), file_sha256(src), 1, 1.0, "cht", str(tmp_root / "out" / "a"))
    assert run_task(store, tid, get_engines(cfg), cfg) == TaskStatus.done
    import json; call = json.loads((tmp_root / "calls.jsonl").read_text().splitlines()[0])
    assert call["source"].startswith("src") or call["source"].startswith("seg_")
    doc = store.find_document(file_sha256(src), str(tmp_root / "out" / "a"))
    assert doc["work_copy_path"] and doc["work_copy_expires_at"] > time.time() + 6 * 86400

def test_purge_expired(tmp_root):
    store = Store(tmp_root / "data" / "aidoc.db"); w = tmp_root / "data" / "work" / "t"; w.mkdir(parents=True); (w / "src.pdf").write_bytes(b"x")
    store.upsert_document(sha256="a" * 64, source_path="a", output_dir="o", engine="e", quality={}, pages=1, lang="cht", aidoc_version="0.1.0",
                          status="ok", work_copy_path=str(w), work_copy_expires_at=time.time() - 1)
    assert purge_expired_work_copies(store, time.time()) == 1 and not w.exists()
    assert store.list_documents()[0]["work_copy_path"] is None
```

- [x] **Step 2: Run** → FAIL. **Step 3: Implement**. **Step 4: Run** → pass. **Step 5: Commit** `feat: source staging with sha verification and work-copy retention`.

---

### Task 7: Startup recovery

**Files:**
- Create: `src/aidoc/recovery.py`, `tests/unit/test_recovery.py`
- Modify: `src/aidoc/cli.py`, `src/aidoc/batch.py` (call `recover_on_startup` before running), `src/aidoc/store.py` (`list_tasks(status_in=[...])`, `tasks_with_pid()`)

**Interfaces:**
- Produces: `recover_on_startup(store, config, log: Callable[[str], None] = print) -> dict` returning `{"killed": [pids], "requeued": [task_ids], "segments_reset": n, "tmp_removed": [paths], "trash_removed": [paths], "orphaned": [doc_ids], "work_purged": n}`. Order is mandatory: (1) kill orphans, (2) requeue, (3) clean `.tmp/`/`.trash/`, (4) mark orphaned documents (sidecar missing on disk), (5) purge expired work copies.
- Kill rule: task rows with `pid IS NOT NULL` and status in `probing|converting|checking`; kill only when `psutil.pid_exists(pid) and procs.is_aidoc_runner(pid)`; always null the pid.
- Requeue: those tasks → `queued`, `pid NULL`; their segments with `status == converting` → `queued`, `output_path NULL`; `done` segments untouched.
- Clean: for each distinct `Path(tasks.output_dir).parent` (and `config.output_root()`): remove `.tmp/<x>` where `x` is not a non-terminal task id; remove everything under `.trash/`.

- [x] **Step 1: Failing tests**

```python
# tests/unit/test_recovery.py
import json, os, subprocess, sys, time
import psutil, pytest
from pathlib import Path
from aidoc import paths
from aidoc.config import load_config
from aidoc.models import ConvertOptions
from aidoc.recovery import recover_on_startup
from aidoc.store import Store

_counter = [0]
def mk(store, root, status, pid=None, segs=()):
    """Create a task row in the given status; each call gets a unique sha256 and output_dir."""
    _counter[0] += 1
    job = store.create_job(ConvertOptions(output_dir=root / "out"), "cli")
    tid, _ = store.create_task(job, "a.pdf", os.urandom(32).hex(), 1, 1.0, "cht", str(root / "out" / f"d{_counter[0]}"))
    store.update_task(tid, status=status, pid=pid)
    if segs:
        ids = store.create_segments(tid, [(1, 40), (41, 45)])
        for i, s in zip(ids, segs):
            store.update_segment(i, status=s, output_path="p" if s != "queued" else None)
    return tid

def test_orphan_runner_killed_but_unrelated_pid_spared(tmp_root):
    store = Store(tmp_root / "data" / "aidoc.db")
    runner = subprocess.Popen([sys.executable, str(paths.runner_script("fake"))], stdin=subprocess.PIPE, stderr=subprocess.DEVNULL, stdout=subprocess.DEVNULL)
    other = subprocess.Popen([sys.executable, "-c", "import time; time.sleep(30)"])
    t1 = mk(store, tmp_root, "converting", pid=runner.pid); t2 = mk(store, tmp_root, "converting", pid=other.pid)
    res = recover_on_startup(store, load_config(), log=lambda s: None)
    assert res["killed"] == [runner.pid]; runner.wait(10)
    assert psutil.pid_exists(other.pid) and other.poll() is None; other.kill()
    assert store.get_task(t1)["status"] == "queued" and store.get_task(t1)["pid"] is None and store.get_task(t2)["pid"] is None

def test_requeue_resets_only_converting_segments(tmp_root):
    store = Store(tmp_root / "data" / "aidoc.db")
    t = mk(store, tmp_root, "converting", segs=("done", "converting"))
    recover_on_startup(store, load_config(), log=lambda s: None)
    segs = store.list_segments(t)
    assert [(s["status"], s["output_path"]) for s in segs] == [("done", "p"), ("queued", None)]

def test_tmp_and_trash_cleanup(tmp_root):
    store = Store(tmp_root / "data" / "aidoc.db"); out = tmp_root / "out"
    live = mk(store, tmp_root, "queued")
    for d in [out / ".tmp" / live, out / ".tmp" / "stale", out / ".trash" / "x"]: d.mkdir(parents=True); (d / "f").write_text("x")
    res = recover_on_startup(store, load_config(), log=lambda s: None)
    assert (out / ".tmp" / live).exists() and not (out / ".tmp" / "stale").exists() and not (out / ".trash" / "x").exists()
    assert res["tmp_removed"] == [str(out / ".tmp" / "stale")]

def test_orphaned_documents_marked(tmp_root):
    store = Store(tmp_root / "data" / "aidoc.db"); out = tmp_root / "out"
    (out / "keep").mkdir(); (out / "keep" / "keep.json").write_text("{}")
    for name in ("keep", "gone"):
        store.upsert_document(sha256=name * 16, source_path=name, output_dir=str(out / name), engine="e", quality={}, pages=1, lang="cht",
                              aidoc_version="0.1.0", status="ok", work_copy_path=None, work_copy_expires_at=None)
    res = recover_on_startup(store, load_config(), log=lambda s: None)
    st = {d["source_path"]: d["status"] for d in store.list_documents()}
    assert st == {"keep": "ok", "gone": "orphaned"} and len(res["orphaned"]) == 1
```

- [x] **Step 2: Run** → FAIL. **Step 3: Implement**; wire `recover_on_startup` into `cli.py` for `batch` and `convert` (log lines to stderr only when something was done). **Step 4: Run** → pass. **Step 5: Commit** `feat: startup recovery of orphans, states and temp dirs`.

---

### Task 8: Task reuse for failed/cancelled, `requeue_task`, CLI batch resume

**Files:**
- Modify: `src/aidoc/store.py`, `src/aidoc/batch.py`, `src/aidoc/cli.py`
- Modify tests: `tests/unit/test_store.py` (add), `tests/unit/test_batch.py` (add)

**Interfaces:**
- Produces: `Store.create_task` now: existing row with status in `queued|probing|converting|checking|failed|cancelled` → reuse (re-parent, set `queued`, keep segments, reset `pid`, keep `tried`/`attempt` history; `failed(input)` rows are reused too — the file may have been restored); `done|low|skipped` → replace as before. `Store.requeue_task(task_id, *, reset_segments: bool, new_sha: str | None = None) -> None` (status `queued`, `error_* NULL`, `pid NULL`; `reset_segments` → `reset_segments()`; `new_sha` updates `sha256/size/mtime` for "convert with new version"). `batch.run_batch` skips tasks whose status is already terminal at loop time only if `skipped`/`done`/`low` (i.e. resumed jobs continue queued ones). CLI `aidoc cancel <job_id>` — P2 implements the in-process no-op message `"no server running; use Ctrl+C"` (P3 replaces with server call).

- [x] **Step 1: Failing tests** (append)

```python
# tests/unit/test_store.py (append)
def test_failed_and_cancelled_reused_keeping_segments(store):
    j = store.create_job(ConvertOptions(output_dir=Path("out")), "cli")
    tid, _ = store.create_task(j, "a.pdf", "k" * 64, 1, 1.0, "cht", "out/k")
    ids = store.create_segments(tid, [(1, 40), (41, 45)]); store.update_segment(ids[0], status="done", output_path="p")
    store.update_task(tid, status=TaskStatus.cancelled, pid=123)
    tid2, reused = store.create_task(j, "a.pdf", "k" * 64, 1, 1.0, "cht", "out/k")
    assert reused and tid2 == tid and store.get_task(tid)["status"] == "queued" and store.get_task(tid)["pid"] is None
    assert store.list_segments(tid)[0]["status"] == "done"
    store.requeue_task(tid, reset_segments=True, new_sha="m" * 64)
    assert store.list_segments(tid)[0]["status"] == "queued" and store.get_task(tid)["sha256"] == "m" * 64
```

```python
# tests/unit/test_batch.py (append)
def test_batch_resumes_unfinished_segments(tmp_root, fixtures, monkeypatch):
    """Interrupted run: segment 0 done, task cancelled while segment 1 runs; re-running the batch converts only segment 1."""
    import threading, time
    from aidoc.pipeline import run_task
    sc = write_scenario(tmp_root / "sc.json", rules=[{"match": {"segment_idx": 1}, "behavior": "slow_ok"}], slow_s=30)
    fake_env(monkeypatch, sc)
    d = tmp_root / "in"; d.mkdir(); shutil.copy(fixtures / "big.pdf", d / "big.pdf"); out = tmp_root / "out"
    cfg = load_config(); store = Store(tmp_root / "data" / "aidoc.db"); opts = ConvertOptions(output_dir=out)
    src = d / "big.pdf"; job = store.create_job(opts, "cli")
    from aidoc.names import file_sha256
    tid, _ = store.create_task(job, str(src), file_sha256(src), src.stat().st_size, src.stat().st_mtime, "cht", str(out / "big"))
    ev = threading.Event()
    def watch():
        while store.list_segments(tid) == [] or store.list_segments(tid)[0]["status"] != "done": time.sleep(0.05)
        ev.set()
    threading.Thread(target=watch, daemon=True).start()
    assert run_task(store, tid, get_engines(cfg), cfg, cancel=ev).value == "cancelled"
    write_scenario(tmp_root / "sc.json")                                          # fast again
    assert main(["batch", str(d), "-o", str(out)]) == 0
    calls = [json.loads(l) for l in (tmp_root / "calls.jsonl").read_text().splitlines()]
    assert [c["pages"] for c in calls].count([1, 40]) == 1 and [c["pages"] for c in calls].count([41, 45]) == 2
    t2 = store.list_tasks(store.list_jobs()[0]["id"])[0]
    assert t2["id"] == tid and t2["status"] == "done" and (out / "big" / "big.md").exists()
```

(The `[41, 45]` count is 2 because the cancelled slow call was logged before it was killed, then re-run.)

- [x] **Step 2: Run** → FAIL. **Step 3: Implement**. **Step 4: Run** → pass. **Step 5: Commit** `feat: reuse failed/cancelled tasks and resume batches from unfinished segments`.

---

### Task 9: Reliability suite (process-level)

**Files:**
- Create: `tests/reliability/__init__.py`, `tests/reliability/conftest.py`, `tests/reliability/test_kill_batch.py`, `tests/reliability/test_kill_engine.py`, `tests/reliability/test_write_failure.py`, `tests/reliability/test_force_open_handle.py`, `tests/reliability/test_source_guard.py`

**Interfaces:**
- `tests/reliability/conftest.py` provides `batch_proc(root, input_dir, out, scenario) -> subprocess.Popen` launching `[sys.executable, "-m", "aidoc.cli", "batch", ...]` with env `AIDOC_ROOT/AIDOC_DATA/AIDOC_FAKE_ENGINES/AIDOC_FAKE_SCENARIO` and `wait_for(pred, timeout)` polling helper; `db(root) -> Store`.

- [x] **Step 1: Write the tests**

```python
# tests/reliability/test_kill_batch.py
import json, shutil, sys, time
from aidoc import procs
from aidoc.store import Store
from tests.fakes.scenario import write_scenario

def test_batch_killed_mid_conversion_resumes(tmp_root, fixtures, batch_proc, wait_for):
    d = tmp_root / "in"; d.mkdir(); shutil.copy(fixtures / "big.pdf", d / "big.pdf"); out = tmp_root / "out"
    sc = write_scenario(tmp_root / "sc.json", rules=[{"match": {"segment_idx": 1}, "behavior": "slow_ok"}], slow_s=20)
    p = batch_proc(tmp_root, d, out, sc)
    store = Store(tmp_root / "data" / "aidoc.db")
    wait_for(lambda: any(s["status"] == "done" for t in store.list_tasks() for s in store.list_segments(t["id"])), 60)
    task = store.list_tasks()[0]; runner_pid = task["pid"]; assert runner_pid
    procs.kill_tree(p.pid); p.wait(10)                          # server/CLI dies; runner may survive as orphan
    store2 = Store(tmp_root / "data" / "aidoc.db")
    assert store2.get_task(task["id"])["status"] == "converting"   # DB shows the interrupted state
    sc = write_scenario(tmp_root / "sc.json")                       # fast again
    p2 = batch_proc(tmp_root, d, out, sc); assert p2.wait(120) == 0
    import psutil; assert not psutil.pid_exists(runner_pid) or not procs.is_aidoc_runner(runner_pid)
    t = store2.get_task(task["id"]); assert t["status"] == "done"
    calls = [json.loads(l) for l in (tmp_root / "calls.jsonl").read_text().splitlines()]
    assert [c["pages"] for c in calls].count([1, 40]) == 1          # segment 0 not re-run
    assert (out / "big" / "big.md").exists() and not (out / ".tmp").exists() or not any((out / ".tmp").iterdir())
```

```python
# tests/reliability/test_kill_engine.py
import shutil, threading, time
from aidoc import procs
from aidoc.config import load_config
from aidoc.engines.registry import get_engines
from aidoc.models import ConvertOptions, TaskStatus
from aidoc.names import file_sha256
from aidoc.pipeline import run_task
from aidoc.retry import RetryPolicy
from aidoc.store import Store
from tests.fakes.scenario import write_scenario, fake_env

def test_killed_engine_is_transient_and_retried(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json", rules=[{"match": {"attempt": 1}, "behavior": "slow_ok"}], slow_s=20))
    monkeypatch.setattr("aidoc.pipeline.RETRY_POLICY", RetryPolicy(backoff=(0.1, 0.1)))
    cfg = load_config(); store = Store(tmp_root / "data" / "aidoc.db")
    src = tmp_root / "a.pdf"; shutil.copy(fixtures / "text.pdf", src)
    job = store.create_job(ConvertOptions(output_dir=tmp_root / "out"), "cli")
    tid, _ = store.create_task(job, str(src), file_sha256(src), 1, 1.0, "cht", str(tmp_root / "out" / "a"))
    def killer():
        while not (store.get_task(tid) or {}).get("pid"): time.sleep(0.05)
        time.sleep(0.5); procs.kill_tree(store.get_task(tid)["pid"])
    threading.Thread(target=killer, daemon=True).start()
    assert run_task(store, tid, get_engines(cfg), cfg) == TaskStatus.done
    t = store.get_task(tid); assert t["tried"][0]["error_kind"] == "transient" and t["attempt"] == 2 and t["engine"] == "docling"
```

```python
# tests/reliability/test_write_failure.py
import shutil
from aidoc import fsops
from aidoc.config import load_config
from aidoc.engines.registry import get_engines
from aidoc.models import ConvertOptions, TaskStatus
from aidoc.names import file_sha256
from aidoc.pipeline import run_task
from aidoc.recovery import recover_on_startup
from aidoc.store import Store
from tests.fakes.scenario import write_scenario, fake_env

def _task(tmp_root, fixtures, store):
    src = tmp_root / "a.pdf"; shutil.copy(fixtures / "text.pdf", src)
    job = store.create_job(ConvertOptions(output_dir=tmp_root / "out"), "cli")
    return store.create_task(job, str(src), file_sha256(src), 1, 1.0, "cht", str(tmp_root / "out" / "a"))[0]

def test_failure_during_write_leaves_no_final_dir(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json")); cfg = load_config(); store = Store(tmp_root / "data" / "aidoc.db")
    tid = _task(tmp_root, fixtures, store)
    orig = fsops.atomic_write_json
    def boom(path, obj):
        if path.name.endswith(".json") and ".tmp" in str(path): raise OSError(28, "No space left on device")
        return orig(path, obj)
    monkeypatch.setattr("aidoc.output.fsops.atomic_write_json", boom)
    assert run_task(store, tid, get_engines(cfg), cfg) == TaskStatus.failed
    assert not (tmp_root / "out" / "a").exists()
    assert store.get_task(tid)["error_kind"] in ("transient", "input")
    res = recover_on_startup(store, cfg, log=lambda s: None)
    assert not (tmp_root / "out" / ".tmp" / tid).exists()

def test_finalize_busy_is_transient(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json")); cfg = load_config(); store = Store(tmp_root / "data" / "aidoc.db")
    tid = _task(tmp_root, fixtures, store)
    monkeypatch.setattr("aidoc.output.fsops.replace_dir_three_step", lambda *a, **k: (_ for _ in ()).throw(fsops.FsBusyError(16, "busy")))
    assert run_task(store, tid, get_engines(cfg), cfg) == TaskStatus.failed
    assert store.get_task(tid)["error_kind"] == "transient" and not (tmp_root / "out" / "a").exists()
```

```python
# tests/reliability/test_force_open_handle.py
import shutil, sys, threading, time, json
import pytest
from aidoc.cli import main
from tests.fakes.scenario import write_scenario, fake_env
pytestmark = pytest.mark.skipif(sys.platform != "win32", reason="Windows share-mode semantics")

def test_force_with_open_handle_succeeds_after_backoff(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json"))
    src = tmp_root / "a.pdf"; shutil.copy(fixtures / "text.pdf", src); out = tmp_root / "out"
    assert main(["convert", str(src), "-o", str(out)]) == 0
    fh = open(out / "a" / "a.md", "r"); threading.Timer(0.6, fh.close).start()
    assert main(["convert", str(src), "-o", str(out), "--force"]) == 0
    assert not (out / ".trash").exists() or not any((out / ".trash").iterdir())
    assert (out / "a" / "a.md").exists()

def test_force_with_long_held_handle_is_transient(tmp_root, fixtures, monkeypatch):
    fake_env(monkeypatch, write_scenario(tmp_root / "sc.json"))
    monkeypatch.setattr("aidoc.fsops.time.sleep", lambda s: None)
    src = tmp_root / "a.pdf"; shutil.copy(fixtures / "text.pdf", src); out = tmp_root / "out"
    assert main(["convert", str(src), "-o", str(out)]) == 0
    with open(out / "a" / "a.md", "r"):
        assert main(["convert", str(src), "-o", str(out), "--force", "--json"]) == 1
    from aidoc.store import Store
    t = Store(tmp_root / "data" / "aidoc.db").list_tasks()[0]
    assert t["error_kind"] == "transient" and (out / "a" / "a.md").exists()
    assert not (out / ".trash").exists() or not any((out / ".trash").iterdir())
```

`test_source_guard.py`: same as `test_pipeline_source_changed` but through `main(["batch", ...])`, asserting the manifest `error` starts with `input: source_missing` / `input: source_changed`.

- [x] **Step 2: Run** `uv run pytest tests/reliability -v` → all pass (fix pipeline/recovery bugs they expose; these tests are the spec's §10 reliability list). **Step 3: Commit** `test: reliability suite with fake engine`.

---

### Task 10: RAG chunking and `aidoc chunk`

**Files:**
- Create: `src/aidoc/chunk.py`, `tests/unit/test_chunk.py`
- Modify: `src/aidoc/cli.py`

**Interfaces:**
- Produces: `@dataclass Chunk(id: str, source: str, heading_path: list[str], page_start: int | None, page_end: int | None, text: str, oversized: bool = False)` with `.to_json()` (omit `oversized` when False); `count_tokens(text) -> int` (tiktoken `cl100k_base`, `TIKTOKEN_CACHE_DIR = data/tiktoken`); `chunk_markdown(md: str, source: str, max_tokens: int = 800, counter=count_tokens) -> list[Chunk]`; `chunk_output_dir(out_root: Path, max_tokens: int = 800, only: str | None = None) -> Path` (iterates `list_output_dirs`, reads `<stem>.md`, `source` from the sidecar; writes `out_root/chunks.jsonl` atomically; returns path); CLI `aidoc chunk <out_dir> [--max-tokens 800] [--doc STEM]` prints `wrote N chunks to <path>`.

Algorithm: tokenise into blocks — page marker lines (update `page`), ATX headings (`#{1,6} `), fenced code (``` … ```), GFM table (consecutive lines starting with `|`), HTML table (`<table` … `</table>`), display math (`$$` … `$$`), paragraphs (blank-line separated). Heading changes the `heading_path` (stack by level; deeper levels appended, same/higher levels pop). Accumulate blocks into the current chunk while `tokens(chunk + block) <= max_tokens` and the heading path is unchanged; otherwise flush. A single block larger than `max_tokens` becomes its own chunk with `oversized=True` (tables/math/code never split; long paragraphs likewise flagged). `page_start/page_end` = min/max page of the blocks (None when the document has no markers). Chunk text = blocks joined by `\n\n`, prefixed by nothing (heading text is in `heading_path`). `id = f"{stem}#{n:04d}"`.

- [x] **Step 1: Failing tests**

```python
# tests/unit/test_chunk.py
import json
from aidoc.chunk import chunk_markdown, chunk_output_dir, Chunk

words = lambda t: len(t.split())   # deterministic counter for tests
MD = """<!-- page: 1 -->
# Title
intro para one two three
## Sec A
para a1 word word word
<!-- page: 2 -->
para a2 word word word word word
| h1 | h2 |
| --- | --- |
| 1 | 2 |
## Sec B
$$
x = 1
$$
para b
"""

def test_heading_paths_and_pages():
    cs = chunk_markdown(MD, "doc.pdf", max_tokens=8, counter=words)
    assert cs[0].heading_path == ["Title"] and cs[0].page_start == 1 and cs[0].text == "intro para one two three"
    a = [c for c in cs if c.heading_path == ["Title", "Sec A"]]
    assert a[0].page_start == 1 and a[-1].page_end == 2
    assert any("| h1 | h2 |" in c.text for c in a)
    b = [c for c in cs if c.heading_path == ["Title", "Sec B"]]
    assert b[0].text.startswith("$$") and b[0].page_start == 2

def test_table_never_split_and_oversized():
    table = "| a | b |\n| --- | --- |\n" + "\n".join(f"| {i} | {i} |" for i in range(50))
    cs = chunk_markdown("# T\n" + table, "d", max_tokens=10, counter=words)
    assert len(cs) == 1 and cs[0].oversized and cs[0].text == table

def test_no_pages_and_no_headings():
    cs = chunk_markdown("just text\n\nmore text", "d", max_tokens=100, counter=words)
    assert cs[0].page_start is None and cs[0].heading_path == [] and cs[0].to_json().get("oversized") is None

def test_ids_and_json():
    cs = chunk_markdown(MD, "doc.pdf", max_tokens=8, counter=words)
    assert cs[0].id == "doc#0000" and cs[1].id == "doc#0001"
    assert set(cs[0].to_json()) == {"id", "source", "heading_path", "page_start", "page_end", "text"}

def test_chunk_output_dir(tmp_path):
    d = tmp_path / "a"; d.mkdir(); (d / "a.md").write_text(MD, encoding="utf-8"); (d / "a.json").write_text(json.dumps({"source": "a.pdf"}))
    (tmp_path / ".tmp").mkdir()
    p = chunk_output_dir(tmp_path, max_tokens=800, counter=words)
    lines = [json.loads(l) for l in p.read_text(encoding="utf-8").splitlines()]
    assert p.name == "chunks.jsonl" and lines[0]["source"] == "a.pdf" and not (tmp_path / ".chunks.jsonl.tmp").exists()
```

(`chunk_output_dir` accepts `counter` too, defaulting to `count_tokens`.)

- [x] **Step 2: Run** → FAIL. **Step 3: Implement**. **Step 4: Run** → pass; then a manual run `uv run aidoc chunk out --max-tokens 800` on the P1 output must print a count and produce valid JSONL (this exercises real tiktoken; requires `aidoc setup` to have cached the BPE or network). **Step 5: Commit** `feat: RAG chunking and aidoc chunk`.

---

### Task 11: Claude Code skill

**Files:**
- Create: `skill/SKILL.md`

- [x] **Step 1: Write the skill**

```markdown
---
name: aidoc-convert
description: Convert a document (PDF, scanned PDF, image, docx/pptx/xlsx, html, epub...) into AI-friendly Markdown with the aidoc CLI. Use when the user says 「轉成 markdown」「把這份文件轉給 AI 讀」「convert this doc」「幫我讀這個 PDF」 or asks to read/summarise a non-text file.
---

# aidoc convert

All routing/engine decisions live in the CLI. This skill only calls it and reads the result.

## Steps
1. Resolve the file path the user means. If it does not exist, ask.
2. Run (from the ai-friendly-doc checkout, or with `AIDOC_ROOT` pointing to it):
   `uv run --project <AIDOC_ROOT> aidoc convert "<file>" -o "<AIDOC_ROOT>/out" --json`
   - Add `--lang en` only if the user says the document is English-only.
   - Add `--force` only if the user asks to re-convert.
   - Add `--allow-online-audio` only if the user explicitly accepts sending audio to Google.
3. Parse the JSON on stdout. Exit code 1 → report `error_kind`/`error_msg` verbatim and stop. If the message says `run aidoc setup <engine>`, tell the user that command.
4. Read `<output_dir>/<stem>.md` (path = the `output_dir` field). Use it to answer the user's request.
5. If `quality.level == "low"`, say so up front and list `quality.reasons` (e.g. `chars_per_page`, `garbage_ratio`, `missing_table`) so the user knows the text may be incomplete. Mention `tried` (which engines were attempted).
6. Never paste the whole Markdown back unless asked; summarise or quote the relevant parts. Page numbers are available from `<!-- page: N -->` markers — cite them.

## Notes
- Large PDFs are converted in 40-page segments; an interrupted run resumes when the same command is re-run.
- Already-converted files (same sha256) return `status: "skipped"` immediately; the `.md` is still valid.
```

- [x] **Step 2: Verify** by reading it once through the checklist: no engine logic, trigger words from spec §7 present, low-quality handling present. **Step 3: Commit** `feat: Claude Code skill for aidoc convert`.

---

### Task 12: Slow integration — real 45-page segmentation

**Files:**
- Create: `tests/integration/test_segments_real.py`
- Modify: `tests/integration/test_e2e_fixtures.py` (`test_big_pdf_whole_document_p1` now expects `segments == 2`)

- [x] **Step 1: Test**

```python
# tests/integration/test_segments_real.py
import json, shutil, pytest
from aidoc.cli import main
pytestmark = pytest.mark.slow

def test_big_pdf_segmented_with_docling(tmp_root, fixtures):
    src = tmp_root / "big.pdf"; shutil.copy(fixtures / "big.pdf", src)
    assert main(["convert", str(src), "-o", str(tmp_root / "out"), "--json"]) == 0
    sc = json.loads((tmp_root / "out" / "big" / "big.json").read_text(encoding="utf-8"))
    assert sc["segments"] == 2 and sc["pages"] == 45 and sc["quality"]["level"] == "ok" and sc["engine"] == "docling"
    md = (tmp_root / "out" / "big" / "big.md").read_text(encoding="utf-8")
    assert "<!-- page: 41 -->" in md and "第 45 頁" in md and md.count("<!-- page: ") == 45

def test_big_pdf_segmented_with_mineru_forced(tmp_root, fixtures):
    src = tmp_root / "big.pdf"; shutil.copy(fixtures / "big.pdf", src)
    assert main(["convert", str(src), "-o", str(tmp_root / "out"), "--engine", "mineru", "--json"]) == 0
    sc = json.loads((tmp_root / "out" / "big" / "big.json").read_text(encoding="utf-8"))
    assert sc["segments"] == 2 and sc["quality"]["level"] == "ok"
```

- [x] **Step 2: Run** `uv run pytest -m slow tests/integration/test_segments_real.py -v` → pass; observe in the log that only one runner process was started per engine (log the pid once in `pipeline` when a session opens). **Step 3: Commit** `test: real segmented conversion`.

---

## Phase acceptance (P2)

- [x] `uv run pytest -m "not slow" -q` → all pass, including `tests/reliability/` (Windows-only tests run on the host).
- [x] `uv run pytest -m slow -q` → all pass.
- [x] Manual interruption drill: start `uv run aidoc batch tests/fixtures -o out_p2 --engine docling` (real engine), wait until `data/aidoc.db` shows `big.pdf` segment 0 `done`, press Ctrl+C, confirm `tasklist | findstr python` shows the docling runner is gone within a few seconds (or that re-running kills it: the recovery log line `killed orphan runner pid=…` appears), re-run the same command → finishes, and `out_p2/big/big.json` has `"segments": 2`; `tasks.attempt` for `big.pdf` shows the session count.
- [x] `uv run aidoc chunk out_p2 --max-tokens 600` → `wrote N chunks to out_p2\chunks.jsonl`; every line has `heading_path`, `page_start`, `page_end`; the ruled table from `text.pdf` appears intact in exactly one chunk.
- [x] Delete `out_p2/text/` manually, run `uv run aidoc convert tests/fixtures/text.pdf -o out_p2` → re-converts (no false cache hit), and the previous `documents` row was marked `orphaned` then replaced.
- [x] `skill/SKILL.md` exists; simulate the skill by running its command line by hand on `tests/fixtures/sample.docx` and reading the JSON.
- [x] Spec §1 success criterion "resume from interruption without half-written output" holds: after the drill above, `out_p2/.tmp/` and `out_p2/.trash/` are empty or absent.

---

## Implementation notes / deviations

Recorded while executing P2 on 2026-09-24 (same host as P1), commits `06db0dd..HEAD` on master. The decisions are
listed below; every item that affects behaviour has a test.

**Where the plan was adapted (spec-faithful, no test weakened)**
- T2: `FakeEngine.engine_opts` and `oom_downgrade` now delegate to the real engine class and add `fake_engine`.
  This lets the T5 OOM test see `tier` and `page_batch_size` exactly as production does (index A9: fakes follow the
  real engines' rules). The `RunnerSession` host runs with cwd `data/work`, since one process serves many segment
  workdirs and all request paths are absolute.
- T4 / final review: several watcher threads share one Store, so `Store` now wraps its connection in
  `_LockedConnection`. Every statement runs and is fully fetched under one RLock. My first ruling
  (`check_same_thread=False` without a lock) was wrong: concurrent use raised `sqlite3.InterfaceError`, which made a
  cancel test flaky. P3 can keep or rename the `threadsafe` flag; the lock is always on.
- T4: two problems in the plan's `watch()` helpers were fixed, with the assertions unchanged:
  - they read `list_segments(tid)[0]` before any rows existed, so an `IndexError` killed the thread and the test hung;
  - they could cancel before the slow segment-1 call was logged, a race on the `[41, 45]` count.
- T4: the per-segment quick check runs only when all three hold:
  - the document has more than one segment;
  - a fallback engine is still left;
  - `has_table_lines` is treated as false for the segment, because it is a whole-document fact and a segment with no
    table must not fail on it.

  A single-segment document keeps P1's best-of. The last engine always finishes, and the whole-document check then
  decides the result (`low` rather than `failed`).
- T4: a PDF with only one segment is not split. It is sent with `pages=None`, which spec §3 defines as the whole
  document.
- T4: resume reuses done segments only if the same engine produced them. `edges.json` records the engine (new field
  `SegmentPart.engine`), and the engine order restarts at that engine. When a low best-of candidate is kept, its
  assets are copied to `work/<task>/best/` before the segment dirs are discarded.
  Store methods added (all additive): `delete_segments`, `get_segment`, `update_document`, `tasks_with_pid`,
  `list_tasks(status_in=)`, `requeue_task`.
- T6: the source is staged before probe and cache lookup, so `source_missing` and `source_changed` are caught on
  every run.
  - The probe reads the work copy.
  - A cache hit deletes a work dir that no document references.
  - `documents.work_copy_path` is the work dir (the P3 plan `rmtree`s it).
  - An already staged copy with the right sha is reused even if the original has since moved.
  - A read-only source is copied, never hardlinked (see final review I2).
  - Any other `OSError` becomes `input: source_unreadable`.
- T7: recovery leaves alone any task whose liveness lock is held by another running CLI (see verifier 4). Spec §8.4
  assumed a single process; without this, a second CLI would kill the first one's runner.
  - Output roots are de-duplicated by realpath.
  - Recovery runs at the start of `aidoc convert` and `aidoc batch`, and writes to stderr only when it acted.
- T8: when a row is reused, `error_kind` and `error_msg` are cleared. Without a server, `aidoc cancel` prints
  "no server running; use Ctrl+C …" to stderr and exits 1.
- T9: `db(root)` is a fixture factory, and `batch_proc` logs to `<root>/batch<n>.out` / `.err`. I added
  `test_orphan_runner_of_killed_batch_is_killed_by_next_start`: the plan's test kills the whole process tree, so
  nothing else tested orphan killing end to end.
- T10: chunking rules:
  - a page marker ends a paragraph;
  - a block that spans a marker (HTML table, `$$`) records both pages;
  - fenced code is kept verbatim;
  - chunk ids use the output dir name;
  - `--doc X` replaces only X's lines in `chunks.jsonl`. This replaces my earlier ruling after final review M3.
- T11: step 4 of SKILL.md names the actual Markdown path, `<output_dir>/<dir name>.md`. P1 names files after the
  output dir, including `-sha8` collision suffixes.
- T12: Docling treats the top line of each page ("第 N 頁 / Page N") as page-header furniture and drops it on every
  page, as it did in P1. The Docling test therefore checks 45 page bodies instead of "第 45 頁"; the MinerU test keeps
  that check. The log line `runner started pid=` confirms one runner per engine. `test_big_pdf_whole_document_p1`
  now expects `segments == 2`.

**P1 deferred items fixed (with tests)**
- `.tmp.pdf`, `.trash.pdf` and hidden stems: a stem never starts with `.` (so `.tmp` becomes `_tmp`) and never
  equals `.tmp`, `.trash`, `_manifest.jsonl` or `chunks.jsonl`.
- Reserved names now include `CONIN$`, `CONOUT$`, `COM0-9`, `COM¹²³`, `LPT0-9` and `LPT¹²³`, also when followed by
  spaces before the dot.
- Fake `slow_ok` sleeps `delay_s` (index §6). `slow_s` overrides it for `slow_ok` only. There is no hidden extra 5 s.
- An unknown `AIDOC_ERROR` kind raises `EngineError(engine, "[runner error kind 'x'] …")` instead of `ValueError`.

**P1 verifier findings fixed in P2 (each with a failing test first)**
1. Unreadable or vanished inputs (`batch.register_source`):
   - an `OSError` while statting or hashing creates a task `failed(input: source_unreadable | source_missing)`;
   - that row uses a placeholder key `"!" + 63 hex` built from path, job and code, not a content sha;
   - the batch continues and exits 2;
   - if collecting inputs crashes, the tasks already created are marked failed and the job `done` before the error
     is re-raised;
   - `aidoc convert` uses the same helper: clean JSON or line output, exit 1.
2. Long stems: `MAX_STEM = 150`, and `names.output_stem()` truncates and appends `-<sha8>`. Any `OSError` while
   writing or finalising output removes `out/.tmp/<task>` and ends as `failed(transient: output write failed …)`.
3. `probe.blank_pages` is now exact: a cheap text/image/drawing check runs on every page. The sampled pages still
   drive the other metrics.
4. Concurrent converts of one file: a per-task OS file lock `data/work/.locks/<task_id>.lock` (`aidoc.tasklock`,
   msvcrt/flock), which the OS releases when the process dies, so no PID is trusted.
   - `Store.create_task` raises `TaskBusyError` for an active row whose lock is held; the new request ends as
     `failed(input: already_converting …)`.
   - `run_task` never runs a task that is locked elsewhere.
   - Recovery skips live tasks.
5. `lang=en` with RapidOCR: a check in the docling env showed that `RapidOcrOptions(lang=["en"])` and
   `["chinese_cht"]` load the same multilingual `PP-OCRv6_rec_small`, plus the same det and cls models. English-only
   recognisers can only be used by pinning model paths, and they gave no measurable gain.
   - **Spec §13.2 revised (needs spec-owner sign-off):** English scans go to Docling first only when
     `docling_ocr = "easyocr"`. With the default RapidOCR, `lang=en` routes like `cht`.
   - Spec line 28 and a spike B addendum record the evidence.
   - P1's `test_scanned_en_routes_to_docling_with_lang_en` now runs with a temporary `docling_ocr = "easyocr"`
     config; the new slow test `tests/integration/test_lang_en_real.py` covers the default.
6. Garbage ratio counts only:
   - U+FFFD, private-use code points, C0/C1 controls and unassigned code points;
   - mojibake runs: at least 4 characters over a Latin-1/cp1252 alphabet, containing a Latin-1 symbol.

   Stars, arrows, bullets, leader dots, circled digits and umlauts are not garbage.
7. No code change was needed. P3 can reach work-copy retention through `sources.purge_expired_work_copies(store, now)`,
   which recovery step 5 also calls. Work dirs of failed or forced runs are left for P3 maintenance.
8. Fixed the vacuous `A and B or C` assertion in `test_fsops`.
9. A1 tests: the clock starts at `AIDOC_READY` (tested with the fake's `startup_delay_s`), and no READY within the
   startup cap gives a transient error with the runner killed.
10. Added a batch test with a per-file I/O error (see 1).

**Final whole-branch review (fresh reviewer): fixed**
- I1: a CUDA OOM raised inside a live runner used to arrive as `kind=engine`, so the OOM downgrade never ran with the
  real runners. `_proto.serve` now reports OOM exceptions as `kind=transient`, with the OOM marker kept in the
  message, and the host sets `EngineError.oom`. The protocol shape is unchanged. New fake behaviour: `oom_raise`.
- I2: removing a hardlinked work copy chmod-ed away the read-only flag of the user's source, because NTFS attributes
  are shared between links. Read-only sources are now copied instead of linked, and `_on_rm_error` never chmods a
  file with more than one link.
- M3 (re-graded to Important): `aidoc chunk --doc X` used to delete every other document's chunks.
- M7 (re-graded to Important): a source locked at run time ended as `engine: internal error`; it is now
  `input: source_unreadable`.
- M1: the cancel-during-backoff test now proves Review Focus 4's 1 s requirement.
- Store thread-safety: see the T4 entry above.

**Deferred minors (from the final review)**
- M2: resume ignores a changed `lang` or other options between runs (only matters with EasyOCR).
- M4: recovery could kill another process's live runner if a dead task's recorded PID was reused by exactly such a
  runner.
- M5: the runner starts a moment before its PID is recorded.
- M6: a missing env python, or a runner that crashes at startup, ends as an internal error or three transient
  startups instead of falling back at once.
- M8: a `fail` retry decision leaves the in-flight segment `converting`.
- M9: work-copy retention keeps the whole `data/work/<task>`, not only `src.*`.
- M10: `MAX_STEM = 150` can still exceed MAX_PATH without LongPathsEnabled (setup already warns about this).
- M11: placeholder failed rows pile up on reruns over an unreadable file.
- M12: Ctrl+C in `convert` or `batch` prints a traceback; recovery handles the leftover state.
- M13: a form feed counts as garbage (the effect is negligible).

**Also found**
- `segment.py` imported `fitz`. Its deprecation warning went to **stdout** and broke `aidoc convert --json`. It now
  imports `pymupdf as fitz`, like `probe.py`.

**Acceptance evidence (2026-09-24)**
- `uv run pytest -m "not slow" -q`: 220 passed (211 unit + 9 reliability). `uv run pytest -m slow -q`: 30 passed and
  1 skipped (the manual-fixtures test, because `tests/fixtures/manual/` is empty).
- Interruption drill (real Docling, `out_p2`):
  - Ctrl+Break was sent to the batch's process group once segment 0 of big.pdf was `done`. The runner was gone
    within 3 s, and the DB showed `converting [(0, done), (1, converting)]`.
  - The re-run logged `recovery: requeued 1 interrupted task(s)` and finished big.pdf with `segments: 2`, 45 pages
    and level ok. `tasks.attempt = 2` (two sessions), and `out_p2/.tmp` and `.trash` were absent.
  - Exit code 2 is expected: bad.exe, corrupt, encrypted and make_fixtures.py fail under `--engine docling`.
- `aidoc chunk out_p2 --max-tokens 600` printed `wrote 28 chunks to out_p2\chunks.jsonl`. Every row has
  `heading_path`, `page_start` and `page_end`. The ruled table from text.pdf is intact in exactly one chunk
  (`text#0001`, pages 2–3).
- After deleting `out_p2/text/`, `aidoc convert tests/fixtures/text.pdf -o out_p2` printed
  `recovery: marked 1 document(s) orphaned` and re-converted (`done docling 1.0`); the documents row went back to
  `ok`.
- Running the skill's command on `sample.docx` returned JSON that parsed, and the Markdown was read from
  `<output_dir>/<dir name>.md`.
