# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Alfred

Personal assistant for Bryan Soo — quarterly goals, daily to-dos, and life admin (finance
tracking is Phase 2, not built yet).

**Source of truth for requirements:** [`alfred_prd.md`](alfred_prd.md). Read it before making
any decision about scope, data model, or architecture — this file is only a map of what's in
the repo and how to work with it, not a restatement of the PRD.

**Operational instructions for the assistant itself:**
[`.claude/skills/alfred/SKILL.md`](.claude/skills/alfred/SKILL.md). That file is the behavioral
spec — persona, entry points, goal logic, scoring formulas, to-do/recurring-admin logic,
quarter-end handling. It's the file that actually changes week to week as the system evolves;
this file stays high-level on purpose.

## Architecture in one paragraph

There is no backend and no local database. All structured data (goals, goal progress, tasks,
workouts, recurring admin, plus narrative notes) lives in a hosted document database provided by
the Artifact tool's `db` capability, attached to the published `dashboard.html` artifact. Chat
(via `Artifact action="read_db"/"write_db"`) and a daily brief invoked on demand (same mechanism,
stateless between runs — there is deliberately no scheduled routine, see "No automation" below)
read/write that store from a Code-tab session. This split — hosted store instead of a local
SQLite file — was a deliberate pivot from the PRD's original plan, made specifically so the
dashboard and any logging also work from Claude on mobile, not just from a session with
filesystem access to this Mac. See PRD Section 3.2 and `narrative/changelog` for the full
reasoning.

**As of 2026-09-14, the dashboard page itself is a full, independent entry point, not just a
read/write widget page.** Alongside `db` it now declares the `sample` (ask Claude) and `mcp`
(call Bryan's connected Gmail/Calendar/Drive/**Parallel Search**, the last added later the same
day specifically to give the dashboard chat a general web-fetching path — see "Sub-goals" in
`SKILL.md`) runtime capabilities, so its "Ask Alfred" chat box can hold an open-ended conversation
and trigger live syncs directly from whatever device has the page open — no Code-tab session
required. See `SKILL.md`'s "5. Dashboard chat" for the mechanism
and `dashboard.html`'s `ALFRED_CONTEXT` constant for the ported instructions. This means there
are now *four* places that read/write the store: chat, the on-demand daily brief, the dashboard's
own data widgets, and the dashboard's own chat — the last of these being new and running with its
own capability grants rather than a Code-tab session's tool access.

**Hard constraint on the dashboard chat's tool list: `sample` allows at most 16 tools per call.**
`dashboard.html`'s `TOOLS` array is at exactly that ceiling (a generic `readCollection` covers
every collection read; the rest are the specific writes worth keeping in code). Adding a 17th
tool doesn't degrade gracefully — every chat message starts failing with `invalid_request`, in a
way that doesn't obviously point back to tool count. Count the list before adding to it; prefer
extending an existing generic tool over adding a new named one.

