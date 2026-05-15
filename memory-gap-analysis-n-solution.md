# IDA Memory Implementation Plan

**Reference:** `docs/memory-gap-analysis.md`
**Planning:** May 7, 2026 (Thu) — implementation starts May 8

---

## Status as of May 11, 2026

| Item | Status | Notes |
|---|---|---|
| EX-4 — Cheatsheet foundation | ✅ Done | Tasks 1.1–1.5 complete |
| EX-5 — agent_memory schema | ✅ Done | Migration `9f8e7d6c5b4a` applied; USER scope, memory_type (4 values), content_text, confidence, size_chars, version |
| HabitAgent (EX-2 phase 1) | ✅ Done | Ahead of schedule. Writes to `chat.habit`. Migration `9f8e7d6c5b4a` applied. Architecture differs from original plan — see updated EX-2 section. |
| Task 1.6 — Cheatsheet size cap | Deferred | Optional; implement if soak reveals overflow |
| CC bugs (Tasks 2.1–2.2) | ⬜ Not started | |
| EP-1 — chat history search (Tasks 2.3–2.4) | ⬜ Not started | |
| EP-4 — ConsolidationAgent | ⬜ Not started | Now also responsible for promoting `chat.habit` → USER agent_memory |
| EX-1 — Session-start injection | ⬜ Not started | |
| EP-2 — Memory search | ⬜ Not started | |

---

## Dependency Chain

```
EX-4 (cheatsheet quality) ✅
        ↓
EP-4 (ConsolidationAgent)  ←  EX-5 (schema migration) ✅
        ↓                               ↓
EX-1 (cross-session injection)   EX-2 phase 2 (ConsolidationAgent promotes chat.habit → USER memory)
        ↓
EP-2 (memory search)

EP-1 (chat history search) — independent, no prerequisites

HabitAgent ✅ → chat.habit → ConsolidationAgent (EP-4) → agent_memory USER scope
```

EX-4 is the critical path. Nothing downstream is reliable until cheatsheet extraction
quality is validated on real well sessions.

EX-2 HabitAgent (phase 1) is complete ahead of schedule. Phase 2 — promoting `chat.habit`
to USER-scoped `agent_memory` — is ConsolidationAgent's responsibility (EP-4).

---

## ✅ May 8–15: EX-4 — Cheatsheet Foundation

**Goal:** Reliable, structured cheatsheet extraction. Every downstream component depends on this.

### ✅ Task 1.1 — Fix message priority

**File:** `backend/app/agents/workers/cheatsheet_agent.py:204`

**Implemented:** Curator receives raw `message or compressed_message` — raw first so the curator can see tool result blocks and correctly assign `verified` vs `inferred` confidence. History loading (user query lookup) uses `compressed_message or message`. **Note:** direction reversed from original plan — raw is correct for the curator because confidence detection depends on tool result block structure, which compression destroys.

---

### ✅ Task 1.2 — Scope curator to scrolled-out turns

**File:** `backend/app/agents/workers/cheatsheet_agent.py`

**Implemented:** `_get_tail_start_id()` fetches the last `TAIL_WINDOW_SIZE+1` (3) AGENT RESPONSE records. Tail window = 2 complete exchanges. If ≤2 exchanges exist or last activity > 1 hour, `tail_start_id = sys.maxsize` (bypass). Records with `id >= tail_start_id` are skipped.

---

### ✅ Task 1.3 — Structured JSON storage

**Files:** `backend/app/services/cheatsheet/cheatsheet_service.py`, `backend/app/services/cheatsheet/cheatsheet_schema.py`

**Implemented:** `chat.cheatsheet` stores JSON with two buckets: `data_insights` (entity-specific, requires `well` field) and `key_facts`. **`user_habits` bucket removed** — user habits are now HabitAgent's domain, extracted per session from the full transcript, not per exchange. `render_to_markdown()` produces injection-ready markdown. Backward-compatible: legacy plain-text cheatsheets returned as-is until replaced on next write.

---

### ✅ Task 1.4 — Update curator prompt

**File:** `backend/app/services/cheatsheet/cheatsheet_curator_prompt.py`

**Implemented:** Two-bucket output (`data_insights` / `key_facts`). Confidence tiers (`verified` / `inferred` / `conflicted`). Numeric precision rules (verbatim copy of tool result values). Conflict detection (prior entry updated to `conflicted`, new `conflicted` entry added with `prior_record_id`). Domain knowledge rule: capture non-universal terms verbatim; skip only universal unambiguous concepts (e.g. NPT, ROP, BHA).

