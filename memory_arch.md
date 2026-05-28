# IDA Memory System

## Memory Architecture Framework

Memory is a **plugin service**, not a property of any individual agent. The same store, loaders, and tools are available to every agent — new agents call the existing API; they do not own storage.

IDA separates **what to remember** (four layers) from **how it is produced and consumed** (three planes). Capture runs asynchronously and never blocks a user request. Retrieve assembles context per request and never invokes Capture directly. **Store** is the only shared boundary between them.

### Four memory layers

Each layer has a different lifetime and cognitive role. Upper layers are hotter and more volatile; lower layers are distilled and persistent.

```mermaid
flowchart TB
    subgraph HOT ["In-request context (bounded)"]
        STM["Short-term window\nlast N exchanges\ncompressed_message or message"]
        CS_INJ["Session cheatsheet\nchat.cheatsheet markdown"]
    end

    subgraph WARM ["Session → project (async capture)"]
        CS_STORE[("chat.cheatsheet JSON\nepisodic — this chat")]
    end

    subgraph COLD ["Cross-session (persistent)"]
        PM[("agent_memory PROJECT\nentity_* · project_facts\nproject_lessons")]
        UP[("agent_memory PROJECT+user\nuser_profile")]
    end

    subgraph RAW ["Authoritative transcript"]
        CR[("chat_record\n+ RAG embeddings")]
    end

    CR --> STM
    CR --> CS_STORE
    CS_STORE --> CS_INJ
    CS_STORE --> PM
    CR --> UP
    PM --> STM
    UP --> STM

    style HOT fill:#e8f4fc,stroke:#2980b9
    style WARM fill:#fef9e7,stroke:#d4ac0d
    style COLD fill:#eafaf1,stroke:#27ae60
    style RAW fill:#f4f6f6,stroke:#7f8c8d
```

| Layer | Contents | Lifetime | Store | Analogy |
|---|---|---|---|---|
| **Chat history** | Raw or compressed exchanges | Session window | `chat_record`, RAG | Working memory |
| **Cheatsheet** | Structured session findings | Per chat | `chat.cheatsheet` | Episodic |
| **Project memory** | Verified entities, facts, lessons | Per project | `agent_memory` PROJECT | Semantic |
| **User profile** | Style and workflow preferences | Per user+project | `agent_memory` + `user_id` | Procedural |

### Three planes (Capture · Store · Retrieve)

```mermaid
flowchart LR
    MSG(["Historian\nrecord_saved"])

    subgraph CAPTURE ["Capture — async background agents"]
        direction TB
        CC["ContextCompressor"]
        CSA["CheatsheetAgent"]
        CONS["ConsolidationAgent"]
        HA["HabitAgent"]
    end

    subgraph STORE ["Store"]
        direction TB
        RAG[("RAG index")]
        CR[("chat_record")]
        CS[("chat.cheatsheet")]
        AM[("agent_memory")]
    end

    subgraph RETRIEVE ["Retrieve — per user request"]
        direction TB
        MS["MemoryService"]
        PMT["ProjectMemoryToolbox"]
    end

    AGENTS(["Orchestrator\nsub-agents"])

    MSG --> CC & CSA
    CC --> RAG & CR
    CSA --> CS
    CS --> CONS
    CONS --> AM
    CR --> HA
    HA --> AM

    CR & CS & AM --> MS
    RAG & CS & AM --> PMT
    MS & PMT --> AGENTS
```

| Plane | Responsibility | Never does |
|---|---|---|
| **Capture** | Distil conversations into store artifacts | Block the request path |
| **Store** | Persist and index by scope/type | Choose what agents say |
| **Retrieve** | Load and format memory for prompts/tools | Write memory directly |

### Pipeline goal

```mermaid
flowchart LR
    E1["1. Extract per-exchange\nCheatsheetAgent"] --> E2["2. Compress + index\nContextCompressor"]
    E2 --> E3["3. Consolidate session\nConsolidationAgent"]
    E3 --> E4["4. Profile user\nHabitAgent"]
    E4 --> E5["5. Inject next session\nMemoryService + tools"]

    style E1 fill:#f9e4b7
    style E3 fill:#f9e4b7
    style E4 fill:#f9e4b7
    style E5 fill:#d4edda
```

---

## Main Components

