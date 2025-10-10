# SME Agent Testing Framework Workflow

## Mermaid Workflow Diagram

```mermaid
graph TD
    A[Start SME Testing Framework] --> B[Phase 1: Build Dataset]
    
    B --> C[Create 100 Test Cases]
    C --> D[RAG Cases: 35%<br/>File Cases: 35%<br/>General Cases: 20%<br/>Meta Cases: 10%]
    
    D --> E[For RAG/General/Meta Cases]
    E --> F[Ask Ida Directly]
    F --> G[Human-in-Loop Review]
    G --> H[Modify Answers if Needed]
    H --> I[Wrap in JSON Format]
    
    D --> J[For File Cases]
    J --> K[Upload Specific File<br/>e.g., NNM101.pdf]
    K --> L[Ask Ida File-Related Questions]
    L --> M[Human-in-Loop Review]
    M --> N[Modify Answers if Needed]
    N --> O[Wrap in JSON Format]
    
    I --> P[Phase 2: Design Metrics]
    O --> P
    
    P --> Q[Agent Functionality Metrics]
    Q --> R[Context Precision RAG]
    Q --> S[Context Recall RAG]
    Q --> T[Faithfulness RAG/File]
    Q --> U[Semantic Similarity All]
    Q --> V[Answer Correctness All]
    Q --> W[Answer Completeness All]
    Q --> X[Answer Relevancy All]
    Q --> Y[Overall Score Weighted]
    
    P --> Z[Agent Routing Metrics]
    Z --> AA[Routing Accuracy<br/>Correct Routes / Total Routes]
    
    Y --> BB[Phase 3: Design Evaluators]
    AA --> BB
    
    BB --> CC[Multi-LLM Evaluator System]
    CC --> DD[General Quality Evaluator<br/>Weight: 30%]
    CC --> EE[Technical Accuracy Evaluator<br/>Weight: 25%]
    CC --> FF[RAG Evaluation Evaluator<br/>Weight: 20%]
    CC --> GG[Semantic Analysis Evaluator<br/>Weight: 15%]
    CC --> HH[Coherence Assessment Evaluator<br/>Weight: 10%]
    
    DD --> II[Phase 4: Build Automatic Test Workflow]
    EE --> II
    FF --> II
    GG --> II
    HH --> II
    
    II --> JJ[Load Test Cases from JSON]
    JJ --> KK[Create SME Agent Instance]
    KK --> LL[Initialize Answer Comparison Service]
    
    LL --> MM[For Each Test Case Loop]
    MM --> NN[Extract Test Data]
    NN --> OO[Query Question]
    NN --> PP[Expected Answer]
    NN --> QQ[Expected Route]
    NN --> RR[Context Information]
    
    OO --> SS[Test Routing Decision]
    SS --> TT[Get Actual Route]
    TT --> UU[Generate Response]
    
    UU --> VV[Multi-Evaluator Assessment]
    VV --> WW[General Quality Evaluation<br/>Correctness, Completeness, Relevancy]
    VV --> XX[Technical Accuracy Evaluation<br/>Domain Knowledge, Terminology]
    VV --> YY[RAG Evaluation<br/>Context Precision, Recall, Faithfulness]
    VV --> ZZ[Semantic Analysis<br/>Similarity, Meaning Preservation]
    VV --> AAA[Coherence Assessment<br/>Flow, Clarity, Organization]
    
    WW --> BBB[Weighted Aggregation]
    XX --> BBB
    YY --> BBB
    ZZ --> BBB
    AAA --> BBB
    
    BBB --> CCC[Calculate Final Metrics]
    CCC --> DDD[Determine Pass/Fail]
    DDD --> EEE[Store Test Result]
    
    EEE --> FFF{More Test Cases?}
    FFF -->|Yes| MM
    FFF -->|No| GGG[Generate Comprehensive Report]
    
    GGG --> HHH[Calculate Statistics]
    HHH --> III[Route Accuracy Analysis]
    HHH --> JJJ[Answer Quality Analysis]
    HHH --> KKK[Individual Evaluator Results]
    HHH --> LLL[Consensus Analysis]
    HHH --> MMM[Performance Metrics]
    
    III --> NNN[Save Results to JSON]
    JJJ --> NNN
    KKK --> NNN
    LLL --> NNN
    MMM --> NNN
    
    NNN --> OOO[Print Formatted Report]
    OOO --> PPP[Phase 5: Further Improvement]
    
    PPP --> QQQ[Continuous Enhancement Strategies]
    QQQ --> RRR[End Testing Framework]
    
    %% Styling
    classDef startEnd fill:#e1f5fe,stroke:#01579b,stroke-width:3px
    classDef phase fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef dataset fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef metrics fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef evaluator fill:#e3f2fd,stroke:#0277bd,stroke-width:2px
    classDef workflow fill:#fce4ec,stroke:#880e4f,stroke-width:2px
    classDef decision fill:#fff8e1,stroke:#f57c00,stroke-width:2px
    classDef report fill:#f1f8e9,stroke:#33691e,stroke-width:2px
    
    class A,RRR startEnd
    class B,P,BB,II,PPP phase
    class C,D,E,F,G,H,I,J,K,L,M,N,O dataset
    class Q,R,S,T,U,V,W,X,Y,Z,AA metrics
    class CC,DD,EE,FF,GG,HH,WW,XX,YY,ZZ,AAA evaluator
    class FFF decision
    class GGG,HHH,III,JJJ,KKK,LLL,MMM,NNN,OOO,QQQ report
```

