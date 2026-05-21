# IDA Memory Architecture

## Overview

IDA uses a four-layer memory model. Each layer has a defined scope, lifetime, and purpose. They are not interchangeable.

| Layer | Scope | Mechanism | Lifetime | Written by |
|---|---|---|---|---|
| ContextApi | One tool chain | `allowed_read/write_from_context_items` | Single request | Tool functions |
| Raw chat history | One session | `load_message_history()` | In-memory, per call | Framework |
| Cheatsheet | One conversation | `chat.cheatsheet` (JSON) | Persistent, per chat | CheatsheetAgent |
| Agent memory | Cross-session | `agent_memory` table | Persistent, scoped | Background agents |

The background agents — CheatsheetAgent, ConsolidationAgent, HabitAgent — are the bridge between ephemeral conversation and long-term memory. They run continuously alongside the main application and never block a user request.

---

## Session Boundary Model

Three idle thresholds define the memory lifecycle for a session:

```
Active session
  └─ exchanges scroll out of TAIL_WINDOW (12) → curated immediately

t=0       user stops
t=40 min  IDLE_BYPASS_THRESHOLD → CheatsheetAgent flushes tail records
t=60 min  ConsolidationAgent → promotes verified cheatsheet entries to PROJECT memory
t=60 min  HabitAgent → extracts user habits → USER memory
```

The 20-minute gap between cheatsheet flush (40 min) and consolidation (60 min) is intentional: consolidation reads the cheatsheet, so the cheatsheet must be fully settled first.

---

## CheatsheetAgent

**Purpose:** Maintain a structured, per-chat memory of confirmed findings, data quality issues, and lessons learned. Updated incrementally after each exchange.

**Storage:** `chat.cheatsheet` — a JSON blob with three typed buckets:

```json
{
  "data_insights":   [{"content": "...", "confidence": "verified|inferred|conflicted", "well": "NNM-101"}],
  "key_facts":       [{"content": "...", "confidence": "verified|inferred|conflicted"}],
  "lessons_learned": [{"content": "...", "confidence": "verified|inferred", "well": "NNM-101"}]
}
```

**Confidence rules:**
- `verified` — value appears verbatim in a tool result block
- `inferred` — derived from agent narrative, not a tool result
- `conflicted` — same well + metric has two different values; both entries preserved

### Workflow

```mermaid
flowchart TD
    A([message_bus: record_saved]) --> B{AGENT RESPONSE?}
    B -- No --> Z([ignore])
    B -- Yes --> C[enqueue chat_id, wake main loop]

    C --> D[drain queue, dedup by chat_id]
    D --> E[_process_pending_for_chat]

    E --> F[get_chat_records\noldest_first=True, limit=TAIL_N+1]
    F --> G{any records?}
    G -- No --> H([return False])
    G -- Yes --> I{oldest in tail?}

    I -- No, scrolled out --> L
    I -- Yes, cursor_ts=None\nshort chat --> L
    I -- Yes, cursor_ts set --> J{idle > 40 min?}

    J -- No --> K([defer, return False])
    J -- Yes, bypass --> L

    L{record in _in_flight?}
    L -- Yes --> M([blocked, return False])
    L -- No --> N[add to _in_flight\nsubmit to executor]

    N --> O[_curate_record]
    O --> P[find preceding user query]
    P --> Q{query found?}
    Q -- No --> R[advance cursor\nreturn False]
    Q -- Yes --> S[CheatsheetService.update_cheatsheet\nLLM curator call ~15s]
    S --> T[advance cheatsheet_cursor_ts]
    T --> U[discard from _in_flight]

    style S fill:#f9f,stroke:#333
```

**Key implementation details:**
- `oldest_first=True` on `get_chat_records` ensures FIFO processing (fixes a DESC LIMIT bug that caused permanent skips)
- `_in_flight: set[int]` prevents the same record being submitted twice while a slow LLM call is running across a 2s main loop cycle
- `max_workers=4` allows up to 4 chats to curate in parallel; `_in_flight` serialises within each individual chat
- Fallback poll every 120s recovers missed events after server restarts
- Tail window size is configurable via `agent_config["tail_window_size"]`; defaults to 12

---

## ConsolidationAgent

**Purpose:** Promote verified cheatsheet findings into persistent PROJECT-scoped `agent_memory` after a session goes idle. Enables cross-session recall without re-reading transcripts.