---

### ✅ Task 1.5 — Parse and store curator output

**Files:** `backend/app/services/cheatsheet/cheatsheet_service.py`, `backend/app/agents/workers/cheatsheet_agent.py`

**Implemented:** `update_cheatsheet()` passes raw JSON as `[[PREVIOUS_CHEATSHEET]]`, parses `<cheatsheet>` output, stamps `record_id` on new entries, saves via `CheatsheetService`. Malformed output is logged and skipped without crashing the poll loop.

---

### ⬜ Task 1.6 — Cheatsheet size cap *(optional — implement if soak shows overflow)*

**Files:** `backend/app/services/cheatsheet/cheatsheet_service.py`, `backend/app/services/cheatsheet/cheatsheet_curator_prompt.py`

**Context:** In normal operation the cheatsheet should stay well under the limits — the ConsolidationAgent promotes entries to `agent_memory` at session end, and the curator deduplicates. Overflow only occurs in unusually long sessions or if deduplication degrades. A reflector LLM pass would be principled but is expensive on every poll and premature before the soak reveals actual overflow patterns. Start with deterministic eviction.

**Limits:**

| Bucket | Max entries |
|---|---|
| `data_insights` | 50 |
| `key_facts` | 20 |

**Change — two layers:**

1. **Soft hint in curator prompt:** Add rule: "If a bucket is near its limit, prefer updating an existing same-well entry over adding a new one." The LLM self-limits before eviction kicks in.

2. **Hard cap in service layer:** In `update_cheatsheet()`, after parsing the curator output into `CheatsheetData`, enforce limits deterministically — no LLM call. Eviction order within each bucket:
   1. `conflicted` entries (unresolved disputes — lowest signal)
   2. `inferred` entries, oldest first (lowest `record_id`)
   3. `verified` entries, oldest first — only if still over the limit after the above

   Never drop a `verified` entry unless all `conflicted` and `inferred` entries are gone first.

**Deferred:** A semantic reflector (second LLM pass to rank entries by domain importance) is deferred to Q3 if the soak reveals that mechanical eviction is dropping the wrong entries.

**Expected result:** Bucket sizes are bounded. The LLM self-limits in most cases; the service enforces the hard cap for edge cases without an extra LLM call.

---

### Validation (ongoing from May 8, not a one-time gate)

- Run real well sessions with NPT analysis; inspect raw `chat.cheatsheet` JSON after each.
- Verify: NPT%, ROP, depth appear verbatim in `data_insights` entries.
- Verify: long-message turns use `compressed_message` as curator input (log check).
- Verify: turns still inside the tail window have no corresponding cheatsheet entries.
- Verify: `verified` entries each have a `record_id` matching a tool-result turn.

---

## ⬜ May 18–22 (Mon–Fri): Wiring + Enum Fix + ConsolidationAgent Start

**Goal:** Wire completed tools into agent instructions. Fix the SQLModel enum bug that blocks typed memory writes. Begin ConsolidationAgent.

**Note — Tasks 2.1–2.3 completed in `feature/memory-arch`:**
- **Task 2.1 (CC dedup guard):** Implemented as idempotent design — `get_indexed_chat_record_ids` check runs before every embed. No cursor needed.
- **Task 2.2 (Separate CC cursors):** Superseded — CC is now fully cursor-free. Both bugs (restart re-embed, failed-compression retry) are covered by idempotency: indexing skips already-indexed IDs; compression skips records with `compressed_message` already set.
- **Task 2.3 (`tool_search_chat_history`):** Registered in `chat_record_data_toolbox.py`. `chat_id=None` gives project-wide search.

---

### Task 2.4 — Wire `tool_search_chat_history` into IDA's AGENT.md

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

**Change:** Add instruction: before asking the user to confirm data established in a prior turn, call `tool_search_chat_history` first.

**Expected result:** Agent stops re-asking for data it saw in a prior session.

---

### Task 2.5 — Fix SQLModel enum bug for `memory_type` writes

**File:** `backend/app/db/agents_memory.py`

**Problem:** SQLAlchemy sends enum `.name` (e.g. `USER_PROFILE`) instead of `.value` (`user_profile`) to the Postgres native enum column. Any agent that sets `memory_type=AgentMemoryType.DATA_INSIGHT` will get a Postgres constraint error. This blocks ConsolidationAgent writing typed memories.

