---
node_id: AI-IMP-001-3
tags:
  - IMP-LIST
  - Implementation
  - phase-1
  - tools
  - roll-check
  - dice
kanban_status: planned
depends_on: [[AI-IMP-001-2-provider-abstraction-and-openrouter-client]]
parent_epic: [[AI-EPIC-001-project-scaffolding-and-bare-api-loop]]
confidence_score: 0.9
date_created: 2026-03-03
date_completed:
---

# AI-IMP-001-3-tool-executor-and-roll-check

## Tool Executor Framework and roll_check Implementation

The binary needs a way to register tools, dispatch incoming tool calls from the API to the right handler, and return results. This ticket builds that framework and implements `roll_check` as the first real tool.

`roll_check` is the ideal first tool because it's stateless — pure dice math with no dependency on `state.json`. It proves the full tool-call round-trip (API requests tool → binary executes → result returned → API narrates) without pulling in state persistence from EPIC-002.

**Done when:** A `Tool` interface and registry exist. `roll_check` is registered and can be dispatched by name. Given `{ action_dice: 3, danger_dice: 1 }`, it returns `{ net_dice: 2, rolls: [4, 6], highest: 6, botch: false }` (with actual random rolls). Given `{ action_dice: 2, danger_dice: 3 }`, it returns a botch.

### Out of Scope

- All other tools (`add_glitch`, `update_scene`, `spend_resist`, etc.) — those come in EPIC-002
- State persistence — `roll_check` doesn't read or write `state.json`
- Oracle table interpretation — the binary returns raw numbers, the prompt tells the AI what they mean (RFC §5 decision)
- Seeded/deterministic random for testing (nice-to-have, not required for Phase 1)

### Design/Approach

**Tool interface:**

```go
type Tool interface {
    Name() string
    Description() string
    Parameters() map[string]interface{} // JSON Schema for the tool's input
    Execute(args json.RawMessage) (json.RawMessage, error)
}
```

Each tool is a struct that implements this interface. `Name()` returns the string the AI uses to call it. `Parameters()` returns the JSON Schema that gets sent to the API so the model knows the expected input shape. `Execute()` takes raw JSON arguments, parses them, does its work, and returns raw JSON output.

**Why `json.RawMessage`?** It's Go's type for "JSON bytes I haven't parsed yet." This lets the registry dispatch generically without knowing each tool's specific input/output types. Each tool parses its own arguments internally.

**Registry:**

```go
type Registry struct {
    tools map[string]Tool
}

func NewRegistry(tools ...Tool) *Registry
func (r *Registry) Get(name string) (Tool, bool)
func (r *Registry) Definitions() []api.ToolDef  // for API requests
func (r *Registry) Execute(name string, args json.RawMessage) (json.RawMessage, error)
```

`Definitions()` converts all registered tools into `api.ToolDef` structs for inclusion in API requests. This is called once at startup and the result is reused.

**`roll_check` logic (from RFC §5):**

```
Input:  { action_dice: int, danger_dice: int, context: string }

net = max(0, action_dice - danger_dice)

if net == 0:
    return { net_dice: 0, rolls: [], highest: 1, botch: true }

rolls = [random d6 for each net die]
highest = max(rolls)

return { net_dice: net, rolls: rolls, highest: highest, botch: false }
```

The `context` field is descriptive text for logging/display. The binary doesn't interpret it.

**Random number generation:** Use `math/rand/v2` (Go 1.22+, auto-seeded, no global lock). If Go version < 1.22, fall back to `math/rand` with explicit seeding. No need for `crypto/rand` — this is a game, not cryptography.

### Files to Touch

`internal/tools/registry.go`: new — `Tool` interface, `Registry` struct, `NewRegistry()`, `Get()`, `Execute()`, `Definitions()`
`internal/tools/rollcheck.go`: new — `RollCheck` struct implementing `Tool`, dice rolling logic

### Implementation Checklist

<CRITICAL_RULE>
Before marking an item complete on the checklist MUST **stop** and **think**. Have you validated all aspects are **implemented** and **tested**?
</CRITICAL_RULE>

- [ ] Define `Tool` interface in `internal/tools/registry.go` with `Name()`, `Description()`, `Parameters()`, `Execute()` methods
- [ ] Implement `Registry` struct with `tools map[string]Tool`
- [ ] Implement `NewRegistry(tools ...Tool)` constructor that registers tools by name
- [ ] Implement `Registry.Get(name string) (Tool, bool)` for lookup
- [ ] Implement `Registry.Execute(name string, args json.RawMessage) (json.RawMessage, error)` — looks up tool, calls Execute, returns result or error if tool not found
- [ ] Implement `Registry.Definitions() []api.ToolDef` — converts all tools to API-ready definitions
- [ ] Create `RollCheck` struct in `internal/tools/rollcheck.go`
- [ ] Implement `RollCheck.Name()` returning `"roll_check"`
- [ ] Implement `RollCheck.Description()` returning a clear description of the tool's purpose
- [ ] Implement `RollCheck.Parameters()` returning JSON Schema: `action_dice` (int, required), `danger_dice` (int, required), `context` (string, required)
- [ ] Implement `RollCheck.Execute()`:
  - Parse input JSON into typed struct
  - Calculate `net = max(0, action_dice - danger_dice)`
  - If `net == 0`: return botch result `{ net_dice: 0, rolls: [], highest: 1, botch: true }`
  - Otherwise: roll `net` d6s, find highest, return `{ net_dice, rolls, highest, botch: false }`
- [ ] Write a unit test for `roll_check`: verify botch when danger >= action, verify net dice calculation, verify highest is max of rolls, verify rolls length equals net_dice
- [ ] Verify `go test ./internal/tools/...` passes
- [ ] Verify `go vet ./internal/tools/...` passes

### Acceptance Criteria

**Scenario:** Rolling with more action dice than danger dice.
**GIVEN** a `RollCheck` tool is registered in the registry.
**WHEN** `Execute` is called with `{ "action_dice": 3, "danger_dice": 1, "context": "sneak past guard" }`.
**THEN** the result has `net_dice: 2`, `rolls` is an array of 2 integers each between 1-6, `highest` equals the max of `rolls`, and `botch` is `false`.

**Scenario:** Rolling with equal or more danger dice (botch).
**GIVEN** a `RollCheck` tool is registered.
**WHEN** `Execute` is called with `{ "action_dice": 2, "danger_dice": 3, "context": "fight the bear" }`.
**THEN** the result has `net_dice: 0`, `rolls` is empty, `highest: 1`, and `botch: true`.

**Scenario:** Dispatching by name through the registry.
**GIVEN** a registry with `roll_check` registered.
**WHEN** `Registry.Execute("roll_check", ...)` is called.
**THEN** the correct tool handles the call.
**AND** `Registry.Execute("nonexistent", ...)` returns an error.

**Scenario:** Tool definitions for API requests.
**GIVEN** a registry with `roll_check` registered.
**WHEN** `Registry.Definitions()` is called.
**THEN** it returns a `[]api.ToolDef` with one entry whose `Name` is `"roll_check"` and `Parameters` is a valid JSON Schema object.

### Issues Encountered
<!--
The comments under the 'Issues Encountered' heading are the only comments you MUST not remove
This section is filled out post work as you fill out the checklists.
You SHOULD document any issues encountered and resolved during the sprint.
You MUST document any failed implementations, blockers or missing tests.
-->
