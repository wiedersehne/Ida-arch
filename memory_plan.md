# IDA Memory Implementation Plan

**Reference:** `docs/memory-architecture.md`, `docs/memory-gap-analysis.md`
**Planning:** May 7, 2026 · Updated May 22, 2026

---

## Goal

Give IDA persistent memory across sessions. A drilling engineer remembers what was
established about a well last week — IDA should too. The pipeline has five stages:

1. **Extract per-exchange** — CheatsheetAgent distils each turn into structured findings
2. **Extract per-session** — HabitAgent profiles user behaviour at session end
3. **Consolidate at session close** — ConsolidationAgent promotes durable facts to `agent_memory`
4. **Inject at session start** — MemoryService loads the right memories into each reasoning node
5. **On-demand retrieval** — tools let the Execute node pull additional context mid-run

The agent architecture is a **single Ida agent** running a five-node loop (Understand → Plan → Execute → Evaluate → Answer). There are no sub-agents.

---

## Dependency Chain

```
Steps 1–3 (new tools) — parallel, independent
        ↓
Step 4 (MemoryService) — depends on Steps 1–3 for internal loaders
        ↓
Step 5 (wire into Ida) — depends on Step 4; highest risk

Concern #3 fix (cheatsheet confidence filtering) — independent, no prerequisites
Ev quality rubric — independent, AGENT.md only
Task 2.1 (CC re-index) — independent, carried from prior plan
```

Steps 1–3 are the critical path. MemoryService quality depends on the tools it composes.
Step 5 changes prompt construction for every node — feature flag required.

---

## Completed Through May 22

### Cheatsheet pipeline
- **Structured JSON storage** (`cheatsheet_schema.py`): three-bucket schema —
  `data_insights`, `key_facts`, `lessons_learned` — each entry with `confidence`,
  `record_id`, optional `well`
- **Curator prompt** updated: `verified` / `inferred` / `conflicted` tiers; numeric
  precision rule; conflict detection; domain knowledge verbatim capture
- **CheatsheetAgent 3-phase tail logic**: Phase 1 (young, curate immediately), Phase 2
  (accumulation, defer until `tail_n` unprocessed), Phase 3 (sliding window, curate
  oldest). 49 tests passing.
- **Message priority fix**: curator receives raw `message`; history loading uses
  `compressed_message or message`
- **Curator output parsing**: XML tag extraction, `record_id` stamping, malformed output
  logged and skipped

### Background agents
- **ConsolidationAgent** implemented: promotes `verified` data_insights (per-well),
  `verified` key_facts (project_facts), `verified + inferred` lessons to PROJECT-scoped
  `agent_memory`. LLM synthesis with `_dedupe` fallback. Cursor: `consolidation_cursor_ts`.
- **HabitAgent** implemented: extracts user behaviour from idle-session transcripts into
  per-chat `chat.habit`. Polls every 300s; fires at 60-min idle.
- Both agents registered in `registry.py`

### Schema
- `agent_memory`: `memory_type`, `user_id`, `content_text`, `confidence`, `size_chars`,
  `version` columns added; GIN FTS index on `content_text`; `AgentMemoryType` enum
- `chat`: `cheatsheet_cursor_ts`, `consolidation_cursor_ts`, `habit`, `habit_cursor_ts`
  columns added

### CC fix (Task 2.1 — partial)
- Context compressor updated; CC re-index after compression may still need validation

---

## May 25–30: Steps 1–4 (New Tools + MemoryService)

**Goal:** All three new tools implemented and tested. MemoryService implemented with
full `bundle_for_node` mapping. No wiring into Ida yet — that is Step 5.

---

### Task 3.1 — `tool_read_cheatsheet` (Step 1)

**File:** `backend/app/agents/tools/toolbox/mem_storage_toolbox.py`

The existing `tool_read_memory` reads from `agent_memory` (cross-session). It does not
touch `chat.cheatsheet` (per-chat). A dedicated tool is needed.

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
bucket. Return `"(no cheatsheet entries)"` on empty.

**Primary use in Execute:** verify a known fact before running an expensive retrieval
tool. `tool_read_cheatsheet(confidence="verified", well="NNM-101")` → if complete, skip
the DDR/WellView query.

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
parameters for entity/keyword matching — this is the retrieval path for `MemoryService`
and for LLM-driven lookups in Execute.

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
is unchanged when `query` is absent.

**Phase 2 (later):** pgvector cosine similarity on `content_text` embedding. Out of
scope for this week — Phase 1 is sufficient at current project scale.

---

### Task 3.4 — `MemoryService` (Step 4)

**File:** `backend/app/services/memory/memory_service.py` (new)

The service assembles the initial prompt context for each node before each LLM call.
Tools handle on-demand access during execution; the service handles static injection
and top-K retrieval at node entry.

