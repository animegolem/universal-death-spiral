---
node_id: AI-IMP-001-2
tags:
  - IMP-LIST
  - Implementation
  - phase-1
  - api
  - provider
  - openrouter
kanban_status: planned
depends_on: [[AI-IMP-001-1-project-scaffolding]]
parent_epic: [[AI-EPIC-001-project-scaffolding-and-bare-api-loop]]
confidence_score: 0.85
date_created: 2026-03-03
date_completed:
---

# AI-IMP-001-2-provider-abstraction-and-openrouter-client

## Provider Interface and OpenRouter Implementation

The binary needs to talk to an LLM API. Rather than hardcoding OpenRouter calls everywhere, we define a `Provider` interface that any API backend can implement. The first (and only, for now) implementation targets OpenRouter using the `sashabaranov/go-openai` library.

This abstraction matters because the RFC identifies prompt caching via the direct Anthropic API as a future optimization. By coding against the interface now, swapping providers later is a contained change — implement the interface, swap the constructor in `main.go`, done.

**Done when:** A `Provider` interface exists with typed message/tool structs, and an OpenRouter implementation can send a chat completion request with tool definitions and receive a response (including tool calls). Verified with a simple integration test or manual call.

### Out of Scope

- Streaming (`ChatCompletionStream` is declared on the interface but returns "not implemented" — used in EPIC-005)
- Retry logic or rate limiting (EPIC-008 polish)
- Anthropic direct API provider (future work)
- Token counting or context budget management (EPIC-002+)

### Design/Approach

**Internal message format: OpenAI-shaped.** OpenRouter speaks the OpenAI chat completions format natively, and `go-openai` provides all the types. We define our own thin types that mirror this structure so the rest of the codebase doesn't import `go-openai` directly. This keeps the provider boundary clean — only `openrouter.go` imports the library.

**Key types:**

```go
// ChatMessage represents a single message in the conversation.
type ChatMessage struct {
    Role       string     // "system", "user", "assistant", "tool"
    Content    string     // text content
    ToolCalls  []ToolCall // present when role="assistant" and model wants tools
    ToolCallID string     // present when role="tool" (echoes the ID from ToolCall)
}

// ToolCall represents a tool invocation requested by the model.
type ToolCall struct {
    ID       string // opaque — may be "call_..." or "toolu_..."
    Function FunctionCall
}

// FunctionCall is the name + arguments of a tool call.
type FunctionCall struct {
    Name      string // tool name (e.g., "roll_check")
    Arguments string // raw JSON string of arguments
}

// ToolDef defines a tool for the API request.
type ToolDef struct {
    Name        string
    Description string
    Parameters  map[string]interface{} // JSON Schema object
}
```

**OpenRouter quirks handled:**
- `max_tokens` is always set (Anthropic models require it)
- `tools` array is re-sent on every request (OpenRouter re-validates)
- Tool call IDs are opaque strings (no prefix assumptions)
- Custom HTTP transport adds `HTTP-Referer` and `X-Title` headers

**Why `go-openai`?** It's the most mature Go client for OpenAI-compatible APIs (~9k GitHub stars). It has full type definitions for tool calls, streaming, and all message roles. It works with OpenRouter by changing the base URL. The alternative was hand-rolling HTTP calls, which would mean reimplementing all the type marshaling that `go-openai` already handles.

### Files to Touch

`internal/api/provider.go`: new — `Provider` interface, `ChatMessage`, `ToolCall`, `ToolDef`, `ChatRequest`, `ChatResponse` types
`internal/api/openrouter.go`: new — `OpenRouterProvider` struct implementing `Provider`, custom HTTP transport

### Implementation Checklist

<CRITICAL_RULE>
Before marking an item complete on the checklist MUST **stop** and **think**. Have you validated all aspects are **implemented** and **tested**?
</CRITICAL_RULE>

- [ ] Define `Provider` interface in `internal/api/provider.go` with `ChatCompletion(ctx, ChatRequest) (ChatResponse, error)` and `ChatCompletionStream(ctx, ChatRequest) (ChatStream, error)`
- [ ] Define `ChatMessage` struct: `Role`, `Content`, `ToolCalls []ToolCall`, `ToolCallID`
- [ ] Define `ToolCall` struct: `ID`, `Function FunctionCall`
- [ ] Define `FunctionCall` struct: `Name`, `Arguments` (raw JSON string)
- [ ] Define `ToolDef` struct: `Name`, `Description`, `Parameters` (JSON Schema as `map[string]interface{}`)
- [ ] Define `ChatRequest` struct: `Model`, `Messages []ChatMessage`, `Tools []ToolDef`, `MaxTokens`, `Temperature`
- [ ] Define `ChatResponse` struct: `Message ChatMessage`, `FinishReason string`, `Usage` (prompt/completion/total tokens)
- [ ] Implement `OpenRouterProvider` in `internal/api/openrouter.go` using `go-openai` client
- [ ] Implement constructor `NewOpenRouterProvider(apiKey, model string, maxTokens int)` that configures `go-openai` with `BaseURL = "https://openrouter.ai/api/v1"`
- [ ] Implement custom `http.RoundTripper` that adds `HTTP-Referer` and `X-Title: Universal Death Spiral` headers
- [ ] Implement `ChatCompletion` method: convert our types → `go-openai` types, call API, convert response back
- [ ] Handle conversion of `go-openai` tool call responses back to our `ToolCall` type (opaque IDs)
- [ ] Implement `ChatCompletionStream` as a stub returning `errors.New("streaming not implemented")`
- [ ] Verify the provider compiles with `go build ./internal/api/...`

### Acceptance Criteria

**Scenario:** The provider sends a basic chat message.
**GIVEN** a valid OpenRouter API key and the provider is initialized.
**WHEN** `ChatCompletion` is called with a single user message and no tools.
**THEN** a `ChatResponse` is returned with `FinishReason = "stop"` and non-empty `Message.Content`.

**Scenario:** The provider sends a request with tool definitions.
**GIVEN** a valid API key and tool definitions for `roll_check`.
**WHEN** `ChatCompletion` is called with a user message like "roll 3 action dice vs 2 danger dice."
**THEN** the response has `FinishReason = "tool_calls"` and `Message.ToolCalls` contains a call to `roll_check`.
**AND** the `ToolCall.ID` is a non-empty string.
**AND** `ToolCall.Function.Arguments` is valid JSON.

**Scenario:** Streaming is called before implementation.
**GIVEN** any valid provider instance.
**WHEN** `ChatCompletionStream` is called.
**THEN** an error is returned with message "streaming not implemented".

### Issues Encountered
<!--
The comments under the 'Issues Encountered' heading are the only comments you MUST not remove
This section is filled out post work as you fill out the checklists.
You SHOULD document any issues encountered and resolved during the sprint.
You MUST document any failed implementations, blockers or missing tests.
-->
