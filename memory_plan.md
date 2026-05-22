# IDA Memory Implementation Plan

**Reference:** `docs/memory-gap-analysis.md`, `docs/memory-architecture.md`
**Planning:** May 7, 2026 · Updated May 22, 2026

---

## Goal

Give IDA persistent memory across sessions. A drilling engineer remembers what was
established about a well last week — IDA should too. The pipeline has four stages:

1. **Extract per-exchange** — CheatsheetAgent distils each turn into structured findings
2. **Extract per-session** — HabitAgent profiles user behaviour at session end
3. **Consolidate at session close** — ConsolidationAgent promotes durable facts to `agent_memory`
4. **Inject at session start** — orchestrator loads PROJECT + USER memory before answering

The agent architecture is a **single Ida agent** running a five-node loop
(Understand → Plan → Execute → Evaluate → Answer). There are no sub-agents.
Memory injection follows the map in `docs/memory-architecture.md`.

---

## Dependency Chain

```
EX-4 (cheatsheet quality)
        ↓
EP-4 (ConsolidationAgent)  ←  EX-5 (schema migration)
        ↓                               ↓
Steps 1–3 (new tools)        EX-2 phase 2 (promote chat.habit → USER memory)
        ↓
Step 4 (MemoryService)
        ↓
Step 5 (wire into Ida)

EP-1 (chat history search) — independent, no prerequisites
EP-5 (context overflow) — depends on EP-4 (same agent, second trigger)
HabitAgent (EX-2 phase 1) → chat.habit → ConsolidationAgent → agent_memory USER scope
```

EX-4 is the critical path. ConsolidationAgent quality is wholly dependent on
cheatsheet provenance. Nothing downstream is reliable until cheatsheet extraction
quality is validated on real well sessions.

---

## May 8–15: EX-4 + EX-5 + EX-2 Phase 1

**Goal:** Reliable, structured cheatsheet extraction. Extended agent_memory schema.
Session-end habit extraction. These three are prerequisites for everything downstream.

---

### Task 1.1 — Fix cheatsheet message priority

**File:** `backend/app/agents/workers/cheatsheet_agent.py`

The curator must receive the raw message, not compressed. Confidence detection
(`verified` vs `inferred`) depends on tool result block structure in the raw content,
which compression destroys. History loading (user query lookup) uses
`compressed_message or message` — compressed first to reduce noise.

**Change:** Curator input: `message or compressed_message`. History load: `compressed_message or message`.

---

### Task 1.2 — Scope curator to scrolled-out turns only

**File:** `backend/app/agents/workers/cheatsheet_agent.py`

The cheatsheet preserves what the tail window can no longer hold. Running the curator
on in-tail turns creates redundant entries and wastes LLM calls.

**Change:** Implement `_get_tail_start_id()` — fetch the last `TAIL_WINDOW_SIZE+1` (3)
AGENT RESPONSE records. Tail window = 2 complete exchanges. Skip records with
`id >= tail_start_id`. Bypass condition: ≤ 2 exchanges exist or last activity > 1 hour
(`tail_start_id = sys.maxsize`).

---

### Task 1.3 — Structured JSON storage

**Files:** `backend/app/services/cheatsheet/cheatsheet_schema.py`,
`backend/app/services/cheatsheet/cheatsheet_service.py`

Plain-text cheatsheet has no machine-readable provenance. ConsolidationAgent needs
structured entries with confidence and source record ID to verify before promoting.

**Change:** Store `chat.cheatsheet` as JSON with two buckets:

```json
{
  "data_insights": [
    {"content": "...", "confidence": "verified", "record_id": 847, "well": "NNM-101"}
  ],
  "key_facts": [
    {"content": "...", "confidence": "inferred", "record_id": 801}
  ]
}
```

`data_insights` entries require `well` field (entity-specific). `key_facts` omit it.
`render_to_markdown()` produces injection-ready markdown — same token cost as plain text.
Backward-compatible: legacy plain-text cheatsheets returned as-is until replaced on next write.

Note: `user_habits` bucket is not included — habits are HabitAgent's domain (see Task 7),
not derivable per-exchange by the curator.

---

### Task 1.4 — Update curator prompt

