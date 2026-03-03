---
node_id: AI-IMP-001-4
tags:
  - IMP-LIST
  - Implementation
  - phase-1
  - game-loop
  - stdin-stdout
kanban_status: planned
depends_on: [[AI-IMP-001-3-tool-executor-and-roll-check]]
parent_epic: [[AI-EPIC-001-project-scaffolding-and-bare-api-loop]]
confidence_score: 0.85
date_created: 2026-03-03
date_completed:
---

# AI-IMP-001-4-bare-stdin-stdout-game-loop

## Wire the Bare Stdin/Stdout Game Loop

This is the integration ticket — it wires config, provider, and tools into a working conversation loop. When done, you can actually play (loosely) with Claude through a terminal. No TUI, no state sidebar, no persistence — just raw text in, narration out, with dice rolls happening in between.

This ticket proves the entire Phase 1 thesis: the tool-call round-trip works end to end.

**Done when:** Running the binary with a valid `.env` starts an interactive session. The player types actions, Claude calls `roll_check` when appropriate, the binary rolls dice and returns results, Claude narrates, and the conversation continues until the player exits with Ctrl+C.

### Out of Scope

- TUI rendering (EPIC-005)
- State persistence or `state.json` (EPIC-002)
- Segment logging / JSONL (EPIC-002)
- The real game prompt (EPIC-003) — we use a minimal test prompt
- Streaming responses (EPIC-005)
- Graceful error recovery / retries (EPIC-008)
- Context budget management (EPIC-002+)

### Design/Approach

**The loop** lives in `internal/game/loop.go`. It's a `Loop` struct that holds a `Provider`, a tool `Registry`, and the conversation history (a `[]api.ChatMessage` slice that grows over the session).

**Conversation flow per turn:**

```
1. Read one line from stdin (bufio.Scanner)
2. Append as { role: "user", content: line } to history
3. Build ChatRequest: system prompt + history + tool definitions
4. Send to provider (ChatCompletion)
5. Check finish_reason:
   a. "tool_calls" →
      - Append assistant message (with tool calls) to history
      - For each tool call: execute via registry, append { role: "tool", tool_call_id, content: result } to history
      - Go to step 3 (re-send with updated history)
   b. "stop" →
      - Append assistant message to history
      - Print message content to stdout
      - Go to step 1 (next player input)
   c. "length" → print warning that response was truncated, treat as "stop"
   d. error → print error, continue to step 1
```

**System prompt (minimal test):**
```
You are a Game Master for a solo RPG. When the player attempts
something risky or uncertain, use the roll_check tool. Set
action_dice based on how favorable the situation is (1-4) and
danger_dice based on threats (0-3). Narrate the result based on
the highest die: 6=great success, 5-4=success with complication,
3=failure, 2=bad failure, 1=catastrophe. Keep narration concise.
```

This is a constant string in the game loop package. It's explicitly disposable — the real prompt is EPIC-003.

**Context assembly:** For Phase 1, context is simple — system prompt as the first message, then the full conversation history. No scratchpad injection, no state.json, no summaries. Those come in later phases.

**Signal handling:** The binary should catch `SIGINT` (Ctrl+C) and exit cleanly with a goodbye message rather than dumping a stack trace.

**Wiring in `main.go`:**
```go
func main() {
    cfg := config.Load()              // IMP-001-1
    provider := api.NewOpenRouterProvider(cfg) // IMP-001-2
    registry := tools.NewRegistry(    // IMP-001-3
        tools.NewRollCheck(),
    )
    loop := game.NewLoop(provider, registry) // this IMP
    loop.Run()
}
```

### Files to Touch

`internal/game/loop.go`: new — `Loop` struct, `NewLoop()`, `Run()` method, tool-call dispatch loop
`main.go`: modify — wire config → provider → registry → loop, add signal handling

### Implementation Checklist

<CRITICAL_RULE>
Before marking an item complete on the checklist MUST **stop** and **think**. Have you validated all aspects are **implemented** and **tested**?
</CRITICAL_RULE>

- [ ] Create `Loop` struct in `internal/game/loop.go` with fields: `provider api.Provider`, `registry *tools.Registry`, `history []api.ChatMessage`
- [ ] Define the minimal test system prompt as a package-level constant
- [ ] Implement `NewLoop(provider, registry)` constructor
- [ ] Implement `Run()` method with the main loop:
  - Print welcome message with instructions
  - Read player input line by line from stdin using `bufio.Scanner`
  - Skip empty lines
  - Append user message to history
  - Call `sendAndProcess()` for the API call + tool loop
- [ ] Implement `sendAndProcess()`:
  - Build `ChatRequest` with system message + history + tool definitions
  - Call `provider.ChatCompletion()`
  - If `finish_reason == "tool_calls"`: execute each tool, append results, recurse/loop
  - If `finish_reason == "stop"`: append assistant message, print content
  - Handle `finish_reason == "length"` with a truncation warning
- [ ] Implement tool-call execution within the loop: for each `ToolCall` in the response, call `registry.Execute()`, build a tool result message with matching `ToolCallID`
- [ ] Print tool execution activity to stderr (e.g., `[roll_check] Rolling 2 net dice...`) so the player sees what's happening without it mixing into the narration
- [ ] Add signal handling in `main.go`: catch SIGINT, print goodbye, exit 0
- [ ] Update `main.go` to wire config → provider → registry → loop and call `loop.Run()`
- [ ] Manual integration test: run binary with valid `.env`, type "I try to sneak past the guard", verify Claude calls `roll_check`, binary rolls dice, Claude narrates the result
- [ ] Verify multi-turn conversation works (second action after first completes)
- [ ] Verify Ctrl+C exits cleanly

### Acceptance Criteria

**Scenario:** Complete tool-call round-trip.
**GIVEN** the binary is running with a valid `.env`.
**WHEN** the player types "I try to pick the lock on the chest."
**THEN** Claude responds with a `roll_check` tool call.
**AND** the binary executes the roll and prints a status line to stderr (e.g., `[roll_check] 3 action vs 1 danger → net 2, rolled [4, 5], highest: 5`).
**AND** Claude receives the roll result and narrates the outcome.
**AND** the narration is printed to stdout.

**Scenario:** Free narration (no roll needed).
**GIVEN** the binary is running.
**WHEN** the player types "I look around the room."
**THEN** Claude narrates freely without calling any tools.
**AND** the narration is printed to stdout.

**Scenario:** Multi-turn conversation.
**GIVEN** the binary is running and the player has completed one action.
**WHEN** the player types a second action.
**THEN** Claude's response accounts for the previous turn's context (conversation history is maintained).

**Scenario:** Clean exit.
**GIVEN** the binary is running.
**WHEN** the player presses Ctrl+C.
**THEN** the binary prints a goodbye message and exits with code 0 (no stack trace).

### Issues Encountered
<!--
The comments under the 'Issues Encountered' heading are the only comments you MUST not remove
This section is filled out post work as you fill out the checklists.
You SHOULD document any issues encountered and resolved during the sprint.
You MUST document any failed implementations, blockers or missing tests.
-->
