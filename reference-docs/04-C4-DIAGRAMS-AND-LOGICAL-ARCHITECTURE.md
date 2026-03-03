# C4 Diagrams and Logical Architecture — FIRE-Bench

This document contains C4 model diagrams (Context, Container, Component, Code) and logical architecture diagrams for the FIRE-Bench platform, all rendered in Mermaid format.

---

## 1. C4 Level 1 — System Context Diagram

Shows FIRE-Bench in the context of its external actors and systems.

```mermaid
graph TB
    Researcher([Researcher or ML Engineer<br>Runs evaluations and analyzes results])
    
    subgraph FIRE_Bench_System[FIRE_Bench Evaluation Platform]
        Core[FIRE_Bench Core<br>Python CLI Application]
    end

    LLM_Service([LLM API Service<br>OpenAI compatible endpoint<br>e.g. GPT4 vLLM DeepSeek])
    Judge_Service([Judge Model Service<br>FIRE_RM Reward Model<br>Scores open ended responses])
    FileSystem([Local Filesystem<br>Datasets configs<br>results and caches])

    Researcher -->|Executes CLI commands<br>Configures evaluations| Core
    Core -->|Sends prompts<br>Receives predictions| LLM_Service
    Core -->|Sends scoring requests<br>Receives 1_5 scores| Judge_Service
    Core -->|Reads datasets and configs<br>Writes results and caches| FileSystem
    Core -->|Displays metrics| Researcher
```

---

## 2. C4 Level 2 — Container Diagram

Shows the major containers (executable units and data stores) within the system.

```mermaid
graph TB
    subgraph FIRE_Bench_Platform[FIRE_Bench Platform]
        CLI_App[CLI Application<br>Python with argparse<br>Entry point and argument parsing]
        Pipeline_Engine[Pipeline Engine<br>Python asyncio<br>Orchestrates evaluation workflow]
        Model_Client_Container[Model Client<br>Python with openai SDK<br>Async API communication]
        Evaluator_Container[Evaluator Engine<br>Python<br>Scoring and metrics computation]
        Config_Container[Configuration Manager<br>Python with PyYAML<br>YAML parsing and validation]
        Utils_Container[Utilities<br>Python<br>Logging and path management]
    end

    subgraph Data_Stores[Data Stores]
        Dataset_Store[(Dataset Files<br>JSON JSONL CSV<br>Excel Parquet)]
        Config_Store[(YAML Configuration<br>datasets.yaml)]
        Cache_Store[(Cache Files<br>JSONL format<br>Prompt response pairs)]
        Results_Store[(Result Files<br>JSON format<br>Metrics and details)]
    end

    subgraph External[External Services]
        LLM_API[LLM API<br>OpenAI Compatible]
        Judge_API[Judge Model API<br>FIRE_RM]
    end

    CLI_App --> Pipeline_Engine
    CLI_App --> Config_Container
    Pipeline_Engine --> Model_Client_Container
    Pipeline_Engine --> Evaluator_Container
    Pipeline_Engine --> Config_Container
    Evaluator_Container --> Model_Client_Container

    Config_Container --> Config_Store
    Pipeline_Engine --> Dataset_Store
    Pipeline_Engine --> Cache_Store
    Pipeline_Engine --> Results_Store
    Model_Client_Container --> LLM_API
    Evaluator_Container --> Judge_API

    CLI_App --> Utils_Container
    Pipeline_Engine --> Utils_Container
```

---

## 3. C4 Level 3 — Component Diagram

Shows the internal components of the Pipeline Engine and Evaluator Engine.

### 3.1 Pipeline Engine Components

```mermaid
graph TB
    subgraph Pipeline_Engine[Pipeline Engine]
        EP[EvaluationPipeline<br>Main orchestrator]
        DL[DatasetLoader<br>Multi format file reader]
        MCF[ModelClientFactory<br>Client constructor]
        PMR[process_model_response<br>Think tag stripper]
        Cache[Cache Manager<br>JSONL read write]
        ResultSaver[Result Saver<br>JSON serialization]
    end

    EP --> DL
    EP --> MCF
    EP --> PMR
    EP --> Cache
    EP --> ResultSaver
```

### 3.2 Evaluator Engine Components

```mermaid
graph TB
    subgraph Evaluator_Engine[Evaluator Engine]
        EM[EvaluatorManager<br>Registry with factory dict]
        FIRE_Adapter[FIREMCQEvaluatorAdapter<br>Registered as fire]
        Scene_Adapter[FireSceneEvaluatorAdapter<br>Registered as fire_scene]
    end

    subgraph MCQ_Evaluator[FIRE MCQ Evaluator]
        MCQ[FIREMCQEvaluator<br>Multiple choice scorer]
        ChoiceExtractor[Choice Extraction<br>Regex based answer parser]
        SubtaskStats[Subtask Statistics<br>Per benchmark accuracy]
    end

    subgraph Scene_Evaluator[FIRE Scene Evaluator]
        FSE[FireSceneEvaluator<br>Hybrid evaluation engine]
        STD[SampleTypeDetector<br>Principle vs rule classifier]
        RE[RuleEvaluator<br>Deterministic JSON comparison]
        SA[SceneAggregator<br>Hierarchical score aggregation]
        JM[Judge Model Client<br>FIRE_RM scorer]
        SP[Score Parser<br>JSON score extraction]
    end

    EM --> FIRE_Adapter
    EM --> Scene_Adapter
    FIRE_Adapter --> MCQ
    MCQ --> ChoiceExtractor
    MCQ --> SubtaskStats
    Scene_Adapter --> FSE
    FSE --> STD
    FSE --> RE
    FSE --> SA
    FSE --> JM
    FSE --> SP
```

