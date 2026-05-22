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
  └─ every 3h of cursor gap → ConsolidationAgent fires mid-session (see below)

t=0       user stops
t=40 min  IDLE_BYPASS_THRESHOLD → CheatsheetAgent flushes tail records
t=60 min  ConsolidationAgent → promotes verified cheatsheet entries to PROJECT memory
t=60 min  HabitAgent → extracts user habits → USER memory
```

The 20-minute gap between cheatsheet flush (40 min) and consolidation (60 min) is intentional: consolidation reads the cheatsheet, so the cheatsheet must be fully settled first.

### Long active sessions

ConsolidationAgent normally requires a 60-minute idle gap to fire. For sessions that run continuously without a long break, this would leave all findings in the cheatsheet with no promotion to PROJECT memory until the session ends.

To handle this, ConsolidationAgent also fires when the gap between `cheatsheet_cursor_ts` and `consolidation_cursor_ts` exceeds **3 hours**, regardless of session activity:

```
FIRE when:  cheatsheet_cursor > consolidation_cursor
            AND (idle > 60 min  OR  cursor_gap > 3 hours)
```

A full 8-hour working session therefore triggers mid-session consolidation at least twice. Because ConsolidationAgent advances `consolidation_cursor_ts` after each run, subsequent runs only process new entries — no duplication. Only `verified` entries are promoted, which are tool-result-backed and stable enough to consolidate mid-session.

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
    A([poll every 300s]) --> B[get_chats_needing_consolidation\ncheatsheet_cursor > consolidation_cursor\nAND idle > 60 min OR cursor_gap > 3h]
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

### 3. Simple dedup fallback is not semantic

When the LLM is unavailable, `_dedupe` falls back to case-insensitive string equality. "NPT: 54.2 hrs" and "NPT was 54 hours" are treated as different facts and both promoted. The LLM synthesis prompt handles this correctly; the fallback does not.

### 4. No memory staleness or conflict resolution across chats

If chat A reports NPT = 54h and chat B reports NPT = 71h for the same well, both get promoted to `entity_nnm101.insights` as separate strings (via `_dedupe`). The cheatsheet handles intra-session conflicts with the `conflicted` confidence level, but ConsolidationAgent has no equivalent mechanism across sessions. The LLM synthesis prompt is supposed to detect this ("values vary: X, Y"), but it is not guaranteed.

### 5. Well key normalisation is lossy

`_normalize_entity_key("NNM-101")` → `"nnm101"`. This collapses NNM-101 and NNM101 into the same key, which is intentional. But it also collapses hypothetical wells like NNM-10 and NNM-1-0 (if they existed). The normalisation is simple and works for the current well naming convention, but is not robust to arbitrary naming schemes.

### 6. Hardcoded DB credentials in the live test

`test_cheatsheet_live.py` contains `postgresql://ida:ida1793@localhost:5432/ida`. This is fine for local dev but must not be used as-is in any shared or CI environment.

### 7. HabitAgent fires at the same threshold as ConsolidationAgent (60 min)

Both agents fire at 60-minute idle. In practice this means both run simultaneously after a session ends. HabitAgent reads the full transcript; ConsolidationAgent reads the cheatsheet. There is no coordination — they run independently and write to different tables, which is fine. But the ordering is not guaranteed: HabitAgent could write a user profile before ConsolidationAgent has finished merging facts from the same session. This is harmless today but worth noting if ordering ever matters.

---

## Memory Injection into the Agent Workflow

Ida is a single agent running a five-node reasoning loop: **Understand → Plan → Execute → Evaluate → Answer**. There are no sub-agents. Each node receives a subset of available memory — injected statically into the prompt, or loaded dynamically via tool call. The injection pattern determines what Ida knows at each step.

### Injection map

| Memory source | U | P | Ex | Ev | A |
|---|:---:|:---:|:---:|:---:|:---:|
| Soul + Agent.md | ✓ | ✓ | ✓ | ✓ | — |
| CH + CS | ✓ | ✓ | ◌ | — | — |
| User Profile | ✓ | — | — | — | — |
| Project Memory | ↑ | — | ◌ | — | — |
| Skill.md | — | — | ✓ | — | — |

**✓** inject (static, always in prompt) · **↑** retrieve top-K by entity at node start · **◌** on demand via tool · **—** not loaded

### Workflow diagram

