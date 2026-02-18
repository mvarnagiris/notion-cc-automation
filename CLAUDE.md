# Notion Projects Automation

This repository implements a headless Claude Code automation system that polls a Notion Projects database and performs work based on task status transitions.

## What This Does

A poller watches the Notion **Projects** database (`722af2f84576437c9a0bfcd9e64a40e6`). When a task reaches a trigger status, a headless Claude Code session is launched with the appropriate agent to perform the work, update the Notion page, and advance the status.

The human drives priority and advancement by setting statuses. CC does the work.

## Status Lifecycle

Each task flows through phases. The human sets the initial phase status; CC transitions through Working → Review (or Questions if blocked).

```
📝 Spec          →  [CC starts]  →  🔄 Spec Working   →  👀 Spec Review
                                                        →  ❓ Spec Questions  (human answers in comments, then resets to 📝 Spec)

🎨 Design        →  [CC starts]  →  🔄 Design Working  →  👀 Design Review
                                                        →  ❓ Design Questions

📋 Plan          →  [CC starts]  →  🔄 Plan Working    →  👀 Plan Review
                                                        →  ❓ Plan Questions

💻 Dev           →  [CC starts]  →  🔄 Dev Working     →  👀 Dev Review  (PR URL updated)
                                                        →  ❓ Dev Questions  (human answers in comments, then resets to 💻 Dev)

✅ Done / ❌ Cancelled  — terminal states, never trigger CC
```

**Working statuses** (`🔄 Spec Working`, `🔄 Design Working`, `🔄 Plan Working`, `🔄 Dev Working`) mean CC is actively running. The poller must never re-trigger tasks in these states.

**Review statuses** mean CC finished and a human needs to review. CC does not act on these.

**Questions statuses** mean CC was blocked. Questions are posted as **Notion page comments**. The human answers in comments and manually resets the status to the phase trigger (e.g., `❓ Spec Questions` → `📝 Spec`) to retry.

## Notion Database Schema

Database URL: `https://www.notion.so/722af2f84576437c9a0bfcd9e64a40e6`
Collection ID: `280386f0-df1c-4775-aee7-bfe21724a99a`

| Property | Type | Notes |
|----------|------|-------|
| Title | title | Task name |
| Status | select | Full list above |
| Type | select | `Task` or `Epic` |
| Project | select | `Lumme` or `Septynrankis` |
| Priority | select | `High`, `Medium`, `Low` |
| Epic | relation | Parent epic (same DB) |
| Blocked by | relation | Dependency |
| Blocks | relation | Reverse dependency |
| PR URL | url | Set by Dev agent after PR creation |
| Created | created_time | Auto |
| Last Updated | last_edited_time | Auto |

## Notion Page Structure

Each project page follows this content structure:

```
## Description
<human-written task description — the source of truth for all agents>

## Specification
<written by Spec agent>

## Design
<written by Design agent — includes uploaded screenshots>

## Implementation Plan
<written by Plan agent>

## Development
<written by Dev agent — progress notes and decisions>
```

Agents read earlier sections and write to their own section. Never overwrite the **Description** section.

## Project → Repository Mapping

| Notion Project | Local path | Notes |
|----------------|-----------|-------|
| `Lumme` | `~/projects/identifiers` | Monorepo. Primary work happens in `lumme/`. Shared code lives in `core/` — changes there may affect other apps. |
| `Septynrankis` | `~/projects/foodai` | Standalone repo. |

## Agents

Each phase has a dedicated agent prompt in `agents/`. The poller invokes CC with the appropriate agent file.

### `agents/spec-agent.md`
**Trigger:** `📝 Spec`

Reads the task's **Description** section and writes a formal specification into the **Specification** section including:
- Acceptance criteria (numbered, testable)
- Out of scope items
- Edge cases and error states
- Open questions (if any block progress → post as comment, set `❓ Spec Questions`)

### `agents/design-agent.md`
**Trigger:** `🎨 Design`

Reads Description + Specification. For Compose-based projects:
1. Identifies relevant screens/components from the spec
2. Runs Roborazzi screenshot tests to capture current and proposed states
3. Uploads screenshots to the **Design** section of the Notion page
4. Writes a brief design narrative (component breakdown, theming decisions, variants)

Requires the project repo to be available locally (see repo mapping).

### `agents/plan-agent.md`
**Trigger:** `📋 Plan`