---

## 4. C4 Level 4 — Code Level Diagram

### 4.1 Model Client Internal Architecture

```mermaid
graph TB
    subgraph OpenAIModelClient[OpenAIModelClient]
        Config_Ref[BaseModelConfig reference]
        ClientPool[AsyncOpenAI Client Pool<br>One client per URL]
        Semaphore[asyncio Semaphore<br>Concurrency limiter]
        CacheLock[asyncio Lock<br>Cache write serializer]
        UsageLock[asyncio Lock<br>Token counter serializer]
        TokenCounter[Token Usage Counters<br>Input and output tokens]
    end

    subgraph Request_Flow[Request Processing]
        GenBatch[generate_batch<br>Creates N async tasks]
        RunOne[run_one<br>Acquires semaphore then calls API]
        MakeReq[_make_async_request<br>Builds and sends API request]
        StreamHandler[Stream Handler<br>Collects SSE chunks]
        ProgressCB[Progress Callback<br>Updates tqdm and writes cache]
    end

    GenBatch --> RunOne
    RunOne --> Semaphore
    RunOne --> MakeReq
    MakeReq --> ClientPool
    MakeReq --> StreamHandler
    StreamHandler --> ProgressCB
    ProgressCB --> CacheLock
    MakeReq --> UsageLock
    MakeReq --> TokenCounter
```

---

## 5. Logical Architecture Diagram

Shows the logical grouping of all system capabilities.

```mermaid
graph TB
    subgraph User_Interface[User Interface Layer]
        CLI[Command Line Interface]
        QuickStart[Quick Start Guide]
        DatasetLister[Dataset Listing]
    end

    subgraph Application_Logic[Application Logic Layer]
        direction TB
        subgraph Workflow_Management[Workflow Management]
            PipelineOrch[Pipeline Orchestration]
            CacheManagement[Cache and Resume Management]
            ResultPersistence[Result Persistence]
        end

        subgraph Data_Processing[Data Processing]
            MultiFormatLoading[Multi Format Data Loading]
            EncodingDetection[Encoding Detection]
            DataNormalization[Data Normalization]
        end

        subgraph LLM_Communication[LLM Communication]
            AsyncBatchProcessing[Async Batch Processing]
            LoadBalancing[Multi URL Load Balancing]
            StreamingSupport[Streaming Response Support]
            TokenTracking[Token Usage Tracking]
        end

        subgraph Evaluation_Logic[Evaluation Logic]
            MCQScoring[MCQ Answer Extraction and Scoring]
            RuleBasedJudging[Rule Based JSON Comparison]
            LLMAsJudge[LLM as Judge Principle Scoring]
            ThinkTagStripping[Think Tag Response Processing]
            SceneAggregation[Hierarchical Scene Aggregation]
        end
    end

    subgraph Cross_Cutting[Cross Cutting Concerns]
        ConfigManagement[YAML Configuration Management]
        PathResolution[Project Path Resolution]
        LoggingInfra[Structured Logging]
        ErrorHandling[Graceful Error Handling]
    end

    User_Interface --> Application_Logic
    Application_Logic --> Cross_Cutting
```

---

## 6. Data Flow Diagram

Shows how data flows through the system during a complete evaluation run.

```mermaid
flowchart LR
    subgraph Input[Inputs]
        YAML[datasets.yaml<br>Configuration]
        DS_Files[Dataset Files<br>JSON JSONL CSV etc]
        CLI_Args[CLI Arguments<br>URLs keys datasets]
        Prompt_Template[Prompt Template<br>Python file]
    end

    subgraph Processing[Processing Pipeline]
        Parse[Parse Config<br>and CLI Args]
        Load[Load and Normalize<br>Dataset Samples]
        Prompt[Extract and Format<br>Prompts]
        CacheCheck[Check Response<br>Cache]
        LLMCall[Call LLM API<br>For Uncached Prompts]
        StripThink[Strip Think Tags<br>From Responses]
        Evaluate[Run Evaluator<br>MCQ or Scene]
        Aggregate[Aggregate Metrics<br>and Scores]
    end

    subgraph Output[Outputs]
        MetricsSummary[metrics_summary.json<br>Per dataset metrics]
        DetailedResults[model_dataset.json<br>Per sample results]
        CacheFile[cache json<br>Prompt response pairs]
        ConsoleOutput[Console Output<br>Summary display]
    end

    YAML --> Parse
    CLI_Args --> Parse
    DS_Files --> Load
    Prompt_Template --> Prompt
    Parse --> Load
    Load --> Prompt
    Prompt --> CacheCheck
    CacheCheck -->|Cache miss| LLMCall
    CacheCheck -->|Cache hit| StripThink
    LLMCall --> StripThink
    LLMCall --> CacheFile
    StripThink --> Evaluate
    Evaluate --> Aggregate
    Aggregate --> MetricsSummary
    Aggregate --> DetailedResults
    Aggregate --> ConsoleOutput
```

