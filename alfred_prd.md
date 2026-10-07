# Alfred — Product Requirements Document

**Owner:** Bryan Soo
**Status:** Draft v1 — ground-up rebuild of the existing Alfred system
**Last updated:** 2026-09-14

## 1. Overview

Alfred is Bryan's personal assistant for tracking quarterly goals, daily/life admin, and (in a later phase) personal finances. It exists to answer two kinds of questions on demand — "how am I doing against my goals right now" and "what should I be doing today" — while also proactively nudging Bryan on things he'd otherwise forget (daily check-ins, overdue admin, an approaching quarter-end).

This is a **ground-up rebuild** of an earlier v1 (a Google Sheet tracker plus chat-based briefs and an on-demand dashboard). The v1 skill will be deleted once this version is live. Nothing from v1's data model or scoring logic is assumed to carry over.

### 1.1 Goals of this rebuild
- Move the data layer from Google Sheets to a proper structured store (SQLite) so goals, tasks, workouts, and finances can be queried and computed on reliably (rolling averages, next-due dates, historical rollups).
- Support goals that change every quarter without requiring a rebuild each time.
- Give Bryan one persistent, interactive dashboard page instead of a regenerated snapshot each time.
- Keep a single consistent source of truth shared between chat and the dashboard.

### 1.2 Non-goals (for this version)
- No automated bank feed integration (Era Context was evaluated and explicitly rejected after Amex connection issues). Bank data enters via manually downloaded CSV statements.
- Alfred's core interactive experience (chat, daily brief, dashboard) stays one consistent skill/persona backed by one shared database, rather than being fragmented across multiple independent agents each with their own view of the data. Dedicated subagents are used opportunistically for self-contained side-tasks where independent, parallel work is a genuinely good fit — see Section 3.4 — not for the core loop itself.
- No fully autonomous goal-setting — Alfred prompts Bryan at quarter-end and offers guidance grounded in his data and history (see Section 5.4), but the final goals remain his decision.

## 2. Persona

Deferred. Alfred is loosely voiced as an Alfred Pennyworth–style butler (caretaker, "mission support," moral compass), but the detailed persona — tone, how opinionated he is, exactly how the wise/maternal quality comes through — is intentionally out of scope for this PRD and will get a dedicated, more in-depth design pass later. The one persona-adjacent requirement that *is* in scope now is the advisory role described in Section 5.4: Alfred should be able to offer guidance and opinions, not just record data, even before the full persona is nailed down.

## 3. Architecture

### 3.1 Shape
Alfred's core is **one Claude skill**. This is a deliberate fit, not a default: a skill is the mechanism for packaging a persona plus behavioral instructions and logic so the *same* assistant behaves consistently no matter which entry point invokes it. That's exactly Alfred's requirement — the same assistant is invoked through three entry points, all reading and writing the same underlying data:

1. **Chat** — Bryan talking to Alfred directly (logging a task, asking a question, a check-in response).
2. **A scheduled trigger** — fires a proactive daily brief into chat (and, separately, the quarter-end goal-setting prompt). Each firing is a fresh session with no memory of past conversations; continuity comes entirely from the database, not from session memory.
3. **The dashboard page** — a persisted, interactive page (built as a live artifact) that both displays and can write back to the same data.

A plain script has no persona or conversational layer to invoke this way, and a standalone agent is built for independent, parallel reasoning — a different need from "be the same consistent assistant everywhere." That's why the core is a skill. Section 3.4 covers where a genuinely separate agent *does* fit.

### 3.4 Where dedicated agents fit
Bryan wants to use agents where they're a genuinely good fit, not avoid them altogether. The distinction from Section 3.1 still holds: agents make sense for self-contained, independent side-tasks that don't need to share the core skill's consistency guarantees — not for the main interactive loop (chat/brief/dashboard), which needs to stay one voice with one view of the data.

Two candidates, both optional and separable from the Phase 1/2 core build:

- **Job-search research agent** — given a company or role, goes off and independently researches it (recent news, culture signals, comp bands, whatever's useful) and returns a report. This is genuinely parallel, self-contained work — it doesn't touch scoring or shared state, so running it as a separate agent adds no consistency risk and lets it work without blocking the main conversation.
- **Finance ingestion agent** *(Phase 2)* — given an uploaded CSV, independently parses and categorizes the transactions, then hands the structured result back for the core skill to write to the database. Also self-contained: the agent's job ends at "here are the categorized rows," and the core skill remains the only thing that actually writes to the shared data.

Both are proposals for Bryan to confirm before build — not locked in.

### 3.2 Data store
- **A hosted document database**, provided via the Artifact tool's `db` capability — not a local SQLite file. This was changed from an earlier SQLite-on-Mac plan specifically to support **mobile access**: a local file is only reachable from a session with filesystem access to Bryan's Mac (e.g. Claude Code), which a Claude mobile session does not have. A hosted store is reachable identically from Claude Code on the Mac, a mobile chat session using the Alfred skill, and the published dashboard artifact itself (desktop or mobile browser) — so full functionality (logging, chatting, viewing) works the same from the phone as from the Mac.
- Section 4's tables are implemented as collections of JSON documents in this store rather than SQL tables. Computed values (rolling averages, quarter-to-date sums, next-due dates) are calculated in code rather than via SQL — more code to write, but nothing the PRD's requirements depend on SQL specifically to do.
- All three entry points from Section 3.1 read and write this one hosted store. There is no separate per-surface or per-device copy of the data, and no local server needed — the dashboard is a normal published Artifact using the `db` capability directly.
- Since there's no longer a single local file to back up, Alfred should periodically export JSON snapshots of the store into the Alfred git repo on Bryan's Mac as a backup/audit trail — not as the live store, just a safety net.

### 3.3 Editing model
- **Inline on the dashboard:** frequent, low-stakes edits — ticking off a to-do item, logging an ad-hoc task, re-categorizing a transaction (Phase 2).
- **Through chat only:** structural changes — editing a goal's target or scoring formula, deleting historical records, redefining a category's meaning. These are rare and need Alfred to apply them consistently across historical data, so they're handled conversationally rather than via a UI control.

## 4. Data model (high level)

Exact schema to be finalized during build. The entities below are implemented as **collections of documents in the hosted database** (Section 3.2), not SQL tables — field lists are still accurate, just conceptual rather than literal columns:

- `goals` — id, name, tracking_method (calendar-derived / weighted-score / binary-checkin / milestone-tasks, see 5.5), target, scoring_formula, quarter_start, quarter_end, status
- `goal_progress` — goal_id, period (day/week), score, notes
- `goal_history` — closed-out goals by quarter, for "what did I track in 2026" style queries
- `tasks` — id, description, source (suggested / ad-hoc), linked_goal_id (nullable), status, due/created dates
- `workouts` — date, type, source (Fitness calendar), duration
- `recurring_admin` — task name, interval, last_completed_date, computed next_due_date, category
- `transactions` *(Phase 2)* — date, amount, inferred_category, source_account, income/expense flag
- `investments` *(Phase 2)* — month, account/type, value (captured via monthly check-in, not auto-fetched)

### 4.1 Narrative memory: memory.md and changelog.md
The SQLite tables above are for structured, queryable facts (a workout happened, a transaction occurred). Alongside them, Alfred maintains two plain files for the things that don't fit a row-and-column shape:

- **`memory`** — durable, narrative knowledge about Bryan that Alfred accumulates over time: learned preferences, behavioral patterns noticed (e.g. "tends to run on Thursdays," "prefers evening workouts on weekdays"), and other soft signals that inform suggestions (Section 6) and quarter-end advice (Section 5.4) but aren't naturally a database row. Read and updated by the core skill as it observes behavior.
- **`changelog`** — a running history of changes to the *system itself*: when a goal's scoring formula was adjusted and why, when goals were set or retired each quarter, any other structural change to how Alfred tracks things. This is distinct from `memory` (which is about Bryan) and from `goal_history` (which is raw closed-out goal data) — it's the narrative record of how Alfred and Bryan's tracking have evolved, useful context for both of them to look back on.

Both are documents in the hosted store (Section 3.2), not local files — this is what makes them readable/updatable from every device, same as the rest of Alfred's data.

## 5. Feature: Goal Tracking (Phase 1)

### 5.1 Generic goal model
Because Bryan sets 3–5 new goals each quarter, goals are not hardcoded. Each goal is defined once as an object with:
- Name
- Tracking method (calendar-derived, weighted composite score, or daily binary check-in)
- Target
- Scoring formula
- Active quarter (start/end)

This lets Alfred support an arbitrary new goal next quarter without a rebuild, and lets old goals close out into `goal_history` for later lookback ("what goals did I have in 2026").

### 5.2 Current goals (Q3 2026 example set)

**Exercise 5x/week**
Tracked automatically from Bryan's "Fitness" Google Calendar (separate from his primary calendar). No manual logging needed.

**Find a new job** — scored 0–10, 1 decimal place
Composite of two buckets:
- *Leading indicators (~60% of score):* applications submitted, networking conversations, interview-prep hours logged, project/portfolio work — each measured against a rough weekly target. Effort past target doesn't inflate the sub-score past 10; the aim is consistency, not cramming.
- *Lagging indicators (~40% of score):* responses received, interviews scheduled/completed, offers — weighted more heavily per instance since they're harder to produce, but a quiet outcome week doesn't zero the score if effort was present.

Shown both as a raw weekly score and a rolling 3–4 week trend average, plus a quarter-to-date cumulative view. Early in a quarter (before 3–4 weeks of history exist), the rolling window expands to use whatever weeks are available rather than leaving it blank.

Sources: email, Bryan's networking spreadsheet, and his job tracker feed this automatically. Any job-search-related work logged as an ad-hoc to-do item (e.g. "research X company") gets tagged to this goal, so effort that isn't in those three sources still counts.

*Note: the weighting above is a first-pass draft for Bryan to review and adjust once it's scoring real weeks — not final.*

**Weekly-cap habit goal (≤2/week)**
Daily check-in: Alfred asks once a day whether it happened. Tracked as a rolling 7-day count against the cap of 2, so Alfred can flag when he's close to or over the limit mid-week, not just retroactively.

### 5.3 Query support
The dashboard and chat must both support ad-hoc questions against the goal/workout data, e.g.:
- How many times did I exercise in Q3?
- How many weeks did I hit the "exercise 5x/week" target?
- What kinds of workouts did I do in the last month?
- Graph my weekly progress toward finding a new job.
- What goals did I set in 2026?

### 5.4 Quarter transitions
Alfred proactively prompts Bryan as each quarter approaches its end, giving him time to think before defining the next quarter's 3–5 goals. This is not fully autonomous — Alfred surfaces the prompt and captures the new goal definitions, but goal-setting itself stays a conversation with Bryan.

Within that conversation, Alfred is expected to actively **advise**, not just collect input: drawing on the closing quarter's `goal_history` and `memory.md` (Section 4.1), it should be able to point out patterns ("you consistently under-shot the gym goal on weeks you traveled — want to adjust the target or the definition of a week?"), flag a goal that's effectively done and could retire, or suggest a new goal based on something Bryan has mentioned wanting to work on. The advice is opinionated, but the decision stays his.

### 5.5 Sub-goals (milestone tracking)

Added 2026-09-14, prompted by a concrete case: turning "do the Anthropic Claude Code course"
from a plain ad-hoc to-do into something trackable with its own deadline and a generated study
plan.

Distinct from the quarterly goals above: a sub-goal is a **finite, one-off project with its own
due date**, not a recurring weekly behavior scored against a trailing pace. It reuses the generic
goal object from 5.1 rather than becoming a new entity — a `goals` doc with a new
`tracking_method: "milestone_tasks"`:

- Optional parent — a sub-goal may (but doesn't have to) roll up under an existing quarterly
  goal. Confirmed 2026-09-14: optional per sub-goal, not mandatory, since not every sub-goal has
  an obvious parent.
- Its own due date, independent of `quarter_end` — a sub-goal can be due mid-quarter.
- A real, checkable estimate of scope (e.g. a course's own stated duration) — never a number
  Alfred invents.

**Pacing is time-based, not task-count-based.** Alfred proposes a study plan in minutes/sessions
("two 45-minute sessions this week"), not "N modules/week," grounded in the material's real
scope rather than a guess. Because Bryan — not Alfred — knows how much time he can realistically
give something and how he'd rather chunk it, the pace and cadence are always a chat-confirmed
decision: Alfred proposes, Bryan confirms, before anything is written. Same principle as the
job-search goal's self-relative scoring (5.2): don't invent a numeric target on Bryan's behalf.

**Progress is computed live, never stored** — percent complete is derived from the estimated
time of its completed linked to-do items divided by the sub-goal's total estimate, recomputed on
every view. Same "no persisted aggregate" principle the rest of the system follows.

**Replanning is reactive, not scheduled** — consistent with "no automation" elsewhere in this
PRD, each time Alfred is invoked and a sub-goal is still open, it re-checks actual progress
against the pace needed to hit the due date and adjusts the remaining plan if Bryan has fallen
behind or pulled ahead, rather than a schedule set once and left to go stale.

**Optional scoring effect on a parent goal** — if a sub-goal has a parent, Bryan can choose to
have time spent on it contribute a temporary weighted component to the parent's weekly score,
added when the sub-goal opens and removed (with the parent's other weights restored) when it
closes. This is a structural scoring-formula edit, so it follows 3.3's rule: chat-only, logged in
`narrative/changelog` both when added and when removed.

**Dashboard**: a sub-goal with no parent gets its own card in a "Sub-goals" section; one with a
parent renders nested inside that parent's existing goal card. Exact schema and mechanics live in
`SKILL.md`, not here.

## 6. Feature: Suggested To-Do List (Phase 1)

- Daily suggested to-do list, prioritizing progress toward active goals first, then other life admin (flights, travel, haircuts, dinners, etc.).
- Ad-hoc task logging ("research X company," "email X person," "start X project") via chat, tagged to a goal where relevant (see 5.2).
- **No preferences are pre-defined.** Alfred starts blank and learns patterns over time — e.g. observed day-of-week workout patterns from the calendar, or which suggested tasks Bryan accepts, reschedules, or ignores. This is pattern-detection from logged behavior, not a black-box ML model — suggestions should be explainable (e.g. "you usually run on Thursdays and haven't yet this week").
- **Recurring life admin** (haircuts, etc.) is tracked as interval-since-last-completed rather than fixed calendar dates: each recurring item has an interval and a last-completed date, from which Alfred computes a next-due date and surfaces it in the daily brief / to-do list once it's due or overdue. Completing the linked to-do item resets the clock. (Fixed-date admin, like a specific renewal deadline, is just a calendar event and needs no special logic.)

## 7. Feature: Finance Tracking (Phase 2)

Scoped now, built after Phase 1 is working.

- **Input:** manually downloaded monthly bank statement CSVs (no automated bank connection). Alfred proactively reminds Bryan once a month if it hasn't seen a fresh statement, rather than waiting to be asked.
- **Categorization:** Alfred infers spending categories automatically from the CSV data; Bryan can rename or recategorize afterward (inline on the dashboard, per the editing model in 3.3).
- **Income:** captured from the same CSVs (salary deposits) as a category/flag rather than a separate input.
- **Investments:** not derivable from bank statements, so tracked via a monthly check-in prompt (same pattern as the daily habit check-in) where Alfred asks for current investment status/values.
- **Output:** spending trends, insights, and (later) financial goals such as "save £5,000 this quarter" — which can reuse the generic goal object from Section 5.1 rather than being a separate system. Not an immediate priority.

## 8. Output Surfaces

**Proactive daily brief (chat, scheduled)**
Short push — today's suggested to-dos, any pending check-ins (habit goal; investments at month-end), admin items coming due. Meant to be quick to read, not a full report.

**Live dashboard (persisted, interactive page)**
One page combining goal progress, the to-do checklist, and (Phase 2) the finance view — rather than separate outputs per mode. Stays current rather than being a regenerated snapshot each time it's opened. Supports the inline edits described in Section 3.3 (ticking tasks, logging ad-hoc items, recategorizing transactions).

**Ad-hoc Q&A (chat)**
Direct natural-language questions (Section 5.3 examples) answered in chat, with a chart rendered inline when it genuinely helps.

**Shared state:** anything logged via chat updates the same database the dashboard reads from. Whether an already-open dashboard reflects a chat update immediately or on next refresh is an implementation detail to settle during build — not a constraint on the data model.

Visual design of the dashboard is not a current priority — a clean, legible default is fine, to be refined later.

## 9. Phased Roadmap

**Phase 1 (this build):**
- Generic goal model + the three current goals (exercise, job search, a weekly-cap habit)
- Suggested daily to-do list + ad-hoc task logging + recurring admin
- SQLite data store in the Alfred folder
- Daily brief (chat, scheduled) + live interactive dashboard
- Quarter-end goal-setting prompt

**Phase 2 (after Phase 1 is working):**
- Finance tracking: CSV ingestion (reminder-based), auto-categorization, income, monthly investment check-in, spending trends
- Optional: financial goals folded into the generic goal model (e.g. savings targets)

## 10. Open Items / To Confirm During Build

- Exact weighting for the job-search score (Section 5.2) — draft only, needs review against a few real scored weeks.
- Whether the dashboard should reflect chat-logged updates in real time or on refresh.
- Exact schema details for each table (Section 4) — to be finalized alongside the build, not fully fixed here.
- Frequency of the JSON snapshot export from the hosted store into the Alfred git repo (Section 3.2) as a backup/audit trail.
- Whether to build the job-search research agent and/or the finance ingestion agent (Section 3.4) — proposed, not yet confirmed.
- Detailed persona design (Section 2) — deferred to a dedicated follow-up pass.
- **Confirmed:** data store moved from local SQLite to a hosted document database (Section 3.2), specifically to support full functionality from Claude mobile as well as the Mac.

## 11. Migration

The current Alfred skill (v1: Google Sheet tracker + chat brief/dashboard) will be deleted once this rebuild is live and verified.