**Data structures:**
```python
@dataclass
class NodeContext:
    system: str    # system prompt: soul + agent.md (+ skill.md for P and Ex)
    context: str   # injected context: CH, CS, user profile, retrieved PM
    state: str = ""  # structured output from prior node
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

**Caching:** `soul`, `agent_md`, `skill` files read from disk once per request and
cached in `self._cache`. DB-backed loaders are not cached — must reflect current state.

**Tests:** unit tests for each loader and for `bundle_for_node` for all four nodes.
Mock `project_service` and `agent_memory_service`; assert system/context/state fields.

---

### Validation (May 25–30)

- Call each new tool from a test script; verify output format and filtering
- Instantiate `MemoryService` in isolation; call `bundle_for_node` for each node;
  assert `NodeContext` fields are non-empty and contain the expected sections
- Verify `tool_read_memory(query="NNM-101")` returns the correct `entity_nnm101` entry

---

## Jun 1–6: Step 5 + Confidence Fix + Ev Rubric

**Goal:** Wire MemoryService into Ida. Fix cheatsheet confidence filtering at injection.
Add Ev quality rubric to AGENT.md. Address CC re-index gap if not yet validated.

---

### Task 4.1 — Wire MemoryService into Ida (Step 5)

**Files:** Ida agent Python entry point, `backend/app/agents/skills/ida_agent/AGENT.md`

**Python changes:**
1. Instantiate `MemoryService` at request entry, passing `chat_id`, `project_id`, `user_id`
2. Replace manual prompt construction at each node with `bundle_for_node(node, ...)`
3. Pass `u_state` from U → P, `p_state` from P → Ex, `ex_state` from Ex → Ev

**AGENT.md changes:**
- Add Understand section: what context U receives and what structured state it must emit
  (entities, intent, known facts — these drive PM retrieval at the next call)
- Add Plan section: what context P receives; that it must produce an ordered execution plan
- Add Execute section: tools available for on-demand context (`tool_read_cheatsheet`,
  `tool_read_chat_history`, `tool_read_memory`)
- Add Evaluate section: quality rubric (see Task 4.3) and what persona context is available
- Register `tool_read_cheatsheet` and `tool_read_chat_history` in `register_tools()`

**Risk:** this changes prompt construction for every node. Use a feature flag
(`IDA_MEMORY_SERVICE=true`) until validated on at least two live sessions.

---

### Task 4.2 — Cheatsheet confidence filtering at injection (Concern #3)

**File:** `backend/app/services/memory/memory_service.py`

The cheatsheet is currently injected without filtering. `conflicted` entries — two
contradictory values for the same metric — can mislead a reasoning node that picks one
value without recognising the conflict.

**Change in `load_cheatsheet()`:** inject `verified` and `inferred` entries as-is;
prefix `conflicted` entries with an explicit marker:

```
[CONFLICTED — two values reported: X, Y — do not treat as established fact]
```

This preserves the information while preventing silent misuse. `tool_read_cheatsheet`
in Execute still allows unfiltered access when a skill explicitly needs conflict data.

---

### Task 4.3 — Evaluate quality rubric in AGENT.md

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

The Evaluate node has User Profile injected (persona-aware evaluation). The missing piece
is explicit quality criteria — without them, Ev can only check factual surface correctness.

**Add to AGENT.md Evaluate section:**
- Every numerical claim must cite a specific tool result (record ID, table, or field name)
- Confidence levels must be surfaced when data is inferred rather than verbatim
- `conflicted` cheatsheet entries must be flagged to the user, not resolved silently
- Response detail level and terminology must match the user's profile (injected via MemoryService)
- If the answer does not fully address the query, flag the gap rather than papering over it

---

### Task 4.4 — CC re-index validation (carried from Task 2.1)

**File:** `backend/app/agents/workers/context_compressor.py`

Verify that after Job 2 (compression), the RAG entry for the compressed record is
deleted and re-embedded with `compressed_message`. Run `tool_search_chat_history` on a
known compressed record; confirm the returned content is the compressed version, not the
raw tool dump.

If the re-index is not yet wired, add: after writing `compressed_message`, call
`rag_service.delete_by_record_id(record_id)` then re-embed `compressed_message`.

---

### Validation (Jun 1–6)

- Enable `IDA_MEMORY_SERVICE=true`; run two live sessions on a known well
- Inspect LLM prompt at each node: assert Soul, CH, CS present in U; CS present in P;
  user profile present in Ev
- Verify U's state output contains `entities` that match well names mentioned in the query
- Verify PM retrieval at U returns the correct `entity_*` entries for those wells
- After Ev: confirm conflicted cheatsheet entries are marked, not silently picked

---

## Jun 8–13: SQLModel Enum Fix + Soak Prep

**Goal:** Fix the SQLModel enum bug that blocks `memory_type` writes. Set up soak
instrumentation. Begin real session validation.

---

### Task 5.1 — Fix SQLModel enum bug for `memory_type` writes

**File:** `backend/app/db/agents_memory.py`

SQLAlchemy sends enum `.name` (`DATA_INSIGHT`) instead of `.value` (`data_insight`)
to the Postgres native enum column. Any write with `memory_type=AgentMemoryType.DATA_INSIGHT`
gets a Postgres constraint error, blocking ConsolidationAgent writes.

**Change:** cast to string value before writing: pass `memory_type.value` (or `sa.cast`).
Postgres receives `'data_insight'`, not `'DATA_INSIGHT'`.

---

### Task 5.2 — Soak instrumentation

Add structured logging to ConsolidationAgent and MemoryService:
- Log entity keys, entry counts, and confidence breakdown at each consolidation run
- Log which PM entries were retrieved at U, and whether they matched the queried well
- Log Ev quality verdict (pass / flag / gap) per session

These logs are the primary signal for tuning during the soak period.

---

## Jun 15–27: Context Overflow + Soak

**Goal:** Implement context overflow detection. Run ≥ 6 real well sessions. Tune
promotion thresholds and curator prompt based on soak findings.

---

### Task 6.1 — Context overflow detection + mid-session ConsolidationAgent trigger

**Files:** Ida agent Python entry point,
`backend/app/agents/workers/consolidation_agent.py`

IDA has no mechanism to detect or handle a full context window. A long session hits the
token limit and fails.

**Context budget monitor (add to Ida agent, each turn):**
Estimate token usage: Soul + Agent.md + active Skill.md + CH + CS + PM snapshot.
When approaching 80% of the model's context limit, fire the overflow trigger.

**Mid-session ConsolidationAgent trigger:**
ConsolidationAgent gains a second trigger mode. Policy differs from the 60-min
end-of-session trigger:

| Trigger | Mode | Promotion policy |
|---|---|---|
| 60-min inactivity | Clean end-of-session | `verified` at 0.85; `inferred` cross-checked |
| Context ~80% full | Forced mid-session | `verified` only at 0.85; `inferred` at 0.4, flagged |

**Session continuation on overflow:**
1. ConsolidationAgent runs in mid-session mode, promotes `verified` entries
2. MemoryService seeds the new chat with PM snapshot + current CS + last 12 turns
3. User is notified that the session has continued seamlessly

---

### Soak activities (Jun 15–27)

- Run minimum 6 real well sessions across ≥ 2 wells
- After each session: inspect `agent_memory` — flag false positives (wrong facts at
  `confidence ≥ 0.8`) and false negatives (important findings discarded)
- Tune ConsolidationAgent synthesis prompt: adjust dedup quality; fix numerical conflicts
  not being caught; add domain-specific examples
- Tune curator prompt: fix numeric values still paraphrased; correct misassigned
  confidence tiers
- If cheatsheet grows large: implement per-bucket caps (50 `data_insights`,
  20 `key_facts`) with deterministic eviction: `conflicted` → `inferred` → `verified`

---

## What Carries to Q3

| Item | Why deferred | When |
|---|---|---|
| PM semantic search (Step 3 Phase 2) | pgvector embedding at write time; needs enough PM entries to validate quality | Q3 early — once ≥ 3 wells have PROJECT memory |
| `inferred` auto-promotion | Conservative through Q2; enable only after zero-false-positive soak validation | Q3, after soak |
| Habit promotion across sessions | HabitAgent writes per-chat `chat.habit`; cross-session USER profile merge via ConsolidationAgent deferred until cheatsheet quality soak is done | Q3 mid |
| Backend evaluation (Honcho / Mem0 / Zep) | Defer until current architecture hits measurable limits; cloud backends require operator data approval | Long-term |

---

## Risk Register

| Risk | Impact | Mitigation |
|---|---|---|
| Step 5 breaks prompt construction | All nodes degraded | Feature flag `IDA_MEMORY_SERVICE`; validate on non-prod chat first |
| MemoryService adds latency at node entry | Slower first response | All loaders are fast DB reads; CH+CS already loaded in current code |
| Conflicted cheatsheet entries mislead U | Wrong facts retrieved at session start | Task 4.2: prefix `[CONFLICTED]` at injection time; Ev rubric catches silent picks |
| ConsolidationAgent over-promotes | Wrong facts persist in PROJECT memory | Conservative mode: only `verified` auto-promoted; `inferred` logged for review |
| Not enough real sessions by end of June | EP-2 and `inferred` promotion slip to Q3 | Expected — soak is the plan, not a risk |
| SQLModel enum bug blocks memory_type writes | ConsolidationAgent writes silently fail | Task 5.1: fix before soak begins |

---

## Success Criteria at End of Q2

| Criterion | How to verify |
|---|---|
| Structured cheatsheet works | Inspect `chat.cheatsheet` JSON after NPT session — NPT%, ROP, depth appear verbatim in `data_insights[confidence=verified]` |
| Cross-session memory works | New session on analyzed well — Ida opens with correct entity facts, no user re-prompting |
| Injection map is live | Inspect prompt at each node: Soul+CH+CS+UP at U; Soul+CH+CS+Skill at P; UP at Ev |
| Conflicted entries labeled | A `conflicted` cheatsheet entry appears as `[CONFLICTED — ...]` in injected context, not silently picked |
| Ev rubric active | Ev flags an answer that cites no tool result; flag appears in response |
| PM retrieval works | `tool_read_memory(query="NNM-101")` returns the correct `entity_nnm101` entry |
| No false-confidence errors | All `confidence ≥ 0.8` entries in `agent_memory` are accurate against source tool results |
