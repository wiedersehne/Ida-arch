Idaparser Refactor: Toolbox + Template-Based Workflow

Current State (summary)





Idaparser.py (~3k lines): Single agent with LangGraph file_loading → ddr_processing → embedding_generation → completion. File handlers cover PDF, Excel, CSV, text, JSON, Word, PPT, HTML, XML, images (OCR).



Classification: _classify_document_and_company() returns (doc_type, company_name) with doc_type in ddr | other; result cached by content hash. Company name is used to select extraction prompt via _get_prompt(company_name) (hardcoded eni/statoil/cop/akerbp/chevron/unknown).



Extraction: _map_ddr_to_schema(content, company_name, file_name) uses instructor + DDRCoreSummary and a prompt chosen by company. Post-processing (activity/plan/operation code mapping) uses organization_service config keyed by company.



Parallelism: DDR files processed in parallel; within a long DDR (≥6 pages), segments processed in parallel (existing DDR_PAGE_COUNT_THRESHOLD).



Prompts: Stored in prompt table via prompts.py; Idaparser registers default prompts by name (e.g. ddr_mapping_prompt_eni) and selects by company.

Target Behavior





File types: Unchanged — same handlers for pdf, excel, csv, images, etc.



Classification: doc_type in ddr | well_view | other (add well_view; company no longer used for template identity).



Template by pattern: Each document is assigned a template_id from a pattern fingerprint (e.g. structural/layout hash from first pages). Same pattern → same template → instant reuse. New pattern → create new template record and store a prompt for it.



Prompt per template: Each template has an associated prompt (registered/saved). For new templates, generate a prompt (e.g. from sample + schema) and save it; use that prompt for extraction. Schema: DDR schema (DDRCoreSummary) is ready; well_view schema can be defined later or reuse/extends DDR.



Parallel processing: Keep and, if needed, extend parallel handling for long documents (DDRs and eventually well_view).



Preprocessing: After DDR extraction, run deterministic extractors (activities, progress, cost, casing, mud, BHA) and persist to dedicated tables so DataInsight can query directly instead of re-extracting on each request.



Architecture (high level)

Workflow A: Fixed pipeline

flowchart LR
  subgraph input [Input]
    Files[Files]
  end
  subgraph workflow [Idaparser workflow]
    Load[file_loading]
    Classify[classify_doc_type]
    Template[identify_template]
    Extract[extract_by_template]
    Embed[embedding_generation]
    Done[completion]
  end
  subgraph toolbox [Document processing toolbox]
    LoadDoc[load_document]
    ClassifyDoc[classify_document]
    MatchTemplate[match_or_create_template]
    ExtractDoc[extract_with_template]
  end
  subgraph storage [Storage]
    Templates[(document_templates)]
    Prompts[(prompt)]
    DDR[(DDR / schema)]
  end
  Files --> Load
  Load --> Classify
  Classify --> Template
  Template --> Extract
  Extract --> Embed
  Embed --> Done
  Load -.-> LoadDoc
  Classify -.-> ClassifyDoc
  Template -.-> MatchTemplate
  Extract -.-> ExtractDoc
  MatchTemplate --> Templates
  MatchTemplate --> Prompts
  ExtractDoc --> DDR

Workflow B: LLM orchestrator within graph

flowchart TB
  subgraph graph [LangGraph]
    Ingest[file_ingest]
    Orchestrator[processing_orchestrator]
    Embed[embedding_generation]
    Done[completion]
  end
  subgraph inner [Inside processing_orchestrator]
    LLM[LLM]
    ToolLoop[invoke_llm_with_tools]
    LLM --> ToolLoop
    ToolLoop -->|result| LLM
  end
  subgraph tools [Tools]
    T1[load_file]
    T2[classify_document]
    T3[identify_template]
    T4[extract_document]
    T5[queue_for_embedding]
    T6[finish_processing]
  end
  Ingest --> Orchestrator
  Orchestrator --> Embed
  Embed --> Done
  ToolLoop -.-> tools

LLMToolsManager.invoke_llm_with_tools + DocumentProcessingToolbox. Tools read/write state via ContextApi (pending_files, embedding_queue). LLM picks tools until finish_processing.



Implementation Plan

1. Document template storage





New table (or SQLModel): e.g. DocumentTemplate with:





id (PK), doc_type (ddr | well_view), pattern_signature (string hash/fingerprint), prompt_id (FK to prompt), schema_name (e.g. ddr), org_id/project_id (scope), created_at, updated_at.



Fingerprint strategy: Derive a stable “pattern” from document content so that same operator/well format produces the same fingerprint. Options:





A: First N characters (e.g. 1500) of normalized text (strip numbers/dates) + section headers regex → hash.



