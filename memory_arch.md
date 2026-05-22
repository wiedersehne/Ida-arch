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
| User Profile | ✓ | — | — | ✓ | — |
| Project Memory | ↑ | — | ◌ | — | — |
| Skill.md | — | ✓ | ✓ | — | — |

**✓** inject (static, always in prompt) · **↑** retrieve top-K by entity at node start · **◌** on demand via tool · **—** not loaded · P also receives U's state output (intent, entities, known facts) — not listed as a memory source since it is produced within the workflow

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
    up --> Ev

    pm -.->|top-K| U
    pm -.->|tool| Ex

    sk -.->|tool| P
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
| Project Memory (top-K) | ~1–2k | ↑ | — |
| U's state output | ~1–2k | — | ✓ |
| **Total** | | **~9–15k** | **~8–14k** |

P needs CH and CS directly: the planner must read the raw conversation and established findings to write a non-redundant execution plan, not just U's interpretation of them. PM is the exception — P receives those via U's state (which already carried the relevant top-K entries), so no direct PM injection at P.

**CH and CS overlap.** The cheatsheet is a curated distillation of older exchanges. They are complementary as long as CH is bounded to recent exchanges (last N) — CS covers history prior to the window. No redundancy if this bound is enforced.

### Why each node gets what it gets

**U gets the full picture — including top-K project memory retrieval.** U's job is to synthesize: interpret the query, identify the relevant entities and wells, recall what is already known about them, and produce a structured state for the rest of the workflow. PM is retrieved by entity match at the start of U, not dumped wholesale. The retrieved entries travel forward in U's state output.

**P gets CH, CS, and U's state — but not PM.** The planner needs the raw conversation (CH) and established findings (CS) to write a non-redundant execution plan — U's interpretation of them is not a substitute. PM is the exception: P receives the relevant entries via U's state output (which already captured the top-K retrieved subset), so no direct PM injection at P is needed.

**Soul + Agent.md in all reasoning nodes.** Small, stable, and essential for grounding every step. Omitting them causes behavioral drift.

**User Profile in U and Ev.** In U it shapes how the query is interpreted. In Ev it grounds the evaluation: did the answer match this user's detail level, domain knowledge, and output preferences? Without it, Ev can only check factual correctness, not whether the response was right for this person.

**Skill.md in P and Ex.** P needs the skill's approach and constraints to write a correct execution plan — scheduling tool calls without knowing the skill's required sequence produces an invalid plan. Ex uses it to guide step-by-step execution. Both nodes load it via `tool_load_skill` once the task type is resolved.

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

**2. Re-injecting raw sources at both U and P — partially fixed.**

PM is fully resolved: P receives it only via U's state, not as a direct injection. CH and CS are intentionally re-injected at P: the planner needs the raw conversation and established findings to write a correct execution plan, not just U's synthesis of them. The remaining duplication (CH+CS in both U and P) is a deliberate trade-off — the context cost (~8–14k at P) is acceptable, and removing it would risk under-informed planning.

**3. Cheatsheet is injected without confidence filtering.**

The cheatsheet has three confidence levels: `verified`, `inferred`, and `conflicted`. The injection design injects all of them. Presenting `conflicted` entries — two contradictory values for the same metric — to a reasoning node as flat context is unreliable: the model may pick one value arbitrarily without recognizing the conflict.

Industry practice for structured entity stores (e.g., knowledge graphs, Weaviate filtered retrieval) is to filter or label by confidence at read time. Two options:

- **Hard filter:** inject only `verified` entries into U and P as established facts; skip `inferred` and `conflicted` entirely (they remain available via `tool_read_cheatsheet` for skills that need them).
- **Labeled injection:** include all entries but prefix conflicted entries with an explicit marker: `[CONFLICTED — two values reported: X, Y]`. This preserves the information while preventing silent misuse.

**4. Evaluate grounding — partially fixed.**

User Profile is now injected into Ev, enabling persona-aware evaluation: did the answer match this user's detail level, domain knowledge, and output preferences? The remaining gap — explicit quality criteria (e.g., "every numerical claim must cite a tool result") — belongs in the Evaluate section of `AGENT.md` as a rubric, not as additional memory injection.

---

#### Summary