Reads Description + Specification + Design. Writes a technical implementation plan into the **Implementation Plan** section:
- Files to create or modify (with paths)
- Step-by-step breakdown ordered by dependency
- Test plan

### `agents/dev-agent.md`
**Trigger:** `💻 Dev`

Reads the full page (Description through Implementation Plan). In the mapped repository:
1. Creates a feature branch (`feature/<task-title-slug>`)
2. Implements the plan step by step
3. Writes progress notes to the **Development** section as work proceeds
4. Creates a PR and updates `PR URL` on the Notion page
5. Sets status to `👀 Dev Review`

If blocked during implementation, posts a comment with the questions and sets status to `❓ Dev Questions`. Human answers in comments and resets to `💻 Dev` to retry.

## Skills

Reusable skills in `skills/` are available to all agents via Claude Code's skill system.

| Skill | Purpose |
|-------|---------|
| `skills/notion-update-status.md` | Update a project's Status property |
| `skills/notion-write-section.md` | Write content to a specific page section (Spec/Design/Plan/Dev) |
| `skills/notion-post-comment.md` | Post a comment on a page (used for questions) |
| `skills/notion-upload-images.md` | Upload local images to a Notion page section |
| `skills/notion-set-pr-url.md` | Update the PR URL property |
| `skills/git-create-branch.md` | Create and push a feature branch |
| `skills/git-create-pr.md` | Create a pull request and return its URL |

## Configuration

### Notion

- **Database URL:** `https://www.notion.so/722af2f84576437c9a0bfcd9e64a40e6`
- **Collection ID:** `280386f0-df1c-4775-aee7-bfe21724a99a`

### Poller

- **Interval:** 120 seconds
- **Alternative:** Notion webhooks can replace polling entirely — a webhook fires on every page update, eliminating the delay and unnecessary API calls. Worth switching to once the basic poller is working.

### Environment Variables

```
NOTION_TOKEN=          # Notion integration token
ANTHROPIC_API_KEY=     # For headless CC sessions
```

## Poller Behavior

The poller (to be implemented in `poller/`) must:

1. Query the database for tasks with trigger statuses: `📝 Spec`, `🎨 Design`, `📋 Plan`, `💻 Dev`
2. **Immediately** set the status to the corresponding Working state before launching CC (prevents double-triggering if polling overlaps)
3. Launch a headless CC session: `claude --print -p "$(cat agents/<phase>-agent.md)" ...` passing the Notion page URL as context
4. Log session output with task ID, phase, timestamp
5. Skip tasks in Working, Review, Questions, Done, or Cancelled states
6. Skip tasks where `Blocked by` contains unresolved dependencies (items not in `✅ Done`)

## Headless CC Invocation

Each agent is invoked as:

```bash
claude --print \
  --model claude-opus-4-6 \
  -p "$(cat agents/<phase>-agent.md)

---
## Task Context
- **Page URL:** <notion page url>
- **Project:** <Lumme|Septynrankis>
"
```

The agent prompt is the full contents of the agent file. The poller appends a `## Task Context` block at the end with the page URL and project name. Every agent reads its task context from this block.

The agent then uses the Notion MCP to fetch the full page and read all relevant sections (Description, Specification, etc.) from there. The page is the source of truth — not the invocation context, which is just enough to locate it.

## Repository Structure

```
├── CLAUDE.md                    # This file — source of truth for all config and conventions
├── agents/
│   ├── spec-agent.md            # Spec writing agent prompt
│   ├── design-agent.md          # Design + screenshot agent prompt
│   ├── plan-agent.md            # Implementation planning agent prompt
│   └── dev-agent.md             # Development agent prompt
├── skills/
│   ├── notion-update-status.md
│   ├── notion-write-section.md
│   ├── notion-post-comment.md
│   ├── notion-upload-images.md
│   ├── notion-set-pr-url.md
│   ├── git-create-branch.md
│   └── git-create-pr.md
└── poller/                      # Polling service (to be implemented)
```

## Key Invariants

- **Never overwrite the Description section** — it is the human-written source of truth
- **Always set Working status before starting work** — prevents race conditions in the poller
- **Never act on Review, Done, or Cancelled statuses** — these are terminal or human-gated
- **Questions go to Notion comments**, not inline in sections — keeps page content clean
- **Never commit secrets or credentials** — use environment variables only
- **Dev agent always works on a branch**, never directly on main/master
