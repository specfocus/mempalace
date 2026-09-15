# Design note: `/readyz` write probe (and honest integrity refusals)

**Status:** design only — no PR until Cursor on-demand / cloud-agent quota is available.  
**Repo:** `specfocus/mempalace` (fork), branch `develop`  
**Author context:** measured on 2026-09-10 against WSL local palace + DigitalOcean Droplet  
**Ops companion:** [`DROPLET.md`](../DROPLET.md) (Droplet deploy notes; links here)

This note is the handoff for whoever implements the change. Execute it; do not rediscover today's outage.

---

## Why this exists

On 2026-09-10 both palaces **refused writes** while looking healthy:

| Host | Symptom | `/healthz` | Reads | Auth (no token) |
| --- | --- | --- | --- | --- |
| Droplet `64.227.5.215` | `mempalace_checkpoint` → `errors: N × [Errno 13] Permission denied: '/data/.cache'` | `ok` | worked | `401` as expected |
| Local WSL | JSON-RPC `-32002` `Palace SQLite integrity check failed; refusing tool call` | (process up) | worked | n/a |

Docker reported the Droplet container **healthy**. Nothing on the health surface moved. The next operator who only hits `/healthz` will redo this diagnosis from scratch unless the probe **names which gate refused**.

Separately: local `PRAGMA integrity_check` on `chroma.sqlite3` (and knowledge_graph / logstream) returned **`ok`** on both copies. The Droplet archive was not a corrupt-but-readable DB. The startup gate uses `PRAGMA quick_check` (see `mcp_server.py` / `repair.sqlite_integrity_status`) and can refuse for reasons that are **not** "the database is corrupt" — yet the operator message asserts integrity failure and points at offline repair. That misdiagnosis cost (or would cost) hours.

---

## What `/healthz` means today (do not reinterpret)

Current `/healthz` is **liveness only**: the HTTP process is bound and can answer. It does **not** mean:

- a write would succeed
- the embedding cache / data volume is writable
- the SQLite / FTS5 integrity gate would allow mutating tools
- the peer-writer lock is held
- embeddings are available

Keep `/healthz` as cheap liveness. Do **not** overload it into a write guarantee without an explicit versioned contract change and docs update. Prefer a separate **`/readyz`**.

---

## Goal: `/readyz` answers "would a write succeed right now?"

### Contract

- **Unauthenticated**, like `/healthz` (ops / load balancers / compose healthchecks).
- **HTTP 200** only when every gate a real mutating tool (`mempalace_checkpoint`, `mempalace_add_drawer`, …) would pass **before** creating a lasting drawer.
- **HTTP 503** (preferred) when any gate refuses, with a **machine-readable body that names the failing gate** and an operator-actionable hint.
- **Must not write drawers** (no probe residue in the palace). A probe that leaves memories is worse than no probe.
- **Must not** download embedding models on every poll if avoidable.

### Suggested JSON body (illustrative)

Passing:

```json
{
  "ok": true,
  "liveness": "ok",
  "write": { "ok": true, "gates": ["data_writable", "cache_writable", "sqlite_integrity", "writer_lock"] }
}
```

Failing (Droplet-class):

```json
{
  "ok": false,
  "liveness": "ok",
  "write": {
    "ok": false,
    "failed_gate": "cache_writable",
    "detail": "Permission denied creating or writing under /data/.cache (HOME)",
    "hint": "Host bind-mount parent must be writable by container uid (often 1000). On the Droplet: chown -R 1000:1000 /opt/mempalace/data && /opt/mempalace/fix-data-perms.sh"
  }
}
```

Failing (integrity-gate-class — honest wording):

```json
{
  "ok": false,
  "liveness": "ok",
  "write": {
    "ok": false,
    "failed_gate": "sqlite_integrity",
    "detail": "startup quick_check did not pass (no clean verdict)",
    "sqlite": {
      "checked": false,
      "ok": null,
      "reason": "<from _sqlite_integrity_no_verdict_reason or probe exception>",
      "errors": []
    },
    "hint": "Do not assume DB corruption. Inspect the reason/errors fields. Run PRAGMA integrity_check / quick_check offline if a real verdict is needed; check locks/WAL and peer writers before repair."
  }
}
```

If `checked: true` and `errors` non-empty, then it *is* fair to say the check reported problems — still include the raw `errors` list, not only the word "corrupt".

---

## Gates `/readyz` must exercise (same path as a real write)

Order them so the response can stop at the **first** failure and name it. Align with what `checkpoint` / mutating tools actually hit in `mcp_server.py`:

