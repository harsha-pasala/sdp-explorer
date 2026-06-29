# SDP Explorer — Project Context

Condensed handoff doc. Everything needed to continue building without prior chat history.

## What this is

An interactive web **explorer that demystifies Lakeflow Spark Declarative Pipelines (SDP) features** for architects & devs — a more intuitive, visual alternative to the docs. Built for **field enablement, scaling SDP adoption in Canada**. Targets both Structured Streaming→SDP migration and net-new workloads.

Each feature is a browsable card whose hero is an **animation/interaction showing what the feature does to the data**, plus a "how it works / when to leverage / watch out" digest. It is shipped as a **Databricks App**.

> Product team is separately building the SS-vs-SDP comparison tool, migration assessor, and TCO calculator — **do NOT duplicate those**. This tool is the feature demystifier.

## Guiding principles (locked with user)

- **Communicate VALUE and COMPLEXITY to the customer, not just data.** Trim sample data to the minimum that makes the point. The animation/interaction IS the explanation.
- **NOT a heavy decision tree / taxonomy.** It's a flat, browsable gallery of bite-sized cards.
- **Lead with the demo, then the SQL.** Card section order: title + one-liner → interactive demo → "The flow" (SQL/Python) → "How it works" → "When to leverage it". Explanatory notes go *after* the syntax they reference (e.g. a `BY NAME` note sits under the SQL, not above it).
- **Honest "when NOT to use / watch out"** on every card — the differentiation vs docs.
- **Styling = Databricks docs look**: white bg, DM Sans font, dark-slate ink `#1b3139`, docs-blue `#2272b4` links, lava-red `#ff3621` accent, thin calm borders, airy spacing. Official logo lockup at `static/databricks-logo.svg` (from docs.databricks.com/aws/en/img/logo.svg).

## Architecture / files

```
sdp-explorer/
├── app.py            # FastAPI serving static/ on 0.0.0.0:$DATABRICKS_APP_PORT (deploy server)
├── app.yaml          # Databricks Apps runtime: command [python, app.py]
├── databricks.yml    # DAB: resources.apps.sdp-explorer, target dev (NO development mode — it prefixes app names invalidly)
├── requirements.txt  # fastapi==0.115.0, uvicorn[standard]==0.30.6
├── dev_server.py     # LOCAL ONLY: livereload server on :8000 (gitignored, not deployed)
├── .gitignore        # excludes .venv/, dev_server.py, __pycache__, .databricks/, .DS_Store
├── README.md
├── CONTEXT.md        # this file
└── static/
    ├── index.html         # THE APP — all cards, styles, animation logic in one file
    └── databricks-logo.svg
```

**Everything lives in `static/index.html`** — CSS in one `<style>`, all logic in one `<script>`. No build step, no frameworks.

## Local dev loop

```bash
./.venv/bin/python dev_server.py        # livereload server, http://localhost:8000 (run in background)
```
- `.venv` has fastapi, uvicorn, livereload installed. Edits to `static/index.html` **auto-reload the browser** — no manual refresh, no re-open.
- **Always syntax-check JS after edits** (a parse error blanks the whole app at boot):
  ```bash
  awk '/<script>/{f=1;next} /<\/script>/{f=0} f' static/index.html > /tmp/sdp_check.js && node --check /tmp/sdp_check.js && echo "JS OK"
  ```

## Deploy (Databricks App)

```bash
databricks bundle deploy --profile <PROFILE>
databricks bundle run sdp-explorer --profile <PROFILE>
```
- **NOT yet deployed.** `e2-demo` profile is at its 300-app cap (create failed there; bundle files uploaded but no app created — `databricks bundle destroy --profile e2-demo` to clean). Likely target = Canada workspace (`canada-eh` / `azure-canada` profiles need `databricks auth login` first).
- App name `sdp-explorer` (≤26 chars). Static site → tiny FastAPI server; must bind `0.0.0.0:$DATABRICKS_APP_PORT`.

## App internals (how the SPA works)

