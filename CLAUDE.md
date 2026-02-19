# CLAUDE.md — Project Conventions

## Project Overview

Universal Death Spiral is an adversarial LLM GM'd solo TTRPG engine. A Go TUI binary runs the game client; an LLM (Claude via OpenRouter) serves as Game Master. The prompt defines the rules, the state schema is the interface contract, and a tag-based dice system provides a mechanical death spiral.

See `RAG/RFC/Initial-RFC.md` for the full V1 design.

## Repository Layout

```
RAG/
├── RFC/              # Request-for-comment design docs
├── AI-EPIC/          # Epic tickets (feature-level work items)
├── AI-IMP/           # Implementation tickets (task-level work items)
├── AI-LOG/           # Session development logs
├── templates/        # Templates for EPIC, IMP, and LOG files
├── scripts/
│   └── generate-index.sh   # Builds INDEX.md from ticket frontmatter
└── INDEX.md          # Auto-generated work tracking index (do not edit manually)
```

## Work Tracking System

### Generating the Index

Run `./RAG/scripts/generate-index.sh` to regenerate `RAG/INDEX.md`. The script:

1. Normalizes frontmatter field names across all tickets (fixes drift like `kanban-status` → `kanban_status`)
2. Sorts EPICs into status buckets (In Progress, Planned, Deferred, Completed)
3. Links IMPs to their parent EPICs via `depends_on` or `parent_epic` fields
4. Auto-populates `parent_epic:` backlinks in IMP files
5. Detects anomalies: orphaned IMPs, status mismatches, illegal statuses, large files

### Legal Kanban Statuses

Statuses are **lowercase**. The canonical values (per templates and `generate-index.sh`) are:

#### EPIC statuses

| Status | Meaning |
|---|---|
| `planned` | Scoped but not started (also accepts `backlog` or empty) |
| `in-progress` | Actively being worked (also accepts `in_progress`) |
| `deferred` | Intentionally postponed |
| `completed` | Done (also accepts `complete`). Set `date_completed:` when closing. |

#### IMP statuses

| Status | Meaning |
|---|---|
| `backlog` | Identified but not yet scheduled |
| `planned` | Scheduled for an upcoming sprint |
| `in-progress` | Actively being worked (also accepts `in_progress`) |
| `deferred` | Intentionally postponed |
| `completed` | Done (also accepts `complete`). Set `date_completed:` when closing. |
| `cancelled` | Will not be done |

**Any other value is flagged as an illegal status in the index anomalies section.**

### Ticket Lifecycle Rules

1. **Always update the "Issues Encountered" section** on a ticket with any problems, blockers, deviations, or failed approaches encountered during work. This is mandatory per the IMP template and ensures continuity across sessions.
2. **Close tickets before starting the next sprint.** If work on a ticket is complete, set `kanban_status: completed` and fill in `date_completed:` before moving on to new work. Do not leave stale in-progress tickets behind.
3. **Orphaned IMPs are anomalies.** Every active IMP should link to a parent EPIC via `depends_on:` or `parent_epic:`. Completed one-off IMPs without a parent are acceptable.
4. **Status mismatches are anomalies.** An open IMP under a completed EPIC indicates something was missed. Resolve before proceeding.

### Creating Tickets

- Use templates from `RAG/templates/` (AI-EPIC.md, AI-IMP.md, AI-LOG.md)
- Place files in the corresponding `RAG/AI-EPIC/`, `RAG/AI-IMP/`, or `RAG/AI-LOG/` directories
- Naming convention: `AI-EPIC-NNN-title-in-kebab-case.md`, `AI-IMP-NNN-title-in-kebab-case.md`
- Sub-tickets use: `AI-IMP-NNN-N-title-in-kebab-case.md`

### Session Logs

At the end of each working session, generate a log using the `RAG/templates/AI-LOG.md` template. Logs must:
- Identify all worked tickets
- List major files edited
- Document issues encountered
- Provide next steps as a handoff to the next session

## Sprint Workflow

1. Review `RAG/INDEX.md` (regenerate first) to understand current state
2. Pick up in-progress work or pull from planned backlog
3. Work tickets, updating "Issues Encountered" as problems arise
4. On completion: set `kanban_status: completed`, fill `date_completed:`
5. Generate a session log (`RAG/AI-LOG/`)
6. Regenerate the index before closing out