**Fix:** Cast enum to its string value before writing — pass `memory_type.value` (or use `sa.cast`) so Postgres receives `'data_insight'` not `'DATA_INSIGHT'`.

**Expected result:** ConsolidationAgent can write `DATA_INSIGHT` and `USER_PROFILE` memories without error.

---

### Task 2.6 — ConsolidationAgent skeleton

**File:** New `backend/app/agents/workers/consolidation_agent.py`

**Goal:** Get the agent scaffolded, registered, trigger wired, and the cheatsheet-promotion path functional (habit promotion can follow in May 29 week). Mirrors HabitAgent's polling pattern.

**This week scope:**
1. Agent class + `capabilities()` + registration in `registry.py`
2. 30-min inactivity trigger: query chats where `cheatsheet_cursor_ts` is older than 30 min ago
3. Cheatsheet promotion path (steps 1–5 from Task 4.1) — LLM gate + write `DATA_INSIGHT` memories
4. Minimum-gate: skip chats with < 5 exchanges

**Deferred to May 29:** Habit promotion path (steps 6–10), overflow/mid-session trigger, spawned-IDA logic.

---

### Validation (end of May 22)

- Restart service; embed a message; restart again — verify no duplicate RAG entries.
- In a session with prior NPT analysis: ask about the same well — verify agent retrieves prior turn without re-prompting.
- Trigger ConsolidationAgent manually on a known chat — verify `agent_memory` rows appear with `scope=PROJECT`, `memory_type=data_insight`.

---

## ✅ EX-5 — Schema Migration *(completed ahead of schedule)*

**Goal:** Extend `agent_memory` to support typed, versioned, FTS-searchable, user-scoped memories.

### ✅ Task 3.1 — Schema migration

**Migration:** `9f8e7d6c5b4a_memory_arch_consolidation.py` (consolidated — covers EX-5, EX-2 phase 1, and all chat column additions).

**Delivered:**
- `AgentMemoryScope.USER` added
- `AgentMemoryType` enum: `user_profile`, `data_insight`, `bookkeeping`, `knowledge`
  - `USER_PROFILE` — USER scope, habits and preferences
  - `DATA_INSIGHT` — PROJECT scope, per-well/file findings
  - `BOOKKEEPING` — GLOBAL, background agent cursors
  - `KNOWLEDGE` — ORG/GLOBAL, domain facts
- New nullable columns on `agent_memory`: `memory_type`, `user_id`, `content_text`, `confidence`, `size_chars`, `version`
- Existing records backfilled with `memory_type='bookkeeping'`

### ✅ Task 3.2 — Auto-populate `content_text` and `version` on write

**File:** `backend/app/db/agents_memory.py`

`set_object()` auto-populates `content_text` from the `object` JSONB, sets `size_chars`, increments `version` on upsert.

**Note — known SQLModel enum bug:** SQLAlchemy sends enum `.name` (e.g. `USER_PROFILE`) instead of `.value` (`user_profile`) to the Postgres native enum column when using `memory_type=AgentMemoryType.USER_PROFILE`. Workaround: omit `memory_type` on writes until the column serialization is fixed. All bookkeeping writes (HabitAgent cursors, CheatsheetAgent cursor) are unaffected — they don't set `memory_type`.

---

## ⬜ May 29 – Jun 6 (Fri + Mon–Fri): EP-4 completion + EX-1 — ConsolidationAgent habit path + Session-Start Injection

**Goal:** Complete ConsolidationAgent habit promotion. Wire cross-session memory into IDA's session start. ConsolidationAgent skeleton + cheatsheet path delivered in May 18–22 week.

### Task 4.1 — ConsolidationAgent (habit promotion path + overflow trigger)

**File:** `backend/app/agents/workers/consolidation_agent.py` (skeleton from May 18–22)

**This week scope — complete what was deferred:**

*Habit promotion (USER scope — EX-2 phase 2):*
6. Query chats where `chat.habit IS NOT NULL` and `chat.habit_cursor_ts > last_habit_promoted_ts` (cursor stored in GLOBAL `agent_memory`, key `habit_consolidation_cursor`, value = last promoted timestamp).
7. Feed existing USER profile (`agent_memory` USER scope) + new per-chat habit profiles to LLM.
8. LLM merges: confirms patterns seen across multiple chats, softens contradictions, adds new observations.
9. Write result to `agent_memory` USER scope, `memory_type=USER_PROFILE`, `content_text=merged_profile`.
10. Advance `habit_consolidation_cursor` in GLOBAL `agent_memory`.

