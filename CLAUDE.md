# Aircraft Induction Status Tracker

## What this project is

A fleet-wide dashboard the ~15-analyst aviation induction team uses to track the
**progress of induction work on each newly delivered aircraft** — a per-aircraft
checklist of ~60 tasks grouped into workflow phases, with status, assignee, note,
and completion date on every task. It answers "where does every tail stand, and
what's blocking delivery?"

Tasks gate one another through a defined workflow (setup → pre-delivery imports →
airframe/ARL/major-assembly work → final docs → close-out), and the board rolls
per-aircraft and per-department progress up into fleet-wide status, filtering, and
exportable reports.

## Current state & how to run

- **One file: `index.html`** (~4,770 lines) — all HTML, CSS, and JavaScript inline.
  No framework, no build step, no npm, no dependencies. Vanilla JS + raw `fetch`.
- To run/preview: open `index.html` in a browser (or serve the folder statically).
  There is nothing to compile. Edit `index.html` directly.
- `README.md` is a one-line stub.
- The file was historically versioned in its own name (`aircraft-induction-tracker-v6_NN.html`)
  and uploaded via the GitHub web UI, then renamed to `index.html`. Git history is a
  series of delete/upload/rename commits, not incremental diffs.

## Architecture at a glance

Vanilla single-page app. On load, `init()` → `load()` (fetch from Supabase) →
run one-time data migrations → `render()`. All state lives in two module-level
globals: `DATA` (array of aircraft) and `LOG` (activity log). User edits mutate
`DATA` in place, mark the changed task "dirty", and a debounced `flushSave()`
upserts just the dirty rows to Supabase. A 90-second background poll pulls other
analysts' changes back in.

## Data model

Each aircraft is `{ ac, del, due, type, tasks }`:
- `ac` — tail number (string, also the primary key), `del` — delivery date,
  `due` — complete-by date (delivery + 30 days), `type` — fleet type.
- `tasks` — map of `taskKey → packed string`.

**Packed task string** (pipe-delimited, trailing empties omitted):
`status|note|assignees|doneDate`
- Parse with helpers `ts()` status · `tn()` note · `ta()` assignees (comma-separated)
  · `td()` done date. Build with `packTask(status, note, assignees, doneDate)`.
- Statuses: `pending`, `inprogress`, `done`, `notready` (gated/locked), `na`
  (not applicable — excluded from progress math).

**Task taxonomy** (top-of-`<script>` constants — the schema):
- `TL` — task key → display label (~60 tasks, the master list `ALL_KEYS`).
- `GROUPS` — ordered workflow phases (45-Day Setup → 15-7 Day Pre-Delivery →
  Airframe → ARL/STF/AIR → Major Assemblies → Final Docs/Delivery Day → AW Cert →
  Reg Cert → Close Out → File Retention). Each phase maps to a department + task keys.
- `DEPT_KEYS` — three departments (**Airframe**, **ARL/AIR/STF**, **Major Assemblies**)
  → task keys. Drives the three progress dots per card and the colored report columns.
- `TASK_COLORS` — task key → color bucket (airframe / arl / maj), per the source Word doc.
- `PEER_AUDIT_KEYS` + `PEER_AUDIT_PARENT` — peer-audit / QA tasks and the parent task
  that unlocks each.
- `NONAD_SUBTASK_KEYS` — four sub-tasks of `nonad`; completing all four auto-completes
  the parent.
- `TEAM_MEMBERS` (19 analysts), `FLEET_TYPES` (737 MAX 8/9, A321neo, 787-9 / 787-9 Hi-J,
  Neo PS, XLR).
- `SEED` — 55 hardcoded aircraft, the **real initial fleet snapshot** the tool was
  built from. It only bootstraps an empty Supabase on first load; in production the live
  queue lives in Supabase and has since grown to ~85 aircraft as tails are added through
  the Add-aircraft UI. Editing `SEED` does **not** change live data.

## Task state machine (unlock/lock cascade)

Tasks gate each other. `notready` means "not yet available." The rules:
- `setup` → done unlocks the pre-delivery tasks (`SETUP_UNLOCKS`).
- `arl_import` → done unlocks all main tasks (`ARL_TRIGGER_UNLOCKS`).
- A parent task → done unlocks its peer-audit/QA task.
- All four `nonad` subtasks done → auto-completes `nonad` and unlocks its peer audit.
- Undoing a trigger re-locks downstream tasks to `notready` **only if not yet started**.

Two places implement this: `applyTriggerStates(DATA)` reconciles the whole fleet on
load; `updState()` runs the same cascade on each individual edit and marks every task
the cascade touched as dirty.