All capture agents subscribe to or poll the same conversation stream. Only **ContextCompressor** and **CheatsheetAgent** react to every new message in real time; **ConsolidationAgent** and **HabitAgent** run on idle or cursor-gap schedules.

```mermaid
flowchart TB
    HIST(["HistorianService\nsystem.historian.record_saved"])

    subgraph CC_AGENT ["ContextCompressor"]
        CC_T["Trigger: USER + AGENT records\nevent + 120s fallback"]
        CC_J1["Job 1: RAG embed\nsync · idempotent"]
        CC_J2["Job 2: LLM compress\ngt=2000 chars to lt=300 words\nasync thread pool"]
        CC_T --> CC_J1 --> CC_J2
    end

    subgraph CSA_AGENT ["CheatsheetAgent"]
        CSA_T["Trigger: AGENT RESPONSE only\nevent + 120s fallback"]
        CSA_TW["Tail window: default 6\nconfig tail_window_size e.g. 12"]
        CSA_PH["Phases 1–3 + 40min idle flush\nper-chat thread · dirty re-run"]
        CSA_CUR["CheatsheetService curator\nwrites chat.cheatsheet JSON"]
        CSA_T --> CSA_TW --> CSA_PH --> CSA_CUR
    end

    subgraph CONS_AGENT ["ConsolidationAgent"]
        CONS_T["Trigger: 300s poll\n60min idle OR 3h cursor gap"]
        CONS_PROM["Promote verified findings\nentity_* · project_facts · project_lessons"]
        CONS_T --> CONS_PROM
    end

    subgraph HA_AGENT ["HabitAgent"]
        HA_T["Trigger: 300s poll\n60min idle"]
        HA_PROF["Merge transcript to user_profile\nPROJECT scope + user_id"]
        HA_T --> HA_PROF
    end

    subgraph STORE_BOX ["Store"]
        RAG[("rag_embeddings")]
        CR[("chat_record")]
        CS[("chat.cheatsheet")]
        AM[("agent_memory")]
    end

    subgraph RET_BOX ["Retrieve"]
        MS["MemoryService\nshort · long · project · profile"]
        PMT["tool_recall_project_memory\ntool_search_chat_history\ntool_read_long_term_memory"]
    end

    HIST --> CC_AGENT & CSA_AGENT
    CC_J1 --> RAG
    CC_J2 --> CR
    CSA_CUR --> CS
    CS --> CONS_AGENT
    CONS_PROM --> AM
    CR --> HA_AGENT
    HA_PROF --> AM
    CR & CS & AM --> MS
    RAG & CS & AM --> PMT
```

| Component | Plane | Location | Trigger summary |
|---|---|---|---|
| `ContextCompressor` | Capture | `workers/context_compressor.py` | `record_saved` (USER/AGENT); 120s fallback |
| `CheatsheetAgent` | Capture | `workers/cheatsheet_agent.py` | `record_saved` (AGENT RESPONSE); 120s fallback |
| `CheatsheetService` | Capture | `services/cheatsheet/cheatsheet_service.py` | Called by CheatsheetAgent per exchange |
| `ConsolidationAgent` | Capture | `workers/consolidation_agent.py` | 300s poll; idle 60 min or cheatsheet cursor gap 3 h |
| `HabitAgent` | Capture | `workers/habit_agent.py` | 300s poll; idle 60 min |
| `MemoryStore` | Store | `db/memory_store.py` | CRUD + FTS on `agent_memory` |
| RAG index | Store | `rag/service.py` | Vector + keyword over chat embeddings |
| `MemoryService` | Retrieve | `services/memory/memory_service.py` | Injected at agent start / prompt build |
| `ProjectMemoryToolbox` | Retrieve | `tools/toolbox/project_memory_toolbox.py` | On-demand during tool loops |

---

## Component Map (detail)

