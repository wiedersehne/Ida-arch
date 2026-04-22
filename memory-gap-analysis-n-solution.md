# IDA Memory Gap Analysis

*Benchmarked against Hermes (Nous Research), OpenClaw, and Claude Code*

---

## Memory Layer Framework

| Layer | Definition | Persists across sessions? |
|---|---|---|
| **In-context** | Active context window — injected files, current conversation | No |
| **Episodic** | Memory of specific past events and conversations | Yes |
| **External** | Persistent structured storage — files, DB, vector index | Yes |
| **Parametric** | Knowledge baked into model weights | Yes (immutable) |

This document covers **Episodic** and **External** gaps only.

---

## Current Architecture

```mermaid
flowchart TD
    subgraph InContext ["In-Context — loaded at session start"]
        CS[chat.playbook\ncheatsheet]
        UP[Uploaded files\nproject metadata]
    end

    subgraph Episodic ["Episodic — exists but not searchable"]
        RAG[(RAG index\nsource_type=CHAT\nall messages embedded)]
        CD[Chat record data\nstaged / clean_viz / raw]
    end

    subgraph External ["External — exists but underused"]
        AM[(agent_memory\nGLOBAL scope only\nbookkeeping)]
    end

    subgraph Background ["Background agents"]
        CA[CheatsheetAgent\n20s poll]
        CC[ContextCompressor\n5s poll]
    end

    CC -->|embed ALL messages| RAG
    CC -->|summarize >2000 chars| CD
    CA -->|curate per exchange| CS
    CS -->|injected at start| InContext
    AM -->|cursor tracking only| CA
    AM -->|cursor tracking only| CC
```

### What works
- Cheatsheet (`chat.playbook`) accumulates structured knowledge within a chat via dedicated curator LLM
- ContextCompressor indexes every USER + AGENT message into RAG (`source_type=CHAT`)
- Chat record data toolbox gives structured access to staged results and visualizations
- `agent_memory` DB table with PROJECT / ORG / GLOBAL scopes is built and operational

### What doesn't
The architecture has no episodic retrieval path and no external persistence path. Everything resets between chats. Both the RAG index and `agent_memory` exist but are either incomplete or never written to by sub-agents.

---

## Infrastructure: ContextCompressor

Before addressing gaps, understanding ContextCompressor is essential — it underpins
both the episodic and external memory systems.

It runs continuously (5s poll) and does **two separate jobs**:

```mermaid
flowchart TD
    A[New USER + AGENT records] --> B[Job 1 — Index\nALL messages → RAG\nsource_type=CHAT\nno threshold]
    A --> C{Job 2 — Compress\nlen > 2000 chars?}
    C -->|Yes| D[LLM summarize → 300 words\n→ compressed_message field]
    C -->|No| E[Skip — original only]
    B --> F[(RAG: original message)]
    D --> G[(chat_record.compressed_message)]
    F -. diverge .-> G
```

**Critical bugs — fix before building on top:**

| Bug | Impact | Severity |
|---|---|---|
| Cursor stored as GLOBAL scope | One `last_processed_chat_record_id` shared across all projects on the deployment — one project overwrites another's cursor | **High** |
| RAG indexes original, not compressed | After compression runs, RAG still holds the noisy original. Search returns degraded content | Medium |
| No dedup on re-index | Restart creates duplicate RAG embeddings for the same records — degrades search ranking | Medium |
| Sub-agents ignore `compressed_message` | Only `ida.py` reads `compressed_message or message`. All sub-agents read raw `message` — compression is invisible to them | Medium |
| Two jobs share one cursor | A compression failure still advances the cursor — failed compressions are silently skipped forever | Low |

**Additional structural issue — compression is lossy for structured data:**
The 300-word summary prompt is designed for prose. When the agent response contains
Markdown tables (NPT breakdowns, cost tables, ROP by section), the LLM summarizes
them to bullet points — the precise numeric values that would be most useful for
recall are lost. The RAG index stores the original table-heavy message, but as a
single noisy blob with no awareness of entity/metric structure.

