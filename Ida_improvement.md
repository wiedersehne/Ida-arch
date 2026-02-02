# Insights gained for IDA

## Executive Summary

This document analyzes 6 research papers on LLM agents and tool usage, plus OpenAI's production data agent architecture, extracts practical technical points, and provides recommendations for enhancing the IDA system. The papers cover: tool documentation, prompt optimization, RAG systems, search enhancement, memory management, agent scaling, and production-grade data agent design.

---

## 1. Tool Documentation Improvements (Hsieh et al.)

### Key Technical Points

1. **Tool Documentation Over Demonstrations**
   - Zero-shot prompts with tool documentation achieve performance on par with few-shot prompts
   - Documentation provides neutral, comprehensive tool descriptions without bias from specific examples
   - Eliminates the need for manual demo curation and selection

2. **Documentation Structure**
   - Clear descriptions of tool capabilities and limitations
   - Input/output specifications
   - Usage examples and important notes
   - Parameter descriptions and constraints

### Current IDA Implementation

✅ **Already Implemented:**
- Tool descriptions, metadata and function in `ToolDefinition` (name, description, args_schema, tags. required_params, passthrough_params, allowed_write_to_context_items, allowed_read_to_context_items)

❌ **Missing/Can Improve:**
- No structured tool documentation format beyond basic descriptions
- No explicit documentation for tool capabilities/limitations
- No usage examples in tool definitions
- Tools are bound directly to LLM without enhanced documentation context

### Recommendations for IDA

1. **Enhanced Tool Documentation Format**
   ```python
   class ToolDocumentation(BaseModel):
       name: str
       description: str
       output_spec: Dict[str, Any]  # Expected output format
       usage_examples: List[Dict[str, str]]  # Example correct.incorrect usage
       important_notes: List[str]  # Critical usage information
       related_tools: List[str]  # Tools that work well together
   ```

---

## 2. AVATAR: Optimizing LLM Agents for Tool Usage via Contrastive Reasoning

### Key Technical Points

1. **Contrastive Reasoning Framework**
   - Comparator module contrasts positive vs negative examples
   - Iteratively refines prompts based on execution results
   - Eliminates manual prompt engineering

2. **Automated Prompt Optimization**
   - Learns from successful/failed tool usage patterns
   - Generates comprehensive tool-usage instructions automatically
   - Adapts prompts based on task performance

3. **Implementation Approach**
   - Collect positive examples (successful tool chains)
   - Collect negative examples (failed tool usage)
   - Use comparator to identify key differences
   - Generate optimized prompts highlighting successful patterns

### Current IDA Implementation

✅ **Already Implemented:**
- Tool execution tracking (`ToolExecutionInfo`)
- Error handling and logging
- Context sharing between tools

❌ **Missing/Can Improve:**
- No contrastive learning from tool execution results
- No automated prompt optimization based on success/failure
- No feedback loop from tool execution to prompt improvement

### Recommendations for IDA

1. **Tool Execution Analytics**
   ```python
   class ToolExecutionAnalytics:
       def track_successful_execution(self, tool_name, args, result, context):
           # Store positive examples
           
       def track_failed_execution(self, tool_name, args, error, context):
           # Store negative examples
           
       def generate_contrastive_insights(self):
           # Compare successful vs failed patterns
   ```

2. **Contrastive Prompt Optimizer**
   - Analyze successful tool chains vs failed ones
   - Identify patterns in successful tool usage
   - Automatically refine system prompts and tool descriptions
   - Update tool recommendations based on historical success

3. **Adaptive System Messages**
   - Generate system messages that highlight successful tool usage patterns
   - Include warnings about common failure modes
   - Adapt based on agent performance over time

4. **Tool Chain Learning**
   - Learn which tool sequences work well together
   - Suggest optimal tool chains for common tasks
   - Warn about tool combinations that typically fail

---

## 3. SELF-RAG: Learning to Retrieve, Generate, and Critique

### Key Technical Points

1. **Retrieval-Augmented Generation with Self-Critique**
   - LLM decides when to retrieve (retrieve/no-retrieve)
   - LLM critiques retrieved passages (relevant/irrelevant)
   - LLM critiques its own generation (supported/partially supported/unsupported)

2. **Adaptive Retrieval**
   - Retrieval is task-dependent and query-dependent
   - Model learns when retrieval is necessary
   - Reduces unnecessary retrieval calls

