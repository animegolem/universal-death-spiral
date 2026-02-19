# Solo RPG Engine — V1 RFC

> **Design Philosophy:** The binary is the game client. The prompt is the game rules. The state schema is the interface contract between them.

---

## 1. Vision

A compiled TUI binary that runs a solo RPG experience powered by an LLM Game Master. The player interacts via text. The AI narrates, manages NPCs, tracks the hidden state of the world, and resolves actions through a tag-based dice system with a mechanical death spiral. The system is designed so that **adversity is structural, not discretionary**—the AI never has to choose to be mean, the math handles it.

**V1 is one specific game.** The binary, the state schema, and the prompt are designed together as a single product. The "universal engine" aspiration is explicitly deferred.

---

## 2. System Architecture

### 2.1 Component Overview

```
┌──────────────────────────────────────────────────────┐
│                    Go Binary (TUI)                    │
│                                                      │
│  ┌─────────┐  ┌──────────┐  ┌─────────────────────┐ │
│  │ Bubble  │  │  State   │  │    API Client        │ │
│  │ Tea UI  │  │  Manager │  │  (OpenRouter/Claude) │ │
│  │         │  │          │  │                      │ │
│  │ - Chat  │  │ - JSON   │  │ - Primary (Sonnet)   │ │
│  │ - Side  │  │ - Files  │  │ - Compress (Haiku)   │ │
│  │   bar   │  │ - Git    │  │ - Tool call loop     │ │
│  └─────────┘  └──────────┘  └─────────────────────┘ │
│                                                      │
│  ┌──────────────────────────────────────────────────┐│
│  │              Tool Executor                       ││
│  │  roll_check · manage_tags · advance_time · etc   ││
│  └──────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────┘
         │                           │
         ▼                           ▼
   Project Folder              OpenRouter API
   (local files)            (Claude Sonnet / Haiku)
```

### 2.2 Data Flow Per Turn

```
Player types action
       │
       ▼
Binary assembles context: system prompt + scratchpad + state + 
  summaries + conversation history + player message
       │
       ▼
API Call → Claude Sonnet
       │
       ▼
AI responds with tool calls + narration
       │
       ▼
Binary executes tool calls sequentially:
  - roll_check → rolls dice, returns raw numbers
  - add_glitch → updates state.json
  - update_scene → updates state.json
  - scratchpad → updates scratchpad.md
  - etc.
       │
       ▼
Tool results returned to AI → AI continues generating
       │
       ▼
Final narration displayed to player
TUI sidebar re-renders from state.json
Conversation appended to segment log
Git commit
       │
       ▼
If advance_time was called AND new_day == true:
  ├── Reset resist, advance NPC clocks (mechanical, binary does this)
  ├── Queue compression pass (async Haiku call)
  └── Morning phase narration (AI handles in next turn)
```

---

## 3. Project Structure

```
my-game/
├── .env                          # OPENROUTER_API_KEY=sk-or-...
├── prompt.md                     # Game system prompt (loaded at runtime)
├── state.json                    # Live game state (binary reads/writes)
├── scratchpad.md                 # Hidden AI GM notes (AI reads/writes via tool)
├── logs/
│   ├── day1-seg1.jsonl           # Full message history, one message per line
│   ├── day1-seg2.jsonl
│   └── ...
├── summaries/
│   ├── day1.md                   # Haiku-generated day summary
│   └── ...
└── world/
    ├── npcs/
    │   ├── zogar-sag.md          # NPC detail files (AI-created via tool)
    │   └── gorm.md
    └── locations/
        ├── the-pit.md
        └── river-crossing.md
```

**[DECISION]** The prompt file is named `prompt.md` and loaded from the project folder at runtime. This means you can edit it between sessions without recompiling. The binary ships with a default prompt that gets written to the folder on first run if none exists.

**[DECISION]** Git init happens automatically on project folder creation. Commits happen after every turn (all tool calls resolved). Commit message format: `Day {N} Seg {M}: {first 50 chars of player input}`.