B: LLM-assisted “layout/format description” from first page(s) → hash (slower, more flexible).



Recommendation: Start with A (deterministic, fast). Store fingerprint in pattern_signature; lookup by (doc_type, pattern_signature) (and optionally org/project scope).

Files: New module e.g. backend/app/db/document_templates.py (model + service: create, get_by_signature, list by doc_type).

Extensibility for future doc types: doc_type is not hardcoded. To add a new type (e.g. rig_report): (1) add to classification prompt, (2) define schema (Pydantic model), (3) register in schema registry, (4) add storage (DB table or reuse DDR-like structure), (5) optionally add preprocessed tables. Template + fingerprint flow stays the same.

2. Schema registry (extensibility)





Purpose: Map schema_name → Pydantic model + storage handler + optional post-processing. Enables new doc types without changing core extraction logic.



Registry structure (e.g. in document_processing/schema_registry.py):





SCHEMA_REGISTRY: Dict[str, SchemaConfig] where SchemaConfig = {response_model, storage_fn, post_process_fn, preprocess_extractors}.



ddr → DDRCoreSummary, DDRManager.add_ddr_from_file, [activity/plan/operation code mappings], [6 preprocessed extractors].



well_view → WellViewSchema (TBD), WellViewManager.add (TBD), [], [].



rig_report (future) → RigReportSchema, RigReportManager.add, [], [].



Extraction flow: extract_with_template(template_id, content) looks up template.schema_name → gets response_model from registry → runs instructor → calls storage_fn → optionally post_process_fn → optionally runs preprocess_extractors. DDR-specific code lives in DDR registry entry, not in core.



Classification: Add new doc_type to prompt; no code change needed for template/extraction once registered.

Files: New backend/app/agents/utils/document_schema_registry.py (or similar); each doc type can have its own module (ddr.py, well_view.py) that registers.

3. Document processing toolbox





New toolbox: backend/app/agents/tools/toolbox/document_processing_toolbox.py (or idaparser_toolbox.py) extending ToolboxBase.



Tools: Can be either (a) internal methods called by fixed-pipeline nodes, or (b) LLM-callable when using the orchestrator workflow. For orchestrator: register tools with register_tool and wire to LLMToolsManager; each tool uses ContextApi to read/write state (pending_files, current_file_content, embedding_queue, extraction_results).





load_document: Given file path, return loaded chunks/documents (reuse current handlers: PDF, Excel, CSV, images, etc.).



classify_document: Given document content (and optional candidate list), return doc_type in ddr | well_view | other (and optionally company for code mappings only).



identify_template: Given doc_type and content, compute fingerprint → lookup template; if missing, create template + generate and save prompt → return template_id and prompt_id.



extract_with_template: Generic extraction. Given template_id, content, company_name: lookup template → get schema_name → get response_model and handlers from schema registry → run instructor → storage_fn (persist) → post_process_fn (e.g. DDR code mappings) → preprocess_extractors (e.g. DDR activities/cost). Returns record id. Works for any registered doc_type.



Toolbox gets dependencies: DDRManager, PromptStorageService, document template service, LLMManager (for classification and extraction), file-handler logic (moved or delegated from Idaparser).

Files: New toolbox class; optionally move file-loading/classification helpers into a shared module used by both toolbox and Idaparser for a single source of truth.

4. Classification: add well_view and stop using company for template





Extend _perform_llm_classification (or equivalent in toolbox) to return doc_type in ddr | well_view | other.





Define well_view criteria in the prompt (e.g. well-centric view, well list, well summary — to be refined with you).



Remove company from template/prompt selection. Keep company only as metadata for:





Post-extraction code mappings (activity_codes, plan_codes, operation_codes) from organization config (existing _load_code_mapping(company_name, ...)).



Classification cache key: use content hash (and optionally doc_type) only; company can still be in the cached tuple for code-mapping use.

Files: Idaparser.py (or toolbox classification method); update prompt text and response parsing for three-way doc_type.

5. Template matching and prompt creation





Match: After classification, for ddr (and later well_view), compute pattern_signature from document content. Query DocumentTemplate by (doc_type, pattern_signature) (and scope). If found → use template_id and prompt_id for extraction.



Concurrency: Add unique constraint on (doc_type, pattern_signature, org_id) so concurrent runs creating the same template do not duplicate. On IntegrityError, retry lookup to get the template created by another process.



Create: If no template found:





Create new DocumentTemplate row with new UUID, doc_type, pattern_signature, schema_name (e.g. ddr).



Generate prompt: Either (a) use a generic “DDR extraction” default prompt and save it linked to this template, or (b) one-shot LLM: “Given this document sample and the DDRCoreSummary schema, produce a system prompt that instructs extraction” → save as new prompt, link to template. (b) is more flexible but adds latency and cost for first-time patterns.