*Overflow / mid-session trigger (if time permits):*

**Two trigger conditions — same agent, different behavior:**

| Trigger | Mode | Promotion policy |
|---|---|---|
| 30-min inactivity / session close | Clean end-of-session | `verified` promotes at 0.85 · `inferred` cross-checked then promoted or flagged |
| Context window ~80% full | Forced mid-session | `verified` only at 0.85 · `inferred` written at 0.4 and flagged — session was cut short |

**Context budget monitor** (lightweight check inside IDA each turn):
Estimate token usage: SOUL.md + AGENT.md + active SKILL.md + PROJECT snapshot
+ cheatsheet rendered + tail window. When approaching 80% of the model's context limit,
fire the overflow trigger.

**On overflow — IDA spawns itself:**
1. ConsolidationAgent runs in mid-session mode, promotes `verified` cheatsheet entries.
2. IDA spawns a new instance of itself to continue the conversation.
3. Spawned instance is seeded with: PROJECT memory snapshot (now including newly promoted
   entries) + current cheatsheet + last 12 turns as the opening tail.
4. User is notified that the session has continued seamlessly.

**Conservative default for initial soak:** only auto-promote `verified` entries. All `inferred` promotions logged for manual review until quality is validated.

---

### Task 4.2 — Session-start injection in IDA's AGENT.md

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

**Change:** Add to planning step: when a well name is mentioned, call `tool_read_memory(name="entity_<well>", scope=PROJECT, project_id=...)` before answering. Load `project_summary` if present.

---

### Task 4.3 — Spawned IDA instances load session context

**File:** `backend/app/agents/skills/ida_agent/AGENT.md`

**Change:** When IDA spawns itself for a sub-task, the spawned instance must load at
spawn time:
1. PROJECT memory snapshot — `tool_read_memory(scope=PROJECT, project_id=...)`
2. Current session's cheatsheet — `tool_read_cheatsheet(chat_id=<parent_chat_id>)`

Spawned instances read from the parent session's `chat_id` so they share the same
accumulated knowledge. Their findings (tool results, conclusions) are written back to
the parent chat record and picked up by the CheatsheetAgent on its next poll.

### Validation (end of Jun 6)

- Run 2–3 real sessions on a known well.
- After 30-min idle: inspect `agent_memory` — verify PROJECT entries exist with correct numeric values.
- New session on same well: verify orchestrator opens with prior entity memory injected, no user re-prompting.
- Check 3 promoted `data_insight` entries for accuracy against source tool results.

---

## Jun 9–27: Soak + Tuning

**Goal:** Validate quality across diverse sessions. Tune promotion thresholds. By this point coding is mostly done — this period is about quality validation and prompt iteration.

### What happens here

- Run minimum 6 real well sessions across ≥ 2 wells.
- After each session: inspect `agent_memory` — flag false positives (wrong facts at `confidence ≥ 0.8`) and false negatives (important findings discarded).
- Tune ConsolidationAgent prompt: tighten or loosen `entity_fact` classification, adjust confidence thresholds.
- Tune curator prompt: fix numeric values still being paraphrased, correct misassigned confidence tiers, add domain-specific examples.
- Monitor cheatsheet sizes; if overflow occurs, implement Task 1.6 (per-bucket caps: 50 / 20, deterministic eviction: `conflicted` → `inferred` → `verified`).

**Target by end of June:**
- Zero false positives with `confidence ≥ 0.8`
- Numeric accuracy 100% for `verified` entries
- New session on analyzed well opens with correct prior knowledge without user prompting

---

## ✅ EX-2 — HabitAgent Phase 1 (Session-End Habit Extraction) *(completed ahead of schedule)*

**Files:**
- `backend/app/agents/workers/habit_agent.py`
- `backend/app/services/habit/habit_agent_prompt.py`
- `backend/app/services/habit/__init__.py`
- Migration `9f8e7d6c5b4a_memory_arch_consolidation.py` — adds `chat.habit`, `chat.habit_cursor_ts` columns

**Architecture (differs from original plan):**
Two-stage design rather than writing directly to USER-scoped `agent_memory`:
- **Stage 1 (done):** HabitAgent extracts habits from the session transcript → writes to `chat.habit`
- **Stage 2 (EP-4):** ConsolidationAgent reads `chat.habit` entries across sessions → merges → writes to USER-scoped `agent_memory`