---

## Episodic Memory Gaps

Episodic memory = ability to recall specific past events and conversations.

### EP-1 — Chat History Is Tail-Only, Not Searchable

**Current state:**
Every sub-agent independently loads recent history as a fixed tail of N records.
No agent can search history by content.

| Agent | Depth | Truncation |
|---|---|---|
| `ida.py` | last 12 records | full (uses `compressed_message` when available) |
| `datainsight.py` | last 4 messages | 200 chars each |
| `simulator_agent.py` | tail=5 | 200 chars |
| `subject_matter_expert.py` | tail=10 | — |
| `eowr_agent.py`, `report_generator.py` | varies | — |

Problems:
- Tail-only — "find the turn where we analyzed NNM-101's NPT" is impossible
- Inconsistent depth — agents disagree on what "recent" means
- No centralized abstraction — changing history loading requires editing every agent
- Cross-chat retrieval is absent entirely

**What's already in place:** ContextCompressor embeds every message into RAG with
`source_type=CHAT`. The embedding infrastructure exists. The gap is a missing
retrieval tool.

**Solution:**

*A — `ChatHistoryService` (centralize loading):*
```python
class ChatHistoryService:
    def load_tail(self, chat_id, limit=12) -> List[BaseMessage]:
        # uses compressed_message when available

    def search(self, query, project_id, chat_id=None,
               top_k=10, start_date=None, end_date=None) -> List[ChatRecord]:
        # RagService with source_type=CHAT filter

    def load_relevant(self, query, chat_id, top_k=5) -> List[BaseMessage]:
        # retrieve by relevance, not recency
```

All agents call `ChatHistoryService` — eliminates inconsistency and gives
every agent search capability for free.

*B — `tool_search_chat_history` in `chat_record_data_toolbox.py`:*
```python
def tool_search_chat_history(
    query: str,
    project_id: int,
    chat_id: Optional[int] = None,      # None = project-wide (covers EP-3)
    data_types: Optional[List[str]] = None,
    top_k: int = 10,
    start_date: Optional[str] = None,
    end_date: Optional[str] = None,
) -> str: ...
```

Uses `RagService(source_type=CHAT, project_id=...)` — no new embedding needed.

```mermaid
flowchart LR
    A[Agent needs prior result] --> B[tool_search_chat_history]
    B --> C[RagService\nsource_type=CHAT]
    C --> D[Ranked records + excerpts]
    D --> E{Has structured data?}
    E -->|Yes| F[tool_get_record_data\nfetch staged payload]
    E -->|No| G[Use message text]
```

*C — Update AGENT.md files:*
Add to each sub-agent: before asking the user to confirm data already established,
call `tool_search_chat_history` first.

---

### EP-2 — Structured Data in Chat Is Not Content-Searchable

**New insight — second-round analysis:**
ContextCompressor embeds each message as a single text blob. A response containing
a 30-row NPT table for NNM-101 embeds as one vector representing the whole message.
Searching "NNM-101 stuck pipe duration" retrieves the message but cannot point to
the specific value.

Compare to OpenClaw's `memory_search` — hybrid semantic + keyword over structured
daily notes — and the cheatsheet's own entity-specific Data Insights structure.

The cheatsheet already does structured extraction (entity → metric → value).
But its output never feeds back into the RAG index. The two systems don't talk
to each other.

**Solution:**
When the CheatsheetAgent adds a new Data Insight entry, also write a targeted
RAG embedding for that entity+metric combination:

```python
# After curator runs and new entity data is detected:
rag_service.add_document(RagEmbedding(
    content=f"Well NNM-101: NPT total 54.2h, main causes: stuck pipe 48%, equipment 21%",
    source_type=SourceType.CHEATSHEET,   # new source type
    source_id=chat_record_id,
    project_id=project_id,
    metadata={"entity": "NNM-101", "metric_type": "npt"}
))
```

