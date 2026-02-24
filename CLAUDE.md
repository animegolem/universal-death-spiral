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

## V1 Work Order

Derived from RFC §16 Build Order. EPICs will be created from `RAG/templates/AI-EPIC.md` as each is started — not all up front. Open playtesting questions (RFC §14) are deferred to gameplay iteration.

### Key Design Decisions (from RFC review)

- **API provider:** OpenRouter primary, but build a provider abstraction layer so we can swap to direct Anthropic API (better prompt caching) or others later.
- **NPC world files:** Deferred. `create_npc` writes to `state.json` only in V1. A thin deferred EPIC tracks the future `world/npcs/` markdown + lorebook system.
- **Game-folder git:** Keep in build order — low lift, high value (turn-level undo, state diffs).
- **Prompt tuning:** The prompt is 60% of the project but is pure game design. It's iterated via playtesting once the tool loop and state layer exist.
- **Target audience note:** Primary developer is not a traditional programmer. EPICs and IMPs should explain the "why" behind architectural decisions, not just the "what."

### Build Sequence

| Phase | EPIC | Summary | Status |
|---|---|---|---|
| 1 | Project Scaffolding & Bare API Loop | Go module, project structure, stdin/stdout loop proving tool-call round-trip works. Provider abstraction layer. | planned |
| 2 | Tool Suite & State Persistence | All 10 tools, `state.json` read/write, scratchpad ops, JSONL segment logging. | planned |
| 3 | Game Prompt Authoring | Write `prompt.md` — oracle table, tag rules, GM behavioral instructions. Iterative; needs phases 1+2. | planned |
| 4 | Setup Flow | Hardcoded setup prompt, conversational character creation, transition to play mode. | planned |
| 5 | TUI (Bubble Tea) | Chat pane, state sidebar, dice display, activity indicators. TUI ergonomics TBD when expanded. | planned |
| 6 | Git Integration | Auto-init game folder, auto-commit per turn with formatted messages. | planned |
| 7 | Compression Pass | Haiku call on day transition, summary generation, logbook review injection. | planned |
| 8 | Polish & Error Handling | Graceful recovery, edge cases, UX refinements. | planned |
| 9 | NPC World Files & Lorebook | `create_npc` writes markdown to `world/npcs/`, future retrieval system. | deferred |

## Sprint Workflow

1. Review `RAG/INDEX.md` (regenerate first) to understand current state
2. Pick up in-progress work or pull from planned backlog
3. Work tickets, updating "Issues Encountered" as problems arise
4. On completion: set `kanban_status: completed`, fill `date_completed:`
5. Generate a session log (`RAG/AI-LOG/`)
6. Regenerate the index before closing out