3. **Quality Control**
   - Self-critique tokens guide generation quality
   - Supports multiple retrieval strategies
   - Handles cases where retrieval is not needed

### Current IDA Implementation

✅ **Already Implemented:**
- RAG service with semantic/keyword/hybrid search
- Cross-encoder re-ranking
- Chat history integration
- Source attribution

❌ **Missing/Can Improve:**
- No self-critique mechanism for retrieval decisions
- No adaptive retrieval (always retrieves)
- No quality assessment of generated responses
- No retrieval decision learning

### Recommendations for IDA

1. **Adaptive Retrieval Decision**
   ```python
   class RagService:
       def should_retrieve(self, query: str, context: Dict) -> bool:
           # LLM decides if retrieval is needed
           # Consider: query type, available context, chat history
           
       def critique_retrieval(self, query: str, retrieved_docs: List) -> Dict:
           # Assess relevance of retrieved documents
           
       def critique_generation(self, query: str, response: str, sources: List) -> Dict:
           # Assess if response is well-supported by sources
   ```

2. **Self-Critique Integration**
   - Add critique tokens to LLM prompts
   - Implement retrieval decision logic
   - Add response quality assessment
   - Use critique feedback to improve retrieval and generation

3. **Retrieval Strategy Selection**
   - Learn when to use semantic vs keyword vs hybrid search
   - Adapt retrieval strategy based on query characteristics
   - Cache retrieval decisions for similar queries

4. **Multi-Layered Context for Data (OpenAI Insight)**
   - Extend RAG beyond documents to include table metadata, code analysis, and institutional knowledge
   - Build context layers: table usage, human annotations, code-level understanding, institutional docs, memory, runtime validation
   - Use RAG to retrieve relevant context from all layers efficiently
   - Impact: Dramatically improves data query accuracy and understanding

---

## 4. ReZero: Enhancing LLM Search Ability by Trying

### Key Technical Points

1. **Zero-Shot Error Correction**
   - LLM tries search queries and evaluates results
   - Iteratively refines search queries based on feedback
   - No training data required

2. **Search Query Refinement**
   - Generate initial search query
   - Execute search and evaluate results
   - Refine query if results are unsatisfactory
   - Repeat until satisfactory results or max iterations

3. **Self-Correction Mechanism**
   - LLM evaluates its own search results
   - Identifies gaps or irrelevant results
   - Generates improved queries automatically

### Current IDA Implementation

✅ **Already Implemented:**
- Rewriting query and ask for clarification.
- RAG search with multiple strategies
- Query processing and embedding
- Result re-ranking

❌ **Missing/Can Improve:**
- No iterative query refinement
- No search result evaluation
- No self-correction for search queries
- Single-pass search only

### Recommendations for IDA

1. **Iterative Search Refinement**
   ```python
   class RagService:
       def search_with_refinement(self, query: str, max_iterations: int = 3):
           current_query = query
           for iteration in range(max_iterations):
               results = self.rag_service.hybrid_search(current_query)
               evaluation = self.evaluate_results(query, results)
               
               if evaluation['satisfactory']:
                   return results
               
               current_query = self.refine_query(query, results, evaluation)
           return results
   ```

2. **Search Result Evaluation**
   - LLM evaluates if search results answer the query
   - Identifies missing information
   - Assesses relevance of retrieved documents
   - Determines if additional searches are needed

---

## 5. Memory-R1: Enhancing Large Language Model Agents to Manage and Retrieve Memory

### Key Technical Points

1. **Memory Manager with CRUD Operations**
   - **Add**: Store new memories/information
   - **Update**: Modify existing memories when new information arrives
   - **Delete**: Remove outdated or incorrect memories
   - **Noop (No Operation)**: Decide not to modify memory when information is redundant or irrelevant

2. **Decision-Making for Memory Operations**
   - LLM decides which operation to perform based on new information
   - Compares new information with existing memories
   - Determines if information is new (add), updates existing (update), contradicts (delete/update), or redundant (noop)

3. **Memory State Management**
   - Maintains consistency of stored memories
   - Handles conflicts between old and new information
   - Prevents redundant storage
   - Ensures memory relevance and accuracy

4. **Practical Benefits**
   - Prevents memory bloat from redundant information
   - Keeps memories up-to-date automatically
   - Reduces storage costs
   - Improves retrieval quality by maintaining clean memory state

### Current IDA Implementation