**Sub-goals (added 2026-09-15).** Beyond the quarterly goals, `goals` also supports
`tracking_method: "milestone_tasks"` — a finite, one-off project with its own due date (e.g. "finish
a course by Sept 30"), an optional parent goal, and a real estimated-minutes scope. No new
collection: it's a `goals` doc plus linked `tasks` docs, with progress always computed live from
those tasks' `estimated_minutes`, never stored. The dashboard renders a sub-goal in its own
section, or — if it has a parent — nested behind a collapsible "▸ N sub-goal(s)" toggle on the
parent's card. See `SKILL.md`'s "Sub-goals" section for full mechanics (pacing, replanning,
optional scoring effect on a parent goal).

**`ALFRED_CONTEXT` (the hand-ported instructions the dashboard chat runs on) is a recurring
source of silent bugs, not just a one-time porting task.** Twice in one day (2026-09-14) it was
missing something the full `SKILL.md` has — the fixed Calendar/Sheet IDs, and the instruction to
actually write a synced calendar outlook back to `daily_context` rather than just answering in
chat — and both times the failure looked like reasonable chat behavior (Alfred answering
correctly, or asking a sensible-sounding clarifying question) rather than an error, so neither was
obvious until Bryan noticed the dashboard itself hadn't changed. When editing any sync/scoring/
logging logic that's described in `SKILL.md`, check whether `ALFRED_CONTEXT` needs the identical
edit — porting it once is not sufficient, since the two files have no mechanism keeping them in
sync.

## Repo layout

```
alfred_prd.md                 Product requirements (source of truth for scope/data model)
CLAUDE.md                     This file
dashboard.html                The dashboard page — published as a Claude Artifact, declares
                               the `db` capability, reads/writes the hosted store directly
.claude/skills/alfred/
  SKILL.md                    Operational instructions: entry points, collection schemas,
                               goal-specific sync logic, scoring formulas, to-do/recurring-admin/
                               quarter-end handling
alfred_v2_standalone_skill.md A self-contained export of SKILL.md's content, for pasting into
                               Bryan's personal/account-level Claude skill (see "Skill scope"
                               below) — keep it in sync with SKILL.md by hand; nothing regenerates
                               it automatically
README.md                     Public-facing overview and setup notes (placeholders to fill in)
CLAUDE.local.md               Gitignored. Holds the real artifact URL -- this repo is public, so
                               the URL and source ids are never committed (see below)
alfred_v2_standalone_skill.local.md
                              Gitignored. The standalone export with the real artifact URL
                               filled in, ready to paste into the account-level skill --
                               regenerate it by hand whenever the tracked export changes
backups/                      Periodic JSON snapshots of the hosted store (audit trail only —
                               nothing reads back from these; the live data is never here)
```

The repo also sometimes accumulates untracked, dated duplicates of the standalone export (e.g.
`alfred_v2_standalone_skill_v2.md`, `alfred_v2_standalone_skill_2026-09-14.md`) from prior
export attempts. Only `alfred_v2_standalone_skill.md` (tracked, referenced above) is canonical —
don't read or paste from an untracked variant without checking `git status` first.

There is intentionally no `src/`, no package manifest, and no local database file — see
"Architecture" above.

## Where the live data actually is

The hosted database is **not a file in this repo**. It belongs to the published dashboard
artifact:

```
<YOUR_ARTIFACT_URL>
```

The real URL is deliberately not committed (this repo is public). It lives in the gitignored
`CLAUDE.local.md`, which Claude Code loads automatically alongside this file.

- Read it: `Artifact action="read_db"` with that `url` (`db_op: "get"/"list"/"query"`).
- Write it: `Artifact action="write_db"` with that `url` (`db_op: "set"/"update"/"delete"/"batch"`,
  `"batch"` for more than a couple of writes at once).
- Collections: `goals`, `goal_progress`, `goal_history`, `tasks`, `workouts`, `recurring_admin`,
  `narrative` (docs `memory` and `changelog`), plus `job_applications`, `networking_contacts`,
  and `daily_context` — three collections added after the initial build to support the
  job-search sync and calendar awareness (not in the original PRD schema) — and `config`
  (doc `sources`: the calendar ids and sheet names/file ids, kept in the store rather than in
  the repo). Field-level schema for
  each is in `SKILL.md`, not repeated here.

If this URL ever changes (the artifact gets re-published as a *new* artifact rather than
updated in place — should not happen in normal use, only if someone forgets to pass `url` when
republishing), update it in `CLAUDE.local.md` (and in the account-level skill).

## External data sources Alfred syncs from

Goal tracking pulls from several outside sources, each with a real gotcha worth knowing before
touching the sync logic (full mechanics in `SKILL.md` under "Goal-specific logic"):

- **Fitness Google Calendar** (separate from Bryan's primary calendar) — feeds the exercise goal. Must discard any event whose start
  time is in the future before counting it as a completed workout.
- **Primary Google Calendar** — merged with the Fitness calendar into
  `daily_context`, a rolling 7-day outlook (today through +6 days) that backs the dashboard's
  "Calendar outlook" panel and brief mentions (e.g. an evening BBQ) — not the exercise goal,
  which only counts *completed* Fitness-calendar events, separately.
- **Job tracker Google Sheet** — dates are **M/D/YYYY**.
- **Networking Google Sheet** — dates are **D/M/YYYY**, the opposite convention from the job tracker.
- **Gmail search** — catches real job-search activity (interviews, referral applications) that
  never makes it into either sheet.

The actual ids for all four live in the store's `config/sources` doc, not in this repo — both
`SKILL.md` and `dashboard.html` read them from there.

None of these sync on a schedule — see "No automation" below.

## "Build/run" commands

There's no build step and nothing to install — `dashboard.html` is deployed by publishing it
directly:

- **Update the dashboard page:** edit `dashboard.html`, then republish it with the Artifact tool
  using the same `url` above (never omit `url`, or it creates a second, separate artifact with
  its own empty database).
- **Seed or inspect data by hand:** use `Artifact action="read_db"` / `"write_db"` against the
  URL above — there's no CLI or script for this, it's done directly through the Artifact tool.
- **Export a backup snapshot:** read each collection via `read_db` (`out_dir: backups/` is
  convenient for dumping straight to files) and commit the result under `backups/`. Cadence is
  whatever's agreed with Bryan (weekly is the current default — see `SKILL.md`).
- **Run/preview the dashboard locally:** not meaningful — it only functions when served from
  its published Artifact URL, since that's what grants it the `db` capability. Opening the raw
  HTML file locally will show the page shell with no data and no working edits.
- **If the dashboard looks stale after clicking its link in chat:** the chat-embedded side-panel
  preview doesn't always trigger a real reload of an already-open artifact. Use the page's own
  **Refresh** button rather than assuming a re-click will reload it; opening the artifact fresh
  (a new tab, or the artifacts gallery) does reload correctly.

## Skill scope: this repo vs. Bryan's account

`SKILL.md` here is a **project skill** — it only activates in a Claude Code session whose
working directory is this repo. It is not visible to a plain claude.ai chat (web or mobile app)
at all; project skills are a Claude Code concept, not an account-wide one. Bryan also has (or is
migrating to) a **personal/account-level** skill named "Alfred," which syncs across every device
he's signed into — that's the one a plain "Alfred" in the regular app actually reaches. The two
are separate storage locations with no automatic sync between them.

`alfred_v2_standalone_skill.md` exists specifically to bridge that gap: it's SKILL.md's content,
made self-contained (no relative links to files outside a personal skill's reach), meant to be
pasted into the account-level skill so behavior matches between "Claude Code on this repo" and
"Bryan on his phone." When SKILL.md changes here, that file needs a manual update to match, and
the account-level skill needs a manual re-paste — none of this happens automatically. If asked
to change Alfred's behavior, update `SKILL.md` (the source of truth for ongoing development) and
flag to Bryan that the standalone export and his account-level skill are now behind it.

**A sharper constraint discovered 2026-09-12, on top of the above:** even with the account-level
skill installed, Alfred's data-touching behavior (reading goals, syncing calendar/email, writing
to the dashboard) only works in a session that actually has the Artifact tool's `db` capability
available — having the *data* be cloud-hosted doesn't guarantee every *surface* is equipped with
the tool to reach it. Confirmed by direct test: in Bryan's Claude desktop app, a "Cowork" session
had Gmail/Drive but no Artifact tool and no Calendar; a "Code"-tab session (this repo's own
environment) has both. The app offers no third, plain-chat entry point to test as a control —
only Cowork and Code exist, and only Code works. So in practice, right now, "call Alfred from
anywhere" means "from a Code-tab session" specifically, not literally any Claude surface. Not
yet verified: whether a Claude Code mobile product carries the same tool access as the desktop
Code tab — check that before treating mobile access as fully solved. See the
`narrative/changelog` entry from this date for the full account.

## No automation

There is no scheduled routine or cron job — Bryan deliberately chose to call Alfred on demand
rather than have a daily brief fire automatically. Don't create one unless he explicitly asks.
This is why the exercise/job-search/calendar syncs are designed to run on *every* invocation
that has the relevant tool access, rather than relying on a background job to keep them current.

## Conventions

- Don't invent or hardcode sample data anywhere (goals, tasks, recurring admin) — if something
  hasn't been logged yet, the dashboard and skill should show that honestly as an empty/zero
  state, not a placeholder.
- Structural changes to goals (target, scoring formula) happen through chat only, never as a
  dashboard control (PRD 3.3) — and get a dated entry in the `narrative/changelog` document.
- Don't invent numeric thresholds or targets on Bryan's behalf (e.g. a "weekly target" to score
  against) — ask him, or design the scoring around his own historical data instead (see the
  job-search goal's self-relative scoring in `SKILL.md` for the precedent).
- An ad-hoc task with no `due_date` is a backlog reminder, not something scoped to any week —
  it never counts toward "how many to-dos this week" totals, and a `suggested` task only counts
  once Bryan actually accepts or completes it, never just because Alfred proposed it. See
  `SKILL.md`'s "Query support" for the exact rule.
- Phase 2 (finance tracking, `transactions`/`investments` collections) is designed in the PRD
  but deliberately not implemented — don't add those collections or UI until Bryan asks to start
  Phase 2.
- **A task scheduled for a specific future day (`surface_early: false`, on either an `ad_hoc` or
  `suggested` task) is invisible everywhere on the dashboard — not shown in the Ad-hoc backlog
  either — until that day arrives, when it moves straight into Today's to-do** (changed
  2026-09-15: it used to sit visibly in the Ad-hoc backlog until due, which is what a plain
  deadline task — `surface_early` left at its default — still does; only the scheduled-day case
  changed). Don't assume a task with a future `due_date` should be visible anywhere before then
  without checking `surface_early` first.
- **Never persist a derived time-window aggregate — compute it live from raw records instead.**
  Three separate bugs on 2026-09-14 traced back to the same mistake: a stored rolling/weekly
  count only updates when something writes it, so it silently goes stale the moment time passes
  without a write (the exercise goal showing last week's count on a fresh Monday; the habit
  goal's `rolling_7d_count` staying nonzero long after the entry that caused it had aged out; a
  to-do-counting answer that missed a completed task because the count wasn't recomputed from the
  actual records). The fix in all three cases was the same: query the underlying `goal_progress`/
  `tasks` docs and sum/filter at read time — in `dashboard.html`'s render code and in chat — never
  trust a previously-written aggregate field. See `narrative/changelog`'s 2026-09-14 entries for
  the specifics.