Save prompt via existing PromptStorageService (agent_id = idaparser, name = e.g. ddr_template_{template_id} or stable name), then set template.prompt_id.



Idaparser default prompts: Existing company-based default prompts can remain as fallback or be migrated to one “generic DDR” default used when creating new templates (option (a)) until a generated prompt is available.

Files: Template service, toolbox “identify_template” and “extract_with_template”, Idaparser nodes that call them.

6. Idaparser workflow changes

Option A (fixed pipeline) or Option B (LLM orchestrator) — see Architecture section. Both preserve embedding_generation and completion nodes.





Graph (Option A): Keep high-level structure; rename or add nodes for clarity, e.g.:





file_loading: still load files and get per-file document chunks (using toolbox or current logic).



Classification: Ensure each file gets doc_type (ddr | well_view | other) and optionally company (for code mapping only). Can stay inside file_loading or be a dedicated node.



Template resolution: For each ddr/well_view file, call toolbox’s identify_template → get template_id and prompt_id; attach to state (e.g. documents_to_process[i]["template_id"], prompt_id).



ddr_processing (or generalized extraction node): For each ddr/well_view item, call extract_with_template (template’s prompt + schema). Continue to run parallel at file level and, for long DDRs, at segment level (unchanged). Write results to DDR DB (and later well_view store if needed). For other, skip extraction.



embedding_generation → completion: unchanged in intent.



Graph (Option B): file_ingest → processing_orchestrator → embedding_generation → completion. Orchestrator uses LLMToolsManager + DocumentProcessingToolbox; tools (load_file, classify_document, identify_template, extract_document, queue_for_embedding, finish_processing) read/write state via ContextApi. Parallelism stays inside extract_document tool for long documents.



State: Extend DocumentProcessingAgentState with pending_files, embedding_queue, template_id, prompt_id, doc_type; keep company_name for code mappings.

Files: Idaparser.py (graph and nodes), state.py.

7. Extraction path: template prompt + schema





Replace _get_prompt(company_name) with “get prompt by template”: load prompt text by prompt_id (or template’s linked prompt name).



Keep _map_ddr_to_schema-style flow: instructor + DDRCoreSummary + template’s system prompt. Post-processing (_map_operation_codes_to_main_activity, _map_activity_codes_to_sub_activity, _map_plan_codes_to_down_time, _clean_ddr_descriptions) still use company_name from file metadata (from classification) and organization config.



Long documents: Keep current logic: if page count ≥ threshold, split by page/header, run extraction per segment in parallel, then add_ddr_from_file per segment (or aggregate if schema allows). Toolbox’s extract function should accept a single content string; Idaparser continues to orchestrate splitting and parallelism.

Files: Idaparser.py (extraction and post-processing), toolbox extract method.

8. Preprocessing: denormalized tables for DataInsight

Preprocessing is doc-type-specific and driven by the schema registry. For ddr, after storage_fn (add_ddr_from_file), run the registered preprocess_extractors. For other types, preprocess_extractors may be empty or different.

Extractors to run (from data_analyzers.py and DataInsight tools):





drilling_activities: extract_all_activities_deterministic on time_log - NPT table + activity table



progress_events: extract_progress_events_deterministic on time_log + daily_information - progress/depth table



cost_data: extract_cost_data_deterministic on cost_summary - cost table



casing_data: Direct from content.casing (no analyzer) - list per DDR



mud_data: Direct from content.mud (no analyzer) - list per DDR



bha_data: Direct from content.bha_components (no analyzer) - list per DDR

Input format: Convert each DDR/segment to well_data format expected by extractors: [{operation_date, wellbore_name, well_name, content: ddr_dict}, ...]. For single-segment DDR, list has one item.

New DB tables (e.g. in backend/app/db/preprocessed_ddr.py):





All tables include ddr_id (FK to DDRModel.id) for traceability, re-preprocess, and backfill.



DrillingActivityModel: project_id, ddr_id, well, wellbore, operation_date, start_time, end_time, duration_hr, main_activity, act_code, sub_activity, is_npt, description, etc.



ProgressEventModel: project_id, ddr_id, well, wellbore, operation_date, end_depth_md, end_depth_tvd, daily_cost, drilling_hours, etc.



CostDataModel: project_id, ddr_id, well, wellbore, operation_date, daily_cost, cumulative_cost, cost_per_depth, etc.



CasingDataModel: project_id, ddr_id, well, wellbore, operation_date, casing_type, outer_diameter_in, inner_diameter_in, top_depth, bottom_depth, installation_date, etc.