✅ **Already Implemented:**
- `AgentMemoryService` exists with CRUD operations:
  - `add_object()` - adds new memory
  - `set_object()` - upsert (creates or updates)
  - `update_object()` - updates existing memory
  - `delete_object()` - deletes memory
  - `get_object()` - retrieves memory
  - `list_objects()` - lists memories
- Chat history management
- Context compression (`context_compressor.py`)
- Message bus for state management
- CheatSheet for updated memory across whole chat session.

❌ **Missing/Can Improve:**
- No LLM-based decision making layer for memory operations
- Callers must manually decide add/update/delete/noop
- No automatic comparison of new info with existing memories
- No conflict resolution for contradictory memories
- No redundancy detection before storing memories
- No intelligent decision about when to skip operations (noop)

### Recommendations for IDA

1. **Memory (ChatHistory) Manager with CRUD Operations**
   ```python
   class MemoryOperation(str, Enum):
       ADD = "add"      # Store new memory
       UPDATE = "update"  # Modify existing memory
       DELETE = "delete"  # Remove memory
       NOOP = "noop"    # No operation needed
   
   class MemoryManager:
       def decide_operation(
           self, 
           new_info: str, 
           existing_memories: List[Memory],
           context: Dict[str, Any]
       ) -> Tuple[MemoryOperation, Optional[Memory]]:
           """
           LLM decides which memory operation to perform.
           Returns operation type and target memory (if update/delete).
           """
           # Use LLM to compare new_info with existing_memories
           # Determine if: new (ADD), update existing (UPDATE), 
           # contradicts (DELETE/UPDATE), or redundant (NOOP)
           
       def add_memory(self, content: str, metadata: Dict) -> Memory:
           """Store new memory"""
           
       def update_memory(self, memory_id: str, new_content: str) -> Memory:
           """Update existing memory with new information"""
           
       def delete_memory(self, memory_id: str) -> bool:
           """Remove memory"""
           
       def noop(self, reason: str) -> None:
           """No operation - log reason for skipping"""
   ```

2. **LLM-Based Memory Decision Making**
   - Implement prompt for comparing new information with existing memories
   - Classify relationship: new, update, contradiction, redundant
   - Make operation decision based on classification
   - Handle edge cases (partial matches, conflicting information)

3. **Memory State Management**
   - Track memory versions and updates
   - Resolve conflicts when information contradicts
   - Merge related memories when appropriate
   - Maintain memory consistency across operations

4. **Integration with AgentMemoryService**
   - Create `MemoryManager` wrapper around existing `AgentMemoryService`
   - Add `smart_store()` method that decides operation automatically
   - Keep existing direct methods (`add_object`, `update_object`, etc.) for explicit control
   - Add memory operation logging and analytics
   - Example integration:
   ```python
   class MemoryManager:
       def __init__(self, memory_service: AgentMemoryService, llm_manager: LLMManager):
           self.memory_service = memory_service
           self.llm = llm_manager.get_llm(LLMModelType.FAST)
       
       def smart_store(
           self,
           agent_id: str,
           name: str,
           new_info: Dict[str, Any],
           scope: AgentMemoryScope,
           project_id: Optional[int] = None,
           org_id: Optional[int] = None,
       ) -> Tuple[MemoryOperation, Optional[AgentMemory]]:
           """
           Intelligently decide and execute memory operation.
           Returns operation performed and resulting memory.
           """
           # Get existing memory
           existing = self.memory_service.get_object(
               agent_id, name, scope, project_id, org_id
           )
           
           if not existing:
               # No existing memory - add it
               memory = self.memory_service.add_object(
                   agent_id, name, new_info, scope, project_id, org_id
               )
               return MemoryOperation.ADD, memory
           
           # Use LLM to decide operation
           decision = self._decide_operation(new_info, existing.object)
           
           if decision == MemoryOperation.NOOP:
               logger.info(f"Noop: {name} - {decision.reason}")
               return MemoryOperation.NOOP, existing
           elif decision == MemoryOperation.UPDATE:
               memory = self.memory_service.update_object(
                   existing.id, new_info
               )
               return MemoryOperation.UPDATE, memory
           elif decision == MemoryOperation.DELETE:
               self.memory_service.delete_object(existing.id)
               return MemoryOperation.DELETE, None
           else:  # ADD (shouldn't happen if existing)
               memory = self.memory_service.add_object(
                   agent_id, name, new_info, scope, project_id, org_id
               )
               return MemoryOperation.ADD, memory
   ```

