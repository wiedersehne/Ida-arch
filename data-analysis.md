---
name: data-analysis
description: Structured drilling-data analysis over the Data Layer — NPT, cost, drilling progress/ROP, well comparison, and BHA/casing/mud. Resolves entities and queries the Data Layer (semantic SQL + entity APIs), reasons over the rows, then visualizes and exports.
compatibility: ida_agent
metadata:
  author: ida-team
  version: "2.0"
  category: analysis
  allowed-tools:
    # Data Layer MCP (retrieval/schema) — confirm the controller's group label
    - data_layer
    # IDA-native presentation + session data
    - data_visualization
    - chat_record_data
---

# Data Analysis

Single skill for all complex drilling-data analysis questions. The **Data Layer MCP** is
the source of truth for facts; this skill **discovers the schema, resolves
entities, queries, then reasons over the returned rows itself** and renders the
answer with IDA's visualization tools. 

## When to Use
Activate for any complex quantitative question about a well or set of wells:

- **NPT / lost time** — downtime, trouble time, stuck pipe, equipment failure,
  waiting on weather/services, mud losses, "why did this well run slow?"
- **Cost** — total well cost, cost per meter, cost vs time/depth, phase
  breakdown, AFE vs actual, overruns.
- **Drilling progress** — depth vs time, ROP, section times, flat spots,
  efficiency, "how far along is this well?"
- **Well comparison** — offset benchmarking of any metric across ≥2 wells.
- **BHA / casing / mud** — BHA runs, bit performance, casing/liner shoe depths,
  mud properties (MW, PV/YP, ECD).

## Core Workflow

Run these in order. Steps 1–2 are cheap and prevent hard-coding entity/field
names that may differ per deployment — the Data Layer is schema-driven.

**Step 1 — Discover schema (once per session, then reuse).**
Call `get_semantic_sql_schema` for queryable tables, columns, units, enum hints,
and join notes. Use `list_entity_types` if you don't yet know whether a concept
(Well, Wellbore, BHARun, Trajectory, Activity, NPTEvent, MudCheck) is modeled,
and `describe_core_entity_type` for the exact shape of one entity before querying.
Never assume a column name — read it from the schema.

**Step 2 — Resolve entities.**
Turn well names/IDs from the user into entity references with `find_entity`
(e.g. `entity_type="Well"`, filters by name). This replaces any hard-coded
well-lookup. For "all wells in the project," list via `query_entity_records` or
`get_entity_data_stats`.

**Step 3 — Retrieve.** Pick the lightest tool that answers the question:
- **Simple filter / single entity** → `query_entity_records` (schema-validated
  filters, pagination). Safer than generating SQL.
- **Joins, aggregates, group-by, time-bucketing, JSONB extension fields** →
  `run_semantic_sql` (bounded, read-only SELECT). Consult the Step-1 schema first.
  TimescaleDB hyperfunctions (`time_bucket`, `first`, `last`, `locf`) are
  available on time-series entities.
- **Context bundle (entity + neighborhood + collection rows like Activity /
  NPTEvent / TrajectoryStation)** → `get_entity_context`.
- **Multi-well side-by-side** → `compare_entities` (same type, chosen fields).

**Step 4 — Analyze (the agent, not a tool).**
Compute totals, %s, breakdowns, ROP, cost/m, etc. directly from the returned
rows. Aggregate in `run_semantic_sql` when the dataset is large; reason in-context
when it is small. Ground every number in a returned value — never estimate.

**Step 5 — Visualize (only on the full path).**
Build the chart spec from your analyzed rows and call `tool_generic_plotlyjs`
(or `tool_generic_table_visualization` for tabular, `tool_well_trajectory_plot`
for trajectories). Paste the returned `<div data-viz-id='...'>` tag verbatim,
never in a code block.

**Step 6 — Export (on request or large tables).**
Call `tool_export_table` when the user asks for a download **or** a table exceeds
~10 rows. Do not offer it proactively for a simple single-well summary.