| Decision | Assessment |
|---|---|
| Skill.md in P and Ex | ✓ Correct — planner needs skill constraints |
| User Profile in U and Ev | ✓ Fixed — Ev now persona-aware |
| Background agents, cursor-based | ✓ Correct |
| Tool access for CH/CS/PM in Ex | ✓ Correct |
| PM top-K retrieval in U, state to P | ✓ Fixed — entity-conditioned, not full-dump |
| CH + CS in both U and P; PM via state | ~ Deliberate — CH/CS needed by planner; PM fixed |
| Cheatsheet injected without confidence filter | ✗ Risk of conflicted entries misleading reasoning |
| Ev quality rubric | ✗ Still missing — belongs in Ev section of AGENT.md |

---

## Implementation Plan

Two parallel tracks: (1) the tools the LLM calls mid-execution to pull additional context; (2) the `MemoryService` that assembles the initial prompt context before each LLM call. These are complementary — the service handles static injection and top-K retrieval at node entry; the tools handle on-demand access during execution.

```
Before LLM call        During LLM execution
─────────────────      ──────────────────────
MemoryService          tool_read_chat_history
 └─ bundle_for_node     tool_read_cheatsheet
     CH, CS, Profile    tool_read_memory (top-K)
     Soul, Agent.md
     Skill.md
     PM (top-K)
```

---

### Step 1 — `tool_read_cheatsheet`

**File:** `app/agents/tools/toolbox/mem_storage_toolbox.py`

The current `tool_read_memory` reads from `agent_memory` (cross-session). It does not touch `chat.cheatsheet` (per-chat). A separate tool is needed.

```python
# args_schema
{
  "confidence": {
    "type": "string",
    "enum": ["verified", "inferred", "conflicted"],
    "description": "Filter entries by confidence level. Omit to return all."
  },
  "well": {
    "type": "string",
    "description": "Filter to a specific well name (normalised: NNM-101 → nnm101). Omit for all wells."
  }
}
# passthrough_params: chat_id, project_id, session_id, trace_id
```

Implementation: call `project_service.get_chat_cheatsheet(chat_id)`, parse with `parse_cheatsheet()`, filter by `confidence` and `_normalize_entity_key(well)`, return formatted text grouped by bucket (`data_insights`, `key_facts`, `lessons_learned`).

Primary use in Execute: verify a known fact before running an expensive retrieval tool. If `confidence="verified"` returns complete data for the queried well, the DDR/WellView query can be skipped.

---

### Step 2 — `tool_read_chat_history`

**File:** `app/agents/tools/toolbox/mem_storage_toolbox.py`

The framework injects recent exchanges via `load_message_history()`. This tool provides access to exchanges outside that window — older history or history from a specific point.

```python
# args_schema
{
  "n": {
    "type": "integer",
    "default": 20,
    "description": "Number of exchanges (user+agent pairs) to return."
  },
  "before_record_id": {
    "type": "integer",
    "description": "Return exchanges before this record ID. Use to page backwards through history."
  },
  "oldest_first": {
    "type": "boolean",
    "default": true
  }
}
# passthrough_params: chat_id, project_id, session_id, trace_id
```

Implementation: call `project_service.get_chat_records(chat_id, limit=n*2, before_record_id=..., oldest_first=...)`, pair user and agent records chronologically, return as formatted transcript.

Primary use in Execute: report and reconciliation skills that need to reference exchanges from earlier in the conversation than the injected window covers.

---

### Step 3 — Extend `tool_read_memory` for top-K entity retrieval

**File:** `app/agents/tools/toolbox/mem_storage_toolbox.py` (existing tool)

Current behaviour: exact name lookup (`name="entity_nnm101"`). Add a `query` parameter that performs entity/keyword matching against `name` and `content_text`, returning the top-K most relevant entries.

```python
# additional args_schema fields
{
  "query": {
    "type": "string",
    "description": "Entity or keyword search against memory name and content. Returns top_k matches. Use instead of 'name' for broad retrieval."
  },
  "top_k": {
    "type": "integer",
    "default": 5,
    "description": "Maximum number of entries to return when using 'query'."
  }
}
```

Implementation (Phase 1 — keyword match): when `query` is provided, fetch all PROJECT-scoped entries for `project_id`, rank by: (a) well name present in `name`, (b) keyword overlap with `content_text`. Return top-K.

Implementation (Phase 2 — semantic search): add an `embedding` vector column to `agent_memory`, embed `content_text` at write time (ConsolidationAgent), use pgvector cosine similarity at read time. Phase 1 is sufficient for current project scale.

This is also the tool `MemoryService.retrieve_project_memory()` calls internally — the service is not a separate DB path, it composes the same tools.

---

### Step 4 — `MemoryService`

**File:** `app/services/memory/memory_service.py`