**File:** `backend/app/services/cheatsheet/cheatsheet_curator_prompt.py`

**Change:** Prompt must enforce:
- Two-bucket output (`data_insights` / `key_facts`) matching the schema above
- Confidence tiers: `verified` (value verbatim in tool result), `inferred` (from narrative),
  `conflicted` (contradicts existing entry — keep both with `prior_record_id`)
- **Numeric precision rule:** structured table values (NPT hours, ROP m/hr, depth, cost) must
  be copied exactly — "12.1%" not "significant NPT"
- **Conflict detection:** before adding a new entry for a well/metric already present, emit a
  `conflicted` entry; never silently overwrite
- **Domain knowledge rule:** capture non-universal terms verbatim (operator-specific names,
  local conventions); skip only universal unambiguous concepts (NPT, ROP, BHA)

---

### Task 1.5 — Parse and store curator output

**Files:** `backend/app/services/cheatsheet/cheatsheet_service.py`,
`backend/app/agents/workers/cheatsheet_agent.py`

**Change:** `update_cheatsheet()` receives prior cheatsheet as `[[PREVIOUS_CHEATSHEET]]`,
parses `<cheatsheet>` XML tags from LLM output, stamps `record_id` on new entries, saves
via `CheatsheetService`. Malformed output is logged and skipped — poll loop must not crash.

---

### Task 1.6 — agent_memory schema migration (EX-5)

**Files:** `backend/app/db/agents_memory.py`, migration file

Extend `agent_memory` to support typed, versioned, FTS-searchable, user-scoped memories.

**Change — migration (additive, all new columns nullable):**
- Add `AgentMemoryScope.USER` to the scope enum
- Create `AgentMemoryType` enum: `user_profile`, `data_insight`, `bookkeeping`, `knowledge`
  - `USER_PROFILE` — USER scope: habits, preferences, conventions
  - `DATA_INSIGHT` — PROJECT scope: per-well/file findings and lessons
  - `BOOKKEEPING` — GLOBAL scope: background agent cursors
  - `KNOWLEDGE` — ORG/GLOBAL scope: domain facts
- Add columns: `memory_type` (indexed), `user_id` (FK to user, CASCADE), `content_text` (Text),
  `confidence` (Float), `size_chars` (Integer), `version` (Integer, default 1)
- GIN index on `to_tsvector('english', content_text)` for FTS
- Backfill existing rows: `memory_type = 'bookkeeping'`

**Change — `set_object()` in `agents_memory.py`:**
Auto-populate `content_text` from the `object` JSONB, set `size_chars = len(content_text)`,
increment `version` on each upsert.

**Known issue:** SQLAlchemy sends enum `.name` (`USER_PROFILE`) instead of `.value`
(`user_profile`) to the Postgres native enum column. Omit `memory_type` on writes until
fixed (see Task 9 — May 18 week). All bookkeeping writes are unaffected.

---

### Task 1.7 — HabitAgent (EX-2 phase 1)

**Files:** `backend/app/agents/workers/habit_agent.py`,
`backend/app/services/habit/habit_agent_prompt.py`,
`backend/app/db/project.py` (add `chat.habit`, `chat.habit_cursor_ts` columns)

Extract per-session user behaviour at session end. Two-stage design: HabitAgent writes
to `chat.habit` per chat; ConsolidationAgent later promotes across sessions to USER-scoped
`agent_memory` (EX-2 phase 2, Task 10).

**Agent behaviour:**
1. Poll every 5 minutes (`poll_interval=300`); sleep 5s between chats if work was found
2. Detect idle chats via `get_idle_chats(IDLE_THRESHOLD=1h)` — SQL: `max(response_ts) < now - 1h`
   AND `max(response_ts) > COALESCE(habit_cursor_ts, epoch)`
3. Fetch USER records and AGENT RESPONSE records separately since cursor; merge and sort by `id`
4. Use `compressed_message or message` for AGENT records — strips tool JSON noise
5. Load current `chat.habit` as existing context (`existing_habits`)
6. LLM extracts/updates habit profile across 5 dimensions: QUERY STYLE, INTERACTION STYLE,
   OUTPUT PREFERENCES, DOMAIN FOCUS, EXPERTISE SIGNALS