---

## 7. Evaluation Decision Flow

Shows the branching logic during the evaluation of a single sample.

```mermaid
flowchart TD
    Start([Receive sample with prediction]) --> CheckEmpty{Is prediction<br>empty?}
    CheckEmpty -->|Yes| SetNull[Score = None<br>Skip evaluation]
    CheckEmpty -->|No| DetectType[SampleTypeDetector<br>determine_sample_type]

    DetectType --> TypeCheck{Sample type?}

    TypeCheck -->|principle| FormatJudge[Format judge prompt<br>with instruction ref_answer<br>principle and response]
    TypeCheck -->|rule| ExtractJSON[Extract JSON from<br>model response]
    TypeCheck -->|unknown| SetNull2[Score = None<br>Log warning]

    FormatJudge --> SendJudge[Send to FIRE_RM<br>judge model]
    SendJudge --> ParseScore[Parse score from<br>judge JSON output]
    ParseScore --> ScoreValid{Score parsed<br>successfully?}
    ScoreValid -->|Yes| Normalize[Normalize 1_5 to 0_100]
    ScoreValid -->|No| SetNull3[Score = None<br>Count as failure]

    ExtractJSON --> RouteJudge[Route to specific<br>judge function by<br>data_source keyword]
    RouteJudge --> CompareFields[Compare JSON fields<br>with ground truth]
    CompareFields --> RuleResult{Match?}
    RuleResult -->|Yes| Score100[Score = 100]
    RuleResult -->|No| Score0[Score = 0]

    SetNull --> Aggregate[Add to results]
    SetNull2 --> Aggregate
    SetNull3 --> Aggregate
    Normalize --> Aggregate
    Score100 --> Aggregate
    Score0 --> Aggregate
```

---

## 8. File Structure Diagram

```mermaid
graph TD
    Root[FIRE_Bench Project Root]
    Root --> run_eval[run_evaluation.py<br>Entry point]
    Root --> requirements[requirements.txt<br>Dependencies]
    Root --> config_dir[config directory]
    Root --> dataset_dir[dataset directory]
    Root --> src_dir[src directory]
    Root --> results_dir[results directory<br>Generated at runtime]

    config_dir --> datasets_yaml[datasets.yaml<br>Dataset and default configs]

    dataset_dir --> fire_dir[FIRE directory]
    dataset_dir --> fire_scene_dir[FIRE_SCENE directory]
    fire_dir --> fire_json[FIRE.json<br>14000 plus MCQ questions]
    fire_scene_dir --> prompt_py[prompt.py<br>Judge prompt template]
    fire_scene_dir --> principle_dir[principle directory]
    principle_dir --> scene_jsonl[scene_principle.release.jsonl<br>Scene evaluation data]

    src_dir --> core_dir[core directory]
    src_dir --> utils_dir[utils directory]

    core_dir --> base_py[base.py<br>Abstract classes and data models]
    core_dir --> pipeline_py[pipeline.py<br>Evaluation orchestrator]
    core_dir --> model_client_py[model_client.py<br>Async OpenAI client]
    core_dir --> dataset_loader_py[dataset_loader.py<br>Multi format data reader]
    core_dir --> evaluator_dir[evaluator directory]

    evaluator_dir --> evaluator_py[evaluator.py<br>Registry and adapters]
    evaluator_dir --> fire_evaluator_py[fire_evaluator.py<br>MCQ scoring logic]
    evaluator_dir --> fire_scene_py[fire_scene_evaluator.py<br>Scene evaluation logic]

    utils_dir --> cli_py[cli.py<br>CLI runner]
    utils_dir --> config_py[config.py<br>Config manager]
    utils_dir --> path_manager_py[path_manager.py<br>Path resolution]
    utils_dir --> logging_py[logging_config.py<br>Logging setup]

    results_dir --> run_folder[model_dataset_timestamp<br>Per run output folder]
    run_folder --> metrics_json[metrics_summary.json]
    run_folder --> detail_json[model_dataset.json]
    run_folder --> cache_dir[cache directory]
    run_folder --> irm_cache_dir[irm_cache directory]
    cache_dir --> cache_jsonl[model_dataset.json<br>JSONL cache]
    irm_cache_dir --> irm_jsonl[judge_model.json<br>JSONL judge cache]
```