---

## 4. State Schema

This is the interface contract. The binary renders from it. The prompt references it. Both must agree on this shape.

```json
{
  "meta": {
    "setup_complete": false,
    "genre": "",
    "day": 1,
    "segment": 1,
    "segments_per_day": 4,
    "segment_labels": ["Morning", "Afternoon", "Evening", "Night"]
  },
  "player": {
    "name": "",
    "concept": "",
    "inherent_tags": [],
    "glitch_tags": [],
    "conditions": [],
    "resist": { "current": 2, "max": 2 }
  },
  "gm_pc": null,
  "scene": {
    "location": "",
    "positive_tags": [],
    "negative_tags": [],
    "active_npcs": []
  },
  "npcs": {}
}
```

<details>
<summary><strong>Full NPC entry shape (within <code>npcs</code> object)</strong></summary>

```json
{
  "zogar-sag": {
    "name": "Zogar Sag",
    "concept": "Pictish shaman of the Wolf Clan",
    "positive_tags": ["commands serpents", "feared by all clans"],
    "negative_tags": ["arrogant", "old rivalry with Gorm"],
    "disposition": "hostile",
    "goal": "Capture outsiders for soul-transfer rituals",
    "clock": 2,
    "clock_max": 4,
    "last_location": "wolf-clan-territory",
    "active": true
  }
}
```

The `active` field indicates whether this NPC is in the current scene. `scene.active_npcs` is an array of NPC IDs that should match entries where `active: true`.

</details>

<details>
<summary><strong>GM PC shape (when present)</strong></summary>

```json
{
  "name": "Old Sage Toren",
  "concept": "Wandering storyteller who knows too much",
  "inherent_tags": ["silver tongue", "ancient lore", "frail body"],
  "glitch_tags": [],
  "conditions": []
}
```

The GM PC does not have resist points. The AI narrates them but they mechanically participate in scenes (their tags can contribute to the narrative and to pool construction when helping the player).

</details>

### What the Binary Reads for TUI Rendering

| Field | Sidebar Display |
|---|---|
| `meta.day`, `meta.segment` | `Day 3 · Afternoon` |
| `player.resist` | `RESIST: ●●○` |
| `player.glitch_tags` | Listed under GLITCHES |
| `player.inherent_tags` | Listed under TAGS |
| `player.conditions` | Listed under CONDITIONS |
| `scene.location` | Scene header |
| `scene.positive_tags`, `scene.negative_tags` | Listed under SCENE |

---

## 5. Tool Definitions

Ten tools. Each has a single clear purpose. The binary executes them and returns structured results. The AI interprets results according to the prompt's game rules.

### Hot Path Tools (used most turns)

**`roll_check`** — Roll the dice and return raw results.

```
Input:  { action_dice: int, danger_dice: int, context: string }
Output: { net_dice: int, rolls: [int], highest: int, botch: bool }
```

**[DECISION]** Dice cancellation is pure subtraction. Net dice $= \max(0,\; \text{action} - \text{danger})$. If net $= 0$: botch, `highest = 1`, no roll. Otherwise: roll `net` d6, return all faces and the highest. The AI interprets `highest` against the oracle table defined in the prompt. The `context` field is for display/logging only.

**[DECISION]** The binary does NOT know the oracle table. It returns numbers. The prompt tells the AI what the numbers mean. This keeps game rules prompt-side and iterable without recompilation.

**`add_glitch`** — Add a glitch tag to the player.

```
Input:  { tag: string }
Output: { glitch_tags: [string], count: int }
```

**`flush_glitches`** — Clear all glitch tags. Returns them for narration.

```
Input:  {}
Output: { flushed: [string], count: int }
```

### State Management Tools

**`spend_resist`** — Spend resist points.

```
Input:  { purpose: "negate_glitch" | "clear_condition", condition?: string }
Output: { success: bool, resist_remaining: int, error?: string }
```