- **`NAV`** array → sidebar groups + items. Each item: `{ id, name, status }`. `status`: `ready` (clickable), `soon` (greyed, "soon" pill), `preview` (amber pill), `beta` (amber pill). `renderNav()` builds it; clicking a ready/preview/beta item calls `route(id)`.
- **`DEMOS`** map: `id → renderFn` returning the card's HTML string.
- **`route(id)`** sets `main.innerHTML = DEMOS[id]()`, then calls the card's `wireXxx()` to attach behavior; unbuilt ids render `placeholder()`.
- Boot: `route("replace-where")`.

### Adding a new card (recipe)
1. Set the `NAV` item's `status` to `ready`.
2. Add `"<id>": <renderFn>` to `DEMOS`.
3. In `route()`, add `if (id === "<id>") wire<Xxx>();`.
4. Write `function <renderFn>()` returning HTML (follow section order above).
5. Write `function wire<Xxx>()` for behavior (placed near the other wire fns, before the boot `route(...)` call).
6. Syntax-check.

## Reusable patterns / primitives

- **source→target merge** (REPLACE WHERE): two `.tbl` panels + arrow; rows fly from source into target.
- **timeline lanes** (Rewind & Replay): stacked lanes sharing an x-axis; draggable handle (`pointerdown/move` on a ruler, snap to `frac = k/N`); today-anchored dates via `new Date()` (browser JS — fine here, unlike workflow sandbox).
- **manual stepper** (AUTO CDC family): `◀ Previous` / main step button / `Reset`. Convention: **when done, main button DISABLES and reads "All N applied"** — do NOT turn it into a second Reset. `Previous` re-enables. State derived purely from `applied` count so back/forward is consistent; rows flash via a `prevSig` Set diff.
- **`.cdc-out` tables**: shared table style for change-feed/output tables; `tr.flash` flash animation; column headers tie to SQL clauses.
- **banners**: `.recon` (amber, reconciliation/"engine did something") and `.recon.diff` (blue, informational per-step diff) and `.insight` (blue, static concept). Keep one consistent diff banner per card rather than scattering messages into a narration line.

## Cards built

Sidebar groups: **Datasets · Change data capture · Flows · Data quality · Incremental refresh · Recovery & operations** (removed non-SDP MERGE & dynamic-overwrite).