**Condition to fire:** `cheatsheet_cursor_ts > consolidation_cursor_ts` AND last agent response older than 60 minutes. Both conditions must hold — the cursor comparison ensures only new cheatsheet content is consolidated on subsequent sessions.

**Storage:** `agent_memory` table, scope=PROJECT:
- `entity_{well}` — per-well data insights (`{"well": "nnm101", "insights": [...]}`)
- `project_facts` — data quality gaps, confirmed well characteristics
- `project_lessons` — transferable operational insights

**Promotion rules:**
| Bucket | What gets promoted |
|---|---|
| `data_insights` | `confidence == "verified"` only |
| `key_facts` | `confidence == "verified"` only |
| `lessons_learned` | `confidence in ("verified", "inferred")` |

### Workflow

```mermaid
flowchart TD
    A([poll every 300s]) --> B[get_chats_needing_consolidation\nidle > 60 min\ncheatsheet_cursor > consolidation_cursor]
    B --> C{any chats?}
    C -- No --> A
    C -- Yes --> D[for each chat]

    D --> E[get_chat_cheatsheet]
    E --> F{empty or legacy text?}
    F -- Yes --> G[advance consolidation_cursor\nskip]
    F -- No --> H[parse CheatsheetData]

    H --> I[filter verified data_insights\ngroup by well, normalize key\nNNM-101 → nnm101]
    I --> J[read existing PROJECT memory]
    J --> K[LLM synthesis or _dedupe fallback]
    K --> L[set_object entity_well]

    L --> M[filter verified key_facts]
    M --> N[LLM synthesis or _dedupe]
    N --> O[set_object project_facts]

    O --> P[filter verified + inferred lessons]
    P --> Q[LLM synthesis or _dedupe]
    Q --> R[set_object project_lessons]

    R --> S[advance consolidation_cursor_ts]

    style K fill:#f9f,stroke:#333
    style N fill:#f9f,stroke:#333
    style Q fill:#f9f,stroke:#333
```

**LLM synthesis prompt:** merges existing + new facts, deduplicates, resolves numerical conflicts, and returns a clean JSON array. Falls back to simple case-insensitive `_dedupe` if LLM is unavailable.

---

## HabitAgent

**Purpose:** Learn how each user prefers to work with Ida — query style, output preferences, domain knowledge level — and store this as a USER-scoped profile that persists across all sessions and projects.

**Storage:** `agent_memory` table, scope=USER, name=`user_profile` — a free-text profile structured with markdown headers.

**Condition to fire:** chat idle for 60 minutes with unprocessed records since `habit_cursor_ts`.

### Workflow

```mermaid
flowchart TD
    A([poll every N seconds]) --> B[get_idle_chats\nidle > 60 min]
    B --> C{any chats?}
    C -- No --> A
    C -- Yes --> D[for each idle chat]

    D --> E[get user_id from chat]
    E --> F{user_id found?}
    F -- No --> G[advance cursor, skip]
    F -- Yes --> H[load transcript since habit_cursor_ts\nuser + agent records]

    H --> I{any records?}
    I -- No --> J([return False])
    I -- Yes --> K[get existing USER profile\nfrom agent_memory]

    K --> L[LLM: extract habits from transcript\nmerge with existing profile]
    L --> M{habits tag in response?}
    M -- No --> N[advance cursor\nno write]
    M -- Yes --> O[set_object user_profile\nscope=USER]
    O --> P[advance habit_cursor_ts]

    style L fill:#f9f,stroke:#333
```

---

## What Works Well

**Incrementality.** All three agents use cursor timestamps to track progress. Restarts, crashes, and slow LLM calls are all safe — work resumes from where it left off, never reprocessed.

**Clean confidence model.** The `verified / inferred / conflicted` distinction in the cheatsheet is meaningful and enforced: only `verified` entries reach long-term memory. Conflicted entries are preserved in full — both values are kept, neither is silently dropped.

**Correct FIFO ordering.** The `oldest_first=True` fix ensures the cheatsheet processes exchanges in the order they happened. Before this fix, DESC LIMIT caused the oldest records to be permanently skipped when 4+ exchanges accumulated.

**In-flight dedup.** The `_in_flight` set prevents the same exchange being submitted twice while its LLM call is running. Without it, every 2-second main loop cycle would re-submit the same record, clogging the executor.

**Session boundary model is coherent.** The 40/60-minute thresholds (cheatsheet flush → consolidation) have a 20-minute safety margin. ConsolidationAgent cannot race ahead of CheatsheetAgent.