Costs: `negate_glitch` = 1 point. `clear_condition` = 2 points (only during morning phase; binary checks `meta.segment == 1`). Binary validates sufficient points and returns error if not.

**`update_scene`** — Modify the current scene.

```
Input:  { 
  location?: string,
  set_positive?: [string],    // replace entire array
  set_negative?: [string],    // replace entire array
  add_positive?: [string],    // append
  add_negative?: [string],    // append
  remove_positive?: [string], // remove specific
  remove_negative?: [string]  // remove specific
}
Output: { scene: Scene }
```

**`update_player`** — Modify player's permanent state.

```
Input:  {
  add_inherent?: [string],
  remove_inherent?: [string],
  add_condition?: [string],
  remove_condition?: [string]
}
Output: { player: Player }
```

### NPC Tools

**`create_npc`** — Create a new NPC.

```
Input:  {
  id: string,              // kebab-case identifier
  name: string,
  concept: string,
  positive_tags: [string],
  negative_tags: [string],
  disposition: string,     // "hostile" | "neutral" | "friendly" | "unknown"
  goal: string,
  clock_max: int           // typically 4
}
Output: { npc: NPC }
```

Also writes a markdown file to `world/npcs/{id}.md` with the NPC's full description. The AI provides this as part of the tool call or the binary generates a template from the fields.

**`update_npc`** — Modify an existing NPC.

```
Input:  {
  id: string,
  disposition?: string,
  goal?: string,
  clock_delta?: int,       // +1 or -1, binary clamps to [0, clock_max]
  active?: bool,
  add_positive?: [string],
  remove_positive?: [string],
  add_negative?: [string],
  remove_negative?: [string]
}
Output: { npc: NPC }
```

### System Tools

**`advance_time`** — Move to the next time segment.

```
Input:  { reason: string }
Output: { 
  day: int, 
  segment: int, 
  segment_label: string,
  new_day: bool,
  resist_reset: bool,         // true if new day
  filled_clocks: [{ id: string, name: string, goal: string }],  // NPCs whose clock hit max
  player_conditions: [string] // reminder for morning phase
}
```

On new day, the binary: resets resist to max, advances ALL NPC clocks by 1, checks for filled clocks, resets segment to 1. Filled clocks are reported so the AI can narrate their consequences. **[DECISION]** The binary does NOT reset filled clocks—the AI must call `update_npc` to set a new goal and reset the clock if it chooses.

**`scratchpad`** — Read or write the scratchpad file.

```
Input:  { 
  operation: "read" | "write" | "append",
  section?: string,    // markdown ## header to target, null = whole file
  content?: string     // for write/append
}
Output: { content: string } // current content of section or file
```

**[DECISION]** Section-based operations use `## Header` matching. `write` replaces a section's content. `append` adds to the end of a section. `read` with no section returns the whole file.

---

## 6. API Strategy

### 6.1 Context Assembly

Each API call assembles a message array in this order:

```
System message:
  ┌─────────────────────────────────────────────┐
  │ [STATIC — cached]                           │
  │ Game rules from prompt.md                    │
  │ Tool definitions                             │
  ├─────────────────────────────────────────────┤
  │ [SEMI-STATIC — cached if unchanged]         │
  │ Scratchpad content (full file)              │
  ├─────────────────────────────────────────────┤
  │ [DYNAMIC — changes every turn]              │
  │ Current state.json                           │
  │ Day summaries (all, chronological)          │
  │ Logbook review suggestions (if pending)     │
  └─────────────────────────────────────────────┘

Conversation messages:
  ┌─────────────────────────────────────────────┐
  │ Full conversation for current segment        │
  │ (all user + assistant + tool messages)       │
  └─────────────────────────────────────────────┘
```

### 6.2 Context Budget

**[DECISION]** Target steady state: 70-100K tokens per API call. At current model context sizes (200K-1M), this is well within limits. The binary monitors total context size and triggers compression when it exceeds 60% of the target model's context window.

