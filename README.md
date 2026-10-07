# Alfred

A personal assistant built on Claude: quarterly goals, daily to-dos, and recurring life admin.
There is no backend. All data lives in the hosted document database attached to a published
Claude Artifact (`dashboard.html`), and Alfred reads and writes it from a Claude Code session,
or from the dashboard's own "Ask Alfred" chat box.

## What's here

| File | What it is |
| --- | --- |
| `alfred_prd.md` | Product requirements: scope, data model, phases |
| `dashboard.html` | The dashboard page, published as a Claude Artifact with the `db`, `sample` and `mcp` capabilities |
| `.claude/skills/alfred/SKILL.md` | Alfred's operating instructions as a Claude Code project skill |
| `alfred_v2_standalone_skill.md` | The same instructions, self-contained, for an account-level skill |
| `CLAUDE.md` | Map of the repo and working conventions for Claude Code |

## Using it yourself

Nothing personal is committed, so three things need filling in:

1. **Publish `dashboard.html`** as a Claude Artifact, declaring `db`, `sample`, and `mcp`
   (Gmail, Google Calendar, Google Drive, and a web-search connector).
2. **Put that artifact's URL** wherever the docs say `<YOUR_ARTIFACT_URL>`. In this repo that
   means a gitignored `CLAUDE.local.md`; in the standalone skill, replace the placeholder inline.
3. **Create a `config/sources` doc** in the artifact's database:

   ```json
   {
     "fitness_calendar_id": "…@group.calendar.google.com",
     "primary_calendar_id": "you@example.com",
     "job_sheet_id": "<Google Sheet file id>",
     "job_sheet_name": "<sheet title>",
     "networking_sheet_id": "<Google Sheet file id>",
     "networking_sheet_name": "<sheet title>"
   }
   ```

Goals themselves are data, not code: add `goals` docs through chat, as described in `SKILL.md`.
