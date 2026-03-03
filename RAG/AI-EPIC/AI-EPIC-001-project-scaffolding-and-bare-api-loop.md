---
node_id: AI-EPIC-001
tags:
  - EPIC
  - AI
  - phase-1
  - scaffolding
  - api-loop
date_created: 2026-03-03
date_completed:
kanban_status: planned
AI_IMP_spawned:
  - AI-IMP-001-1
  - AI-IMP-001-2
  - AI-IMP-001-3
  - AI-IMP-001-4
---

# AI-EPIC-001-project-scaffolding-and-bare-api-loop

## Problem Statement/Feature Scope

No project code exists yet. Before building state management, a TUI, or game logic, we need to prove the fundamental loop works: a player types something, Claude responds with tool calls, the binary executes those tools, returns results, and Claude narrates the outcome. This is the single riskiest integration point — if the tool-call round-trip doesn't work cleanly through OpenRouter, nothing else matters.

A secondary goal is establishing the Go project structure and a provider abstraction layer so we aren't locked to a single API provider. OpenRouter is primary, but the architecture should allow swapping to the direct Anthropic API (for prompt caching) or others later.

## Proposed Solution(s)

Build a Go binary that runs a bare stdin/stdout conversation loop against Claude via OpenRouter. The binary:

1. Loads configuration from `.env` (API key, model name)
2. Sends player input to Claude with tool definitions attached
3. When Claude responds with tool calls, executes them locally and returns results
4. Loops until Claude produces final narration, then prints it
5. Repeats for the next player input

The implementation is split into four layers:
- **Config** — `.env` loading via `joho/godotenv`
- **Provider** — Abstract `Provider` interface with an OpenRouter implementation using `sashabaranov/go-openai`
- **Tools** — Registry + dispatch framework with `roll_check` as the first real tool
- **Game loop** — Stdin/stdout conversation loop wiring everything together

A minimal test prompt (not the real game prompt) triggers tool usage. The real prompt is authored in EPIC-003.

See `RAG/RFC/Initial-RFC.md` §2, §5, §6, §16 for the design context.

## Path(s) Not Taken

- **Streaming responses:** Deferred to EPIC-005 (TUI). Non-streaming simplifies the tool-call loop for Phase 1. The `Provider` interface declares `ChatCompletionStream` but the loop doesn't use it yet.
- **State persistence:** No `state.json` read/write. `roll_check` is stateless. State comes in EPIC-002.
- **Real game prompt:** A throwaway test prompt is used. The full game prompt is EPIC-003.
- **Hand-rolled HTTP client:** We use `sashabaranov/go-openai` rather than building our own. Less control, but dramatically less code and the tool-call types are already defined.
- **Official OpenRouter SDK:** Beta quality with sparse tool-call documentation. Too risky for the foundation layer.

## Success Metrics

1. `go build .` compiles without errors and `go vet ./...` passes.
2. With a valid `.env`, the binary starts and accepts player input from stdin.
3. Typing a risky action (e.g., "I try to sneak past the guard") causes Claude to call `roll_check`.
4. The binary executes the dice roll, returns the result, and Claude narrates the outcome.
5. The conversation continues across multiple turns without error.
6. Ctrl+C exits cleanly.

## Requirements

### Functional Requirements

- [ ] FR-1: The binary shall load API configuration from a `.env` file (API key, model, max tokens).
- [ ] FR-2: The binary shall define a `Provider` interface abstracting LLM API calls, with an OpenRouter implementation.
- [ ] FR-3: The binary shall define a `Tool` interface and registry for dispatching tool calls by name.
- [ ] FR-4: The binary shall implement `roll_check` per RFC §5 — net dice calculation, d6 rolling, botch detection.
- [ ] FR-5: The binary shall run a stdin/stdout loop: read input → send to Claude with tools → execute tool calls → return results → print narration → repeat.
- [ ] FR-6: The tool-call loop shall handle multiple sequential tool calls in a single turn (Claude calls tool A, gets result, calls tool B, etc.).
- [ ] FR-7: The binary shall use a minimal test system prompt sufficient to trigger tool usage.

### Non-Functional Requirements

- NFR-1: All application code lives under `internal/` (Go convention — compiler-enforced encapsulation).
- NFR-2: The provider interface must be implementation-agnostic. No OpenRouter-specific types leak into the tool executor or game loop.
- NFR-3: Internal message types use the OpenAI-compatible format (this is OpenRouter's native format). A future Anthropic provider translates at the boundary.
- NFR-4: Tool call IDs are treated as opaque strings (OpenRouter may return `toolu_` or `call_` prefixes).
- NFR-5: `max_tokens` is set explicitly on every API request (Anthropic model requirement).
- NFR-6: Tools array is re-sent on every request (OpenRouter requirement).

## Implementation Breakdown

| IMP | Title | Status |
|---|---|---|
| [[AI-IMP-001-1-project-scaffolding]] | Project Scaffolding | planned |
| [[AI-IMP-001-2-provider-abstraction-and-openrouter-client]] | Provider Abstraction & OpenRouter Client | planned |
| [[AI-IMP-001-3-tool-executor-and-roll-check]] | Tool Executor Framework & roll_check | planned |
| [[AI-IMP-001-4-bare-stdin-stdout-game-loop]] | Bare Stdin/Stdout Game Loop | planned |