| id | group | status | what it shows |
|----|-------|--------|---------------|
| `replace-where` | Flows | ready | **REPLACE WHERE** (label, not "FLOW REPLACE WHERE" — that stays only in the SQL). Source→target merge animation. 5-row today-anchored sample (2 old + last 3 days) demonstrating predicate `date >= current_date()-7`. `BY NAME` note sits under the SQL. Button "Run Refresh". |
| `rewind-replay` | Recovery & operations | preview | **Rewind & Replay** (Streaming Time Travel renamed → just "Rewind and Replay"). Interactive: drag rewind point T; live value meter (rewind vs full-refresh % compute saved); 3 lanes (source events / pipeline checkpoint / target revenue) sharing a today-anchored date axis; on run: ①checkpoint rollback ②erase target after T ③replay through fixed code (code badge buggy→fixed, bug marker ✗→✓ patched). Under-rewinding leaves leftover corrupted rows = consistency teaching moment. Speed selector (Slow default). |
| `auto-cdc` | Change data capture | ready | **AUTO CDC** (`APPLY CHANGES`). Change feed → target. **SCD 1 / SCD 2 toggle**. Manual stepper, **timestamps** (not seq#). Signature moment: a late, out-of-order UPDATE that arrives *after* a DELETE — SCD1 ignores it (stays deleted, no resurrection), SCD2 slots it into history. Column headers tie to `KEYS`/`APPLY AS DELETE WHEN`/`SEQUENCE BY`. |
| `auto-cdc-snapshot` | Change data capture | ready | **AUTO CDC FROM SNAPSHOT** (SCD2 only). Left = plain full snapshots (NO annotations/animation — raw input; a deleted key is simply absent). Right = SCD2 built by diffing consecutive snapshots (all value/flash here). One consistent blue diff banner per snapshot incl. delete-by-absence. Python `create_auto_cdc_from_snapshot_flow(stored_as_scd_type=2)`. |
| `auto-cdc-bitemporal` | Change data capture | beta | **AUTO CDC bitemporal** — replicates docs example exactly (company A, XFv1→v2→v3 out-of-order→delete). Step-through builds the two-axis table: business validity `__START_AT`/`__END_AT` + system awareness `__SYSTEM_START_AT`/`__SYSTEM_END_AT`; superseded (system-closed) beliefs greyed. SQL: `SEQUENCE BY` + `SYSTEM SEQUENCE BY` + `STORED AS BITEMPORAL`. **+ "The payoff" as-of-knowledge grid** (`wireBitemporalAsOf`): 4×4 interactive matrix, cols = business time (when true) × rows = system time (what we knew by); clickable cells color-coded by believed value (XFv1/v2/v3/deleted) with crosshair + plain-English readout. Reading *down a column* (business 12:10: XFv1→XFv3 once the out-of-order correction lands) makes the "same moment, different belief over time" concept land — the payoff a single-timeline SCD2 can't show. Grid data hand-reconstructed from the table; opens on (12:22, 12:10). |

**Not yet built** (status `soon`): `streaming-table`, `materialized-view` (Datasets); `append-flow` (Flows); `expectations`, `quarantine` (Data quality); `enzyme` (Incremental refresh).

## Key SDP feature facts (verified against docs)

- **REPLACE WHERE** (SDP form): `CREATE OR REFRESH STREAMING TABLE … SCHEDULE EVERY 1 DAY … FLOW REPLACE WHERE <predicate> BY NAME SELECT …`. Matched rows deleted → source query recomputed for that range → inserted, atomically. `BY NAME` **required**. Don't put the predicate in the SELECT — engine applies it. (Not Delta DML `INSERT … REPLACE WHERE`.)
- **AUTO CDC**: `AUTO CDC INTO t FROM STREAM(src) KEYS(...) APPLY AS DELETE WHEN ... SEQUENCE BY ... STORED AS SCD TYPE 1|2`. `SEQUENCE BY` makes it out-of-order safe (sequence order, not arrival). Late event with lower sequence than applied = stale.
- **AUTO CDC FROM SNAPSHOT**: Python `dlt.create_auto_cdc_from_snapshot_flow(target, source, keys, stored_as_scd_type=2)`. Diffs consecutive full snapshots; **deletes inferred from rows that disappear**. Source must be the complete table.
- **AUTO CDC bitemporal** (Beta): docs https://docs.databricks.com/aws/en/ldp/cdc#how-bitemporal-auto-cdc-works . Two time axes (business/valid vs system/awareness). On each change: close prior belief on system axis (`__SYSTEM_END_AT`) + insert corrected rows stamped with new system time; out-of-order corrects history in place. Sequencing cols must be sortable, no NULLs.
- **Rewind & Replay**: triggered SDP only (no continuous/SS); sources Delta+Kafka (no Auto Loader); sinks ST+MV; no stateful operators; CLI `databricks pipelines start-update <id> --json '{"cause":"API_CALL","rewind_spec":{rewind_timestamp,dry_run:false,datasets:[{identifier}]}}'` (CLI >0.283.0); rewind = metadata-only restore (Delta version+offsets+state), pipeline idle after; replay on restart; non-destructive; ~7-day/100-batch retention; can't rewind past FULL REFRESH.

## CSS tokens (in `:root`)

`--bg #fff` · `--ink #1b3139` · `--ink-2 #475a63` · `--muted #6b7c85` · `--line #e2e7ea` · `--panel #f7f8f9` · `--blue #2272b4` · `--blue-bg #eaf3fa` · `--red #ff3621` · greens `--keep #2f8a5b`/`--keep-soft`/`--keep-line` · oranges `--target #c2410c`/soft/line · blues `--src`/soft/line. Fonts: DM Sans (text), DM Mono (code/data).
