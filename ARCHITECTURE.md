# Architecture

> Developer-facing internals of `pi-context-prune`. For the user-facing explanation of *why* pruning works and its cache tradeoffs, see [PRUNING.md](./PRUNING.md).

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PI CODING AGENT RUNTIME                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────────────┐   │
│  │  User    │───►│  Agent   │───►│  Tools   │───►│  Tool Results        │   │
│  │  Message │    │  (LLM)   │    │  Execute │    │  (raw output)        │   │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┬───────────┘   │
│                       ▲                                      │              │
│                       │                                      ▼              │
│                       │                          ┌───────────────────────┐  │
│                       │                          │  SESSION BRANCH       │  │
│                       │                          │  (full history)       │  │
│                       │                          └───────────┬───────────┘  │
│                       │                                      │              │
│       ┌───────────────┼──────────────────────────────────────┤              │
│       │               │                                      │              │
│       │    ┌──────────┴──────────┐              ┌────────────▼───────────┐  │
│       │    │   "context" EVENT   │◄─────────────│  pi-context-prune      │  │
│       │    │   (filtered msgs)   │   INTERCEPT  │  EXTENSION             │  │
│       │    └─────────────────────┘              └────────────────────────┘  │
│       │                                                                     │
│       └─────────────────────────────────────────────────────────────────────┘
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Module Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    pi-context-prune EXTENSION INTERNALS                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐     ┌──────────────────┐     ┌─────────────────────┐  │
│  │  index.ts        │     │  config.ts       │     │  types.ts           │  │
│  │  (orchestrator)  │────►│  (settings.json) │     │  (shared constants) │  │
│  │                  │     └──────────────────┘     └─────────────────────┘  │
│  │  Event handlers: │                                                       │
│  │  • session_start │     ┌──────────────────┐                              │
│  │  • turn_end      │────►│  batch-capture   │  Serialize turn data         │
│  │  • message_end   │     │  .ts             │  into CapturedBatch          │
│  │  • context       │     └────────┬─────────┘                              │
│  │  • agent_end     │              │                                        │
│  │  • before_agent_ │              ▼                                        │
│  │    start         │     ┌──────────────────┐                              │
│  └──────────┬───────┘     │  summarizer.ts   │  LLM call → markdown         │
│             │             │  (stream-based)  │  summary text                │
│             │             └────────┬─────────┘                              │
│             │                      │                                        │
│             │                      ▼                                        │
│             │             ┌───────────────────┐                             │
│             │             │  indexer.ts       │  Map<toolCallId, Record>    │
│             │             │  (ToolCallIndexer)│  + session persistence      │
│             │             └────────┬──────────┘                             │
│             │                      │                                        │
│             ▼                      ▼                                        │
│     ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────────┐   │
│     │  pruner.ts      │  │  query-tool.ts   │  │  context-prune-tool.ts │   │
│     │  (filter msgs)  │  │  (recover data)  │  │  (agentic trigger)     │   │
│     └─────────────────┘  └──────────────────┘  └────────────────────────┘   │
│                                                                             │
│     ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────────┐   │
│     │  reminder.ts    │  │  frontier.ts     │  │  stats.ts              │   │
│     │  (<pruner-note>)│  │  (boundary track)│  │  (token/cost stats)    │   │
│     └─────────────────┘  └──────────────────┘  └────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: Step by Step (agent-message mode — default)

### Phase 1: Accumulation (cache-friendly — no context changes)

```
    User: "Build me a React table component"
         │
         ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  AGENT LOOP — tool-calling turns                                │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                 │
    │  Turn 1: read_file("src/App.tsx")                               │
    │     └──► turn_end fires                                         │
    │          └──► captureBatch() → CapturedBatch pushed to queue    │
    │          └──► NO flush (mode = agent-message)                   │
    │          └──► Notify: "1 turn queued"                           │
    │                                                                 │
    │  Turn 2: read_file("package.json")                              │
    │     └──► turn_end fires                                         │
    │          └──► captureBatch() → CapturedBatch pushed to queue    │
    │          └──► NO flush                                          │
    │          └──► Notify: "2 turns queued"                          │
    │                                                                 │
    │  Turn 3: web_search() + read_file()                             │
    │     └──► turn_end fires                                         │
    │          └──► captureBatch() → pushed                           │
    │          └──► NO flush                                          │
    │          └──► "3 turns queued"                                  │
    │                                                                 │
    │  Turn 4: edit_file() + bash("npm run build")                    │
    │     └──► turn_end fires                                         │
    │          └──► captureBatch() → pushed                           │
    │          └──► NO flush                                          │
    │          └──► "4 turns queued"                                  │
    │                                                                 │
    │  ═══════════════════════════════════════════════════════════════│
    │  DURING THIS ENTIRE PHASE:                                      │
    │  • The "context" event returns UNCHANGED messages               │
    │  • Prefix cache stays HOT across all turns                      │
    │  • Each new LLM request adds only the new tool result to suffix │
    │  ═══════════════════════════════════════════════════════════════│
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### Phase 2: Trigger (agent sends final text response)

```
    ┌─────────────────────────────────────────────────────────────────┐
    │  Turn 5: Assistant sends FINAL TEXT (no tool calls)             │
    │                                                                 │
    │  "I've built the DataTable component with sorting and           │
    │   pagination. The build is passing now."                        │
    │                                                                 │
    │     └──► message_end fires                                      │
    │          └──► isFinalAssistantMessage(msg) === true             │
    │          └──► mode === "agent-message" ✓                        │
    │          └──► flushPending(ctx, { delivery: "session" })        │
    │                                                                 │
    └──────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
