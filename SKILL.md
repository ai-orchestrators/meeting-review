---
name: meeting-review
description: "**Meeting Review & PM Update**: Pull a meeting transcript from a connected MCP (Fathom, Fireflies, Otter, Read AI, Grain, Notion meeting notes, or any other meeting MCP) or take a manual paste, extract every PM-relevant item, and sync changes across configured downstream systems (project state files, Notion tasks, Reclaim, dashboards). Use whenever the user says 'meeting review', 'process this meeting', 'review this transcript', 'update from meeting', 'sync meeting notes', or uploads/references a transcript that needs to be turned into project updates. The skill runs a one-time setup on first use to learn transcript sources, business context, downstream systems, and output preferences, then handles the full loop on every later run: fetch transcript → analyse → clarify gaps → propose updates → apply on approval."
allowedTools: [AskUserQuestion, Read, Write, Edit, Bash, ToolSearch]
---

# Meeting Review Skill

Extracts actionable PM data from a meeting transcript and walks through updating every configured system. Every change is reviewed before it's applied.

The skill runs in **7 stages**: a one-time setup (Stage 0), then six review-and-update stages on every invocation. Don't skip ahead.

**Core principle:** the skill itself contains zero personal or business-specific data. All identity, transcript sources, downstream systems, and output preferences are collected per user during Stage 0 and stored in `config.json`. Subsequent runs read from config.

---

## Stage 0: First-Run Setup

**Goal:** On the very first invocation, configure the skill for the user's transcript sources, business context, downstream systems, and output preferences. On every later run, this stage is silent.

### Step 0.1: Check for Config

Look for `config.json` in the same directory as this `SKILL.md`. Three possible states:

- **No `config.json`** → setup needed. Go to Step 0.2.
- **`config.json` exists, `"configured": false`** → setup incomplete. Go to Step 0.2.
- **`config.json` exists, `"configured": true`** → setup complete. Load it and skip to Stage 1.

### Step 0.2: Run Setup Flow

Tell the user: "This looks like the first time you're using this skill. One-time setup to learn your transcript sources, business context, and how you want the review to look. Should take about 3 minutes."

Use `AskUserQuestion` for each block. If it isn't loaded, fetch with ToolSearch (`query: "select:AskUserQuestion"`) first. Bundle related questions in a single call where it makes sense.

Run the blocks in order. The transcript-source answer in Block B gates a couple of source-specific sub-questions; everything else is independent.

#### Block A — Identity & locale

1. **What name should I use to refer to you in this skill's prompts?** (free text)
2. **What language variant do you write in?** (en-AU / en-US / en-GB / en-CA / en-IE / en-NZ / other)

#### Block B — Transcript source(s)