5. **Data-Specific Memory (OpenAI Insight)**
   - Extend memory system to capture data query corrections and constraints
   - Automatically detect learning opportunities from query failures
   - Store non-obvious filters and constraints (e.g., "table X excludes test wells")
   - Scoped memory: global (all users), project-specific, personal preferences
   - Automatically suggest saving learnings when corrections occur
   - Impact: Memory system improves data query accuracy over time

---

## 6. ReasoningBank: Scaling Agent Reasoning

### Key Technical Points

1. **Reasoning Pattern Library**
   - Catalog of common reasoning patterns
   - Reusable reasoning templates
   - Pattern matching and application

2. **Scalable Reasoning**
   - Decompose complex reasoning into patterns
   - Compose patterns for complex tasks
   - Learn new patterns from successful reasoning

3. **Pattern-Based Agent Design**
   - Agents can leverage proven reasoning patterns
   - Faster development of new agents
   - Better consistency across agents

4. **Benchmark and Evaluation**
   - Standardized reasoning benchmarks
   - Pattern effectiveness metrics
   - Agent performance tracking

### Current IDA Implementation

✅ **Already Implemented:**
- Multiple specialized agents (SME, DataInsight, etc.)
- Agent factory pattern
- Tool-based reasoning chains
- Agent registry
- CheatSheet

❌ **Missing/Can Improve:**
- No reasoning pattern library
- No reusable reasoning templates
- Limited pattern-based agent composition
- No reasoning benchmark framework

### Recommendations for IDA

1. **Reasoning Pattern Library**
   ```python
   class ReasoningPattern:
       name: str
       description: str
       steps: List[ReasoningStep]
       tools: List[str]
       examples: List[Dict]
       
   class ReasoningPatternLibrary:
       patterns: Dict[str, ReasoningPattern]
       
       def match_pattern(self, task: str) -> List[ReasoningPattern]:
           # Find applicable patterns
           
       def compose_patterns(self, patterns: List[ReasoningPattern]) -> ReasoningChain:
           # Combine patterns for complex tasks
   ```

---

## 7. OpenAI's Production Data Agent: Lessons from Real-World Deployment

### Key Technical Points

1. **Multi-Layered Context System (6 Layers)**
   - **Layer 1: Table Usage**: Metadata, lineage, query history, common joins
   - **Layer 2: Human Annotations**: Expert descriptions, business meaning, known caveats
   - **Layer 3: Codex Enrichment**: Code-level understanding of how data is created
   - **Layer 4: Institutional Knowledge**: Slack, Google Docs, Notion for company context
   - **Layer 5: Memory**: Learned corrections and constraints (scoped: global/personal)
   - **Layer 6: Runtime Context**: Live queries for real-time validation
   - Context aggregated offline into embeddings, retrieved via RAG at query time

2. **Code-Level Data Understanding**
   - Analyzes codebase to understand how tables are created (not just schemas)
   - Extracts: data freshness, business intent, data scope, downstream usage
   - Automatically refreshes when code changes
   - Distinguishes similar tables (e.g., logged-in vs logged-out users)
   - Critical insight: "Meaning lives in code" - schemas alone aren't enough

3. **Self-Learning Memory System**
   - Automatically detects learning opportunities from corrections
   - Stores non-obvious constraints and filters
   - Scoped at global and personal levels
   - Suggests saving learnings proactively
   - Improves with every interaction

4. **Systematic Evaluation**
   - Uses Evals API for continuous quality monitoring
   - Curated question-answer pairs with "golden" SQL queries
   - Compares both SQL and results (not just string matching)
   - Catches regressions early (canaries in production)
   - Enables confident iteration

5. **Conversational & Iterative**
   - Handles both quick answers and deep exploration
   - Carries full context across turns
   - Allows mid-analysis redirection
   - Proactively asks clarifying questions
   - Self-validates intermediate results and self-corrects

6. **Workflows for Recurring Analyses**
   - Packages common analyses into reusable instruction sets
   - Examples: weekly business reports, table validations
   - Ensures consistency across users
   - Encodes best practices once

7. **Key Lessons Learned**
   - **Less is More**: Restricted overlapping tools to reduce ambiguity
   - **Guide the Goal, Not the Path**: Higher-level guidance, let model choose execution
   - **Meaning Lives in Code**: Code-level understanding is critical for data