Active conversation window: **current segment + 4 previous segments** (roughly one full day of play). Older segments exist in log files but are NOT included in context unless explicitly needed.

**[DECISION]** Day summaries are cumulative and always included. They're small (~500 tokens each). After 10+ days, if summary tokens become significant, the compression pass also compresses old summaries into a "story so far" block.

### 6.3 Prompt Caching

The static portion of the system prompt (game rules, tool definitions) should be identical across turns and eligible for Anthropic's prompt caching. The binary should set the appropriate cache control markers if using the Anthropic API directly. Via OpenRouter, caching behavior depends on the underlying provider.

---

## 7. Setup Phase

### 7.1 Detection

Binary checks if `state.json` exists in the project folder:
- **No state.json:** Initialize with default empty state, enter setup mode.
- **state.json exists, `setup_complete: false`:** Resume setup (conversation history in `logs/setup.jsonl`).
- **state.json exists, `setup_complete: true`:** Enter play mode. Load `prompt.md`.

### 7.2 Setup Prompt

**[DECISION]** The setup prompt is a hardcoded string constant in the binary. It does NOT load from file. This ensures a consistent onboarding experience regardless of what game prompt is in the project folder.

The setup prompt instructs a conversational assistant to:

1. **Welcome and collect genre/setting** — Free text conversation. AI asks clarifying questions until it has a clear picture.

2. **Collect player character** — Name, concept, 3-5 inherent tags. The AI suggests tags based on the concept, the player approves/modifies. AI warns if fewer than 3 tags are proposed.

3. **Optional GM PC** — AI asks if the player wants a persistent companion/guide character. If yes, same process: name, concept, tags.

4. **Confirmation** — AI presents the character sheet(s) in a formatted block. Player confirms or requests changes.

5. **AI-only generation** — Once confirmed, AI generates (written to scratchpad via `scratchpad` tool):
   - 5-10 principles (2:1 broad:narrow ratio)
   - 2-4 NPC/factions with goals and clocks (created via `create_npc` tool)
   - 3-5 potential arcs or dramatic questions
   - Opening scene concept

6. **Transition** — AI sets `setup_complete: true` via state update. Binary detects this on next turn, switches to game prompt from `prompt.md`, and the AI narrates the opening scene.

### 7.3 The Transition Moment

When the binary detects `setup_complete` flipped to `true`:
1. Clear conversation history (setup conversation goes to `logs/setup.jsonl`)
2. Load `prompt.md` as the new system prompt
3. Inject state + scratchpad (now populated from setup) into context
4. The first "turn" of play has no player message—the system prompt instructs the AI to narrate the opening scene based on scratchpad contents
5. Play begins

---

## 8. Play Loop

### 8.1 Standard Turn

```
1. Player types an action or dialogue
2. AI reads state + scratchpad from system context
3. AI assesses the situation:
   a. Is this action uncertain with interesting success AND failure?
      → If NO: narrate the outcome freely, maybe update_scene, done
      → If YES: proceed to roll
   b. AI identifies relevant tags:
      - Positive: which player inherent + scene positive + GM PC tags help?
      - Negative: player glitch tags + scene negative + opposing NPC positive tags
   c. AI PRESENTS the tag assessment to the player:
      "I see [keen tracker] and [morning mist] helping (+2), 
       against [twisted ankle] and [patrol route] (-2). 
       That's 3 Action vs 2 Danger. Sound right?"
   d. Player confirms or negotiates
4. AI calls roll_check(action: 3, danger: 2, context: "sneak past the scout")
5. Binary rolls, returns { net_dice: 1, rolls: [4], highest: 4, botch: false }
6. AI consults oracle table (in prompt) → 4 = "Yes, but..."
7. AI calls add_glitch("guard heard a splash") — oracle says partial success gains a glitch
8. AI narrates the "Yes, but..." incorporating the specific tags
9. Player sees: narration + dice display + updated sidebar
10. Binary commits state + appends to segment log + git commit
```