MudDataModel: project_id, ddr_id, well, wellbore, operation_date, mud_type, depth_md, mud_density, mud_viscosity, etc.



BHADataModel: project_id, ddr_id, well, wellbore, operation_date, component_name, joint_count, component_length, outer_diameter_in, inner_diameter_in, cumulative_length, connection_type, etc.

Idaparser changes: After add_ddr_from_file (returns row with id), build well_data from structured_ddr(s), call the three extractors, directly extract casing/mud/bha lists, persist all to the new tables with ddr_id. Run preprocessing in a separate block so DDR commit is independent; on preprocessing failure, log and do not rollback DDR.

DataInsight changes: Add query methods for each table. In extract_well_data or tool entry points, check preprocessed tables first; if data exists, build activity_well_results, depth_well_results, cost_well_results, casing_well_results, mud_well_results, bha_well_results from DB. Fallback: if no preprocessed data, use current flow (load DDR, run extractors).

Backfill: Add preprocess_project_ddrs(project_id) to preprocess existing DDRs that lack preprocessed rows. Enables gradual migration without blocking.

Files: New backend/app/db/preprocessed_ddr.py; modify Idaparser.py, datainsight.py, data_insight_toolbox.py.

Note: tool_extract_bha_data_from_ddr uses RAG - separate. BHA preprocessing here is for content.bha_components only.

9. Well_view handling





Classification: well_view is a new label; add criteria to the classification prompt (e.g. “well view”, “well list”, “well summary” — to be defined).



Processing: For now, well_view can be treated like “other” for extraction (embedding only) until a well_view schema and template prompts are defined; or a minimal schema can be added and reuse the same template/extraction pipeline with a different schema_name. No company-based template; same pattern-based template + prompt approach.

Files: Classification prompt in Idaparser/toolbox; optionally schema and extraction branch for well_view later.

10. Toolbox registration and agent wiring





Register the new toolbox in tools/toolbox/init.py.



Idaparser receives the toolbox (or the underlying services) via constructor/dependency injection: either pass the toolbox instance from registry when creating IDAParser, or pass template service + prompt service and keep file/classification logic inside Idaparser but call the same template/prompt APIs. Recommendation: inject template service and prompt service into Idaparser and implement the “toolbox” as a thin layer that uses these services so the agent remains the single orchestrator and the toolbox can still be reused by tests or other callers.

Files: registry.py, Idaparser __init__, toolbox __init__.py.



Migration and backward compatibility





Existing projects: Documents already processed with company-based prompts have no stored “template”; they can keep using existing behavior if we add a fallback: when no template is found for a fingerprint, fall back to current company-based prompt selection until a template is created for that document. Alternatively, run a one-time “template discovery” over existing files to create templates from current company-based prompts.



Prompts: Keep existing default prompts (ddr_mapping_prompt_eni, etc.) as fallbacks for “generic” or “legacy” template creation (e.g. first time we see a pattern we could clone the best-match default by company if available, then phase out company key).



Open decisions (for discussion)





Well_view definition: Exact criteria (and schema) for classifying and processing well_view docs.



Fingerprint design: Pure text hash (A) vs LLM-assisted layout hash (B); and exact normalization (e.g. how much to strip, which sections to include).



New-template prompt: Use a single generic DDR prompt for all new templates (a) vs LLM-generated prompt from sample (b).



Scope of templates: org-level vs project-level (current thinking: org or project in DocumentTemplate for multi-tenant isolation).



Toolbox as “tools” vs internal API: Whether the document-processing steps are exposed as LangChain tools for another agent or only as an internal API used by Idaparser nodes; the plan above keeps them as internal API with optional toolbox abstraction for reuse and tests.



Idempotency for re-uploads: Skip extraction if same file (content hash) already processed, or always re-process and overwrite? Configurable per project?



Preprocessing registry: Pluggable registry for data types (extractor + table) vs hardcoded list of six types. Registry enables adding survey, trajectory, etc. without code changes.



Workflow choice: Fixed pipeline (Option A) vs LLM orchestrator (Option B). Option A: simpler, lower latency, deterministic. Option B: flexible, handles edge cases, consistent with DataInsight/ReportGenerator. Could implement both behind a config flag.



File and dependency summary







Area



New



Modified





DB



document_templates.py, preprocessed_ddr.py, document_schema_registry.py



—





Toolbox



document_processing_toolbox.py (or idaparser_toolbox.py)



toolbox/__init__.py





Agent



—



Idaparser.py, state.py, datainsight.py, data_insight_toolbox.py





Factory



—



registry.py (inject template/prompt services if needed)





Prompts



—



Classification prompt (ddr/well_view/other); optional new default “generic DDR”

No change to ddr.py schema (DDRCoreSummary); optional future well_view schema in same or new module.
