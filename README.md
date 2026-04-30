# meeting-review

A Claude Code / Claude Cowork skill that turns meeting transcripts into actionable PM updates and syncs them across your stack.

## Why this exists

Most meeting-summary tools give you a tidy list of action items in their own UI. The action items don't make it to your task tracker, your project state file, your calendar, or your dashboard. So you re-type everything, or you don't.

This skill closes that loop. It pulls a transcript from any connected meeting MCP (Fathom, Fireflies, Otter, Read AI, Grain, Notion meeting notes) or accepts a manual paste, extracts every PM-relevant item, and walks through proposed updates to each downstream system one at a time. **Every change is reviewed before it's applied.**

## What you get

A skill that runs in two phases:

### First run: setup (one-time)

Configures:

- **Transcript sources.** Which meeting MCPs you have connected, and how to fetch from each.
- **Business context.** Who's in your meetings (clients, internal team, stakeholders) so the skill can correctly attribute action items.
- **Downstream systems.** Where updates go: COS files (`cos.yaml`), Notion task databases, Reclaim tasks, calendar events, dashboards, custom files.
- **Output preferences.** Format for summaries, urgency thresholds, what to ask vs auto-apply.

The setup writes a `config.json` alongside the skill so future runs skip discovery.

### Every later run: process

1. **Fetch transcript** from your chosen source (or paste).
2. **Analyse** for PM-relevant items: action items, decisions, blockers, status changes, deadlines, scope changes, new risks.
3. **Clarify gaps.** If the transcript is ambiguous (who owns this?, when is this due?), the skill asks before applying.
4. **Propose updates.** One downstream system at a time, the skill shows the diff and asks for approval.
5. **Apply on approval.** Writes only the approved changes. Logs everything.

## Use cases

| Trigger | Outcome |
|---|---|
| Drop a Fathom link or transcript | Full PM pipeline runs |
| "Process this meeting" | Same as above with manual context |
| "Update from Tuesday's call" | Pulls latest meeting from configured source |
| "Sync meeting notes" | Reapplies a previously-processed transcript to new downstream targets |

## Supported transcript sources

Anything reachable via MCP. Tested with:

- **Fathom** (via Fathom MCP)
- **Fireflies**
- **Otter.ai**
- **Read AI**
- **Grain**
- **Notion meeting notes** (via Notion MCP)
- **Manual paste** (fallback for any source)

The skill detects which MCPs are connected at runtime and uses what's available.

## Supported downstream systems

- `cos.yaml` files (project state)
- Notion databases (tasks, decisions, projects)
- Reclaim tasks
- Google Calendar events
- Dashboards (Next.js, custom)
- Slack DMs / channels (for status updates)
- Plain markdown files (handoff notes, decision logs)

Each one is configured during setup. The skill never touches a system that wasn't approved.

## Installation

### Claude Code (CLI)

User-level:

```bash
git clone https://github.com/ai-orchestrators/meeting-review.git ~/.claude/skills/meeting-review
```

Project-level:

```bash
git clone https://github.com/ai-orchestrators/meeting-review.git .claude/skills/meeting-review
```

### Claude Cowork (Desktop)

Drop the `meeting-review/` directory into `~/.claude/skills/`.

### First run

Run the skill once. It walks through setup interactively and writes `config.json`. Plan ~10 minutes for setup. Future runs are fast.

A `config.example.json` ships with the repo so you can see the shape before running setup.

## Trigger phrases

- "meeting review"
- "process this meeting"
- "review this transcript"
- "update from meeting"
- "sync meeting notes"
- "extract action items"

## License

MIT
