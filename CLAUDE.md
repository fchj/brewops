# BrewOps

Telemetry app for the office coffee machines. Python with FastAPI, data in SQLite.

## Architecture

One process, four layers, data flows one way: **ingest → db → api → frontend**.

- `src/brewops/ingest/` — CSV loader (`loader.py`) + CLI entry points (`cli.py`). Parses files from
  an inbox folder, validates rows, writes into SQLite via the same functions the API uses.
- `src/brewops/db/` — SQLite access, stdlib `sqlite3`, no ORM.
  - `schema.py` — table definitions and reference data (`machines`, `drink_types`).
  - `queries.py` — all reads/writes go through here; nothing else touches SQL directly.
  - `connection.py` — connects to `$BREWOPS_DB` (default `./brewops.db`).
- `src/brewops/api/main.py` — FastAPI app exposing JSON endpoints under `/api/*`, and serves
  `src/brewops/frontend/` as static files at `/`.
- `src/brewops/frontend/` — vanilla JS/HTML/CSS, no build step, no framework. `app.js` fetches
  from `/api/*` and renders the dashboard directly into the DOM.

Two machines generate telemetry automatically (`has_telemetry = true`); "Old Faithful" has no
sensor and is logged by hand — see ingestion paths below.

## Ingestion paths

Both paths write into the same `brew_events`/`maintenance_events` tables, distinguished by
`source` ('csv' vs 'manual'):

1. **CSV batch ingest** (`uv run ingest [path]`, default `data/inbox/`) — reads files by filename
   prefix: `brews_*.csv` (source=csv), `manual_*.csv` (source=manual, exports of the paper log
   kept next to Old Faithful), `maintenance_*.csv`. Unrecognized filenames are skipped; bad rows
   are rejected and reported, the rest of the file still loads. `uv run seed` wipes and rebuilds
   the whole DB from `data/inbox/`.
2. **Manual entry via the API/frontend** — `POST /api/brews` and `POST /api/maintenance`, used by
   the dashboard's logging forms. Always `source='manual'` for brews.

Timestamps everywhere are naive local time, stored as `'YYYY-MM-DD HH:MM:SS'`. The API accepts
`datetime-local` strings too and normalizes them; both paths reject future timestamps.

## Running and testing

```
uv run start   # serve the app at http://localhost:8123
uv run seed    # rebuild brewops.db from data/inbox/
uv run ingest [path]   # ingest one file or folder without resetting the DB
uv run pytest  # tests/, one file per layer (test_db, test_ingest, test_api, test_frontend)
```

On this machine, Defender's Attack Surface Reduction blocks the `uv run` shim with
"Adgang nægtet (os error 5)". Workaround: use the `.cmd` shims directly, e.g.
`.venv/Scripts/start.cmd`, `.venv/Scripts/seed.cmd`.

## Conventions

- All SQL lives in `db/queries.py`; the API and ingest layers call those functions rather than
  writing their own statements.
- A read that composes a UI-facing payload (e.g. `get_machine_health`) does all its joins/lookups
  in one function and returns a single merged dict — the API layer stays a thin pass-through with
  no aggregation logic of its own.
- Reference data (`machines`, `drink_types`) is seeded via `INSERT OR IGNORE` in `init_db`, so it's
  safe to call on every app startup.
- New drink types need matching entries in `schema.py` (DB), any relevant docs, and the frontend
  picks them up automatically via `/api/drink-types` — see the `add-drink-type` skill.
- Tests use a real SQLite file in `tmp_path`, not mocks — see the `conn` fixture in `tests/test_db.py`.
