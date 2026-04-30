# meeting-review

A portable meeting-review skill. Pulls transcripts from a connected MCP (Fathom, Fireflies, Otter, Read AI, Grain, Notion meeting notes, or any other meeting MCP) or accepts a manual paste, extracts every PM-relevant item, and walks through updating downstream systems (COS files, Notion tasks, Reclaim, dashboards).

The skill itself contains zero personal or business-specific data. Everything that varies by user — transcript sources, business context, downstream systems, output preferences — is collected on first run and stored in `config.json`.

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | The skill itself. Stage 0 handles first-run setup. Stages 1-6 do the work on every run. |
| `config.example.json` | Template config. Never edited directly — copied to `config.json` during setup. |
| `config.json` | User-specific config. Created on first run by Stage 0. Read on every later run. **Do not commit this file** if the skill ships in a public plugin. |

## How first-run setup works

1. On invocation, Stage 0 checks for `config.json`.
2. If missing or `"configured": false`, runs through five blocks of questions via `AskUserQuestion`:
   - **A** — Identity & locale (name, language variant)
   - **B** — Transcript source(s): Fathom, Fireflies, Otter, Read AI, Grain, Notion meeting notes, another MCP, manual paste — pick one or many
   - **C** — Business context: single brand vs multi-brand vs agency vs internal, project taxonomy, active-projects source
   - **D** — Downstream systems: COS files, Notion tasks DB, Reclaim, dashboard, team owners
   - **E** — Output preferences: detail level, sections to include, format, tone, save-path template
3. Saves answers to `config.json` with `"configured": true`.
4. Continues to Stage 1 and runs the first review.

On every later run, Stage 0 silently loads config and skips straight to Stage 1.

## Transcript sources at a glance

Every MCP-based connector uses the same uniform shape: `namespace_prefix`, `list_tool`, `fetch_tool`, `lookback_days`. Only Fathom ships with verified defaults pre-filled — the rest get filled during Stage 0.

| Source | Type | Notes |
|---|---|---|
| Fathom | MCP | Defaults pre-filled (`mcp__fathom__list_meetings` / `get_transcript`). Override if your install differs. |
| Fireflies.ai | MCP | User provides namespace prefix and tool names during setup. |
| Otter.ai | MCP | User provides namespace prefix and tool names during setup. |
| Read AI | MCP | User provides namespace prefix and tool names during setup. |
| Grain | MCP | User provides namespace prefix and tool names during setup. |
| Notion meeting notes | Notion MCP | Different shape — queries a Notion database for recent meeting notes pages. |
| Another MCP connector | MCP | Catch-all for Granola, Avoma, Krisp, Gong, Zoom, etc. User provides service name + namespace + tool names. |
| Manual paste / file | None | Always available. Use for any source without a connected MCP — paste text, drop a file path, or upload. |

You can enable as many sources as you use. When more than one is enabled, the skill asks which source the current transcript is from before fetching. If you don't know an MCP's tool names, run `ToolSearch` with the service name (e.g. `"fireflies"`) to discover them, then complete setup.

## Re-running setup

To force the setup flow again:
- Edit `config.json` and set `"configured": false`, or
- Delete `config.json`

Next invocation will trigger Stage 0.

## Adding a new system or transcript source later

1. Add a new entry under `transcript_sources` or `systems` in `config.example.json` with `enabled: false` by default.
2. Add a question for it in Stage 0 of `SKILL.md` (Block B for transcript sources, Block D for downstream systems).
3. For transcript sources: add a fetch branch in Stage 1.2.
4. For downstream systems: add a Stage 4 reference check (if applicable) and a Stage 6 execution step.
5. Wrap new logic with `if <enabled>` so the skill stays portable for users without that system.

## How to integrate with a plugin customizer

When this skill is bundled into a plugin, the customizer can:

1. Detect `config.example.json` in the skill directory.
2. Walk the user through filling it during plugin install.
3. Write the result to `config.json` and set `"configured": true`.

The user gets the same one-time setup whether the skill is installed standalone (Stage 0 handles it on first run) or via plugin install (the customizer handles it up front).