7. Write updated profile to `chat.habit` via `ProjectService.update_chat_habit()`
8. Advance `chat.habit_cursor_ts` to `latest_ts`

**Output format:** Structured text with `## DIMENSION` headers — not JSON.
**Update strategy:** confirms → unchanged; seen again → mark confirmed; contradicts → soften; new → add.
**Cursor:** `chat.habit_cursor_ts` (per-chat TIMESTAMP column) — not stored in `agent_memory`.

---

### Validation

- Run real well sessions with NPT analysis; inspect raw `chat.cheatsheet` JSON after each
- Verify: NPT%, ROP, depth appear verbatim in `data_insights` entries
- Verify: turns still inside the tail window have no corresponding cheatsheet entries
- Verify: `verified` entries each have a `record_id` matching a tool-result turn
- After a session goes idle: verify `chat.habit` written with populated dimensions

---

## May 18–22: CC Fix + Wiring + ConsolidationAgent Start

**Goal:** Fix the last CC reliability gap (RAG holds noisy originals). Wire completed
tools into agent instructions. Fix the SQLModel enum bug. Get ConsolidationAgent's
cheatsheet promotion path working end-to-end.

---

### Task 2.1 — CC: re-index with compressed content after compression

**File:** `backend/app/agents/workers/context_compressor.py`

CC runs two jobs per record: Job 1 embeds the original message into RAG; Job 2 compresses
messages > 2000 chars. After Job 2, the RAG entry still holds the noisy original — tool dumps,
verbose markdown, boilerplate. `tool_search_chat_history` returns degraded content.

CC indexing is idempotent: `get_indexed_chat_record_ids` is checked before embedding, so
restarts do not re-embed. Compression is idempotent: records with `compressed_message` already
set are skipped.

**Change:** After writing `compressed_message` in Job 2, delete the existing RAG embedding for
that `record_id` and re-embed with `compressed_message`. Records not compressed (< 2000 chars)
are not re-indexed — original is the final content.

**Expected result:** RAG `source_type=CHAT` entries always hold the cleanest available version.
`tool_search_chat_history` returns compressed, noise-free excerpts.

---

### Task 2.2 — Fix SQLModel enum bug for `memory_type` writes

**File:** `backend/app/db/agents_memory.py`

SQLAlchemy sends enum `.name` (e.g. `USER_PROFILE`) instead of `.value` (`user_profile`)
to the Postgres native enum column. Any write with `memory_type=AgentMemoryType.DATA_INSIGHT`
gets a Postgres constraint error. This blocks ConsolidationAgent.

**Change:** Before writing `memory_type` to the DB, cast to string value: pass `memory_type.value`
(or use `sa.cast`). Postgres receives `'data_insight'`, not `'DATA_INSIGHT'`.

---

### Task 2.3 — Wire `tool_search_chat_history` into IDA's AGENT.md

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

`tool_search_chat_history` is registered and functional but not referenced in any AGENT.md.
The tool accepts `query`, `project_id`, `chat_id` (None = project-wide), `top_k`, `start_date`,
`end_date`. Uses `RagService(source_type=CHAT)` — no new embedding required.

**Change:** Add to orchestrator planning instructions: before asking the user to confirm data
established in a prior turn, call `tool_search_chat_history` first.

---

### Task 2.4 — ConsolidationAgent: skeleton + cheatsheet promotion path

**File:** New `backend/app/agents/workers/consolidation_agent.py`

**Trigger:** Poll for chats idle ≥ 30 minutes. Minimum gate: skip chats with < 5 exchanges
(avoids triggering on brief interactions).

**Cheatsheet promotion path (PROJECT scope):**
1. Load `chat.cheatsheet` JSON for the session
2. Load last 20 chat records for context
3. LLM gate — classify each cheatsheet entry:
   - `entity_fact` → `agent_memory` PROJECT scope, `memory_type=DATA_INSIGHT`
   - `project_lesson` → `agent_memory` PROJECT scope, `memory_type=DATA_INSIGHT`
   - `discard` → stays in cheatsheet only