### 8.2 Morning Phase (World Tick)

Triggered when `advance_time` results in `new_day: true`.

**Mechanical (binary handles):**
- Reset `player.resist.current` to `player.resist.max`
- Advance all NPC clocks by 1
- Report filled clocks and current conditions in tool response

**Narrative (AI handles, next turn):**
The prompt instructs the AI to present a morning phase:

```
"Dawn breaks over the forest camp. You flex your fingers, 
feeling [slightly] more ready to face the day.

RESIST REFRESHED: ●● (2/2)

You're still nursing that [broken rib] from the fall. 
Spend 2 resist now to recover, or save your strength?

[Meanwhile, in the world...] 
Zogar Sag's hunters have been closing in. 
(His clock filled — the AI narrates the consequence as a new scene element)"
```

The player can then spend resist on conditions or save it. The AI narrates consequences of any filled clocks and sets the scene for the new day.

### 8.3 Time and Segments

| Segment | Label | Triggered By |
|---|---|---|
| 1 | Morning | New day |
| 2 | Afternoon | Player-initiated location change |
| 3 | Evening | Player-initiated location change |
| 4 | Night | Player-initiated location change |

**[DECISION]** Time advances when the player moves to a new significant location, or when a botch/critical failure forces a dramatic scene change. The AI calls `advance_time` in either case. The binary increments the segment and handles day rollover.

**[DECISION]** A full botch (result = 1 / No and...) can trigger an involuntary location change narrated by the AI. This means pacing isn't entirely in the player's hands—catastrophic failure can eat your time. This prevents the player from camping one location indefinitely.

---

## 9. Game Mechanics (Prompt-Defined)

These rules live in `prompt.md`, not in the binary. The binary provides the tools. The prompt tells the AI how and when to use them.

### 9.1 Tag System

**Three categories of tags, all mechanically identical (they add dice to pools):**

| Tag Type | Persists | Source | Examples |
|---|---|---|---|
| **Inherent** (player) | Permanent | Character creation | `keen tracker`, `father's longbow` |
| **Environmental** (scene) | Until scene changes | AI sets on location change | `morning mist`, `strong current` |
| **Glitch** (player) | Until flushed | Gained on partial success | `twisted ankle`, `guard heard a splash` |

**Positive tags** → add to action pool. **Negative tags** → add to danger pool.

NPC positive tags become danger dice when that NPC opposes the player. NPC negative tags can become action dice when the player exploits them.

**Conditions** are mechanically identical to permanent negative tags. They always contribute danger dice when relevant. They can only be cleared by spending 2 resist during morning phase.

### 9.2 Oracle Table

| Highest Die | Oracle | Tag Effect |
|---|---|---|
| **6** | **Yes, and...** | Succeed + remove one glitch tag (if any) |
| **5** | **Yes, but...** | Succeed + gain a glitch tag |
| **4** | **Yes, but...** | Succeed + gain a glitch tag |
| **3** | **No, but...** | Fail, no additional penalty |
| **2** | **No...** | Fail + flush all glitches into narration |
| **1 / Botch** | **No, and...** | Fail + flush glitches + gain a Condition |

**The spiral:** Two-thirds of successes (4-5) cost a glitch tag. Each glitch adds a danger die to the next roll. Fewer net dice → lower expected result → more glitches or failure. Eventually you fail, glitches flush, and you start clean but with narrative consequences (and possibly a Condition).

### 9.3 Resist Mechanic

- **2 resist per day.** Resets each morning.
- **Spend 1:** Negate a glitch tag you would have gained (on a 4-5 result). You still succeeded. The AI narrates a near-miss instead of a complication.
- **Spend 2:** During morning phase only. Clear one Condition.
- **Cannot** be spent to avoid failure results. Cannot remove existing glitches.

**[DECISION]** 2/day is the starting value. This gives the player meaningful but limited control. They can absorb two partial successes per day OR clear one condition. Not both.