```

### Phase 3: Flush (summarize + index + inject)

```
    ┌─────────────────────────────────────────────────────────────────┐
    │  flushPending() — THE CORE PRUNE OPERATION                      │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                 │
    │  Step 1: CAPTURE                                                │
    │  ─────────────────                                              │
    │  • capturePendingBatches(ctx)                                   │
    │    → scans session branch for all unsummarized tool results     │
    │    → applies frontier trimming (skip already-handled ranges)    │
    │    → applies groupBatchesByMode("agent-message")                │
    │      → merges turns sharing same userTurnGroup into 1 batch     │
    │                                                                 │
    │  Step 2: SUMMARIZE (parallel LLM calls)                         │
    │  ──────────────────────────────────────                         │
    │  For EACH batch:                                                │
    │    • serializeBatchForSummarizer(batch)                         │
    │      → renders tool calls as readable text (max 2000 chars each)│
    │    • summarizeBatch(batch, config, ctx)                         │
    │      → resolves model (default or custom)                       │
    │      → gets API key via ctx.modelRegistry                       │
    │      → streams LLM response (system prompt + serialized batch)  │
    │      → returns { summaryText, usage }                           │
    │                                                                 │
    │  Step 3: SIZE CHECK (skip-oversized guard)                      │
    │  ─────────────────────────────────────────                      │
    │  • If summary.length > rawCharCount → SKIP pruning that batch   │
    │  • Raw text stays in context (it's actually smaller!)           │
    │  • Frontier still advances past it                              │
    │                                                                 │
    │  Step 4: PERSIST INDEX                                          │
    │  ─────────────────────                                          │
    │  For each tool call in the batch:                               │
    │    indexer.index.set(toolCallId, {                              │
    │      toolCallId, toolName, args, resultText,                    │
    │      isError, turnIndex, timestamp                              │
    │    })                                                           │
    │    pi.appendEntry(CUSTOM_TYPE_INDEX, { toolCalls: [...] })      │
    │    ↑ persisted to session file — survives restart/branch switch │
    │                                                                 │
    │  Step 5: INJECT SUMMARY MESSAGE                                 │
    │  ──────────────────────────────                                 │
    │  • delivery "session":                                          │
    │    sessionManager.appendCustomMessageEntry(                     │
    │      CUSTOM_TYPE_SUMMARY, summaryText, display=true, details    │
    │    )                                                            │
    │  • delivery "runtime":                                          │
    │    pi.sendMessage({ customType, content, display, details },    │
    │      { deliverAs: "steer" }  ← arrives before next LLM call     │
    │    )                                                            │
    │                                                                 │
    │  Step 6: ADVANCE FRONTIER                                       │
    │  ─────────────────────────                                      │
    │  frontier.advance({                                             │
    │    lastAttemptedToolCallId,                                     │
    │    lastAttemptedTurnIndex,                                      │
    │    outcome: "summarized" | "skipped-oversized"                  │
    │  })                                                             │
    │  → prevents double-summarization on subsequent flushes          │
    │                                                                 │
    │  Step 7: UPDATE STATS                                           │
    │  ─────────────────────                                          │
    │  statsAccum.add(usage) → persist to session                     │
    │  setPruneStatusWidget → footer shows new state                  │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### Phase 4: Filter (next LLM request sees pruned context)

```
    ┌─────────────────────────────────────────────────────────────────┐
    │  User sends next message: "Now add column filtering"            │
    │                                                                 │
    │  Pi prepares the context for the LLM...                         │
    │     └──► "context" event fires                                  │
    │                                                                 │
    │  pi.on("context", (event) => {                                  │
    │    // indexer now has tc-001..tc-009 marked as summarized       │
    │    const pruned = pruneMessages(event.messages, indexer)        │
    │    //                                                           │
    │    // WHAT pruneMessages DOES:                                  │
    │    // messages.filter(msg =>                                    │
    │    //   !(msg.role === "toolResult" &&                          │
    │    //     indexer.isSummarized(msg.toolCallId))                 │
    │    // )                                                         │
    │    //                                                           │
    │    // KEEPS: system, user, assistant (with toolCall blocks),    │
    │    //        summary messages, unsummarized tool results        │
    │    // REMOVES: toolResult messages for indexed IDs              │
    │    //                                                           │
    │    return { messages: pruned }                                  │
    │  })                                                             │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

---

## What the LLM Sees (Before vs After)

```
══════════════════════════════════════════════════════════════════════════
  BEFORE PRUNING (what the LLM would see without the extension)
══════════════════════════════════════════════════════════════════════════

  msg[0]  role: system      "You are Pi, a coding assistant..."
  msg[1]  role: user        "Build a React table component..."
  msg[2]  role: assistant   [toolCall: read_file, id: tc-001]
  msg[3]  role: toolResult  "export default function App()..."        45 tok
  msg[4]  role: assistant   [toolCall: read_file, id: tc-002]
  msg[5]  role: toolResult  "{ dependencies: { react: ^18..."        120 tok
  msg[6]  role: assistant   [toolCall: web_search + read_file]
  msg[7]  role: toolResult  "React Table v7 docs... 3400 chars"    3,400 tok
  msg[8]  role: toolResult  "import { useState }..."                 200 tok
  msg[9]  role: assistant   [toolCall: edit_file, id: tc-005]
  msg[10] role: toolResult  "✔ File updated"                          15 tok
  msg[11] role: assistant   [toolCall: bash + read_file]
  msg[12] role: toolResult  "BUILD OUTPUT... 2800 chars"           2,800 tok
  msg[13] role: toolResult  "Updated file contents..."               180 tok
  msg[14] role: assistant   "Done! I built the component..."
  msg[15] role: user        "Now add column filtering"

  TOTAL: ~7,000+ tokens of stale tool output

══════════════════════════════════════════════════════════════════════════
  AFTER PRUNING (what the LLM actually sees)
══════════════════════════════════════════════════════════════════════════

  msg[0]  role: system      "You are Pi, a coding assistant..."
  msg[1]  role: user        "Build a React table component..."
  msg[2]  role: assistant   [toolCall: read_file, id: tc-001]       ← KEPT
  msg[3]  ░░░░░░░░░░░░░░░░░ REMOVED (tc-001 in index) ░░░░░░░░░░
  msg[4]  role: assistant   [toolCall: read_file, id: tc-002]       ← KEPT
  msg[5]  ░░░░░░░░░░░░░░░░░ REMOVED (tc-002 in index) ░░░░░░░░░░
  msg[6]  role: assistant   [toolCall: web_search + read_file]      ← KEPT
  msg[7]  ░░░░░░░░░░░░░░░░░ REMOVED (tc-003 in index) ░░░░░░░░░░
  msg[8]  ░░░░░░░░░░░░░░░░░ REMOVED (tc-004 in index) ░░░░░░░░░░
  msg[9]  role: assistant   [toolCall: edit_file, id: tc-005]       ← KEPT
  msg[10] ░░░░░░░░░░░░░░░░░ REMOVED (tc-005 in index) ░░░░░░░░░░
  msg[11] role: assistant   [toolCall: bash + read_file]            ← KEPT
  msg[12] ░░░░░░░░░░░░░░░░░ REMOVED (tc-006 in index) ░░░░░░░░░░
  msg[13] ░░░░░░░░░░░░░░░░░ REMOVED (tc-007 in index) ░░░░░░░░░░
  msg[14] role: assistant   "Done! I built the component..."
  msg[15] role: steer       [SUMMARY: ~200 tokens]
  msg[16] role: user        "Now add column filtering"

  TOTAL: ~500 tokens (96% reduction)
  Assistant toolCall blocks KEPT → model can call context_tree_query by ID

══════════════════════════════════════════════════════════════════════════
```

---

## Prefix Cache Behavior (Request by Request)

```
  REQUEST 1 (Turn 1):
  [system][user]                          │ [assistant toolCall]
  ◄──── computed (MISS — first request) ─►│

  REQUEST 2 (Turn 2):
  [system][user][asst tc-001][result-001] │ [asst tc-002]
  ◄──── HIT on prefix from Req 1 ───────►│ only new suffix

  REQUEST 3..5 (Turns 3-5):
  Growing prefix, always a cache HIT ✓

  ════════ PRUNE FIRES (after final text reply) ════════

  REQUEST 6 (user's next message, post-prune):
  [sys][user][asst][asst]...[summary]     │ [user "add filter"]
  ◄──── MISS (prefix changed) ──────────► │
  Provider stores this new prefix

  REQUEST 7+ (next agent loop):
  Same prefix as Req 6 → HIT ✓

  ┌──────────────────────────────────────────────────────────────┐
  │  During agent loop:         ALL CACHE HITS                   │
  │  First request after prune: ONE CACHE MISS                   │
  │  Subsequent requests:       CACHE HITS again                 │
  │                                                              │
  │  vs. hitting compaction without pruning:                     │
  │  → TOTAL cache destruction + permanent data loss             │
  └──────────────────────────────────────────────────────────────┘
```

---

## Two-Layer Memory Model

```
  ╔═══════════════════════════════════════════════════════════════╗
  ║  LAYER 1: HOT MEMORY (active LLM context)                     ║
  ║                                                               ║
  ║  • System prompt                                              ║
  ║  • User messages                                              ║
  ║  • Assistant messages (with toolCall ID blocks KEPT)          ║
  ║  • Summary messages (compact markdown)                        ║
  ║  • CURRENT turn's unsummarized tool results                   ║
  ║                                                               ║
  ║  Size: ~500-2000 tokens (bounded)                             ║
  ╚═══════════════════════════════════════════════════════════════╝
                         │
                         │ context_tree_query(toolCallIds)
                         ▼
  ╔═══════════════════════════════════════════════════════════════╗
  ║  LAYER 2: COLD MEMORY (pruner index)                          ║
  ║                                                               ║
  ║  Map<toolCallId, {                                            ║
  ║    toolCallId, toolName, args, resultText,                    ║
  ║    isError, turnIndex, timestamp                              ║
  ║  }>                                                           ║
  ║                                                               ║
  ║  Size: unbounded (all historical tool outputs preserved)      ║
  ║  Access: on-demand via context_tree_query tool                ║
  ║  Persistence: session custom entries (CUSTOM_TYPE_INDEX)      ║
  ╚═══════════════════════════════════════════════════════════════╝
```

---

## Event Handler Wiring

| Pi Event | Handler Behavior |
|----------|-----------------|
| `session_start` | Load config, rebuild indexer/stats/frontier from session entries, clear queue, sync tool activation |
| `session_tree` | Same rebuild (branch changed) |
| `before_agent_start` | If agentic-auto: append system prompt instructions |
| `turn_end` | Capture batch → push to queue. If `every-turn`: flush immediately. Else: notify pending count |
| `tool_execution_end` | If `context_tag` called && mode is `on-context-tag`: flush |
| `message_end` | If mode is `agent-message` && final text reply: flush |
| `agent_end` | Update status widget only (no LLM work — session may be disposing) |
| `context` | `pruneMessages()` removes indexed toolResults; optionally annotates with `<pruner-note>` |

---

## Persistence Model

The extension persists state as custom session entries (JSONL):

| Entry Type | Purpose |
|-----------|---------|
| `context-prune-index` | Full tool call records (toolCallId, name, args, resultText, etc.) |
| `context-prune-summary` | Summary message content + display metadata |
| `context-prune-stats` | Cumulative summarizer token/cost snapshots |
| `context-prune-frontier` | Last attempted prune boundary (prevents double-summarization) |

On `session_start` / `session_tree`, all entries are scanned to rebuild runtime state. This means the extension survives restarts and branch switches without any external database.

---

## Mode Comparison

| Mode | Trigger | Cache Busts | Use Case |
|------|---------|-------------|----------|
| `every-turn` | After each tool turn | Every turn | Debugging only |
| `agent-message` | Agent's final text reply | 1 per exchange | **Default** — best balance |
| `on-context-tag` | `context_tag()` tool called | At milestones | Save-point workflows |
| `on-demand` | User runs `/pruner now` | Manual | Maximum cache preservation |
| `agentic-auto` | LLM calls `context_prune` | LLM decides | Long autonomous loops |

---

## The Tradeoff

```
WITHOUT PRUNING:
  • Context grows unbounded → hits provider's max window
  • Compaction fires → ENTIRE prefix destroyed + data lost permanently
  • Cost: full-price tokens on every request for stale data

WITH PRUNING (agent-message mode):
  • 1 cache miss per user↔agent exchange
  • Context stays bounded (~500-2000 tokens of history)
  • No compaction ever needed
  • Original data always recoverable via context_tree_query
  • Cost: 1 cheap summarizer LLM call per exchange

  Net: small periodic cache bust ≪ catastrophic compaction bust
```