This gives search results that point to specific facts, not just message blobs.

---

### EP-3 — No Cross-Chat Retrieval

Covered by `tool_search_chat_history(chat_id=None)` from EP-1 —
project-wide search across all chats using existing RAG embeddings.

---

### EP-4 — No End-of-Session Consolidation

**Current state:**
The CheatsheetAgent curates turn-by-turn. There is no session-end event
that promotes session conclusions to persistent memory. Compare to
OpenClaw's Dreaming pipeline.

**Solution — ConsolidationAgent** (extends CheatsheetAgent, triggered on
session close or 30-minute inactivity):

```mermaid
flowchart TD
    A[Session end trigger] --> B[Load chat.playbook\n+ last 20 records]
    B --> C[Consolidation LLM:\nWhat is durable beyond this chat?]
    C --> D{Per entry}
    D -->|Entity finding| E[→ agent_memory\nentity_NNM101\nscope=PROJECT]
    D -->|Project lesson| F[→ agent_memory\nproject_lessons\nscope=PROJECT]
    D -->|Chat-specific| G[Stay in playbook only]
```

---

## External Memory Gaps

External memory = persistent structured storage reused across sessions.

### EX-1 — Cheatsheet Does Not Cross Session Boundaries

**Current state:**
The cheatsheet (`chat.playbook`) is scoped to a single chat. A new chat in
the same project starts blank. PROJECT-scoped `agent_memory` exists but
nothing writes to it. IDA forgets everything between sessions.

**Additional insight — sub-agents don't read the playbook either:**
Even within a session, only the orchestrator (`ida_agent`) references
`tool_read_playbook`. Sub-agents (`data_insight_agent`, `sme_agent`, etc.)
never load the cheatsheet. The curated knowledge is only used by the
orchestrator's planning step — not by the agents doing the actual work.

**Solution:**

*Session persistence* — ConsolidationAgent (EP-4) writes to PROJECT-scoped
`agent_memory` at session end.

*Session start injection* — orchestrator reads PROJECT memories before
answering first query:
```
# ida_agent AGENT.md — plannable step:
tool_read_memory(name="entity_<well>", scope=PROJECT, project_id=...)
```

*Sub-agent access* — add `tool_read_playbook` call to each sub-agent's
AGENT.md so they benefit from accumulated knowledge within the session.

---

### EX-2 — No User Profile

**Current state:**
No persistent record of user preferences, depth units, operator conventions,
or recurring workflows. Hermes solves this with `USER.md` (injected every
session). Claude Code uses user-scoped CLAUDE.md.

**Solution:**
Store user profile in `agent_memory` (USER scope — requires schema change,
see EX-4). Written when user states a preference or the agent detects a
consistent pattern:

```python
{
  "depth_unit": "m",
  "preferred_output": "concise_tables",
  "operator": "ENI Congo",
  "primary_wells": ["NNM-101", "NNM-102"],
  "terminology_overrides": {"NPT": "lost time"},
  "last_active_project": 42
}
```

Inject as a lightweight block at session start alongside the project index.
Expose as a user-editable panel equivalent to Hermes' `USER.md`.

---

### EX-3 — No Progressive Memory Loading

**Current state:**
The full cheatsheet injects into context every turn regardless of query scope.
As the project accumulates wells, context cost grows linearly.

**Two-tier solution:**

*Tier 1 — Index (`chat.playbook`, always loaded, target < 500 tokens):*
```markdown
### Active Wells
- NNM-101: 45 days, NPT 12% — details: agent_memory:entity_NNM101
- NNM-102: in progress

### Project Knowledge
- Operator NPT threshold: >5% triggers review
- WellView exports use metres for this project

### Lessons
- tool_retrieve_data must precede tool_analyze_data
```