4. `verified` entries: promote directly, `confidence = 0.85`
5. `inferred` entries: cross-check against source record via `tool_get_record_data(record_id)`.
   Promote at `confidence = 0.85` if confirmed; write at `confidence = 0.4` and flag if unconfirmable

**Conservative default:** only auto-promote `verified` entries in Q2. `inferred` promotions
logged for manual review until soak confirms quality.

**Register in:** `backend/app/agents/factory/registry.py`

Habit promotion path deferred to Task 3.1. Overflow trigger deferred to Task 4.1.

---

### Validation

- Restart service; verify no duplicate RAG entries for the same record
- Compress a long record; query via `tool_search_chat_history` — verify compressed text returned,
  not raw tool dump
- Trigger ConsolidationAgent manually on a known chat — verify `agent_memory` rows appear with
  `scope=PROJECT`, `memory_type=data_insight`, numeric values matching source tool results

---

## May 25–Jun 6: Memory Injection Pipeline

**Goal:** Implement the three new tools that support on-demand context access. Build
MemoryService to assemble per-node prompt context. Wire into Ida behind a feature flag.
Fix cheatsheet confidence filtering at injection. Add Ev quality rubric.

---

### Task 3.1 — `tool_read_cheatsheet` (Step 1)

**File:** `backend/app/agents/tools/toolbox/mem_storage_toolbox.py`

The existing `tool_read_memory` reads from `agent_memory` (cross-session). It does not
touch `chat.cheatsheet` (per-chat). A dedicated tool is needed for Execute-node access.

**Args schema:**
```python
{
  "confidence": {
    "type": "string",
    "enum": ["verified", "inferred", "conflicted"],
    "description": "Filter entries by confidence level. Omit to return all."
  },
  "well": {
    "type": "string",
    "description": "Filter to a specific well (normalised: NNM-101 → nnm101). Omit for all wells."
  }
}
# passthrough_params: chat_id, project_id, session_id, trace_id
```

**Implementation:** `project_service.get_chat_cheatsheet(chat_id)` → `parse_cheatsheet()`
→ filter by `confidence` and `_normalize_entity_key(well)` → return markdown grouped by
bucket (`data_insights`, `key_facts`, `lessons_learned`).

**Primary use in Execute:** verify a known fact before running an expensive retrieval tool.
`tool_read_cheatsheet(confidence="verified", well="NNM-101")` — if complete, skip the
DDR/WellView query.

---

### Task 3.2 — `tool_read_chat_history` (Step 2)

**File:** `backend/app/agents/tools/toolbox/mem_storage_toolbox.py`

The framework injects recent exchanges via `load_message_history()`. This tool provides
access to exchanges outside that window — older history or history from a specific point.

**Args schema:**
```python
{
  "n": {
    "type": "integer",
    "default": 20,
    "description": "Number of exchanges (user+agent pairs) to return."
  },
  "before_record_id": {
    "type": "integer",
    "description": "Return exchanges before this record ID. Omit to start from the most recent."
  },
  "oldest_first": {
    "type": "boolean",
    "default": true
  }
}
# passthrough_params: chat_id, project_id, session_id, trace_id
```

**Implementation:** `project_service.get_chat_records(chat_id, limit=n*2,
before_record_id=..., oldest_first=...)`, pair user and agent records chronologically,
return as formatted transcript with `[User]` / `[Ida]` prefixes. Prefers
`compressed_message or message` for each record.

**Primary use in Execute:** report and reconciliation skills that need context beyond
the injected window — e.g. a report skill referencing an exchange from 20 turns back.

---

### Task 3.3 — Extend `tool_read_memory` for top-K retrieval (Step 3)

**File:** `backend/app/agents/tools/toolbox/mem_storage_toolbox.py` (existing tool)

Current behaviour: exact name lookup (`name="entity_nnm101"`). Add `query` and `top_k`
parameters for entity/keyword matching — used by MemoryService internally and by LLM-driven
lookups in Execute.

**Additional args schema fields:**
```python
{
  "query": {
    "type": "string",
    "description": "Entity or keyword search against memory name and content. Returns top_k matches. Use instead of 'name' for broad retrieval."
  },
  "top_k": {
    "type": "integer",
    "default": 5,
    "description": "Maximum entries to return when using 'query'."
  }
}
```

