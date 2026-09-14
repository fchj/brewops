# Implementation plan: dashboard date filter (ticket 005)

Goal: let dashboard users pick a date range; the total/per-drink/per-day numbers, the
timeline chart, and the machine cards all reflect only brews in that range. See
`tickets/005-date-filter.md`.

Scope decision (already made, don't re-litigate): the date range filters brew *activity*
numbers — `get_stats` entirely, and on machine cards: `brew_count`, `last_brew`,
`specialty`. It does **not** filter `last_maintenance` or `recent_errors` — those are
"current machine status," not "activity in the period," and should always show the
latest regardless of the selected range.

Query parameter names: `start` and `end`, both optional, both naive local timestamps in
`'YYYY-MM-DD HH:MM:SS'` or `'YYYY-MM-DDTHH:MM'` form (same formats `parse_timestamp`
already accepts in `src/brewops/api/main.py`). Missing `start` = no lower bound (all
history). Missing `end` = no upper bound (through now). Both missing = today's
unfiltered behavior, unchanged.

---

## 1. `src/brewops/db/queries.py`

### `get_stats`

Current signature: `get_stats(conn: sqlite3.Connection) -> dict[str, Any]`

New signature: `get_stats(conn: sqlite3.Connection, start: str | None = None, end: str | None = None) -> dict[str, Any]`

Change all three sub-queries to respect the range:

- **total**: add `WHERE timestamp >= ? AND timestamp <= ?` — but only include each bound
  when it's not `None`. Simplest correct approach: build the WHERE clause and param list
  conditionally, e.g.:

  ```python
  conditions = []
  params = []
  if start is not None:
      conditions.append("timestamp >= ?")
      params.append(start)
  if end is not None:
      conditions.append("timestamp <= ?")
      params.append(end)
  where_clause = f"WHERE {' AND '.join(conditions)}" if conditions else ""
  ```

  Use `where_clause` / `params` in the `total` query.

- **per_drink**: this one is a `LEFT JOIN` specifically so drink types with **zero**
  brews still appear in the result (see `test_stats_math`'s
  `assert per_drink["cappuccino"] == 0`). Do **not** add the date condition as a plain
  `WHERE` clause after the join — that would silently drop rows for drink types with
  zero brews *in range* (SQLite would exclude them because the WHERE runs after the
  join, and the LEFT JOIN's whole point is to keep drink types with no matching row).
  Put the date condition **inside the `ON` clause instead**, e.g.:

  ```sql
  SELECT dt.name, dt.label, COUNT(be.id) AS count
  FROM drink_types dt
  LEFT JOIN brew_events be
    ON be.drink_type = dt.name AND be.timestamp >= ? AND be.timestamp <= ?
  GROUP BY dt.id
  ORDER BY dt.id
  ```

  (drop whichever of the two conditions has no bound, same conditional-building
  approach as above — build the `ON` clause suffix the same way you build
  `where_clause`, just append it after `ON be.drink_type = dt.name`).

- **per_day**: same conditional `WHERE timestamp >= ? AND timestamp <= ?` as `total`.

Reuse the same `conditions`/`params` construction for `total` and `per_day` (they're
identical); build the `per_drink` ON-clause params separately since they're positioned
differently in the query.

### `get_machine_health`

Current signature: `get_machine_health(conn: sqlite3.Connection, machine_id: int) -> dict[str, Any] | None`

New signature: `get_machine_health(conn, machine_id: int, start: str | None = None, end: str | None = None) -> dict[str, Any] | None`

- The `brews` query (`COUNT(*)`, `MAX(timestamp)` — feeds `brew_count` and `last_brew`)
  needs the same conditional date bounds added to its `WHERE machine_id = ?` clause
  (`AND timestamp >= ?` / `AND timestamp <= ?` as applicable).
- The `specialty` query (`GROUP BY dt.id ORDER BY count DESC ... LIMIT 1`) needs the
  same conditional bounds added to its `WHERE be.machine_id = ?` clause.
- `last_maintenance` and `recent_errors` queries: **leave unchanged, no date params.**

---

## 2. `src/brewops/api/main.py`

### `GET /api/stats`

```python
@app.get("/api/stats")
def stats(
    start: str | None = None,
    end: str | None = None,
    conn: sqlite3.Connection = Depends(get_db),
):
    start, end = normalize_range(start, end)
    return queries.get_stats(conn, start, end)
```

FastAPI turns `start`/`end` function params into `?start=...&end=...` query params
automatically — no Pydantic model needed for a GET.

### `GET /api/machines/{machine_id}`

Same pattern — add `start: str | None = None, end: str | None = None` params, normalize,
pass through to `queries.get_machine_health(conn, machine_id, start, end)`.

### New helper: `normalize_range`

Add near `parse_timestamp` (around line 41). Reuse `parse_timestamp`'s format-parsing
logic but **without** the "reject future timestamps" rule (a future `end` should just
mean "through now," not be an error) and **without** requiring the value to be present.

```python
def normalize_range(start: str | None, end: str | None) -> tuple[str | None, str | None]:
    """Parse optional start/end query params into storage-format timestamps.
    Unlike parse_timestamp, does not reject future values and treats blank as unset."""

    def parse_bound(value: str | None) -> str | None:
        if not value:
            return None
        for fmt in TIMESTAMP_FORMATS:
            try:
                return datetime.strptime(value.strip(), fmt).strftime("%Y-%m-%d %H:%M:%S")
            except ValueError:
                continue
        raise HTTPException(400, f"unparsable timestamp {value!r}")

    start = parse_bound(start)
    end = parse_bound(end)
    if start is not None and end is not None and start > end:
        raise HTTPException(400, "start must not be after end")
    return start, end
```

Note: if the frontend sends a plain date like `2026-06-01` (from `<input type="date">`),
add `"%Y-%m-%d"` to the accepted formats, or have the frontend pad end-of-day itself
(see section 3 — recommended: frontend pads, so `end` is inclusive of the whole day).

---

## 3. Frontend

### `src/brewops/frontend/index.html`

Add a filter control inside `<section id="dashboard">`, before `.stat-tiles`:

```html
<div class="panel date-filter">
  <label for="filter-start">From</label>
  <input type="date" id="filter-start">
  <label for="filter-end">To</label>
  <input type="date" id="filter-end">
  <button type="button" id="filter-apply">Apply</button>
  <button type="button" id="filter-clear">Clear</button>
</div>
```

Plain `<input type="date">` (not `datetime-local`) — users pick whole days, not
times. Leave both inputs blank by default (= unfiltered, today's behavior).

### `src/brewops/frontend/app.js`

App.js currently has these functions (top to bottom): `fetchJSON`, `renderDrinkBars`,
`renderTimeline`, `renderMachineCards`, `loadDashboard`, `localNow`, `fillSelect`,
`setupForms`, `submitForm`, plus two bare calls at the bottom
(`loadDashboard().catch(...)`, `setupForms().catch(...)`). Below is what changes in
each — everything not listed stays untouched.

**New module-level state**, add right after the `fetchJSON` function (before
`renderDrinkBars`):

```js
let currentRange = { start: null, end: null };
```

This is what "the current filter" means for the rest of the file. It must live outside
any function so it survives across repeated `loadDashboard()` calls (including the one
`submitForm` triggers after logging a brew — the filter must still be applied to that
refresh, not silently reset to unfiltered).

**`fetchJSON`** — no change.

**`renderDrinkBars(perDrink)`** — no change. Already handles the all-zero case
(`Math.max(1, ...perDrink.map(...))` prevents divide-by-zero), so a filtered range
with no brews just renders every bar at 0 width. Verify this still holds, don't rewrite it.

**`renderTimeline(perDay)`** — no change to the rendering logic, but its existing
early-return behavior becomes the answer to "what does the chart do when the range has
zero brews": `if (perDay.length === 0) return;` at the top means the `<svg>` is cleared
(`svg.innerHTML = ""`) and nothing is drawn — the chart panel is just a blank box, no
error, no placeholder text. This is functionally correct but silent. Recommend adding
one thing here, since users applying a real filter (not just idling on page load) will
hit this: after the early return, set a simple "no brews in this range" text node into
the SVG (or a sibling element) instead of leaving it blank, e.g.:

```js
function renderTimeline(perDay) {
  const svg = document.getElementById("timeline");
  svg.innerHTML = "";
  if (perDay.length === 0) {
    const text = document.createElementNS("http://www.w3.org/2000/svg", "text");
    text.setAttribute("x", "50%");
    text.setAttribute("y", "50%");
    text.setAttribute("text-anchor", "middle");
    text.setAttribute("class", "timeline-empty");
    text.textContent = "No brews in this range";
    svg.appendChild(text);
    return;
  }
  // ...unchanged bar-drawing loop below
}
```

Add a matching `.timeline-empty` rule in `style.css` (fill color consistent with
existing muted text, check what class the codebase already uses for hint/muted text —
e.g. `.hint` in index.html — and reuse that color rather than inventing a new one).

**`renderMachineCards(healths)`** — no change. Each machine's card already renders
`m.brew_count` and `m.last_brew` directly from whatever the API returned; if a machine
had zero brews in range, `brew_count` is `0` and `last_brew` is `null`, and the existing
template already handles `null` (`m.last_brew ? m.last_brew.slice(0, 16) : "never"`).
Verify this still holds under a filtered zero-brew range, don't rewrite it.

**`loadDashboard()`** — this is the one function with real logic changes. Build a query
string from `currentRange` and append it to both fetch calls:

```js
async function loadDashboard() {
  const qs = buildRangeQuery();
  const stats = await fetchJSON("/api/stats" + qs);
  document.getElementById("total-brews").textContent = stats.total_brews;
  const lastDay = stats.per_day[stats.per_day.length - 1];
  document.getElementById("brews-today").textContent = lastDay ? lastDay.count : 0;
  renderDrinkBars(stats.per_drink);
  renderTimeline(stats.per_day);

  const machines = await fetchJSON("/api/machines");
  document.getElementById("machine-count").textContent = machines.length;
  const healths = await Promise.all(
    machines.map((m) => fetchJSON(`/api/machines/${m.id}${qs}`))
  );
  renderMachineCards(healths);
}
```

Note `/api/machines` itself (the list, not per-machine health) is *not* filtered — it's
just "which machines exist," unrelated to the date range — so that call stays as-is.

`brews-today`'s existing fallback (`lastDay ? lastDay.count : 0`) already produces `0`
when `stats.per_day` is `[]`, which is exactly the empty-range case. Verify this still
holds, don't rewrite it.

**New function `buildRangeQuery()`** — add above `loadDashboard`:

```js
function buildRangeQuery() {
  const params = new URLSearchParams();
  if (currentRange.start) params.set("start", currentRange.start);
  if (currentRange.end) params.set("end", currentRange.end);
  const qs = params.toString();
  return qs ? `?${qs}` : "";
}
```

**`localNow`, `fillSelect`** — no change.

**`setupForms()`** — no change to its existing body, but add the two filter-control
event listeners here (it's already the function responsible for wiring up dashboard
controls on load):

```js
document.getElementById("filter-apply").addEventListener("click", () => {
  const startDate = document.getElementById("filter-start").value;
  const endDate = document.getElementById("filter-end").value;
  currentRange = {
    start: startDate ? `${startDate} 00:00:00` : null,
    end: endDate ? `${endDate} 23:59:59` : null,
  };
  loadDashboard().catch((error) => console.error("Dashboard failed to load:", error));
});

document.getElementById("filter-clear").addEventListener("click", () => {
  document.getElementById("filter-start").value = "";
  document.getElementById("filter-end").value = "";
  currentRange = { start: null, end: null };
  loadDashboard().catch((error) => console.error("Dashboard failed to load:", error));
});
```

**Inclusive end-of-day**: `<input type="date">` gives `"2026-06-01"` with no time
component. If sent as-is for `end`, `timestamp <= '2026-06-01'` would exclude every
brew logged after midnight that day (string comparison: `'2026-06-01 08:00:00' > '2026-06-01'`).
The padding above (`" 00:00:00"` / `" 23:59:59"`) handles this — it's done once, in the
`filter-apply` handler, not repeated elsewhere.

**`submitForm`** — no change. It already calls `loadDashboard()` on success (line ~152
in the current file), and since `loadDashboard` now reads module-level `currentRange`,
the post-submit refresh automatically respects whatever filter is active. Verify this
still holds, don't rewrite it.

**Bottom-of-file calls** (`loadDashboard().catch(...)`, `setupForms().catch(...)`) — no
change.

### `src/brewops/frontend/style.css`

Add minimal styling for `.date-filter` consistent with existing `.panel` spacing — check
existing `.panel` rules before adding new ones; don't introduce a new layout pattern.

---

## Edge cases to handle explicitly

1. **No range set (both blank)** — must produce byte-identical behavior to today. This
   is the regression to watch for: every existing test in `test_db.py`/`test_api.py`
   that calls `get_stats(conn)` / `GET /api/stats` with no params must keep passing
   unmodified.
2. **Only `start` given, no `end`** — open-ended upper bound, includes brews up to now
   (and beyond, technically — no future-timestamp rejection on this path, unlike
   `parse_timestamp`).
3. **Only `end` given, no `start`** — open-ended lower bound, "everything up to X."
4. **`start` after `end`** — must return 400 from the API layer (`normalize_range`),
   not a silently-empty result set.
5. **Range with zero brews in it** (e.g. a week nobody used the machines) — `get_stats`
   must return `total_brews: 0`, `per_drink` with every drink type present at
   `count: 0` (not an empty list — this is what the `LEFT JOIN`/`ON`-clause approach
   guarantees), and `per_day: []` (empty list is correct here — there's nothing to
   `GROUP BY`). On the frontend this hits `renderTimeline`'s early-return path (see
   section 3) — with the recommended change it renders "No brews in this range" text
   instead of an unexplained blank box; without that change it's just a blank `<svg>`,
   which is a worse but not broken UX. `loadDashboard`'s `brews-today` tile must handle
   `stats.per_day.length === 0` → `lastDay` is `undefined` → falls back to `0`, already
   handled by the existing `lastDay ? lastDay.count : 0` — verify this still holds
   under a filtered empty range too.
6. **Range excludes a drink type's only brews** — `per_drink` for that drink must show
   `count: 0`, not be missing from the array (this is the reason for the `ON`-clause
   placement above, not a plain `WHERE`).
7. **Machine with zero brews in range but brews outside it** — `get_machine_health`
   should show `brew_count: 0`, `last_brew: null`, `specialty: null` for the range, while
   `last_maintenance` and `recent_errors` still show real (unfiltered) data if any exist.
   Don't let a machine disappear from the dashboard just because it was idle in range.
8. **Malformed date param** (e.g. `?start=banana`) — 400 with a message containing
   `"unparsable"`, matching the existing convention in `test_post_brew_validation`.
9. **Boundary inclusivity** — a brew exactly at `start` or exactly at `end` must be
   included (`>=` / `<=`, not `>` / `<`). Write a test asserting this explicitly since
   it's an easy off-by-one to get wrong.

---

## Verification steps

1. **Unit tests — `tests/test_db.py`**: add cases modeled on the existing `test_stats_math`
   fixture pattern (insert known brews at known timestamps via `insert_brew`, then call
   `get_stats(conn, start, end)`), covering:
   - a range that includes a subset of the seeded brews (assert exact counts)
   - a range with no brews in it (assert `total_brews == 0`, every drink type present
     at 0, `per_day == []`)
   - `start`-only and `end`-only (open-ended each direction)
   - a boundary test: brew timestamp exactly equal to `start` and exactly equal to `end`
     both counted
   - same coverage for `get_machine_health` with a range: brew_count/last_brew/specialty
     change, last_maintenance/recent_errors don't

2. **API tests — `tests/test_api.py`**: using the existing `db` fixture (already seeds
   brews at `2026-06-01 08:00:00`, `2026-06-01 09:00:00`, `2026-06-02 10:00:00`):
   - `GET /api/stats?start=2026-06-02+00:00:00` → `total_brews == 1`
   - `GET /api/stats?start=2026-06-05&end=2026-06-01` → 400 (start after end)
   - `GET /api/stats?start=banana` → 400, `"unparsable"` in detail
   - same shape of tests for `GET /api/machines/1?start=...&end=...`

3. **Run the full suite**: `.venv/Scripts/pytest.cmd` (see CLAUDE.md — Defender ASR
   blocks the `uv run` shim on this machine, use the `.cmd` shim directly). Confirm
   the pre-existing tests in `test_db.py`/`test_api.py`/`test_frontend.py` all still
   pass unmodified — they call these functions/endpoints with no date params, which
   must remain equivalent to "no filter."

4. **Manual check in the running app**: `.venv/Scripts/start.cmd`, open
   `http://localhost:8123`, confirm:
   - with no filter set, dashboard numbers match what you see today (no regression)
   - setting a narrow range updates the tiles, drink bars, timeline, and machine cards
   - setting a range with no data in it shows zeroes/empty chart, not an error or a
     crash (check the browser console)
   - Clear button resets to the unfiltered view
   - logging a new brew via the form while a range is active either still shows in the
     range (if the new timestamp falls inside it) or doesn't (if outside it) — confirms
     `loadDashboard()`'s post-submit refresh respects `currentRange`