This mirrors the cheatsheet pattern exactly: per-chat extraction → consolidated persistent memory.

**What HabitAgent does:**
1. Polls every 5 minutes (`poll_interval=300`); sleeps 5s between chats if work was found.
2. Detects idle chats via `get_idle_chats(IDLE_THRESHOLD)` — returns chats where latest AGENT RESPONSE timestamp is older than 1 hour and newer than `chat.habit_cursor_ts`.
3. Fetches USER records and AGENT RESPONSE records separately since cursor, merges and sorts by `id`.
4. Loads current `chat.habit` as existing context (`existing_habits`).
5. LLM extracts/updates habit profile across 5 dimensions: QUERY STYLE, INTERACTION STYLE, OUTPUT PREFERENCES, DOMAIN FOCUS, EXPERTISE SIGNALS.
6. Writes updated profile to `chat.habit` via `ProjectService.update_chat_habit()`.
7. Advances `chat.habit_cursor_ts` (per-chat TIMESTAMP column) to `latest_ts` via `update_chat_habit_cursor_ts()`. Not stored in `agent_memory`.

**Output format:** Structured text with `## DIMENSION` headers — not JSON. Leaf values are free-text observations; JSON would add no programmatic value here since ConsolidationAgent merges by content, not by typed fields.

**Update strategy:** confirms → unchanged, strengthens (seen again) → mark confirmed, contradicts → soften, new → add. Nothing is replaced.

**Re-entry behaviour:** User returning after idle starts a second session in the same chat. HabitAgent skips until the second idle, then processes only new exchanges (since cursor), merges with `chat.habit` already written.

**`user_habits` bucket removed from cheatsheet curator:** Habits are per-session patterns across exchanges, not per-exchange observations. The curator cannot detect recurrent preferences from a single `(query, answer)` pair.

**Tested on:** chats 383, 384, 385 (project 235) — habits written to `chat.habit` and verified in DB.

## ⬜ EX-2 Phase 2 — Habit Consolidation to USER Memory

Handled by ConsolidationAgent (EP-4, steps 6–10 above). No separate agent needed.

---

## What Carries to Future

| Item | Why not Q2 | When |
|---|---|---|
| EP-2 — `tool_search_memory` | Needs enough data in `agent_memory` to validate; premature if ConsolidationAgent soak is still ongoing | Q3 early — once ≥ 3 wells have PROJECT memory |
| `inferred` auto-promotion in ConsolidationAgent | Conservative mode through Q2; enable only after zero false-positive validation | Q3, after soak confirms quality |
| EX-2 phase 2 — Habit consolidation to USER memory | HabitAgent (phase 1) done. Phase 2 is ConsolidationAgent steps 6–10. | EP-4 (May 29) |
| EX-3 — Structured project memory | Refinement of ConsolidationAgent write structure; needs real data from Q2 soak to validate what works | Q3 mid |
| Backend evaluation (Honcho / Mem0 / QMD) | Defer until current architecture hits measurable limits | Long-term |

---

## Risk Register

| Risk | Impact | Mitigation |
|---|---|---|
| Curator JSON output is malformed | Parse error, entries lost | Validate output; fallback to plain-text append on parse failure; log all failures |
| ConsolidationAgent over-promotes wrong facts | Wrong facts persist in PROJECT memory | Conservative mode: only `verified` entries auto-promoted in Q2; `inferred` logged for review |
| Schema migration breaks existing bookkeeping cursors | CC and CheatsheetAgent lose cursor position | Additive-only migration; no existing column altered |
| Soak reveals systematic curator errors | Prompt debugging consumes tuning time | Prompt iteration is fast (no Python changes); 2-day round-trip per fix |
| Not enough real sessions to fully validate ConsolidationAgent by end of June | EP-2 and `inferred` auto-promotion slip further | Expected — soak is the plan, not a risk. Q2 success is measured by first session correctly recalled, not full pipeline validated |

---

## Success Criteria at End of Q2

| Criterion | How to verify |
|---|---|
| Numeric values verbatim in cheatsheet | Inspect `chat.cheatsheet` JSON after NPT session — NPT%, ROP, depth match source tool results exactly |
| Cross-session memory works | New session on analyzed well — IDA opens with correct entity facts, no user re-prompting |
| No false-confidence errors | All `confidence ≥ 0.8` entries in `agent_memory` are accurate |
| History search works | `tool_search_chat_history` returns the correct turn for a content-based query |
