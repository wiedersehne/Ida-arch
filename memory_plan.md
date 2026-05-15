# IDA Memory Implementation Plan

**Reference:** `docs/memory-gap-analysis.md`
**Planning:** May 7, 2026

---

## Goal

Give IDA persistent memory across sessions. A drilling engineer remembers what was
established about a well last week — IDA should too. The pipeline has four stages:

1. **Extract per-exchange** — CheatsheetAgent distils each turn into structured findings
2. **Extract per-session** — HabitAgent profiles user behaviour at session end
3. **Consolidate at session close** — ConsolidationAgent promotes durable facts to `agent_memory`
4. **Inject at session start** — orchestrator loads PROJECT + USER memory before answering

---

## Dependency Chain

```
EX-4 (cheatsheet quality)
        ↓
EP-4 (ConsolidationAgent)  ←  EX-5 (schema migration)
        ↓                               ↓
EX-1 (cross-session injection)   EX-2 phase 2 (promote chat.habit → USER memory)
        ↓
EP-2 (memory search — Q3)

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

### Task 1 — Fix cheatsheet message priority

**File:** `backend/app/agents/workers/cheatsheet_agent.py`

The curator must receive the raw message, not compressed. Confidence detection
(`verified` vs `inferred`) depends on tool result block structure in the raw content,
which compression destroys. History loading (user query lookup) uses
`compressed_message or message` — compressed first to reduce noise.

**Change:** Curator input: `message or compressed_message`. History load: `compressed_message or message`.

---

### Task 2 — Scope curator to scrolled-out turns only

**File:** `backend/app/agents/workers/cheatsheet_agent.py`

The cheatsheet preserves what the tail window can no longer hold. Running the curator
on in-tail turns creates redundant entries and wastes LLM calls.

**Change:** Implement `_get_tail_start_id()` — fetch the last `TAIL_WINDOW_SIZE+1` (3)
AGENT RESPONSE records. Tail window = 2 complete exchanges. Skip records with
`id >= tail_start_id`. Bypass condition: ≤ 2 exchanges exist or last activity > 1 hour
(`tail_start_id = sys.maxsize`).

---

### Task 3 — Structured JSON storage

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

### Task 4 — Update curator prompt

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

### Task 5 — Parse and store curator output

**Files:** `backend/app/services/cheatsheet/cheatsheet_service.py`,
`backend/app/agents/workers/cheatsheet_agent.py`

**Change:** `update_cheatsheet()` receives prior cheatsheet as `[[PREVIOUS_CHEATSHEET]]`,
parses `<cheatsheet>` XML tags from LLM output, stamps `record_id` on new entries, saves
via `CheatsheetService`. Malformed output is logged and skipped — poll loop must not crash.

---

### Task 6 — agent_memory schema migration (EX-5)

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

### Task 7 — HabitAgent (EX-2 phase 1)

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

### Task 8 — CC: re-index with compressed content after compression

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

### Task 9 — Fix SQLModel enum bug for `memory_type` writes

**File:** `backend/app/db/agents_memory.py`

SQLAlchemy sends enum `.name` (e.g. `USER_PROFILE`) instead of `.value` (`user_profile`)
to the Postgres native enum column. Any write with `memory_type=AgentMemoryType.DATA_INSIGHT`
gets a Postgres constraint error. This blocks ConsolidationAgent.

**Change:** Before writing `memory_type` to the DB, cast to string value: pass `memory_type.value`
(or use `sa.cast`). Postgres receives `'data_insight'`, not `'DATA_INSIGHT'`.

---

### Task 10 — Wire `tool_search_chat_history` into IDA's AGENT.md

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

`tool_search_chat_history` is registered and functional but not referenced in any AGENT.md.
The tool accepts `query`, `project_id`, `chat_id` (None = project-wide), `top_k`, `start_date`,
`end_date`. Uses `RagService(source_type=CHAT)` — no new embedding required.

**Change:** Add to orchestrator planning instructions: before asking the user to confirm data
established in a prior turn, call `tool_search_chat_history` first.

---

### Task 11 — ConsolidationAgent: skeleton + cheatsheet promotion path

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

Habit promotion path and overflow trigger deferred to Tasks 12 and 16.

---

### Validation

- Restart service; verify no duplicate RAG entries for the same record
- Compress a long record; query via `tool_search_chat_history` — verify compressed text returned,
  not raw tool dump
- Trigger ConsolidationAgent manually on a known chat — verify `agent_memory` rows appear with
  `scope=PROJECT`, `memory_type=data_insight`, numeric values matching source tool results

---

## May 29–Jun 6: ConsolidationAgent Habit Path + EX-1 + Sub-Agent Improvements

**Goal:** Complete ConsolidationAgent with habit promotion. Wire cross-session memory into
IDA's session start. Fix sub-agent history loading quality.

---

### Task 12 — ConsolidationAgent: habit promotion path (EX-2 phase 2)

**File:** `backend/app/agents/workers/consolidation_agent.py`

**Habit promotion (USER scope):**
6. Query chats where `chat.habit IS NOT NULL` and `chat.habit_cursor_ts > last_habit_promoted_ts`.
   Cursor stored in GLOBAL `agent_memory`, key `habit_consolidation_cursor`, value = last promoted timestamp
7. Feed existing USER profile (`agent_memory` USER scope) + new per-chat `chat.habit` entries to LLM
8. LLM merges: confirms patterns seen across multiple chats, softens contradictions, adds new observations
9. Write merged profile to `agent_memory` USER scope, `memory_type=USER_PROFILE`, `content_text=merged_profile`
10. Advance `habit_consolidation_cursor` in GLOBAL `agent_memory`

---

### Task 13 — Session-start memory injection (EX-1)

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

PROJECT-scoped `agent_memory` exists and is written by ConsolidationAgent (Task 11) but
the orchestrator never reads it at session start.

**Change:** Add to planning step: when a well name is mentioned, call
`tool_read_memory(name="entity_<well>", scope=PROJECT, project_id=...)` before answering.
Read `project_index` if present for project-wide context.

---

### Task 14 — Sub-agents: use `compressed_message` + read cheatsheet

**Files:** `backend/app/agents/skills/data_insight_agent/AGENT.md`,
`sme_agent/AGENT.md`, `viz_agent/AGENT.md`

Two gaps from the gap analysis:

1. **Sub-agents ignore `compressed_message`** — all sub-agents load chat history using
   raw `message`. Only the orchestrator reads `compressed_message or message`. After
   compression runs, sub-agents still see the original noisy content.
   **Change:** Audit each sub-agent's history loading path. Prefer `compressed_message`
   over `message` wherever prior turns are loaded.

2. **Sub-agents don't read the cheatsheet** — only the orchestrator calls `tool_read_cheatsheet`.
   Sub-agents doing the actual analysis (`data_insight_agent`, `sme_agent`) have no visibility
   into accumulated findings for the session. A sub-agent analyzing NPT should know what was
   already established about this well.
   **Change:** Add `tool_read_cheatsheet(chat_id=...)` to each sub-agent's AGENT.md planning
   section, called before the first tool call.

---

### Task 15 — Spawned IDA instances load session context

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

When IDA spawns itself for a sub-task, the spawned instance starts with no knowledge of
the parent session's accumulated findings.

**Change:** At spawn time, the spawned instance must load:
1. PROJECT memory snapshot — `tool_read_memory(scope=PROJECT, project_id=...)`
2. Current session's cheatsheet — `tool_read_cheatsheet(chat_id=<parent_chat_id>)`

Spawned instances write findings back to the parent `chat_id` — CheatsheetAgent picks
them up on its next poll.

---

### Validation

- Run 2–3 real sessions on a known well
- After 30-min idle: inspect `agent_memory` — verify PROJECT entries with correct numeric values
- New session on same well: verify orchestrator opens with prior entity memory, no user re-prompting
- Check 3 promoted `data_insight` entries for accuracy against source tool results

---

## Jun 9–27: EP-5 + Soak + Tuning

**Goal:** Implement context overflow handling. Validate quality across diverse real sessions.
Tune promotion thresholds and curator prompt.

---

### Task 16 — EP-5: Context overflow detection + mid-session ConsolidationAgent trigger

**Files:** `backend/app/agents/skills/ida_agent/AGENT.md`,
`backend/app/agents/workers/consolidation_agent.py`

IDA has no mechanism to detect or handle a full context window. A long session hits the
token limit and fails.

**Context budget monitor (add to orchestrator, each turn):**
Estimate token usage: SOUL.md + AGENT.md + active SKILL.md + PROJECT snapshot + cheatsheet
rendered + tail window. When approaching 80% of the model's context limit, fire overflow trigger.

**Mid-session ConsolidationAgent trigger (add to consolidation_agent.py):**
ConsolidationAgent gains a second trigger mode: `overflow`. Policy differs from the
30-min end-of-session trigger:

| Trigger | Mode | Promotion policy |
|---|---|---|
| 30-min inactivity | Clean end-of-session | `verified` at 0.85 · `inferred` cross-checked then promoted or flagged |
| Context ~80% full | Forced mid-session | `verified` only at 0.85 · `inferred` at 0.4 and flagged |

**Session continuation on overflow:**
1. ConsolidationAgent runs in mid-session mode, promotes `verified` entries
2. IDA opens a new chat, seeds it with: PROJECT memory snapshot + current cheatsheet + last 12 turns
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
- If cheatsheet overflow occurs: implement Task 1.6 — per-bucket caps (50 `data_insights`,
  20 `key_facts`) with deterministic eviction: `conflicted` → `inferred` → `verified`

---

### Task 1.6 — Cheatsheet size cap *(implement only if soak reveals overflow)*

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
| EP-2 — `tool_search_memory` | Needs enough data in `agent_memory` to validate quality; premature while ConsolidationAgent soak is ongoing | Q3 early — once ≥ 3 wells have PROJECT memory |
| `inferred` auto-promotion in ConsolidationAgent | Conservative mode through Q2; enable only after zero false-positive validation in soak | Q3, after soak confirms quality |
| EX-3 — Structured project memory | Refinement of ConsolidationAgent write structure (two-tier: project index + per-well entity records); needs real soak data to validate what structure works | Q3 mid |
| Backend evaluation (Honcho / Mem0 / QMD) | Defer until current architecture hits measurable limits; cloud backends require operator data approval | Long-term |

---

## Risk Register

| Risk | Impact | Mitigation |
|---|---|---|
| Curator JSON output is malformed | Parse error, entries lost | Validate output; fallback to plain-text append on parse failure; log all failures |
| ConsolidationAgent over-promotes wrong facts | Wrong facts persist in PROJECT memory across sessions | Conservative mode: only `verified` entries auto-promoted in Q2; `inferred` logged for review |
| Soak reveals systematic curator errors | Prompt debugging consumes tuning time | Prompt iteration is fast (no Python changes); 2-day round-trip per fix |
| Not enough real sessions to fully validate ConsolidationAgent by end of June | EP-2 and `inferred` auto-promotion slip to Q3 | Expected — soak is the plan, not a risk. Q2 success = first session correctly recalled, not full pipeline validated |
| Sub-agent cheatsheet reads increase latency | Each sub-agent adds one tool call at session start | Cheatsheet is small and cached in chat table — negligible in practice |

---

## Success Criteria at End of Q2

| Criterion | How to verify |
|---|---|
| Numeric values verbatim in cheatsheet | Inspect `chat.cheatsheet` JSON after NPT session — NPT%, ROP, depth match source tool results exactly |
| History search works | `tool_search_chat_history` returns the correct turn for a content-based query, with compressed content not raw tool dump |
| Cross-session memory works | New session on analyzed well — IDA opens with correct entity facts, no user re-prompting |
| No false-confidence errors | All `confidence ≥ 0.8` entries in `agent_memory` are accurate |
| Habit profile built | After 2+ sessions: `chat.habit` populated for each chat; USER-scoped `agent_memory` written by ConsolidationAgent |