| Component | Plane | Location | What it does |
|---|---|---|---|
| `ContextCompressor` | Capture | `workers/context_compressor.py` | Embeds every message into the RAG index; compresses long messages to ≤300 words |
| `CheatsheetAgent` | Capture | `workers/cheatsheet_agent.py` | Extracts structured findings from each exchange into `chat.cheatsheet` |
| `CheatsheetService` | Capture | `services/cheatsheet/cheatsheet_service.py` | Orchestrates the LLM curator for a single update |
| `ConsolidationAgent` | Capture | `workers/consolidation_agent.py` | Promotes verified cheatsheet entries to PROJECT-scoped `agent_memory` |
| `HabitAgent` | Capture | `workers/habit_agent.py` | Extracts per-user behavioral patterns at session end into `agent_memory` |
| `MemoryStore` | Store | `db/memory_store.py` | CRUD + FTS over the `agent_memory` table |
| RAG index | Store | `rag/service.py` | Vector + keyword search over embedded `chat_record` messages |
| `chat_record` | Store | `db/project.py` | Raw message history; holds `compressed_message` after compressor runs |
| `chat.cheatsheet` | Store | `db/project.py` (chat row) | Per-chat JSON: curated session findings |
| `MemoryService` | Retrieve | `services/memory/memory_service.py` | Assembles memory strings for LLM prompt injection |
| `ProjectMemoryToolbox` | Retrieve | `tools/toolbox/project_memory_toolbox.py` | Agent-callable tools over memory and chat history |

---

## Memory Layers

