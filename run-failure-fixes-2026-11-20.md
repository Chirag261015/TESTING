# Run failure fixes — 20 Nov 2026

Three small code changes for Advise Workbench (`dev-master`). Please apply them to your own copy of the repo and restart your backend.

| # | Error you see on the run | File to change |
|---|---|---|
| 1 | "Check the drafts before using them … Reason the checks stopped: consuming input failed: server closed the connection unexpectedly" | `src/workflows/checkpoint.py` |
| 2 | "Check the drafts before using them … Reason the checks stopped: Generation was interrupted by a server restart" | `src/workflows/orchestrator.py` (+ a test) |
| 3 | Backend console: "Failed to export metrics batch due to timeout" | `src/core/observability/otel_tracing.py` |

**Before you edit:** your backend runs with `uvicorn --reload`. On Windows, saving a file under `src/` sends Ctrl+C to the whole `run_agent` console, which stops the backend, worker and frontend. Make the edits, then restart with `scripts\run_agent.cmd`.

---

## 1. Checkpoint save fails: "server closed the connection unexpectedly"

**Why it happens**

At the end of generation, the run saves a LangGraph checkpoint to PostgreSQL. The checkpointer uses its own small connection pool, separate from the app's main database connections. That pool has no TCP keepalives and doesn't check connections before using them. During generation (10–35 minutes) its connection sits idle, and the network between our machines and the shared database drops it. The next checkpoint write then fails, and the whole run is marked failed even though the PPTX/DOCX was already produced.

Evidence: 8 runs failed this way on 19–20 Nov (Priyanka 5, Chirag 2, Debarchana 1). The stored traceback ends in `langgraph/checkpoint/postgres/__init__.py put → cur.executemany(UPSERT_CHECKPOINT_BLOBS_SQL)` with `psycopg.OperationalError`. The PostgreSQL server never crashed. Its log shows the connection being reset from the client machine in the same second the run failed. The app's main connections (`src/core/db/session.py`) already have keepalives for exactly this reason; the checkpoint pool was missed.

**Change** — `src/workflows/checkpoint.py`, in `get_checkpointer()`:

```diff
             url = settings.database_url.replace("postgresql+psycopg://", "postgresql://")
             # A small pool: run worker threads checkpoint at the same time.
-            pool = ConnectionPool(url, min_size=1, max_size=5, kwargs={"autocommit": True, "prepare_threshold": 0, "row_factory": dict_row})
+            # Keepalives and a check on checkout: a run leaves this pool idle through long LLM phases, and the
+            # network drops idle connections, so the next checkpoint write failed with "server closed the connection".
+            pool = ConnectionPool(
+                url,
+                min_size=1,
+                max_size=5,
+                check=ConnectionPool.check_connection,
+                kwargs={
+                    "autocommit": True,
+                    "prepare_threshold": 0,
+                    "row_factory": dict_row,
+                    "keepalives": 1,
+                    "keepalives_idle": 30,
+                    "keepalives_interval": 10,
+                    "keepalives_count": 5,
+                },
+            )
```

`check=` needs `psycopg_pool` 3.2 or newer. The repo's venv has 3.3.3. Check yours with:

```
.venv\Scripts\python.exe -c "import psycopg_pool; print(psycopg_pool.__version__)"
```

**Status:** not applied or tested yet on any machine. After you apply it, generate one deck and check that the run finishes and gets past design review.

---

## 2. Run fails while waiting in design review: "interrupted by a server restart"

**Why it happens**

After the checks, a run waits in design review (`hitl_review`) for up to an hour. While waiting it stays `status="running"` and logs nothing. When any backend connected to the shared database starts or reloads, it marks every "running" run that has been quiet for 10 minutes as orphaned and fails it. That includes runs that are alive and waiting on someone else's machine. The stuck-run watchdog already skips design-review runs; the restart check didn't.

**Change** — `src/workflows/orchestrator.py`

Add this function just above `def settle_orphan_suspects()`:

```python
def _design_review_wait_sec() -> int:
    from src.workflows.phase5_hitl_review import MAX_WAIT_SECONDS  # local: phase5 imports this module

    return MAX_WAIT_SECONDS
```

Inside `settle_orphan_suspects()`, add the second `if` between the two existing lines shown:

```diff
                 ref = _last_run_activity(session, run)
                 if ref is not None and ref > cutoff:
                     continue
+                if run.phase_5_status == "pending" and ref is not None and ref > utcnow() - timedelta(seconds=_design_review_wait_sec()):
+                    # Design review logs nothing while it waits, and may be waiting on another server that shares
+                    # this database: keep checking each tick, and only treat it as orphaned once the review would have timed out.
+                    continue
                 if not _resume_from_checkpoint(session, run):
```

`utcnow` and `timedelta` are already imported in that file.

**Test** — append to `tests/unit/test_run_stuck_recovery.py`:

```python
def test_startup_reconcile_spares_design_review_until_its_timeout(monkeypatch: pytest.MonkeyPatch) -> None:
    init_db()
    monkeypatch.setattr(settings, "run_queue_startup_reconcile", True)
    monkeypatch.setattr(settings, "run_orphan_idle_sec", 300)
    monkeypatch.setattr(run_worker, "enqueue_run_execution", lambda *a, **k: True)
    monkeypatch.setattr(run_worker, "_resume_from_checkpoint", lambda *a, **k: False)  # e.g. the checkpoint is on another server
    session = SessionLocal()
    try:
        session.add(User(id="u_review2", email="review2@test.local", hashed_password="x"))
        session.add(Project(id="p_review2", name="ReviewRestart", created_by="u_review2"))
        _seed_run(session, run_id="r_review_restart", project_id="p_review2", status="running", last_event_age_sec=1200)
        session.flush()
        session.get(Run, "r_review_restart").phase_5_status = "pending"
        session.commit()
    finally:
        session.close()

    run_worker.reconcile_stalled_approved_runs_on_startup()
    verify = SessionLocal()
    try:
        assert verify.get(Run, "r_review_restart").status == "running"
    finally:
        verify.close()

    # Once the review would have timed out, a later watchdog tick treats it as orphaned.
    monkeypatch.setattr(run_worker, "_design_review_wait_sec", lambda: 600)
    run_worker.settle_orphan_suspects()
    verify = SessionLocal()
    try:
        assert verify.get(Run, "r_review_restart").status == "failed"
    finally:
        verify.close()
```

Run: `.venv\Scripts\python.exe -m pytest tests/unit/test_run_stuck_recovery.py -q`. On the shared server all 9 tests pass.

**Status:** applied and tested on the shared server (172.31.65.20).

---

## 3. "Failed to export metrics batch due to timeout"

**Why it happens**

The backend sends metrics directly to OpenObserve with a 5-second timeout. OpenObserve's write-ahead log sometimes takes up to ~5 seconds to write, so some exports time out and those metric points are dropped. Runs and users are not affected; it's log noise and small gaps in the dashboards.

**Change** — `src/core/observability/otel_tracing.py`, in `_init_openobserve_metrics_and_logs()`:

```diff
             certificate_file=_ca_file(),
-            timeout=5  # 5 second timeout for metric exports
+            timeout=10,  # OpenObserve WAL writes have been seen taking ~5s; the SDK default leaves headroom
         ),
```

**Status:** applied on the shared server.

---

## After applying

1. Run the tests: `.venv\Scripts\python.exe -m pytest tests/unit/test_run_stuck_recovery.py -q`
2. Restart: `scripts\run_agent.cmd`
3. Generate one deck end to end and approve it in design review.

Every backend shares one database, so fix 2 only protects your runs fully once **every** teammate's backend has it.