```mermaid
flowchart TD
    soul(["Soul + Agent.md"])
    ep(["CH · CS"])
    up(["User Profile"])
    pm(["Project Mem"])
    sk(["Skill.md"])

    U["U"] -->|state| P["P"] --> Ex["Ex"] --> Ev["Ev"] --> A["A"]

    soul --> U
    soul --> P
    soul --> Ex
    soul --> Ev

    ep --> U
    ep --> P
    ep -.->|tool| Ex

    up --> U

    pm -.->|top-K| U
    pm -.->|tool| Ex

    sk -.->|tool| Ex

    style soul fill:#dde,stroke:#99a
    style ep fill:#dfd,stroke:#9a9
    style up fill:#dfd,stroke:#9a9
    style pm fill:#ffd,stroke:#aa9
    style sk fill:#ffd,stroke:#aa9
```

### Context load

| Source | Typical size | U | P |
|---|---|:---:|:---:|
| Soul + Agent.md | ~2–3k | ✓ | ✓ |
| CH (last 12 exchanges) | ~4–6k | ✓ | ✓ |
| CS | ~1–3k | ✓ | ✓ |
| User Profile | ~0.5–1k | ✓ | — |
| Project Memory | top-K only, ~1–2k | ↑ | via state |
| **Total** | **~9–15k** | | |

P receives U's state output rather than a fresh PM injection — no additional tokens beyond what U already synthesized. This keeps P lean and avoids re-reading raw sources.

**CH and CS overlap.** The cheatsheet is a curated distillation of older exchanges. They are complementary as long as CH is bounded to recent exchanges (last N) — CS covers history prior to the window. No redundancy if this bound is enforced.

### Why each node gets what it gets

**U gets the full picture — including top-K project memory retrieval.** U's job is to synthesize: interpret the query, identify the relevant entities and wells, recall what is already known about them, and produce a structured state for the rest of the workflow. PM is retrieved by entity match at the start of U, not dumped wholesale. The retrieved entries travel forward in U's state output.

**P consumes U's state — no raw re-injection.** P receives Soul/Agent.md plus U's output (intent, entities, known facts including retrieved PM). It does not re-read CH, CS, or PM from source. Re-injecting the same raw sources at P is the context-stuffing anti-pattern: P reasons better over a compact, synthesized state than over 15k tokens of repeated context.

**Soul + Agent.md in all reasoning nodes.** Small, stable, and essential for grounding every step. Omitting them causes behavioral drift.

**User Profile in U only.** Preferences shape how the query is interpreted, not how tool calls are planned. By P, the intent is resolved; user style is irrelevant to scheduling.

**Ex gets Skill.md on demand.** The right skill is unknown until the plan is formed. `tool_load_skill` fires at the start of Execute once the task type is clear.

**CH, CS, and PM optionally in Ex.** Most skills don't need them — U/P already provided the context. But a skill verifying a specific fact mid-execution, looking up a long-ago exchange, or retrieving memory for a newly identified entity can do so via tool without re-injecting the full payload.

### Tools for CH, CS, and Project Memory access in Execute

Three tools support on-demand context loading in Execute:

**`tool_read_cheatsheet`** — reads `chat.cheatsheet`, optionally filtered by confidence level or well name. Primary use: verify a fact before running an expensive retrieval tool. Example: `tool_read_cheatsheet(confidence="verified", well="NNM-101")` returns known facts about that well; if complete, skip the DDR query.

**`tool_read_chat_history`** — reads N raw message exchanges from `chat_record` with optional offset. Needed when a skill requires context beyond the exchanges injected in P — for example, a report skill referencing an exchange from 20 turns back.

**`tool_read_memory`** — already exists; reads from `agent_memory` by scope, name, and memory type. Covers project memory retrieval in Execute for skills that need to look up a specific entity that wasn't in the injected project memory summary.

`tool_read_cheatsheet` and `tool_read_chat_history` are new additions; both are thin wrappers over existing DB queries and belong in `mem_storage_toolbox.py`. `tool_read_memory` is already implemented.

### Current vs target state

| Memory source | Write path | Read path |
|---|---|---|
| Soul.md + Agent.md | Manual (checked in) | Injected in current prompts ✓ |
| Chat History | Framework (`load_message_history`) | Loaded in U ✓ |
| Cheatsheet | CheatsheetAgent ✓ | **Not yet injected in U** |
| User Profile | HabitAgent ✓ | **Not yet read** |
| Project Memory | ConsolidationAgent ✓ | **Not yet retrieved in U** (entity-match top-K) |
| Skill.md | Manual (checked in) | Loaded on demand in Ex ✓ |

The write infrastructure is complete. The remaining work is activation: inject cheatsheet, user profile, and project memory into the U/P prompts in `AGENT.md`.

---

### Industry practice review

#### What aligns well

**Skill.md on demand (RAG-over-instructions).** Dynamically loading task instructions based on the resolved intent is the correct pattern, used in production by OpenAI function schemas, LangChain tool descriptions, and retrieval-augmented instruction systems. Loading Skill.md upfront at every node would be expensive and brittle.