### Current IDA Implementation

✅ **Already Implemented:**
- Data retrieval and parsing (DDR, multi-format files)
- RAG service with multiple search strategies
- AgentMemoryService with CRUD operations
- DataInsight toolbox for drilling data analysis
- Tool-based architecture with dynamic tool selection
- Answer comparison service for quality assessment

❌ **Missing/Can Improve:**
- No multi-layered context system for data
- No code-level data enrichment (Codex-style)
- Memory system not optimized for data corrections
- No systematic evaluation of data queries
- No workflow system for recurring analyses
- Limited self-validation and correction for data queries
- No institutional knowledge integration

### Recommendations for IDA

1. **Multi-Layered Context System for Data**
   ```python
   class DataContextService:
       """Aggregates 6 layers of context for data queries"""
       
       # Layer 1: Table Usage
       def get_table_metadata(self, table_name: str) -> Dict:
           """Schema, lineage, query history"""
           
       # Layer 2: Human Annotations
       def get_table_annotations(self, table_name: str) -> Dict:
           """Expert descriptions, caveats"""
           
       # Layer 3: Codex Enrichment
       def enrich_from_code(self, table_name: str) -> Dict:
           """Code-level understanding of data creation"""
           
       # Layer 4: Institutional Knowledge
       def search_institutional_knowledge(self, query: str) -> List[Dict]:
           """Search Slack, Docs, Notion"""
           
       # Layer 5: Memory
       def get_data_memory(self, table_name: str) -> List[Dict]:
           """Learned corrections and constraints"""
           
       # Layer 6: Runtime Context
       def inspect_table_live(self, table_name: str) -> Dict:
           """Live queries for validation"""
       
       def get_enriched_context(self, query: str) -> Dict:
           """Aggregate all layers via RAG"""
   ```

2. **Code-Level Data Understanding (Codex Enrichment)**
   - Use codebase search to find table creation/usage code
   - Parse data pipeline code (Spark, Python scripts)
   - Extract: freshness, scope, business logic, downstream usage
   - Store enriched metadata, refresh on code changes
   - Critical for distinguishing similar tables

3. **Data-Specific Memory System**
   - Extend `AgentMemoryService` with data-specific methods
   - Automatically detect learning opportunities from query failures
   - Store corrections and constraints with appropriate scope
   - Apply learned constraints automatically in future queries

4. **Systematic Evaluation for Data Queries**
   - Create evaluation framework for data queries
   - Curated question-answer pairs with expected SQL
   - Compare both SQL and results (not just strings)
   - Integrate with CI/CD as canaries
   - Use LLM grader for nuanced comparison

5. **Self-Validation & Correction**
   - Add validation nodes to data agent workflows
   - Detect anomalies (zero rows, suspicious nulls, etc.)
   - Self-correct queries based on error analysis
   - Iterative refinement until satisfactory result

6. **Workflow System**
   - Create `DataWorkflowService` for reusable analyses
   - Package common analyses (weekly reports, validations)
   - Ensure consistency across users
   - Encode best practices once

7. **Institutional Knowledge Integration**
   - Extend RAG to include Slack, Google Docs, Notion
   - Store with proper access control
   - Access canonical metric definitions, launch context, terminology

---

## Implementation Priority Recommendations

### High Priority (Immediate Impact)

1. **Enhanced Tool Documentation** (Paper 1)
   - Add structured documentation format
   - Include capabilities, limitations, examples
   - Impact: Better zero-shot tool usage, reduced need for demos

2. **Adaptive RAG with Self-Critique** (Paper 3)
   - Implement retrieval decision logic
   - Add response quality assessment
   - Impact: Better retrieval efficiency and response quality

3. **Iterative Search Refinement** (Paper 4)
   - Add ReZero-style query refinement
   - Implement search result evaluation
   - Impact: Improved search quality without training

4. **Multi-Layered Context for Data** (OpenAI Data Agent)
   - Build 6-layer context system (table usage, annotations, code, institutional, memory, runtime)
   - Implement Codex-style code-level data enrichment
   - Aggregate context via RAG for efficient retrieval
   - Impact: Dramatically improves data query accuracy and understanding

5. **Self-Validation & Correction** (OpenAI Data Agent + ReZero)
   - Add validation nodes to detect anomalies
   - Implement self-correction for data queries
   - Iterative refinement until satisfactory
   - Impact: Reduces errors, enables autonomous data exploration