*Tier 2 — Entity detail records (`agent_memory` PROJECT scope, on demand):*
```python
name = "entity_NNM101"
scope = PROJECT
memory_type = "entity_insight"
object = {
    "well": "NNM-101",
    "npt_total_hours": 54.2,
    "npt_pct": 12.1,
    "main_npt_causes": ["stuck pipe", "equipment failure"],
    "rop_by_section": {"26in": 13.9, "17.5in": 11.8},
    "data_quality_notes": "WellView export missing mud log sheet",
    "last_updated_chat": 42,
    "confidence": 0.92
}
```

```mermaid
flowchart TD
    A[Session start] --> B[Tier 1: load index\n< 500 tokens always]
    B --> C[User query]
    C --> D{Mentions specific well?}
    D -->|Yes| E[tool_read_memory\nentity_WELLNAME → Tier 2]
    D -->|No| F[Index sufficient]
    E --> G[Full well context in context]
```

**CheatsheetAgent write strategy:**
- Every exchange → update Tier 1 index (entity name + one-liner only)
- New Data Insight detected → upsert Tier 2 entity record in `agent_memory`

---

### EX-4 — Cheatsheet Accuracy Cannot Be Verified

**Current state:**
The curator LLM can misinterpret data. Wrong values persist due to the
APPEND-ONLY rule. No human review. No confidence tagging. No source tracing.

**Solutions:**

1. **Source tagging** — tag each Data Insight entry with the chat record ID
   it derived from. Enables tracing suspect values back to the source turn.

2. **Confidence tiers:**
   - `[verified]` — value from a tool result with clear source
   - `[inferred]` — derived by curator from partial data
   - `[conflicted]` — conflicts with prior entry; both shown with sources

3. **Cheatsheet review UI** — `/cheatsheet` command showing current content
   with edit/delete, equivalent to Claude Code's `/memory`.

4. **Conflict detection** — curator must surface conflicting values with
   sources rather than silently resolving them.

---

### EX-5 — agent_memory Schema Is Under-Specified

**Current schema problems:**

| Problem | Detail |
|---|---|
| No `memory_type` field | User profiles, entity insights, bookkeeping all land in the same JSONB — no way to query by type |
| No FTS on content | Only `name` is indexed — content search requires expensive JSONB operators |
| No `user_id` scope | No per-user memory within a project — user profiles must use naming conventions |
| No versioning | `set_object` overwrites silently — wrong values have no recovery path |
| No size tracking | JSONB grows unboundedly — no context budget enforcement |

**Proposed schema (additive migration — all new fields nullable):**

```python
class AgentMemoryScope(StrEnum):
    GLOBAL = "GLOBAL"
    PROJECT = "PROJECT"
    ORG = "ORG"
    USER = "USER"                        # new — per-user across projects

class AgentMemoryType(StrEnum):
    USER_PROFILE = "user_profile"        # USER scope
    PROJECT_SUMMARY = "project_summary"  # PROJECT scope
    ENTITY_INSIGHT = "entity_insight"    # PROJECT scope — per-well/file
    LESSON_LEARNED = "lesson_learned"    # PROJECT / ORG scope
    BOOKKEEPING = "bookkeeping"          # GLOBAL — background agent state
    KNOWLEDGE = "knowledge"              # ORG / GLOBAL — domain facts

class AgentMemoryModel(SQLModel, table=True):
    # existing fields unchanged
    id, name, description, agent_id, scope, project_id, org_id, object, created_at, updated_at

    # new fields
    memory_type: AgentMemoryType = Field(index=True)
    user_id: Optional[int] = Field(default=None, foreign_key="user.id")
    content_text: Optional[str]          # FTS-indexed flat text, auto-populated from object
    confidence: Optional[float]          # 0.0–1.0
    size_chars: Optional[int]            # context budget tracking
    version: int = Field(default=1)      # incremented on each update
```