See [Four memory layers](#four-memory-layers) for the stack diagram. Layer details below.

### Chat history

`chat_record` stores every USER and AGENT message. The short-term window (last 12–16 exchanges) is the primary continuity mechanism within a request. Each row has:
- `message` — original text
- `compressed_message` — ≤300-word summary (written by ContextCompressor when `len(message) ≥ 2000`)

Retrieval always prefers `compressed_message or message`. The raw message is retained for audit; the compressed version is what enters the context window.

### Cheatsheet

`chat.cheatsheet` is a per-chat JSON blob written incrementally by CheatsheetAgent. It holds structured findings in three buckets:

```json
{
  "data_insights":   [{"entity": "NNM-101", "content": "NPT 54h during 8½\" section", "confidence": "verified", "record_id": 847}],
  "key_facts":       [{"id": "kf_3a1b2c",  "content": "WellView export truncated at 3200m", "confidence": "inferred", "record_id": 801}],
  "lessons_learned": [{"id": "ll_9f8e7d",  "content": "Pre-treat with LCM before depleted zones", "confidence": "inferred"}]
}
```

**Confidence model:**
- `verified` — value appears verbatim in a tool result block
- `inferred` — derived from narrative, not directly confirmed by data
- `conflicted` — same metric has two different values; both entries preserved, neither dropped

`data_insights` entries are keyed by entity (well, formation, BHA) — no stable `id`. `key_facts` and `lessons_learned` entries carry a stable `id` (`kf_*`, `ll_*`) so the curator can update them across turns rather than duplicate.

### Project memory

`agent_memory` rows with `scope=PROJECT` hold distilled cross-session knowledge. Three named slots:

| Name pattern | Type | Contents |
|---|---|---|
| `entity_{well}` | `DATA_INSIGHT` | Merged, deduplicated per-entity numeric findings |
| `project_facts` | `KEY_FACTS` | Non-quantitative session findings: data gaps, user knowledge gaps, naming conventions |
| `project_lessons` | `LESSON_LEARNED` | Transferable operational insights |

Each record has a `content_text` field (plain-text rendering) that is FTS-indexed in PostgreSQL, enabling keyword retrieval via `MemoryStore.search_objects`.

### User profile

`agent_memory` rows with `scope=PROJECT` and a `user_id`. Name: `user_profile`. Type: `USER_PROFILE`. Written by HabitAgent after sessions go idle.

Five dimensions captured: query style, interaction style, output preferences, domain focus, expertise signals. Free-form text; no schema enforcement. FTS-indexed.

> **Scope:** `USER_PROFILE` uses `scope=PROJECT` with `project_id + user_id` to keep preferences project-contextual. A user's preferences on a shallow gas project may differ from a deep HP/HT project.

---

## Capture Layer

Background agents and how they connect to the store (detail flows below).

```mermaid
flowchart LR
    RS(["record_saved"])

    RS --> CC["ContextCompressor\nUSER + AGENT"]
    RS --> CSA["CheatsheetAgent\nAGENT RESPONSE"]

    CC --> RAG[("RAG")]
    CC --> CR[("chat_record\ncompressed_message")]

    CSA --> CS[("chat.cheatsheet")]

    CS --> CONS["ConsolidationAgent\n300s poll"]
    CONS --> AM[("agent_memory\nPROJECT")]

    CR --> HA["HabitAgent\n300s poll"]
    HA --> UP[("user_profile")]

    style CC fill:#d4edda
    style CSA fill:#f9e4b7
    style CONS fill:#f9e4b7
    style HA fill:#f9e4b7
```

### ContextCompressor

**Location:** `workers/context_compressor.py`

**Trigger:** subscribes to `system.historian.record_saved` bus event; 120s fallback poll catches records missed across restarts. The fallback cursor is in-memory only — restarts re-scan from 0, but both jobs are idempotent.

```mermaid
flowchart TD
    A(["record_saved event\nor 120s fallback poll"]) --> B{USER or AGENT\nmessage?}
    B -- No --> Z([ignore])
    B -- Yes --> C["enqueue (record_id, project_id)"]

    C --> D["Main loop drains queue"]
    D --> E["_process_record"]

    E --> F["Job 1 — Index synchronous\ncheck get_indexed_chat_record_ids\nskip if already indexed\nadd_document_batch then generate_embeddings"]

    E --> G{len msg &gt;= 2000\nAND no compressed_message?}
    G -- No --> H([done])
    G -- Yes --> I["Job 2 — submit to\nsingle-worker thread pool\nnon-blocking"]
    I --> J["LLM call MICRO_SMART\nsummarize to 300 words max"]
    J --> K["update_chat_record\n_compressed_message"]

    style F fill:#d4edda,stroke:#28a745
    style J fill:#f9e4b7,stroke:#e6a817
```

Job 1 (indexing) runs immediately so the RAG index is always fresh. Job 2 (compression) is submitted to a single-worker thread pool so the main loop is never blocked by an LLM call.

**Idempotency:** indexing checks `get_indexed_chat_record_ids` before embedding. Compression skips records with `compressed_message` already set.

---

### CheatsheetAgent + CheatsheetService

**Location:** `workers/cheatsheet_agent.py`, `services/cheatsheet/cheatsheet_service.py`

**Trigger:** same `system.historian.record_saved` bus event; 120s fallback poll. Only fires on AGENT RESPONSE records.

**Per-chat threading:** each chat gets its own thread. A `_dirty_chats` flag handles the race: if a new response arrives while the thread is running, the thread re-submits itself on exit. No events are dropped.

#### Tail window logic

The cheatsheet is a curated summary, not a log. The last `tail_n` exchanges are kept as a **working window** (default **6** in code; set `tail_window_size: 12` in agent config for long drilling Q&A sessions). The tail logic controls when each record becomes eligible for curation:

| Phase | Condition | Action |
|---|---|---|
| **Phase 1 — young** | total records ≤ `tail_n` | Curate each exchange immediately |
| **Phase 2 — accumulating** | total > `tail_n`, unprocessed ≤ `tail_n` | Defer — wait for the tail to fill |
| **Phase 3 — sliding window** | unprocessed > `tail_n` | Oldest unprocessed scrolled past tail — curate immediately |
| **Idle bypass** | last exchange > 40 min ago | Flush all remaining tail records regardless of phase |

```mermaid
flowchart TD
    A(["record_saved event\nor 120s fallback poll"]) --> B{AGENT RESPONSE?}
    B -- No --> Z([ignore])
    B -- Yes --> C{chat in\n_active_chats?}
    C -- Yes --> D["mark _dirty_chats\nwill re-run on exit"]
    C -- No --> E["add to _active_chats\nsubmit thread"]

    E --> F["_get_next_record\ntail window logic"]
    F --> G{Phase?}
    G -- "Phase 1\ntotal lte tail_n" --> H["curate immediately"]
    G -- "Phase 2\nunprocessed lte tail_n\ntotal gt tail_n" --> I{idle &gt; 40 min?}
    I -- No --> J([defer])
    I -- Yes --> H
    G -- "Phase 3\nunprocessed > tail_n" --> H

    H --> K["find preceding user query"]
    K --> L["CheatsheetService\n.update_cheatsheet\nLLM curator ~15s"]
    L --> M["advance\ncheatsheet_cursor_ts"]
    M --> N{more records\nto curate?}
    N -- Yes --> F
    N -- No --> O{in _dirty_chats?}
    O -- Yes --> P["discard dirty flag\nre-submit thread"]
    O -- No --> Q(["remove from _active_chats\ndone"])

    style L fill:#f9e4b7,stroke:#e6a817
```

#### Curation

`CheatsheetService.update_cheatsheet` runs one LLM curator call per eligible exchange:

1. Load current cheatsheet JSON (or `{}` if empty)
2. LLM call: `[user_query, agent_response, current_cheatsheet]` → updated cheatsheet JSON
3. Parse and validate with `parse_cheatsheet`
4. Stamp `id` on new `key_facts` / `lessons_learned` entries (`kf_*`, `ll_*`)
5. Stamp `record_id` on all new entries (existing entries carry their `record_id` forward unchanged)
6. Save via `project_service.save_chat_cheatsheet`
7. Advance `cheatsheet_cursor_ts`

---

### Compressor + Cheatsheet: context window management

These two capture jobs solve the same problem from different angles: as a session grows, the raw transcript eventually exceeds the context window.

```mermaid
flowchart LR
    TR(["session grows\n40 exchanges\nlong tool outputs"])

    TR --> CC_BOX
    TR --> CSA_BOX

    subgraph CC_BOX ["ContextCompressor"]
        CC1["each long message\ngt= 2000 chars"]
        CC2["compressed to\n300 words max"]
        CC1 --> CC2
    end

    subgraph CSA_BOX ["CheatsheetAgent"]
        CSA1["each exchange\ncurated by LLM"]
        CSA2["structured findings\n~2 KB JSON blob"]
        CSA1 --> CSA2
    end

    CC2 --> WIN["context window\nlast N compressed exchanges\n= bounded working memory"]
    CSA2 --> WIN2["context window\ncheatsheet = all session\nknowledge in ~2 KB"]

    WIN --> OK(["context window stays\nwithin budget\nregardless of session length"])
    WIN2 --> OK
```

Without the compressor, long tool outputs overflow the chat history window. Without the cheatsheet, recovering facts from turn 3 requires injecting the full transcript. Together: the context window always holds the last N compressed exchanges and the cheatsheet. Neither grows unboundedly.

---

### ConsolidationAgent

**Location:** `workers/consolidation_agent.py`

**Trigger:** polls every 300s. Fires on a chat when:
- `cheatsheet_cursor_ts > consolidation_cursor_ts` (new cheatsheet content exists), AND
- Either idle > 60 min OR cursor gap > 3h (for long active sessions)

The 60-minute threshold is 20 minutes after the cheatsheet idle flush (40 min), giving CheatsheetAgent time to fully settle before Consolidation reads it.

**What gets promoted:**

| Cheatsheet bucket | Promotion filter | Target memory |
|---|---|---|
| `data_insights` | `confidence == "verified"` | `entity_{key}` per entity (merged) |
| `key_facts` | `confidence in ("verified", "inferred")` | `project_facts` (merged) |
| `lessons_learned` | all entries | `project_lessons` (merged) |

`inferred` key_facts are included because they represent user knowledge gaps and data quality observations — valuable even without direct tool confirmation. `inferred` data_insights are excluded because numeric claims without tool grounding should not enter permanent memory.

```mermaid
flowchart TD
    A(["poll every 300s"]) --> B["get_chats_needing_consolidation\nidle > 60min OR cursor_gap > 3h"]
    B --> C{any chats?}
    C -- No --> A
    C -- Yes --> D["for each chat: parse chat.cheatsheet"]

    D --> E{empty or\nlegacy text?}
    E -- Yes --> F["advance cursor · skip"]
    E -- No --> G

    G["filter data_insights\nconfidence == verified\ngroup by entity"] --> H["read entity_{key}\nfrom agent_memory"]
    H --> I["_synthesize\nLLM merge or _dedupe"]
    I --> J["set_object entity_{key}\nscope=PROJECT · confidence=0.85"]

    J --> K["filter key_facts\nverified + inferred"]
    K --> L["_synthesize\nLLM merge · budget 400 tokens"]
    L --> M["set_object project_facts"]

    M --> N["all lessons_learned"]
    N --> O["_synthesize\nLLM merge · budget 200 tokens"]
    O --> P["set_object project_lessons"]

    P --> Q["advance consolidation_cursor_ts"]

    style I fill:#f9e4b7,stroke:#e6a817
    style L fill:#f9e4b7,stroke:#e6a817
    style O fill:#f9e4b7,stroke:#e6a817
```

**Entity key normalization:** `NNM-101`, `NNM_101`, `NNM 101` → `nnm101`. Separators stripped, lowercased.

**LLM synthesis:** existing memory + new candidates → synthesis LLM (MICRO_SMART). Merge duplicates (keep most specific), resolve numerical conflicts (append "values vary: X, Y" when unresolvable), drop subsumed entries, preserve every distinct fact. Returns a JSON array of strings. Falls back to case-insensitive `_dedupe` if LLM is unavailable.

Token budgets (400 / 200 tokens) are hints in the synthesis prompt — the LLM is trusted to stay within them. No post-synthesis trimming.

---

### HabitAgent

**Location:** `workers/habit_agent.py`

**Trigger:** polls every 300s. Fires when `max(response_ts) < now - 60min` AND `max(response_ts) > habit_cursor_ts` — i.e., the session is idle but has unprocessed exchanges since the last run.

```mermaid
flowchart TD
    A(["poll every 300s"]) --> B["get_idle_chats\nmax response_ts < now − 60min\nAND > habit_cursor_ts"]
    B --> C{any idle chats?}
    C -- No --> A
    C -- Yes --> D["for each chat"]

    D --> E["get_user_id_from_chat\nsender_name → user table"]
    E --> F{user_id found?}
    F -- No --> G["advance cursor · skip\nsystem or API chat"]
    F -- Yes --> H["fetch USER records\nfetch AGENT RESPONSE records\nboth since habit_cursor_ts\nlimit 200 · sort by id ASC"]

    H --> I{any records?}
    I -- No --> A
    I -- Yes --> J["load existing user_profile\nagent_memory\nscope=PROJECT · user_id"]

    J --> K["LLM: extract habits from transcript\nmerge with existing profile\n5 dimensions: query style · interaction\noutput prefs · domain focus · expertise"]
    K --> L{"&lt;habits&gt; tags\nin response?"}
    L -- No --> M["keep existing · advance cursor"]
    L -- Yes --> N{profile != none?}
    N -- No --> M
    N -- Yes --> O["set_object user_profile\nscope=PROJECT · user_id\nmemory_type=USER_PROFILE"]
    O --> P["advance habit_cursor_ts"]

    style K fill:#f9e4b7,stroke:#e6a817
```

**Profile update strategy:** existing pattern confirmed → unchanged; confirmed multiple times → append "(confirmed)"; new evidence contradicts it → soften; new pattern → append. One line per observation. Five dimensions: query style, interaction style, output preferences, domain focus, expertise signals.

**`get_user_id_from_chat`** derives the user ID by looking up `sender_name` (user email) from USER records and joining to the `user` table. System or API chats with no USER records get `user_id=None` — cursor is advanced and the chat is silently skipped.

---

## Storage Layer

### MemoryStore

**Location:** `db/memory_store.py`

CRUD service over the `agent_memory` PostgreSQL table. Stateless — takes an `Engine` at construction.

**Primary key pattern:** `(agent_id, name, scope, project_id, org_id, user_id)`. `set_object` upserts on this composite key, incrementing `version` on each update.

**API surface:**

```python
# Upsert — creates or updates, increments version
memory_store.set_object(agent_id, name, object, scope, memory_type,
                        project_id, user_id, content_text, confidence)

# Point read
memory_store.get_object(agent_id, name, scope, project_id, user_id) → Memory | None

# FTS search (PostgreSQL plainto_tsquery, ranked by ts_rank)
memory_store.search_objects(query, project_id, scope, memory_type, limit) → list[Memory]

# List all for an agent
memory_store.list_objects(agent_id, scope, memory_type, project_id, limit) → list[Memory]
```

**Schema:**

```
agent_memory
  agent_id      TEXT        — "consolidation_agent", "habit_agent", ...
  name          TEXT        — "entity_nnm101", "project_facts", "user_profile"
  scope         ENUM        — GLOBAL / ORG / PROJECT / USER
  memory_type   ENUM        — DATA_INSIGHT / KEY_FACTS / LESSON_LEARNED / USER_PROFILE / BOOKKEEPING
  project_id    INT         — required for scope=PROJECT
  user_id       INT         — required for user_profile entries
  object        JSONB       — structured payload ({"insights": [...], "facts": [...], ...})
  content_text  TEXT        — plain-text rendering of object, FTS-indexed
  confidence    FLOAT       — 0.0–1.0 (written by ConsolidationAgent for data insights)
  version       INT         — incremented on each upsert
  size_chars    INT         — len(content_text), for context budget tracking
```

`content_text` is the FTS surface — always a human-readable rendering of `object`. For entity memories: `"Entity: nnm101\n- NPT 54h\n- ROP 13 m/hr"`. For user_profile: the raw profile text. FTS queries operate on content, not JSONB keys.

**Scopes:** `GLOBAL` for bookkeeping cursors (background agents tracking last-processed IDs); `PROJECT` for entity memories, facts, lessons, and user profiles; `ORG` and `USER` scopes exist but are not currently used.

---

## Retrieval Layer

### MemoryService

**Location:** `services/memory/memory_service.py`

Coordinator for in-context memory assembly. Takes `project_service` and `memory_store` as injected dependencies — works with null dependencies (returns `"(none)"` safely). Never raises.

**Four loaders:**

```python
svc = MemoryService(project_service=ps, memory_store=ms, tail_n=12, top_k=3)

# Last tail_n exchanges as "User: ... / Ida: ..." pairs (chronological ASC)
svc.load_short_term_memory(chat_id) → str

# Current session's cheatsheet rendered as markdown
svc.load_long_term_memory(chat_id) → str

# Top-K PROJECT-scoped memories matching query via FTS
svc.load_project_memory(project_id, query) → str

# user_profile from agent_memory (PROJECT scope, project_id + user_id)
svc.load_user_profile(user_id, project_id) → str
```

**`load_short_term_memory`:** fetches `tail_n + 4` records (buffer for system/tool messages), iterates ASC (records return oldest-first). Formats as `Role: text` pairs. Prefers `compressed_message or message`.

**`load_long_term_memory`:** fetches `chat.cheatsheet` JSON, parses with `parse_cheatsheet`, renders via `render_to_markdown`. Returns `"(none)"` for empty cheatsheets.

**`load_project_memory`:** calls `MemoryStore.search_objects(scope=PROJECT, limit=top_k)`. Formats each hit as `[{memory_type}] {name}\n{content_text}`. Returns `"(none)"` on no results.

**`load_user_profile`:** calls `MemoryStore.get_object(agent_id="habit_agent", name="user_profile", scope=PROJECT, project_id, user_id)`. Returns `content_text` directly.

---

### ProjectMemoryToolbox

**Location:** `tools/toolbox/project_memory_toolbox.py`

Agent-callable tools for on-demand memory access during LLM execution. Three tools:

#### `tool_recall_project_memory`

```
Search PROJECT-scoped agent_memory via FTS.
Input:  query (str), top_k (int, default 5, max 20)
Output: formatted hits "[{type}] {name}\n{content_text}", separated by "---"
Filter: USER_PROFILE entries excluded (preferences must not pollute analytical results)
Passthrough: project_id
```

#### `tool_search_chat_history`

```
Hybrid (semantic + keyword) search over RAG-indexed chat records.
Input:  query (str), top_k (int, default 8, max 20)
Output: "[record_id={id}]\n{content}" excerpts, separated by "---"
Filter: source_type=CHAT, project_id (cross-chat within project)
Passthrough: project_id
```

#### `tool_read_long_term_memory`

```
Read current session's cheatsheet as markdown.
Input:  (none)
Output: render_to_markdown(parse_cheatsheet(raw)) or "(none)"
Passthrough: chat_id
```

---

## Session Boundary Timeline

Capture agents use **staggered idle thresholds** so each stage reads settled upstream data.

```mermaid
timeline
    title After user stops sending messages
    section Immediate
        Last response : Historian saves AGENT RESPONSE
    section 40 minutes
        Cheatsheet flush : CheatsheetAgent idle bypass
                         : Remaining tail curated
                         : chat.cheatsheet settled
    section 60 minutes
        Consolidation : ConsolidationAgent promotes to agent_memory
        Habit profile : HabitAgent merges user_profile
```

The **20-minute gap** between cheatsheet flush (40 min) and consolidation/habit (60 min) is intentional: ConsolidationAgent must read a fully settled cheatsheet.

For **long active sessions** (no 60 min idle), ConsolidationAgent also fires when `cheatsheet_cursor_ts − consolidation_cursor_ts > 3 hours`, so project memory does not stall mid-session.