1. **`data_writable`** — palace data root (under `HOME` / `MEMPALACE_PALACE_PATH`) is creatable/writable by the server uid.
2. **`cache_writable`** — embedding / HF / transformers cache under `HOME` (today: `/data/.cache` when `HOME=/data`) can be created and written. Today's Droplet outage was exactly this: parent `/opt/mempalace/data` was `root:root`, container uid `1000`.
3. **`sqlite_integrity`** — reuse the existing gate (`_ensure_sqlite_integrity_status` / `_sqlite_integrity_payload`), including the internal distinction already present:
   - verdict exists and clean
   - verdict exists and failed (`errors`)
   - **no verdict** (`checked: false`, `reason` / `_sqlite_integrity_no_verdict_reason`)
   - probe exception (`check_error`)
4. **`writer_lock`** — peer-writer / mine lock state (`_MCP_WRITER_READ_ONLY`, lock failed). If mutating tools would refuse because another writer holds the palace, say so explicitly.
5. **Any other pre-mutation gate** `checkpoint` uses before persisting a drawer (keep this list honest as code evolves; do not invent gates the write path does not hit).

**Explicitly out of scope for every poll:** creating a drawer, diary entry, or KG row; forcing an embedding model download.

Optional later: a deeper `/readyz?deep=1` that does a transactional write+rollback or a disposable probe key if the codebase gains one — not required for v1 if the gates above match production refusals.

---

## Companion fix: stop lying in the `-32002` message

Today `_mcp_sqlite_integrity_refusal` always says:

> `Palace SQLite integrity check failed; refusing tool call until the palace is repaired`

Internally, `sqlite_integrity_status` / `_sqlite_integrity_payload` already distinguish:

- clean verdict
- failed verdict (`errors`)
- **no verdict** (`checked: false` + `reason`)
- probe failure (`check_error`)

The refusal message collapses all of that into "integrity check failed" + a hint to repair corruption offline. That sent operators to the database when `PRAGMA integrity_check` was already `ok`.

**When implementing, change the refusal to:**

- If no verdict / probe could not run: *"SQLite integrity probe did not produce a clean verdict; refusing mutating tool call"* + include `reason` / `check_error` in `error.data`.
- If verdict failed: *"SQLite integrity probe reported errors; refusing mutating tool call"* + include `errors`.
- Never say "corrupt" / "failed integrity check" unless a real negative verdict exists.
- Hint should branch: no-verdict → check locks, WAL, peer writers, re-run probe; failed-verdict → backup + `mempalace repair` path.

This is as important as `/readyz`. A green probe with a lying refuse message still burns hours.

---

## Deploy / docs updates (same PR later)

- Remote/team server guide: document `/healthz` = liveness, `/readyz` = write readiness; show example failing bodies.
- Compose examples: healthcheck may keep `/healthz` for restart policy; optionally add a slower readiness check on `/readyz` for orchestration that supports it.
- Droplet note: after extracting palace data into a bind mount, `chown -R 1000:1000` the mount root (see `DROPLET.md` / `/opt/mempalace/fix-data-perms.sh`).

---

## Acceptance criteria

1. With a deliberately root-owned `/data` parent (uid mismatch), `/healthz` stays `ok`, `/readyz` returns **503** with `failed_gate: cache_writable` (or `data_writable`) and a chown-style hint.
2. With the integrity gate in a no-verdict or failed state, `/readyz` returns **503** with `failed_gate: sqlite_integrity` and payload fields that match `_sqlite_integrity_payload` (including `reason` when unchecked).
3. With peer-writer lock denying mutations, `/readyz` names `writer_lock`.
4. A passing `/readyz` leaves **zero** new drawers / diary entries.
5. `-32002` message no longer claims "integrity check failed" when no negative verdict exists.
6. Tests cover the cases above; remote-server docs updated.

---

## Non-goals

- Replacing `/healthz` without a transition plan.
- Requiring auth on `/readyz` for v1 (revisit if the body ever leaks secrets — it must not).
- Automatically running `mempalace repair` from the probe.

---

## Pointers in this tree

- Integrity gate + refusal: `mempalace/mcp_server.py` (`_refresh_sqlite_integrity_status`, `_mcp_sqlite_integrity_refusal`, `_SQLITE_INTEGRITY_ERROR_CODE = -32002`)
- Status helper: `mempalace/repair.py` (`sqlite_integrity_status`)
- Current liveness route: `/healthz` handler in `mcp_server.py`
- Droplet ops: `DROPLET.md`