6. **Reasoning Pattern Library** (Paper 6)
    - Create pattern catalog
    - Enable pattern-based agent composition
    - Impact: Faster agent development, better consistency


### Medium Priority (Significant Improvement)

7. **Contrastive Prompt Optimization** (Paper 2)
   - Track tool execution success/failure
   - Generate optimized prompts automatically
   - Impact: Better tool usage patterns, reduced manual tuning

8. **Data-Specific Memory System** (Paper 5 + OpenAI Data Agent)
   - Implement LLM-based memory decision making (add/update/delete/noop)
   - Extend to data corrections and constraints
   - Automatic learning detection from query failures
   - Scoped memory (global, project, personal)
   - Impact: Memory improves data query accuracy over time

9. **Systematic Evaluation for Data** (OpenAI Data Agent)
   - Create evaluation framework for data queries
   - Compare SQL and results (not just strings)
   - Integrate with CI/CD as canaries
   - Impact: Continuous quality monitoring, early regression detection

10. **Workflow System** (OpenAI Data Agent)
   - Create reusable analysis workflows
   - Package common analyses
   - Impact: Consistency, faster routine work

### Low Priority (Long-term Enhancement)

11. **Institutional Knowledge Integration** (OpenAI Data Agent)
    - Extend RAG to Slack, Google Docs, Notion
    - Access control and caching
    - Impact: Better company context understanding

---

## Integration Points in IDA Architecture

### Current Architecture Strengths

1. **Tool Management System**
   - Well-structured toolbox architecture
   - Good separation of concerns
   - Flexible tool registration

2. **RAG Service**
   - Multiple search strategies
   - Good re-ranking mechanism
   - Source attribution

3. **Agent Factory**
   - Flexible agent creation
   - Good dependency injection
   - Extensible design

### Recommended Integration Points

1. **Execution Analytics Service**
   - New service: `ToolExecutionAnalytics`
   - Integrate with `LLMToolsManager._execute_tool()`
   - Feed into prompt optimizer

2. **Adaptive RAG Enhancement**
   - Enhance `RagService.prompt_llm()`
   - Add retrieval decision logic
   - Add self-critique mechanisms

3. **Memory System Enhancement**
   - Enhance `AgentMemoryService`
   - Add hierarchical memory structure
   - Implement retrieval strategies

4. **Data Context Service** (OpenAI Data Agent)
   - New service: `DataContextService`
   - Integrate with `DDRManager`, `RagService`, `AgentMemoryService`
   - Build 6-layer context system
   - Implement Codex-style code analysis

5. **Codex Data Enrichment Service** (OpenAI Data Agent)
   - New service: `CodexDataEnrichmentService`
   - Analyze codebase for data understanding
   - Parse data pipeline code
   - Store enriched metadata

6. **Data Query Evaluator** (OpenAI Data Agent)
   - New service: `DataQueryEvaluator`
   - Extend `AnswerComparisonService`
   - Compare SQL and results
   - Integrate with CI/CD

7. **Data Workflow Service** (OpenAI Data Agent)
   - New service: `DataWorkflowService`
   - Manage reusable analysis workflows
   - Package common analyses

8. **Self-Validation Nodes** (OpenAI Data Agent)
    - Enhance `DataInsightAgent` workflow
    - Add validation and self-correction nodes
    - Implement iterative refinement

---

## Conclusion

The papers and OpenAI's production data agent provide valuable insights for enhancing IDA's capabilities in tool usage, RAG, search, memory management, reasoning, and data handling. The recommendations focus on practical improvements that can be integrated into IDA's existing architecture without major refactoring.

Key themes across papers and production systems:
- **Zero-shot capabilities**: Reduce reliance on demonstrations
- **Self-improvement**: Learn from execution results and corrections
- **Adaptive behavior**: Make decisions based on context
- **Quality control**: Critique and refine outputs
- **Scalability**: Handle large tool sets and complex tasks
- **Rich context**: Multi-layered context systems and memory (reasoning, strategies) bank dramatically improve accuracy
- **Code-level understanding**: Meaning lives in code, not just schemas
- **Systematic evaluation**: Continuous quality monitoring prevents regressions


These can be implemented incrementally and will provide immediate benefits to IDA's agent capabilities, especially for data handling and analysis tasks.