## Detailed Workflow Description

### Phase 1: Build Dataset (100 Test Cases)
**Purpose**: Create comprehensive test dataset covering all SME agent capabilities

**Test Case Distribution**:
- **RAG Cases (35%)**: Domain knowledge questions requiring retrieval from knowledge base
- **File Cases (35%)**: Questions about uploaded documents requiring file interpretation
- **General Cases (20%)**: General knowledge questions not requiring domain expertise
- **Meta Cases (10%)**: Questions about Ida's identity, capabilities, and role

**Dataset Creation Process**:
1. **For RAG/General/Meta Cases**:
   - Ask Ida directly to get initial answers
   - Human-in-loop review and validation
   - Modify answers if needed for accuracy
   - Wrap in standardized JSON format

2. **For File Cases**:
   - Upload specific files (e.g., NNM101.pdf)
   - Ask Ida file-related questions
   - Human-in-loop review and validation
   - Modify answers if needed for accuracy
   - Wrap in standardized JSON format

### Phase 2: Design Metrics
**Purpose**: Define comprehensive evaluation criteria for both functionality and routing

**Agent Functionality Metrics**:
- **Context Precision (RAG)**: Relevance of retrieved context chunks
- **Context Recall (RAG)**: Completeness of context retrieval
- **Faithfulness (RAG/File)**: Grounding in provided context without hallucination
- **Semantic Similarity (All)**: Cosine similarity between generated and expected answers
- **Answer Correctness (All)**: Factual accuracy and correctness
- **Answer Completeness (All)**: Thoroughness in addressing all question aspects
- **Answer Relevancy (All)**: Directness in addressing the specific question
- **Overall Score**: Weighted average of all metrics

**Agent Routing Metrics**:
- **Routing Accuracy**: Correct routing decisions / Total routing decisions

### Phase 3: Design Evaluators
**Purpose**: Implement multi-LLM evaluation system with specialized evaluators

**Multi-LLM Evaluator System**:
- **General Quality Evaluator** (30% weight): Correctness, completeness, relevancy, coherence
- **Technical Accuracy Evaluator** (25% weight): Oil & gas domain knowledge and terminology
- **RAG Evaluation Evaluator** (20% weight): Context precision, recall, faithfulness
- **Semantic Analysis Evaluator** (15% weight): Meaning similarity and concept alignment
- **Coherence Assessment Evaluator** (10% weight): Structure, clarity, organization

**Key Features**:
- Each evaluator uses specialized LLM models
- Weighted aggregation of results
- Confidence scoring for each evaluation
- Detailed reasoning for audit trail

### Phase 4: Build Automatic Test Workflow
**Purpose**: Implement automated testing pipeline with comprehensive evaluation

**Test Execution Process**:
1. **Load Test Cases**: Read 100 test cases from JSON
2. **Create SME Agent**: Initialize agent with mocked dependencies
3. **Initialize Answer Comparison Service**: Set up multi-evaluator system
4. **For Each Test Case**:
   - Extract test data (question, expected answer, route, context)
   - Test routing decision accuracy
   - Generate response
   - Run multi-evaluator assessment
   - Calculate weighted metrics
   - Determine pass/fail status
   - Store results

### Phase 5: Further Improvement
**Purpose**: Continuous enhancement based on test results

**Enhancement Strategies**:
- Analyze test results for patterns and weaknesses
- Refine evaluator weights based on performance
- Update test cases based on new requirements
- Improve SME agent based on identified issues
- Iterate on evaluation criteria and thresholds

## Key Features

- **Comprehensive Coverage**: Tests all SME agent capabilities (RAG, File, General, Meta)
- **Human-in-Loop Validation**: Ensures high-quality test dataset
- **Multi-LLM Evaluation**: Specialized evaluators for different aspects
- **Weighted Aggregation**: Scientifically sound result combination
- **Detailed Reporting**: Complete analysis and audit trail
- **Continuous Improvement**: Framework for ongoing enhancement
- **Robust Error Handling**: Fallback mechanisms for reliability