**Simple path:** single well + single metric + no chart → Steps 1–4 only
(reuse a cached schema), then respond with a small table + 2–3 sentences. Stop.

## Per-Analysis Guidance

> Entity/field names below are the **expected** Core types; always confirm against
> the Step-1 schema, since cost and casing concepts in particular may be modeled
> as Core entities, extension profiles, or Custom types per deployment.

### NPT
- Entities: `NPTEvent` (and `Activity` for the time log). Group by the
  activity/cause field; sum durations and event counts.
- Report NPT as **both absolute hours and % of relevant time** (drilling time or
  reported span). A higher total isn't worse if the % is lower.
- Chart: pie of hours-by-category for one well; bar of totals per well for many.

### Cost
- Discover the cost entity/fields from the schema (often on `Activity` or a cost
  entity/extension). Aggregate by phase/category with `run_semantic_sql`.
- `cost per meter` = total cost / total depth — note the basis; it's misleading
  across formations. Report **AFE vs actual** side-by-side when both exist.
- Charts: cumulative cost vs time, cost vs depth, phase breakdown.

### Drilling progress / ROP
- Entities: `Activity` (depth-time, drilling hours), `Trajectory` /
  `TrajectoryStation` (MD/TVD), `Wellbore`.
- ROP = depth drilled / on-bottom drilling time. Separate **on-bottom ROP** from
  effective/average ROP and state which you used. Call out flat spots (no depth
  progress while time accrues).
- Chart: depth-vs-time curve; ROP-by-section bar for breakdowns.

### Well comparison
- Use `compare_entities` for the headline metric, then `run_semantic_sql` for
  aggregates the comparison tool can't express.
- **Normalize** before comparing wells of different TD: cost/m, ROP, NPT % — not
  raw totals. State the normalization basis.

### BHA / casing / mud
- Entities: `BHARun` (bit runs, footage, hours), `MudCheck` (MW, PV/YP, ECD),
  and casing/shoe depths (confirm the entity from the schema).
- Per-BHA ROP and footage come from `BHARun`; reconcile against well-level KPIs
  and flag scope mismatches. Report casing/liner **shoe depths in MD** with set dates.

## Output Format

1. **Executive summary** — 1–2 sentences with the headline number(s) and the
   single largest driver/contributor.
2. **Summary table** — always a Markdown table, never a bullet list. Bold the
   largest value in the key numeric and % columns.
3. **Chart tag** — embed the `<div data-viz-id='...'>` immediately after the
   table (full path only).
4. **Narrative** — 2–4 sentences: top contributors, patterns, planned vs
   unplanned / AFE vs actual / on-bottom vs effective where relevant.
5. **Multi-well comparison table** — one compact row per well when ≥2 wells.

## Domain Heuristics

- **Units and currency** come from the schema/source — never convert or assume.
  Quote `mKB`, `m/hr`, `$`, etc. as stored.
- **Use exact category/enum names** from the data; do not rename them.
- **Date filters:** scope tables and narrative to the user's window; if the data
  is full-well, say so.
- **Data quality first:** when a result looks non-physical (TVD≪MD, zeroed
  fields, regressing depth), profile it with `profile_core_entity_data` /
  `get_entity_data_stats`, base the answer on the reliable field (e.g. MD-based
  progress when TVD is corrupt), and state the limitation.
- **Confidence:** report findings as grounded ("the largest NPT share is X");
  offer causes only when labeled as hypotheses.

## Do NOT

- Do **not** assume entity or column names — read them from `get_semantic_sql_schema`
  / `describe_core_entity_type` first.
- Do **not** call `run_semantic_sql` with anything but a single bounded read-only
  SELECT over the registered surface; use `query_entity_records` for simple filters.
- Do **not** call `compare_entities` for a single well.
- Do **not** paste raw tool output — synthesize into tables and narrative.
- Do **not** invent or estimate values — if a field is missing for a well, say so.
- Do **not** wrap chart `<div>` tags in code blocks.