**No LLM calls block user requests.** All curation, consolidation, and habit extraction happen in background threads. The user never waits for memory work.

---

## Honest Problems

### 1. Sub-agents don't read from memory (the biggest gap)

`tool_read_memory` and `tool_upsert_memory` exist and work. No sub-agent AGENT.md currently instructs a memory read at session start. The PROJECT-scoped memories written by ConsolidationAgent and the USER-scoped profiles written by HabitAgent are never loaded into the LLM context. The infrastructure is complete; the activation is missing.

**Impact:** IDA forgets everything between sessions. A returning user gets no benefit from previous consolidated findings.

### 2. Cheatsheet curation lags during fast sessions

Each curator LLM call takes ~15s. During fast interactions (15–20s between exchanges), the cheatsheet can fall several exchanges behind the active conversation. The tail protection defers the last 12 exchanges deliberately, but during the active session the cheatsheet cursor and the raw context window (12–15 exchanges) can leave a gap of uncovered exchanges that appear in neither.

**Specifically:** IDA agent loads 12 raw exchanges, SME agent loads 15. TAIL_N=12 means the 3 oldest of the SME's 15 loaded exchanges are curated AND in raw context — mild redundancy. More importantly, if the cheatsheet lags significantly, exchanges between the cheatsheet cursor and the raw window start are not covered by either.

**Current mitigation:** none. The gap closes at idle flush (40 min).

### 3. ConsolidationAgent only runs at session end

If a session is very long and active (no 60-minute idle gap), ConsolidationAgent never fires during that session. In a full working day of continuous use, all findings stay in the cheatsheet and never reach PROJECT memory until the user stops for an hour.

### 4. Simple dedup fallback is not semantic

When the LLM is unavailable, `_dedupe` falls back to case-insensitive string equality. "NPT: 54.2 hrs" and "NPT was 54 hours" are treated as different facts and both promoted. The LLM synthesis prompt handles this correctly; the fallback does not.

### 5. No memory staleness or conflict resolution across chats

If chat A reports NPT = 54h and chat B reports NPT = 71h for the same well, both get promoted to `entity_nnm101.insights` as separate strings (via `_dedupe`). The cheatsheet handles intra-session conflicts with the `conflicted` confidence level, but ConsolidationAgent has no equivalent mechanism across sessions. The LLM synthesis prompt is supposed to detect this ("values vary: X, Y"), but it is not guaranteed.

### 6. Well key normalisation is lossy

`_normalize_entity_key("NNM-101")` → `"nnm101"`. This collapses NNM-101 and NNM101 into the same key, which is intentional. But it also collapses hypothetical wells like NNM-10 and NNM-1-0 (if they existed). The normalisation is simple and works for the current well naming convention, but is not robust to arbitrary naming schemes.

### 7. Hardcoded DB credentials in the live test

`test_cheatsheet_live.py` contains `postgresql://ida:ida1793@localhost:5432/ida`. This is fine for local dev but must not be used as-is in any shared or CI environment.

### 8. HabitAgent fires at the same threshold as ConsolidationAgent (60 min)

Both agents fire at 60-minute idle. In practice this means both run simultaneously after a session ends. HabitAgent reads the full transcript; ConsolidationAgent reads the cheatsheet. There is no coordination — they run independently and write to different tables, which is fine. But the ordering is not guaranteed: HabitAgent could write a user profile before ConsolidationAgent has finished merging facts from the same session. This is harmless today but worth noting if ordering ever matters.

---

## Appendix: DB Schema (relevant columns)

```
chat
  cheatsheet              TEXT         — JSON CheatsheetData, updated by CheatsheetAgent
  cheatsheet_cursor_ts    TIMESTAMP    — last exchange processed by CheatsheetAgent
  consolidation_cursor_ts TIMESTAMP    — last cheatsheet state consolidated to agent_memory
  habit_cursor_ts         TIMESTAMP    — last exchange processed by HabitAgent

agent_memory
  agent_id    TEXT         — "consolidation_agent", "habit_agent", etc.
  name        TEXT         — "entity_nnm101", "project_facts", "user_profile", ...
  scope       ENUM         — GLOBAL / ORG / PROJECT / USER
  memory_type ENUM         — DATA_INSIGHT / USER_PROFILE / ...
  project_id  INT          — set when scope=PROJECT
  user_id     INT          — set when scope=USER
  object      JSONB        — structured payload
  content_text TEXT        — plain-text rendering for RAG / display
  confidence  FLOAT        — 0–1, written by consolidation_agent
```