### 9.4 Pool Construction

$$\text{Action Dice} = 1\text{ (base)} + \text{relevant positive inherent tags} + \text{relevant positive scene tags}$$

$$\text{Danger Dice} = \text{glitch tags (all, always)} + \text{relevant negative scene tags} + \text{opposing NPC positive tags}$$

$$\text{Net Dice} = \max(0,\; \text{Action} - \text{Danger})$$

If Net Dice $= 0$: automatic Botch (result $= 1$).

Otherwise: roll Net Dice × d6, result $=$ highest face.

**Key word: "relevant."** The AI must justify which tags contribute. The prompt instructs the AI to present this assessment to the player before rolling. The player can challenge the assessment.

Glitch tags are ALWAYS relevant—they represent your accumulating disadvantage and always count.

---

## 10. World Tick

The world tick fires on each new day (when `advance_time` transitions segment past 4). It has two parts:

### 10.1 Mechanical (Binary)

Executed automatically within the `advance_time` tool:
- Reset resist
- Increment all NPC clocks by 1
- Identify NPCs whose clock $\geq$ clock_max
- Return filled clock data to AI

### 10.2 Narrative (AI, via prompt instructions)

The prompt instructs the AI to handle the morning phase:
1. Narrate rest/recovery flavor
2. Present resist status and conditions to player
3. For each filled clock: narrate the consequence of that NPC/faction achieving their goal. This should change scene tags, introduce new threats, or alter the narrative landscape.
4. For filled-clock NPCs: call `update_npc` to set a new goal and reset clock to 0
5. Update scratchpad with any changes to the world state
6. Present the new day's situation to the player

### 10.3 Scratchpad Update

The prompt instructs the AI to maintain specific scratchpad sections:

```markdown
## Principles
[Generated during setup, occasionally updated]

## Active Threads
- Thread descriptions with status

## NPC Notes
- Internal motivations, plans, relationships not captured in state.json

## World State
- Broader world facts established during play

## Resolved
- Completed threads (kept briefly for reference, pruned by compression pass)
```

---

## 11. Compression Pass

### 11.1 Trigger