## Storage & sync (Supabase, no SDK)

Raw `fetch` against Supabase PostgREST. **URL + anon key are hardcoded in the file.**
Two tables:
- `aircraft` — `(ac, data jsonb, updated_at)`. Holds per-aircraft **metadata**
  (type/del/due) plus a special row `ac = '__log__'` storing the activity log.
- `tasks` — `(ac, k, v, updated_at)`, **one row per task**. This is the source of
  truth for task state. Split to row-per-task deliberately so simultaneous analyst
  edits don't clobber each other.

Flow:
- `load()` — reads the aircraft table + paged reads of the tasks table, assembles `DATA`.
- `save(ac, key)` — writes a localStorage mirror immediately, marks dirty, debounces
  `flushSave()` by 1s.
- `flushSave()` — upserts (`Prefer: merge-duplicates`) only dirty task rows + dirty
  aircraft metadata + the log. Snapshots the dirty sets before awaiting so edits made
  mid-flight aren't lost, and re-flushes if more became dirty.
- `backgroundSync()` — every 90s, fetches task rows with `updated_at` newer than the
  last sync, skips rows the user is editing or that are older than the local copy,
  applies the rest, and re-renders. This is the multi-user freshness mechanism
  (polling — no Supabase Realtime here).
- Resilience: localStorage mirror (`act-v3`), a per-change local backup ring buffer
  (`act-v3-backup`, 500 entries), a flush on `visibilitychange → hidden`, and a
  localStorage/`SEED` fallback if Supabase is unreachable. A sync-status pill in the
  header reflects saving/synced/error/local.

## Rendering & UI

- One page: header (title, report buttons, sync pill, "+ Add aircraft") → 4-KPI stats
  row → sticky left **filter sidebar** + main **aircraft-card list**.
- Sidebar filters: sort, aircraft status, fleet type, tail-# search, task status,
  assignee, task. A hidden legacy `.toolbar` block is kept only for JS compatibility;
  filter state is mirrored between the visible sidebar and those hidden elements.
- `render()` is a dispatcher: depending on which filters are active it switches among
  many specialized views (assignee view, completion view, global task-status view,
  single-task view, search view, and assignee×task intersections). `buildCard()`
  renders a normal aircraft card; expanding it shows tasks grouped by phase, each with
  a status dropdown, note input, assignee multi-select, and done-date field.
- Modals: Add Aircraft; Combined Report (Section 1 fleet summary + Section 2 pending
  tasks), printable to PDF via `@media print`.

## Exports & reports

- `exportExcel('all' | 'overdue')` — builds a **real .xlsx from raw XML, no library**,
  color-coded by department.
- `exportReportCSV()` (report modal), `window.print()` for PDF, `exportBackup()` for a
  full JSON snapshot.

## Status derivation

- `pct(a)` — % done across applicable (non-`na`) tasks. `dpt(a, dept)` — % per department.
- `isOver(a)` — past `due` and <100%. `isRisk(a)` — due within 14 days. `astate(a)` →
  `overdue` / `risk` / `ok`, which drives card color, badge, and the progress-fill class.

## Migrations

On load, `init()` runs one-time `apply*` migrations (`applyNewTaskDefaults`,
`applyFleetTypeRename`, `applyE1E2Split`, `applyNewAirframeTasks`, `applyV6Migrations`)
that add/rename/split tasks as the schema evolved. If any reports a change, every row is
marked dirty and persisted through the normal `flushSave()` path. In steady state these
are no-ops.

## Conventions & gotchas

- **Edit `index.html` directly** — no build, no modules, no package manager.
- **Deployed via GitHub Pages** from this repo — the live site is `index.html` at the
  repo root. Deploying an update = replacing that `index.html` (history shows a
  rename-based upload flow through the GitHub web UI).
- **Backend is Supabase** (PostgREST + the `aircraft`/`tasks` tables above); the app
  talks to it directly with the committed anon key.
- **No authentication, by design.** The anon Supabase key is committed and writes are
  open to anyone who has the app. This is intentional: the tracker has no accounts, and
  access is controlled by keeping the app URL and key internal to the team. Lead-only
  writes / auth are explicitly *not* planned.
- **No accounts.** `userName` is a local field used only to stamp the activity log.
- Task state is edited optimistically in `DATA`; the packed-string format is the single
  most important thing to understand before touching task logic.

## Open questions (confirm before assuming)

- Is `index.html` (renamed from `index_2.html`) the canonical file? (History says yes.)