**Implementation (Phase 1 — keyword match):** when `query` is provided, fetch all
PROJECT-scoped entries for `project_id`, rank by: (a) well name substring in `name`,
(b) word overlap with `content_text`. Return top-K. Existing name-lookup behaviour
unchanged when `query` is absent.

**Phase 2 (Q3):** pgvector cosine similarity on `content_text` embedding — deferred until
enough PM entries exist to validate quality.

---

### Task 3.4 — `MemoryService` (Step 4)

**File:** `backend/app/services/memory/memory_service.py` (new)

The service assembles the initial prompt context for each node before the LLM call. It is
the Python-layer counterpart to the tools — tools run inside the LLM execution loop; the
service runs in the framework before each node is invoked.

**Data structures:**
```python
@dataclass
class NodeContext:
    system: str    # system prompt: soul + agent.md (+ skill.md for P and Ex)
    context: str   # injected context: CH, CS, user profile, retrieved PM
    state: str = ""  # structured output from the prior node
```

**Interface:**
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

    def load_soul(self) -> str
    def load_agent_md(self) -> str
    def load_skill(self, skill_name: str) -> str
    def load_chat_history(self, n: int = 12) -> str
    def load_cheatsheet(self, confidence: str = None) -> str
    def load_user_profile(self) -> str
    def retrieve_project_memory(self, entities: list[str], top_k: int = 5) -> str
    def bundle_for_node(self, node: str, **kwargs) -> NodeContext
```

**`bundle_for_node` injection map:**
```python
match node:
    case "U":
        system  = soul + agent_md
        context = chat_history(12) + cheatsheet() + user_profile()
                  + retrieve_project_memory(entities=kwargs["entities"])
    case "P":
        system  = soul + agent_md + skill(kwargs["skill_name"])
        context = chat_history(12) + cheatsheet()
        state   = kwargs["u_state"]   # intent, entities, known facts from U
    case "Ex":
        system  = soul + agent_md + skill(kwargs["skill_name"])
        context = ""                  # tools handle on-demand loading
        state   = kwargs["p_state"]   # execution plan from P
    case "Ev":
        system  = soul + agent_md
        context = user_profile()
        state   = kwargs["ex_state"]  # results from Ex