The service assembles the initial prompt context for each node before the LLM call. It is the Python-layer counterpart to the tool functions — tools run inside the LLM execution loop; the service runs in the framework before each node is invoked.

#### Data structures

```python
@dataclass
class NodeContext:
    system: str       # system prompt: soul + agent.md (+ skill.md for P and Ex)
    context: str      # injected context block: CH, CS, user profile, retrieved PM
    state: str = ""   # structured output from prior node (U→P, P→Ex)
```

#### Interface

```python
class MemoryService:
    def __init__(
        self,
        project_service: ProjectService,
        agent_memory_service: AgentMemoryService,
        chat_id: int,
        project_id: int,
        user_id: int,
    ): ...

    # Individual loaders
    def load_soul(self) -> str
    def load_agent_md(self) -> str
    def load_skill(self, skill_name: str) -> str
    def load_chat_history(self, n: int = 12) -> str
    def load_cheatsheet(self, confidence: str = None) -> str
    def load_user_profile(self) -> str
    def retrieve_project_memory(self, entities: list[str], top_k: int = 5) -> str

    # Node bundles — standard assembly per the injection map
    def bundle_for_node(self, node: str, **kwargs) -> NodeContext
```

#### `bundle_for_node` mapping

```python
match node:
    case "U":
        system  = soul + agent_md
        context = chat_history(12) + cheatsheet() + user_profile()
                  + retrieve_project_memory(entities=kwargs["entities"])
    case "P":
        system  = soul + agent_md + skill(kwargs["skill_name"])
        context = chat_history(12) + cheatsheet()
        state   = kwargs["u_state"]   # structured output from U
    case "Ex":
        system  = soul + agent_md + skill(kwargs["skill_name"])
        context = ""                  # tools handle on-demand loading
        state   = kwargs["p_state"]   # execution plan from P
    case "Ev":
        system  = soul + agent_md
        context = user_profile()
        state   = kwargs["ex_state"]  # results from Ex
```

#### Caching

`soul`, `agent_md`, and `skill` files are read from disk once per request and cached in `self._cache`. DB-backed loaders (`chat_history`, `cheatsheet`, `user_profile`, `retrieve_project_memory`) are not cached — they must reflect current DB state.

#### Where it fits in the call stack

```
User request
  └─ Ida agent (Python)
       ├─ memory_service.bundle_for_node("U", entities=[...])  ← MemoryService
       ├─ LLM call: Understand node
       │    └─ produces u_state (intent, entities, known facts)
       ├─ memory_service.bundle_for_node("P", skill_name=..., u_state=...)
       ├─ LLM call: Plan node
       │    └─ produces p_state (execution plan)
       ├─ memory_service.bundle_for_node("Ex", skill_name=..., p_state=...)
       ├─ LLM call: Execute node
       │    ├─ tool_read_cheatsheet(...)        ← tool (LLM-driven)
       │    ├─ tool_read_memory(query=...)      ← tool (LLM-driven)
       │    └─ produces ex_state (results)
       ├─ memory_service.bundle_for_node("Ev", ex_state=...)
       └─ LLM call: Evaluate node
```

---

### Step 5 — Wire into Ida

Once Steps 1–4 are complete, the activation work is:

1. **Instantiate `MemoryService`** at request entry in the Ida agent Python code, passing `chat_id`, `project_id`, `user_id`.
2. **Replace manual prompt construction** at each node with `bundle_for_node(node, ...)`.
3. **Update `AGENT.md`** — add instructions for each node to describe what context it receives and what structured state it must produce (especially U, which must emit `entities` for PM retrieval at the next call).
4. **Register the two new tools** (`tool_read_cheatsheet`, `tool_read_chat_history`) in `mem_storage_toolbox.py` and wire them via `register_tools()`.

#### Sequencing

| Step | Dependency | Risk |
|---|---|---|
| 1 — `tool_read_cheatsheet` | None | Low — thin wrapper over existing DB query |
| 2 — `tool_read_chat_history` | None | Low — thin wrapper over existing DB query |
| 3 — extend `tool_read_memory` | None | Low — additive, existing name lookup unchanged |
| 4 — `MemoryService` | Steps 1–3 | Medium — new abstraction, needs tests |
| 5 — Wire into Ida | Step 4 | High — changes prompt construction for every node |

Steps 1–3 are independent and can be done in parallel. Step 4 depends on 1–3 only for its internal `retrieve_project_memory` call; the rest of its loaders are standalone. Step 5 is the highest-risk step and should be done behind a feature flag or tested on a non-production chat first.

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
