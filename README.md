# SME Agent Test Workflow

## Mermaid Workflow Diagram

```mermaid
graph TD
    A[Start SME Test Pipeline] --> B[Load Configuration]
    B --> C[Initialize LLM Manager]
    C --> D[Load Test Cases from JSON]
    D --> E[Initialize Multi-Evaluator System]
    
    E --> F[General Quality Evaluator<br/>Weight: 30%]
    E --> G[Technical Accuracy Evaluator<br/>Weight: 25%]
    E --> H[RAG Evaluation Evaluator<br/>Weight: 20%]
    E --> I[Semantic Analysis Evaluator<br/>Weight: 15%]
    E --> J[Coherence Assessment Evaluator<br/>Weight: 10%]
    
    F --> K[Create SME Agent Instance]
    G --> K
    H --> K
    I --> K
    J --> K
    
    K --> L[For Each Test Case]
    L --> M[Extract Test Data]
    M --> N[Query: question]
    M --> O[Expected Answer]
    M --> P[Expected Route]
    M --> Q[Context]
    
    N --> R[Test Routing Decision]
    R --> S[Get Actual Route]
    S --> T[Generate Mock Response]
    
    T --> U[Multi-Evaluator Assessment]
    U --> V[General Quality Evaluation]
    U --> W[Technical Accuracy Evaluation]
    U --> X[RAG Evaluation]
    U --> Y[Semantic Analysis]
    U --> Z[Coherence Assessment]
    
    V --> AA[Weighted Aggregation]
    W --> AA
    X --> AA
    Y --> AA
    Z --> AA
    
    AA --> BB[Calculate Final Metrics]
    BB --> CC[Determine Pass/Fail]
    CC --> DD[Store Test Result]
    
    DD --> EE{More Test Cases?}
    EE -->|Yes| L
    EE -->|No| FF[Generate Comprehensive Report]
    
    FF --> GG[Calculate Statistics]
    GG --> HH[Route Breakdown Analysis]
    HH --> II[Individual Evaluator Results]
    II --> JJ[Consensus Analysis]
    JJ --> KK[Performance Metrics]
    
    KK --> LL[Save Results to JSON]
    LL --> MM[Print Formatted Report]
    MM --> NN[End Test Pipeline]
    
    %% Styling
    classDef startEnd fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef process fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef evaluator fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef report fill:#fce4ec,stroke:#880e4f,stroke-width:2px
    
    class A,NN startEnd
    class B,C,D,K,L,M,N,O,P,Q,R,S,T,AA,BB,CC,DD,FF,GG,HH,II,JJ,KK,LL,MM process
    class F,G,H,I,J,V,W,X,Y,Z evaluator
    class EE decision
    class FF,GG,HH,II,JJ,KK,LL,MM report
```

## Detailed Workflow Description

### Phase 1: Initialization
1. **Start SME Test Pipeline**: Begin automated testing process
2. **Load Configuration**: Load LLM and evaluation settings from config.yaml
3. **Initialize LLM Manager**: Set up Azure OpenAI connections
4. **Load Test Cases**: Read predefined Q&A pairs from JSON file
5. **Initialize Multi-Evaluator System**: Set up 5 specialized evaluators

### Phase 2: Evaluator Setup
- **General Quality Evaluator** (30% weight): Assesses correctness, completeness, relevancy, coherence
- **Technical Accuracy Evaluator** (25% weight): Validates oil & gas domain knowledge
- **RAG Evaluation Evaluator** (20% weight): Measures context precision, recall, faithfulness
- **Semantic Analysis Evaluator** (15% weight): Analyzes meaning similarity and concept alignment
- **Coherence Assessment Evaluator** (10% weight): Evaluates structure and readability

### Phase 3: Test Execution
1. **Create SME Agent**: Initialize agent with mocked dependencies
2. **For Each Test Case**: Iterate through all test cases
3. **Extract Test Data**: Parse question, expected answer, route, and context
4. **Test Routing**: Determine which route the agent would take
5. **Generate Response**: Create mock response based on routing decision
6. **Multi-Evaluator Assessment**: Run all 5 evaluators in parallel

### Phase 4: Evaluation & Aggregation
1. **Individual Evaluations**: Each evaluator provides specialized assessment
2. **Weighted Aggregation**: Combine results using configurable weights
3. **Calculate Final Metrics**: Compute overall scores and confidence levels
4. **Determine Pass/Fail**: Apply thresholds to determine test success
5. **Store Results**: Save individual test results

### Phase 5: Reporting
1. **Generate Comprehensive Report**: Create detailed analysis
2. **Calculate Statistics**: Compute averages, success rates, trends
3. **Route Breakdown**: Analyze performance by routing category
4. **Individual Evaluator Results**: Show detailed evaluator assessments
5. **Consensus Analysis**: Measure agreement between evaluators
6. **Performance Metrics**: Track execution times and efficiency
7. **Save & Display**: Export to JSON and print formatted report

## Key Features

- **Parallel Processing**: Multiple evaluators run simultaneously
- **Weighted Scoring**: Configurable importance for different evaluation aspects
- **Comprehensive Metrics**: 10+ evaluation dimensions
- **Robust Error Handling**: Fallback mechanisms for evaluator failures
- **Detailed Reporting**: Complete audit trail and analysis
- **Scientific Rigor**: Statistically sound aggregation methods