```

**Caching:** `soul`, `agent_md`, `skill` files read from disk once per request and cached
in `self._cache`. DB-backed loaders are not cached — must reflect current state.

**Tests:** unit tests for each loader and for `bundle_for_node` for all four nodes.
Mock `project_service` and `agent_memory_service`; assert system/context/state fields.

---

### Task 3.5 — Wire MemoryService into Ida (Step 5)

**Files:** Ida agent Python entry point, `backend/app/agents/skills/ida_agent/AGENT.md`

**Python changes:**
1. Instantiate `MemoryService` at request entry, passing `chat_id`, `project_id`, `user_id`
2. Replace manual prompt construction at each node with `bundle_for_node(node, ...)`
3. Pass `u_state` from U → P, `p_state` from P → Ex, `ex_state` from Ex → Ev
4. Register `tool_read_cheatsheet` and `tool_read_chat_history` in `register_tools()`

**AGENT.md changes:**
- Understand section: what context U receives; what structured state it must emit
  (entities, intent, known facts — entities drive PM retrieval)
- Plan section: what context P receives; that it must produce an ordered execution plan
- Execute section: tools available for on-demand access (`tool_read_cheatsheet`,
  `tool_read_chat_history`, `tool_read_memory`)
- Evaluate section: quality rubric (see Task 3.6) and what persona context is available

**Risk:** changes prompt construction for every node. Use feature flag
`IDA_MEMORY_SERVICE=true` until validated on at least two live sessions.

---

### Task 3.6 — Cheatsheet confidence filtering at injection

**File:** `backend/app/services/memory/memory_service.py`

The cheatsheet is currently injected without filtering. `conflicted` entries — two
contradictory values for the same metric — can mislead a reasoning node that picks one
value without recognising the conflict.

**Change in `load_cheatsheet()`:** inject `verified` and `inferred` entries as-is;
prefix `conflicted` entries with an explicit marker:

```
[CONFLICTED — two values reported: X, Y — do not treat as established fact]
```

`tool_read_cheatsheet` in Execute still allows unfiltered access when a skill
explicitly needs conflict data.

---

### Task 3.7 — Evaluate quality rubric in AGENT.md

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

User Profile is injected at Ev (persona-aware evaluation). The remaining gap is explicit
quality criteria — without them Ev can only check factual surface correctness.

**Add to AGENT.md Evaluate section:**
- Every numerical claim must cite a specific tool result (record ID, table, or field name)
- Confidence levels must be surfaced when data is inferred rather than verbatim
- `conflicted` cheatsheet entries must be flagged to the user, not resolved silently
- Response detail level and terminology must match the user's profile
- If the answer does not fully address the query, flag the gap rather than papering over it

---

### Task 3.8 — ConsolidationAgent: habit promotion path (EX-2 phase 2)

**File:** `backend/app/agents/workers/consolidation_agent.py`

**Habit promotion (USER scope):**
1. Query chats where `chat.habit IS NOT NULL` and `chat.habit_cursor_ts > last_habit_promoted_ts`.
   Cursor stored in GLOBAL `agent_memory`, key `habit_consolidation_cursor`
2. Feed existing USER profile (`agent_memory` USER scope) + new per-chat `chat.habit` entries to LLM
3. LLM merges: confirms patterns seen across multiple chats, softens contradictions, adds new observations
4. Write merged profile to `agent_memory` USER scope, `memory_type=USER_PROFILE`
5. Advance `habit_consolidation_cursor` in GLOBAL `agent_memory`

---

### Validation (May 25–Jun 6)

- Call each new tool from a test script; verify output format and filtering
- Instantiate `MemoryService`; call `bundle_for_node` for each node; assert fields
- Enable `IDA_MEMORY_SERVICE=true`; run two live sessions on a known well
- Inspect prompt at each node: Soul+CH+CS at U; CH+CS+Skill at P; User Profile at Ev
- Verify PM retrieval at U returns correct `entity_*` entries for queried wells
- Verify `conflicted` entries appear with `[CONFLICTED — ...]` prefix in injected context

---

## Jun 9–27: EP-5 + Soak + Tuning

**Goal:** Implement context overflow handling. Validate quality across diverse real sessions.
Tune promotion thresholds and curator prompt.

---

### Task 4.1 — EP-5: Context overflow detection + mid-session ConsolidationAgent trigger

**Files:** `backend/app/agents/skills/ida_agent/AGENT.md`,
`backend/app/agents/workers/consolidation_agent.py`

IDA has no mechanism to detect or handle a full context window. A long session hits the
token limit and fails.

**Context budget monitor (add to orchestrator, each turn):**
Estimate token usage: Soul + Agent.md + active Skill.md + CH + CS + PM snapshot.
When approaching 80% of the model's context limit, fire overflow trigger.

**Mid-session ConsolidationAgent trigger (add to consolidation_agent.py):**
ConsolidationAgent gains a second trigger mode: `overflow`. Policy differs from the
60-min end-of-session trigger:

| Trigger | Mode | Promotion policy |
|---|---|---|
| 60-min inactivity | Clean end-of-session | `verified` at 0.85 · `inferred` cross-checked then promoted or flagged |
| Context ~80% full | Forced mid-session | `verified` only at 0.85 · `inferred` at 0.4 and flagged |

**Session continuation on overflow:**
1. ConsolidationAgent runs in mid-session mode, promotes `verified` entries
2. MemoryService seeds the new chat with PM snapshot + current CS + last 12 turns
3. User is notified that the session has continued seamlessly

---

### Soak activities (Jun 9–27)

- Run minimum 6 real well sessions across ≥ 2 wells
- After each session: inspect `agent_memory` — flag false positives (wrong facts at
  `confidence ≥ 0.8`) and false negatives (important findings discarded)
- Tune ConsolidationAgent prompt: adjust `entity_fact` / `project_lesson` / `discard`
  classification; adjust confidence thresholds
- Tune curator prompt: fix numeric values still being paraphrased; correct misassigned
  confidence tiers; add domain-specific examples
- If cheatsheet overflow occurs: implement Task 4.2 — per-bucket caps (50 `data_insights`,
  20 `key_facts`) with deterministic eviction: `conflicted` → `inferred` → `verified`

---

### Task 4.2 — Cheatsheet size cap *(implement only if soak reveals overflow)*

**Files:** `backend/app/services/cheatsheet/cheatsheet_service.py`,
`backend/app/services/cheatsheet/cheatsheet_curator_prompt.py`

In normal operation, ConsolidationAgent promotes entries at session end and the curator
deduplicates. Overflow only occurs in unusually long sessions or degraded deduplication.

**Two-layer cap:**
1. **Soft hint in curator prompt:** "If a bucket is near its limit, prefer updating an
   existing same-well entry over adding a new one." LLM self-limits before eviction kicks in.
2. **Hard cap in service layer:** After parsing curator output, enforce limits
   deterministically. Eviction order within each bucket:
   1. `conflicted` entries (unresolved disputes — lowest signal)
   2. `inferred` entries, oldest first (lowest `record_id`)
   3. `verified` entries, oldest first — only if still over the limit after the above

---

## What Carries to Q3

| Item | Why deferred | When |
|---|---|---|
| EP-2 — `tool_search_memory` (semantic) | Needs enough data in `agent_memory` to validate quality; premature while ConsolidationAgent soak is ongoing | Q3 early — once ≥ 3 wells have PROJECT memory |
| `inferred` auto-promotion in ConsolidationAgent | Conservative mode through Q2; enable only after zero false-positive validation in soak | Q3, after soak confirms quality |
| EX-3 — Structured project memory | Refinement of ConsolidationAgent write structure; needs real soak data to validate what structure works | Q3 mid |
| PM semantic search (Step 3 Phase 2) | pgvector embedding at write time; needs enough PM entries | Q3 early |
| Backend evaluation (Honcho / Mem0 / Zep) | Defer until current architecture hits measurable limits; cloud backends require operator data approval | Long-term |

---

## Risk Register

| Risk | Impact | Mitigation |
|---|---|---|
| Curator JSON output is malformed | Parse error, entries lost | Validate output; fallback to plain-text append on parse failure; log all failures |
| ConsolidationAgent over-promotes wrong facts | Wrong facts persist in PROJECT memory across sessions | Conservative mode: only `verified` entries auto-promoted in Q2; `inferred` logged for review |
| Step 5 breaks prompt construction | All nodes degraded | Feature flag `IDA_MEMORY_SERVICE`; validate on non-prod chat first |
| Conflicted cheatsheet entries mislead U | Wrong facts retrieved at session start | Task 3.6: prefix `[CONFLICTED]` at injection time; Ev rubric catches silent picks |
| Soak reveals systematic curator errors | Prompt debugging consumes tuning time | Prompt iteration is fast (no Python changes); 2-day round-trip per fix |
| Not enough real sessions to fully validate ConsolidationAgent by end of June | EP-2 and `inferred` auto-promotion slip to Q3 | Expected — soak is the plan, not a risk. Q2 success = first session correctly recalled, not full pipeline validated |
| SQLModel enum bug blocks memory_type writes | ConsolidationAgent writes silently fail | Task 2.2 fix; validate before soak begins |

---

## Success Criteria at End of Q2

| Criterion | How to verify |
|---|---|
| Numeric values verbatim in cheatsheet | Inspect `chat.cheatsheet` JSON after NPT session — NPT%, ROP, depth match source tool results exactly |
| History search works | `tool_search_chat_history` returns the correct turn for a content-based query, with compressed content not raw tool dump |
| Cross-session memory works | New session on analyzed well — IDA opens with correct entity facts, no user re-prompting |
| Injection map is live | Inspect prompt at each node: Soul+CH+CS at U; CH+CS+Skill at P; User Profile at Ev |
| Conflicted entries labeled | A `conflicted` entry appears as `[CONFLICTED — ...]` in injected context, not silently resolved |
| Ev rubric active | Ev flags an answer that cites no tool result |
| No false-confidence errors | All `confidence ≥ 0.8` entries in `agent_memory` are accurate |
| Habit profile built | After 2+ sessions: `chat.habit` populated; USER-scoped `agent_memory` written by ConsolidationAgent |