**User Profile in U only.** Consistent with how personalization is handled in production assistants. Preferences shape interpretation, not planning logic. Injecting them beyond U adds tokens without value.

**Background agents for memory maintenance.** Running CheatsheetAgent, ConsolidationAgent, and HabitAgent asynchronously — never blocking a user request — is the standard production pattern. Synchronous memory maintenance during inference is an anti-pattern because it adds latency to the critical path.

**Cursor-based incremental processing.** Correct. Timestamp cursors are the industry-standard mechanism for durable, resumable streaming pipelines (used in Kafka consumers, CDC systems, and production agent memory backends like Zep).

**Tool-based access for CH, CS, PM in Execute.** The on-demand retrieval pattern for archival memory is well-established (MemGPT's archival memory, OpenAI's file search). Making tools the access path — rather than always-injecting — is correct for content that is large, selective, and query-specific.

---

#### Concerns

**1. Project memory full-dump — fixed.**

~~The current design injects all project memory into U and P unconditionally.~~ **Resolved:** PM is retrieved by entity match at the start of U (top-K, not full-dump). The retrieved entries travel forward in U's state output to P; P has no direct PM injection. For IDA's current scale, entity-based filtering (by well name extracted from the query) is sufficient. Semantic embedding search over `content_text` is the natural next step when entity matching is insufficient.

**2. Re-injecting the same raw sources at both U and P.**

U and P both receive CH, CS, and project memory. This is the "context stuffing" anti-pattern. In LangGraph, LangChain Expression Language, and ReAct-style frameworks, nodes pass structured state forward — U produces an intent object (interpreted query, relevant entities, known facts) and P consumes that object. P does not re-read the raw sources; it reasons over U's output.

Re-injection has two costs. First, token cost: the same 10–17k tokens are paid twice per user turn. Second, reasoning cost: an LLM presented with the same content in two places (the raw CH and U's synthesis of it) can attend to either, which reduces the reliability of the synthesis step. If U's output is well-structured, P should not need raw CH or CS — it has everything U extracted.

The fix: define a structured `ReasoningState` that U populates (intent, entities, what is already known, what still needs investigation) and P consumes. P only needs Soul/Agent.md, project memory (for planning around cross-session knowledge), and the ReasoningState. Raw CH and CS drop out of P.

**3. Cheatsheet is injected without confidence filtering.**

The cheatsheet has three confidence levels: `verified`, `inferred`, and `conflicted`. The injection design injects all of them. Presenting `conflicted` entries — two contradictory values for the same metric — to a reasoning node as flat context is unreliable: the model may pick one value arbitrarily without recognizing the conflict.

Industry practice for structured entity stores (e.g., knowledge graphs, Weaviate filtered retrieval) is to filter or label by confidence at read time. Two options:

- **Hard filter:** inject only `verified` entries into U and P as established facts; skip `inferred` and `conflicted` entirely (they remain available via `tool_read_cheatsheet` for skills that need them).
- **Labeled injection:** include all entries but prefix conflicted entries with an explicit marker: `[CONFLICTED — two values reported: X, Y]`. This preserves the information while preventing silent misuse.

**4. Evaluate is grounded only in Soul/Agent.md.**

Ev receives Soul.md + Agent.md plus the conversation history (Ex's output flows through). This is thin. In Reflexion (Shinn et al., 2023), CRITIC (Gou et al., 2023), and Self-RAG (Asai et al., 2023), the reflection/evaluation step explicitly receives: the original task, the evaluation criteria, and the generated output. Ev reasoning solely from persona and general agent instructions will produce shallow evaluation — it can check tone and format but not correctness, completeness, or whether the answer is grounded in tool results.

Ev should also receive the original user query and an explicit quality rubric (e.g., "every numerical claim must cite a tool result"). This belongs in the Evaluate section of `AGENT.md`, not as additional memory injection per se — but it is a gap in the current design.

---

#### Summary

| Decision | Assessment |
|---|---|
| Skill.md on demand | ✓ Correct — RAG-over-instructions pattern |
| User Profile in U only | ✓ Correct |
| Background agents, cursor-based | ✓ Correct |
| Tool access for CH/CS/PM in Ex | ✓ Correct |
| PM top-K retrieval in U, state to P | ✓ Fixed — entity-conditioned, not full-dump |
| Re-injecting CH + CS in P | ✗ Context stuffing — P should consume U's state output |
| Cheatsheet injected without confidence filter | ✗ Risk of conflicted entries misleading reasoning |
| Ev grounded only in Soul/Agent.md | ✗ Insufficient — needs task + quality criteria |

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