Fires on day transition (same trigger as world tick, but async—doesn't block the next turn). Uses Haiku/Flash via a separate API call.

### 11.2 Input

The cheap model receives:
- Full conversation logs for the day(s) being compressed
- Current scratchpad
- Current state.json (for NPC clock context)
- Existing day summaries (to avoid duplication)

### 11.3 Output Schema

```json
{
  "summaries": {
    "day_3": "Kael tracked the Wolf Clan raiders to the river crossing..."
  },
  "logbook_review": {
    "stale": [
      { "section": "Active Threads", "entry": "...", "reason": "..." }
    ],
    "consolidate": [
      { "entries": ["...", "..."], "suggested": "..." }
    ],
    "contradictions": [
      { "a": "...", "b": "...", "note": "..." }
    ],
    "missing": [
      "Player promised to return the fisher's nets (Day 2, Seg 3)"
    ]
  }
}
```

### 11.4 Processing

1. Summaries → written to `summaries/dayN.md`
2. `logbook_review` → injected into the next turn's system prompt as a temporary block
3. The primary model reads the review and can act on it via `scratchpad` tool calls
4. The logbook review block is removed after one turn (it's a one-shot suggestion)

---

## 12. TUI Layout

```
┌─────────────────────────────────────┬──────────────────────┐
│                                     │ Day 2 · Afternoon    │
│  [narrative chat, scrollable,       │                      │
│   markdown rendered]                │ RESIST ●●○  (2/2)    │
│                                     │                      │
│  The mist parts as you reach the    │ GLITCHES             │
│  river crossing. Through the reeds  │  · twisted ankle     │
│  you spot a Wolf Clan scout...      │  · lost bearings     │
│                                     │                      │
│  ╔══════════════════════════════╗    │ TAGS                 │
│  ║ Roll: 3 Action vs 2 Danger  ║    │  + keen tracker      │
│  ║ Net: 1d6 → [4]              ║    │  + father's longbow  │
│  ║ Oracle: Yes, but...         ║    │  + wilderness hardened│
│  ╚══════════════════════════════╝    │                      │
│                                     │ CONDITIONS           │
│  You slip past—barely—but your      │  - broken rib        │
│  foot catches a submerged root.     │                      │
│  He hasn't turned yet.              │ SCENE                │
│                                     │  Wolf Clan border    │
│  > New glitch: guard heard splash   │  + shallow ford      │
│                                     │  - patrol route      │
│                                     │  - strong current    │
├─────────────────────────────────────┤                      │
│ ⟳ Rolling dice...                   │                      │
│ ✓ State updated                     │                      │
├─────────────────────────────────────┤                      │
│ > _                                 │                      │
└─────────────────────────────────────┴──────────────────────┘
```

**[DECISION]** Activity indicator (bottom-left above input) shows deterministic tool call status messages. Not AI-generated, not summarized.

Messages: `⟳ Rolling dice...`, `⟳ Updating world...`, `⟳ Consulting the scratchpad...`, `✓ State updated`, `⟳ Compressing history (background)...`

The dice display block is inline in the chat, rendered by the TUI when it detects a `roll_check` tool result in the response.

---

## 13. Prompt Architecture

### 13.1 Setup Prompt (Hardcoded in Binary)

```markdown
# Setup Assistant

You are helping a player set up a new solo RPG campaign. 
Your role is friendly, collaborative, and creative.

## Your Process
1. Ask about genre/setting. Discuss until clear.
2. Ask about their character: name, concept, background.
3. Suggest 3-5 inherent tags based on their concept. 
   Negotiate until the player is happy. Warn if < 3.
4. Ask if they want a GM companion character. If yes, repeat step 2-3.
5. Present a formatted character sheet. Get confirmation.
6. Once confirmed, generate the following WITHOUT showing the player:
   [instructions for principles, arcs, antagonists, NPCs]
   Write these to the scratchpad via tool calls.
   Create starting NPCs via create_npc tool calls.
7. Set setup_complete to true.
8. Describe the opening scene.

## Rules
- Be conversational, not form-like
- Let the player ramble and extract details
- Suggest, don't dictate
- All generated world elements go to scratchpad, not to chat
```

### 13.2 Game Prompt Structure (`prompt.md`)

This is the document the user can edit. It's the DMG. Approximate structure:

```markdown
# [Game Name] — Game Master Prompt

## Identity
You are the Game Master for a solo RPG. You narrate the world, 
portray NPCs, manage scene transitions, and resolve actions through 
the tag-based dice system. You are the world—adversarial when the 
fiction demands it, generous when earned.

## Core Philosophy
[Key principles about GMing style, drawn from the user's design doc]
- The mechanics create adversity. Your job is to narrate it faithfully.
- When the oracle says "No, and..." you LEAN IN. This is where 
  the best stories happen.
- Never fudge. Never soften a result. The dice spoke.
- When no roll is needed, narrate freely and generously.
[etc.]

## Game Rules

### Tags
[Full explanation of inherent, environmental, glitch tags]
[How tags map to dice pools]
[Tag relevance — what "relevant" means, with examples]

### Rolling
[When to roll: only when both success and failure are interesting]
[Pool construction explanation]
[Oracle table with mechanical effects]
[Player-facing tag assessment: always show your work before rolling]

### The Spiral
[How glitches accumulate]
[How failure flushes them]
[How conditions work]
[The emotional arc: tension builds → release → reset]

### Resist
[2 per day, dual use, resets on morning]

### Time
[4 segments, triggered by location change or forced by botch]
[Day transition triggers world tick]

## Turn Structure
[Step by step: what you do when the player acts]
1. Assess: roll or free narration?
2. If rolling: identify tags, present assessment, get confirmation
3. Call roll_check
4. Interpret oracle, call appropriate tag tools
5. Narrate incorporating the tags
6. If scene changes: update_scene
7. If time advances: advance_time

## Morning Phase
[What to do when a new day starts]
[Present resist, conditions, world changes]
[Narrate consequences of filled NPC clocks]

## NPC Guidelines
[How to portray them consistently]
[When to create new NPCs]
[NPC positive tags = danger dice when opposing player]
[NPC negative tags = exploitable]

## Scratchpad Management
[What sections to maintain]
[When to update (after significant events, not every turn)]
[Keep it concise — this is your working notebook, not a novel]

## Narration Guidelines
[Tone guidance based on genre]
[When to be terse vs. expansive]
[How to use tags as writing prompts in failure narration]
[Don't repeat mechanical info the sidebar already shows]
[STOP after presenting a dice check. Wait for confirmation.]

## Glitch Tag Quality
Glitch tags must be:
- Concrete and observable ("guard heard a splash" not "things got worse")
- Forward-looking (suggesting what could go wrong NEXT)
- Specific to the fiction (not generic)

## Turn Verification
Before finalizing your response, verify:
- Every tag I referenced in pool construction exists in state
- Every tag effect has a corresponding tool call
- My narration matches the oracle result I received
- If I narrated a world change, I updated scratchpad or scene
- I did NOT describe the player's thoughts or speak for them
- If I called for a roll, I STOPPED and waited for confirmation
```

**[DECISION]** The prompt does NOT contain the principles, NPC details, or world facts. Those live in the scratchpad and state.json, which are injected separately into the system context. The prompt contains only the rules and behavioral instructions.

---

## 14. Open Questions for Playtesting

These are explicitly NOT decided in this RFC. They'll be resolved through play:

1. **Oracle distribution** — Is 4-5 both being "Yes, but..." too aggressive? Maybe 5 should be clean success. Adjustable in prompt without recompilation.

2. **Relevant tag assessment** — How strictly should the AI gate tag relevance? Too strict = frustrating. Too loose = trivial. The right answer is genre-dependent.

3. **World tick frequency** — Per-day feels right but may be too infrequent if days are long (many segments of play). Could add mid-day ticks on segment 3 transition. Adjustable in prompt.

4. **Glitch flush trigger** — Currently flushes on any failure (2 or 1). Could flush only on botch (1) with regular failure (2) just being a narrative setback without clearing glitches. This changes the spiral shape significantly.

5. **GM PC mechanical participation** — Do GM PC tags actually add to the player's action pool? Or are they purely narrative? Mechanical participation makes the game easier; narrative-only keeps difficulty consistent.

---

## 15. Future Scope (Explicitly Deferred)

- **Lorebook / retrieval system** — Keyword or semantic retrieval from `world/` directory
- **NPC templates** — Random or scoped template selection on `create_npc`
- **Multiple game systems** — Different prompts for different RPG genres
- **State schema flexibility** — Binary renders from a config rather than hardcoded schema
- **Model-agnostic tool calls** — Normalization layer for different API providers
- **Multi-step unbundled tool calls** — Full AI autonomy over the tool call loop
- **Multiplayer** — Probably never

---

## 16. Build Order

1. **Bare API loop** — stdin/stdout, no TUI. Prove the tool call cycle works: player types → AI responds with tool calls → binary executes → AI narrates.
2. **State persistence** — `state.json` read/write, scratchpad file ops, basic logging.
3. **The prompt** — Write and iterate on `prompt.md` until gameplay feel is correct. **This is 60% of the project.** Test with the bare API loop.
4. **Setup flow** — Implement the hardcoded setup prompt and the transition to play.
5. **TUI wrapper** — Bubble Tea chat pane + state sidebar. Wire up the API loop.
6. **Git integration** — Auto-init, auto-commit per turn.
7. **Compression pass** — Haiku call on day transition, summary writing, logbook review injection.
8. **Polish** — Dice display, activity indicators, error handling, graceful recovery.

---