**`content_text` enables PostgreSQL FTS on memory content:**
```sql
SELECT * FROM agent_memory
WHERE scope = 'PROJECT' AND project_id = 42
  AND to_tsvector('english', content_text) @@ plainto_tsquery('NNM-101 NPT');
```

Migration: backfill `memory_type='bookkeeping'` for all existing records —
they are all bookkeeping entries today. All new columns nullable with defaults,
no data loss.

---

## Backend Evaluation — QMD, Honcho, Others

| Backend | What it adds | Dependency | IDA fit |
|---|---|---|---|
| **QMD** | Local-first sidecar, reranking, query expansion | External process | Re-evaluate if RAG search is too slow at scale |
| **Honcho** | Cross-session user modeling, multi-agent awareness | Cloud service | Re-evaluate for multi-org deployments; data policy risk |
| **Mem0** | Semantic memory graph, automatic fact extraction | Cloud / self-hosted | Re-evaluate if entity extraction at EP-2 proves too complex |
| **ChromaDB / Qdrant** | Dedicated vector DB, better ANN at scale | External process | Re-evaluate if PostgreSQL vector store bottlenecks |

**Recommendation: defer all external backends.**
- The RAG service already embeds all chat history. The gap is a missing filter, not a missing backend.
- Drilling data is sensitive — cloud backends (Honcho, Mem0 cloud) require operator approval.
- Fix the ContextCompressor cursor scope bug first. A broken cursor makes any search backend return incomplete results.

---

## Summary

### Gap Classification

| Gap | Layer | Severity | Depends on |
|---|---|---|---|
| **EP-1** Chat history tail-only, not searchable | Episodic | High | Fix CC cursor bug first |
| **EP-2** Structured data not content-searchable | Episodic | Medium | EP-1 infra |
| **EP-3** No cross-chat retrieval | Episodic | Medium | EP-1 (`chat_id=None`) |
| **EP-4** No end-of-session consolidation | Episodic → External | Medium | EX-5 schema |
| **EX-1** Cheatsheet doesn't cross session boundaries | External | High | EP-4 + EX-5 |
| **EX-2** No user profile | External | Medium | EX-5 schema (USER scope) |
| **EX-3** No progressive memory loading | External | Medium | EP-4 + EX-5 |
| **EX-4** Cheatsheet accuracy unverifiable | External | High | Independent |
| **EX-5** agent_memory schema under-specified | External | Medium | Independent |

### Build Order

```
Phase 0 — Fix ContextCompressor (prerequisite for everything)
  ├── Cursor scope: GLOBAL → PROJECT
  ├── Dedup check before re-embedding
  └── Re-index with compressed_message when compression runs

Phase 1 — Episodic retrieval (EP-1, EP-3)
  ├── ChatHistoryService — centralize all agent history loading
  └── tool_search_chat_history — RagService with source_type=CHAT filter

Phase 2 — External persistence foundation (EX-5, EP-4, EX-1)
  ├── Schema migration — add memory_type, user_id, content_text, confidence
  ├── ConsolidationAgent — session-end trigger → PROJECT writes
  └── Sub-agents read playbook — add tool_read_playbook to AGENT.md files

Phase 3 — External memory features (EX-2, EX-3, EP-2)
  ├── User profile — USER scope + preference detection
  ├── Two-tier cheatsheet — index + entity detail records
  └── Cheatsheet → RAG feedback loop (entity embeddings from curator output)

Phase 4 — Quality and UX (EX-4)
  ├── Source tagging + confidence tiers in curator prompt
  └── /cheatsheet review UI

Phase 5 — External backend evaluation (if needed)
  └── QMD / Honcho — only if Phase 1–4 reveal scale or modeling limitations
```

**All phases use existing infrastructure.** No new DB tables or external backends
required in Phases 0–4. The key insight: ContextCompressor already does the
hard work of embedding everything — IDA just lacks the tools to query it.