3. **Where do your meeting transcripts come from?** (multi-select — pick every source you use)

   | Option | How it works |
   |---|---|
   | Fathom | Calls a Fathom transcript MCP (defaults pre-filled, override during setup if your install differs) |
   | Fireflies.ai | Calls a Fireflies transcript MCP — user supplies namespace and tool names |
   | Otter.ai | Calls an Otter transcript MCP — user supplies namespace and tool names |
   | Read AI | Calls a Read AI meeting MCP — user supplies namespace and tool names |
   | Grain | Calls a Grain meeting MCP — user supplies namespace and tool names |
   | Notion meeting notes | Queries a Notion database where meeting notes live |
   | Another MCP connector | Catch-all for any other meeting MCP (Granola, Avoma, Krisp, Gong, Zoom, etc.) — user supplies service name, namespace, and tool names |
   | Manual paste / file | User pastes transcript, drops a file path, or uploads — for any source without a connected MCP |

   "Manual paste / file" is always available — pick it as the sole option if no connectors are wired up. Pick it alongside connectors when some meetings come from sources that don't have an MCP.

   For each **MCP-based** source the user picks (Fathom, Fireflies, Otter, Read AI, Grain, Another MCP), ask the same uniform set in one `AskUserQuestion` round (don't loop one service per round):

   - **MCP namespace prefix** (e.g. `mcp__fireflies__`) — Fathom default `mcp__fathom__` is pre-filled; the user can override
   - **List tool name** — the tool that returns recent meetings (Fathom default `list_meetings` is pre-filled)
   - **Fetch tool name** — the tool that returns a transcript for a chosen meeting (Fathom default `get_transcript` is pre-filled)
   - **Lookback window** — days back to scan when listing meetings (default 7)
   - **Service name** — only for "Another MCP connector": the human-readable label (e.g. "Granola")

   If the user doesn't know the namespace or tool names for a chosen MCP-based source, instruct them to run `ToolSearch` with the service name (e.g. `query: "fireflies"`) to discover the tools, then come back to setup. Don't enable the source with blank fields.

   For **Notion meeting notes**, ask: meeting notes database collection ID (`collection://...`).

   For **Manual paste / file**, ask: optional default folder where pasted transcripts should be saved as `.md` (skip if you don't want copies kept).

#### Block C — Business context

4. **What kind of operation are these meetings for?**
   - Single brand or company
   - Multi-brand portfolio (you run several brands)
   - Agency or consultancy serving external clients
   - Internal team only
   - Other (free text)
5. **How do you organise projects?**
   - By client only
   - By client + project
   - By project only (no client layer)
   - Other (free text)
6. **Do you maintain a registered list of active clients or projects?** If yes, where? (free text — file path, Notion DB ID, etc.). The skill uses this to disambiguate which project a meeting is about.

#### Block D — Downstream systems

7. **Do you use project state files (YAML)?** If yes:
   - Path to the master project state file
   - Glob pattern for per-project state files (skip if you don't have project-level state files)
8. **Do you use Notion for task tracking?** If yes:
   - Tasks DB collection ID (`collection://...`)
   - Project Dashboard DB collection ID (skip if you don't link tasks to projects)
9. **Do you use Reclaim for time-blocking?** If yes:
   - Task naming convention (free text template, e.g. `{project} | {category}` — placeholders are filled at write time. Skip for default `{project}: {task}`.)
10. **Do you have a dashboard or report regeneration command to run after updates?** If yes, what's the command?
11. **Do you have team members who get assigned tasks?** If yes, comma-separated names. (Skip if you handle every assignment yourself.)

#### Block E — Output preferences

12. **Detail level for the review summary:**
    - Concise — decisions and new tasks only
    - Standard — decisions, tasks, blockers, timeline, completed
    - Detailed — everything including context/intel and open questions
13. **Sections to include** (multi-select):
    - Attendees
    - Decisions / approvals
    - Completed items
    - New tasks
    - Blocker updates
    - Timeline / due-date changes
    - Risk register updates
    - Open questions / unresolved threads
    - Context / intel
    - Direct quotes (verbatim, as evidence)
14. **Format style:**
    - Bullets only
    - Mixed (short prose intros + bullets)
    - Prose only
15. **Tone:** neutral / terse / conversational
16. **Where should the rendered review summary be saved?** Free-text path template with placeholders (e.g. `{project_path}/docs/meetings/review-{date}.md`). Available placeholders: `{project_path}`, `{client}`, `{project}`, `{date}`. Or answer "don't save" to skip.

### Step 0.3: Save Config

Read `config.example.json` from the same directory. Use it as a template. Fill in every answer. Set `"configured": true`. Save as `config.json` in the skill directory.

Confirm to the user: "Setup saved. Config is at `config.json` in the skill folder — edit it directly any time, or ask me to re-run setup. Running your first meeting review now."

### Step 0.4: Load Config

Load `config.json`. The rest of the skill reads from `config` to decide which systems to update. Whenever you see `{user_name}`, `{project_state_master_path}`, `{notion_tasks_db_id}` etc. below, substitute the value from config. Skip any stage or step whose enabling system has `enabled: false`.

---

## Stage 1: Locate & Read the Transcript

**Goal:** Get the transcript into context, identify the client/project(s), and extract every PM-relevant item.

### Step 1.1: Decide the Source

Look at `config.transcript_sources`. Three paths:

- **Exactly one source configured** → use it.
- **Multiple sources configured** → ask the user which source this transcript is from (single-select via AskUserQuestion).
- **Only manual configured** → go straight to Step 1.3.

### Step 1.2: Fetch via Connector

Every MCP-based source (Fathom, Fireflies, Otter, Read AI, Grain, Another MCP) follows the same uniform pattern from config:

1. Build the list-tool name as `<namespace_prefix><list_tool>` (e.g. `mcp__fathom__list_meetings`). If that tool isn't loaded, fetch it via ToolSearch first (`query: "select:<full_tool_name>"`).
2. Call it with a lookback equal to `<source>.lookback_days` plus any `extra` filters from config.
3. Present the returned meetings to the user via AskUserQuestion. Confirm the pick.
4. Build the fetch-tool name as `<namespace_prefix><fetch_tool>` and call it with the chosen meeting ID.

**Notion meeting notes** is the one exception — it doesn't use list/fetch tools. Instead: query the configured `database_id` filtered to recent rows → present to user → fetch the chosen page's content.

If the connector returns more than one transcript and the user has not pre-named the meeting, always confirm the pick before downloading.

### Step 1.3: Manual Paste / File

Ask the user to paste the transcript, drop a file path, or upload. Read into context. If the transcript is very long (>~50KB), summarise via a subagent before extraction.

If `config.transcript_sources.manual.save_folder` is set, offer to save a copy of the pasted transcript at `{save_folder}/transcript-{date}.md`.

### Step 1.4: Identify Client / Project / Attendees

From the transcript, determine:
- **Which client / project** this relates to. Use `config.business.active_projects_source` to disambiguate when there are multiple plausible matches.
- **Who was on the call** — names and roles where stated.

Present this back to the user via AskUserQuestion: "This looks like a [Client/Project] meeting with [Attendees]. Correct?" Confirm before continuing.

### Step 1.5: Extract PM Items

Scan the transcript for anything that affects project status. Categorise each item as one of:

- **Completed** — confirmed as done during the meeting
- **Decided / approved** — a decision was made
- **New task** — needs doing, wasn't already tracked
- **Blocker update** — blocker resolved, or a new one identified
- **Timeline change** — date shifted, milestone moved, deadline set
- **Open question** — unresolved thread that needs follow-up
- **Context / intel** — important background, not a task

Capture everything. Filtering happens in Stage 3 and Stage 5. Present the extracted items grouped by category.

---

## Stage 2: Cross-Reference Project State Files

**Goal:** Compare extracted items against current COS state.

**Skip this stage entirely if `systems.project_state.enabled` is false.**

### Step 2.1: Read the Master State File

Read the file at `{project_state_master_path}` and find the relevant client / project section(s). Pull current status, summary, next deliverable, blockers, next up, risks. Note matches against extracted items.

### Step 2.2: Read Project-Level State

If `systems.project_state.project_state_pattern` is set, resolve the pattern for the current client / project and read those files. This is where detailed state lives.

### Step 2.3: Flag Differences

For each extracted item, note one of:
- Already tracked and up to date — no change needed
- Tracked but stale — status, detail, or date needs updating
- Not tracked — needs adding to one or more systems
- Contradicts current data — flag for the user to resolve

---

## Stage 3: Clarify Gaps

**Goal:** Ask about anything ambiguous before proposing changes.

Use AskUserQuestion to address:
- Items where the transcript is unclear about scope, owner, or timing
- Owner assignment for new tasks (offer choices from `team.owners`)
- Priority for new tasks (offer levels from `task_defaults.priority_levels`)
- Whether a "completed" item should be marked complete in task systems or just noted in COS
- Any context items where you're unsure if they're significant enough to capture

Bundle related questions. Aim for 1-2 rounds max. If the transcript is unambiguous, skip this stage and say so.

---

## Stage 4: Cross-Reference Task Systems

**Goal:** Check what already exists in configured task systems so the update plan is accurate.

### Step 4.1: Check Notion (if `systems.notion.enabled`)

Search the Tasks DB at `{notion_tasks_db_id}` for existing tasks tied to this project. For each extracted item:
- Does a matching task exist?
- If yes, current status — does it need updating?
- If no, should one be created?

### Step 4.2: Check Reclaim (if `systems.reclaim.enabled`)

`reclaim_list_tasks` to get active tasks. Filter using the configured naming convention `{task_naming_convention}`. For each extracted item:
- Match against existing tasks?
- Time estimate, due date, or priority changes?
- Any to mark complete?
- Any to create?

If the Reclaim list is large (>50KB), save it to a file and parse with a subagent or script.

---

## Stage 5: Render Summary + Propose Update Plan

**Goal:** Produce the user-facing meeting review summary AND the proposed system updates. Wait for approval before any writes.

### Step 5.1: Render the Summary

Build the summary using `output_preferences`:
- **Sections** — include only what's in `output_preferences.sections`
- **Detail level** — `concise` keeps decisions + new tasks, `standard` adds blockers/timeline/completed, `detailed` adds context/intel/open questions
- **Format** — bullets, mixed, or prose
- **Tone** — neutral, terse, or conversational

Default standard layout (drop sections the user excluded):

```
# Meeting Review — [Client / Project] — [Date]

**Attendees:** ...
**Duration:** ...

## Decisions
- ...

## Completed
- ...

## New Tasks
- ...

## Blockers
- ...

## Timeline
- ...

## Open Questions
- ...

## Context / Intel
- ...
```

### Step 5.2: Propose System Updates

Below the summary, list every proposed change grouped by system. Only include sections for enabled systems.

```
## Proposed Updates

### project state file changes (if enabled)
1. [Client > Project]: update summary — [old → new]
2. [Client > Project]: add to next_up — "[new task]"

### Project state changes (if enabled)
1. Update deliverable status: [item] → [new status]

### Notion (if enabled)
1. Mark complete: "[Task]" (ID: xxx)
2. Update status: "[Task]" → In Progress (ID: xxx)
3. Create: "[Task]" — [priority, est hours]

### Reclaim (if enabled)
1. Mark complete: "[Task]" (ID: xxx)
2. Update due date: "[Task]" → [new date] (ID: xxx)
3. Create: "[Task]" — [priority, hours, due date]
```

### Step 5.3: Approval Gate

Ask the user to approve via AskUserQuestion. Options: "Approved, go ahead", "Needs changes" (free-text the changes).

**Do NOT proceed to Stage 6 until approved.**

---

## Stage 6: Execute Updates

**Goal:** Apply approved changes, one system at a time, then save the summary.

Execute in this order: master state → project state → external task systems → dashboard → save summary.

### Step 6.1: Update Master COS (if enabled)
Edit `{project_state_master_path}` with targeted Edits. Confirm changes.

### Step 6.2: Update Project State (if enabled)
Edit project-level files. Confirm changes.

### Step 6.3: Update Notion (if enabled)
- Mark complete: `notion-update-page` (status property)
- Update existing: `notion-update-page` (relevant properties)
- Create new: `notion-create-pages` with title, status, priority, applicable categories from `task_defaults.categories`. Link to Project Dashboard via `{project_dashboard_db_id}` if configured.

### Step 6.4: Update Reclaim (if enabled)
- Mark complete: `reclaim_mark_complete`
- Update existing: `reclaim_update_task`
- Create new: `reclaim_create_task` using `{task_naming_convention}`, time in 15-minute chunks, timezone-aware due dates, priority from `task_defaults.priority_levels`, category WORK.

### Step 6.5: Regenerate Dashboard (if enabled)
Run `{regenerate_cmd}` from `systems.dashboard`.

### Step 6.6: Save Summary
If `output_preferences.save_path_template` is set, render the placeholders (`{project_path}`, `{client}`, `{project}`, `{date}`) and Write the summary to that path. Confirm the file path.

### Step 6.7: Final Report

```
# Meeting Review Complete — [Client / Project] — [Date]

**Systems updated:**
- [system 1] — [X] changes
- [system 2] — [X] changes

**Summary saved:** [path or "not saved"]

**Key changes:**
- [Most important change 1]
- [Most important change 2]
```

---

## Important Notes

- **Don't assume — ask.** Ambiguous transcript = Stage 3 question, not a guess.
- **Capture the transcript.** If a copy isn't already saved, drop one in the configured folder (or alongside the summary) so future runs can re-reference.
- **Multiple projects from one meeting.** Handle each project's updates separately but present them in one unified plan in Stage 5.
- **Tasks across systems.** If both Notion and Reclaim are enabled, every actionable task should land in both. Don't create in one and forget the other.
- **Language variant.** Use `{language_variant}` from config for spelling and phrasing in all generated text.
- **Re-running setup.** To redo setup, edit `config.json` and set `"configured": false`, or delete `config.json`. Stage 0 will run again on next invocation.
- **Adding a new system or transcript source later.** Edit `config.example.json` to add the new entry shape, then either run setup again or hand-edit `config.json`. Wrap any new logic in this skill with `if systems.<name>.enabled` so the skill stays portable for users without it.
