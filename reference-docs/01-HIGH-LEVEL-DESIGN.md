# High-Level Design (HLD) — FIRE-Bench Evaluation Platform

## 1. Executive Summary

**FIRE-Bench** (Financial Intelligence and Reasoning Evaluation) is an automated benchmarking platform designed to evaluate the financial reasoning capabilities of Large Language Models (LLMs). Built as a collaboration between Duxiaoman Technology (Beijing), Tsinghua University PBC School of Finance, and Renmin University School of Finance, FIRE-Bench combines standardized financial certification exam questions (14,000+) with real-world financial scenario tasks (3,000+) to provide a comprehensive assessment of LLM performance in the financial domain.

The system follows a pipeline architecture that loads datasets, sends prompts to LLMs via OpenAI-compatible APIs, collects responses, evaluates them against ground truth or rubric-based criteria, and produces structured performance reports.

---

## 2. Business Goals and Objectives

| Goal | Description |
|------|-------------|
| **Comprehensive Financial Evaluation** | Assess LLMs across 14 professional financial certification exams (CFA, FRM, CPA, etc.) and 17 real-world financial business scenarios |
| **Dual Evaluation Strategy** | Support both deterministic (multiple-choice) and open-ended (principle-based rubric) evaluation modes |
| **Scalable Benchmarking** | Enable parallel, high-throughput evaluation of models against large datasets with caching and resume capabilities |
| **Reproducibility** | Ensure consistent, repeatable evaluation through configuration-driven design and result persistence |
| **Automated Scoring** | Provide end-to-end automated scoring including LLM-as-a-Judge for open-ended questions |

---

## 3. System Context

```mermaid
graph TB
    subgraph External_Systems
        LLM_API[LLM Model API<br>OpenAI Compatible]
        Judge_API[Judge Model API<br>FIRE_RM Scoring Model]
    end

    subgraph FIRE_Bench_Platform
        CLI[CLI Entry Point]
        Pipeline[Evaluation Pipeline]
        DataLoader[Dataset Loader]
        ModelClient[Model Client]
        Evaluators[Evaluator Engine]
        ResultStore[Result Storage]
    end

    subgraph Data_Assets
        FIRE_Dataset[FIRE MCQ Dataset<br>14000 plus questions]
        FIRE_SCENE_Dataset[FIRE SCENE Dataset<br>3000 plus scenario tasks]
        Config[YAML Configuration]
    end

    User([Researcher or Engineer]) -->|CLI Commands| CLI
    CLI --> Pipeline
    Pipeline --> DataLoader
    Pipeline --> ModelClient
    Pipeline --> Evaluators
    Pipeline --> ResultStore

    DataLoader --> FIRE_Dataset
    DataLoader --> FIRE_SCENE_Dataset
    Pipeline --> Config

    ModelClient -->|Async HTTP| LLM_API
    Evaluators -->|Async HTTP| Judge_API

    ResultStore -->|JSON Files| Disk[(Local Filesystem)]
```

---

## 4. High-Level Architecture

The platform is organized in a layered architecture with clear separation of concerns:

```mermaid
graph TB
    subgraph Presentation_Layer[Presentation Layer]
        CLI_Runner[CLI Runner<br>Argument Parsing]
        Quick_Start[Quick Start Guide]
    end

    subgraph Orchestration_Layer[Orchestration Layer]
        Eval_Pipeline[Evaluation Pipeline<br>Workflow Coordinator]
        Config_Manager[Configuration Manager<br>YAML Parser]
    end

    subgraph Core_Layer[Core Business Logic Layer]
        Dataset_Loader[Dataset Loader<br>Multi Format Reader]
        Model_Client[OpenAI Model Client<br>Async HTTP with Load Balancing]
        Evaluator_Engine[Evaluator Engine<br>Registry Pattern]
    end

    subgraph Evaluator_Modules[Evaluator Modules]
        FIRE_MCQ[FIRE MCQ Evaluator<br>Multiple Choice Scoring]
        FIRE_Scene[FIRE Scene Evaluator<br>Rule and Principle Scoring]
    end

    subgraph Infrastructure_Layer[Infrastructure Layer]
        Path_Manager[Path Manager<br>Project Root Resolution]
        Logging[Logging Config<br>Loguru Setup]
        Cache[Cache System<br>JSONL Persistence]
    end

    Presentation_Layer --> Orchestration_Layer
    Orchestration_Layer --> Core_Layer
    Core_Layer --> Evaluator_Modules
    Core_Layer --> Infrastructure_Layer
    Evaluator_Modules --> Infrastructure_Layer
```

---

## 5. Key Components Overview

### 5.1 CLI Runner
- Entry point for all user interactions
- Parses command-line arguments (model URL, API key, dataset selection, etc.)
- Delegates to the Evaluation Pipeline

### 5.2 Configuration Manager
- Loads and parses YAML configuration files (`config/datasets.yaml`)
- Resolves dataset paths and default model parameters
- Creates typed configuration objects (`BaseDataset`, `BaseModelConfig`)

### 5.3 Evaluation Pipeline
- Central orchestrator that coordinates the full evaluation workflow
- Manages the lifecycle: load data, generate prompts, call LLM, evaluate, save results
- Supports resume from previous runs via cached responses
- Handles result persistence (metrics summaries + detailed per-sample outputs)

### 5.4 Dataset Loader
- Multi-format data reader (JSON, JSONL, CSV, Excel, Parquet)
- Recursive directory scanning for datasets spread across multiple files
- Automatic encoding detection for international character sets

### 5.5 Model Client (OpenAI)
- Async HTTP client using the OpenAI SDK
- Multi-URL load balancing (random client selection)
- Supports both streaming and non-streaming modes
- Semaphore-based concurrency control
- Token usage tracking (input/output)

### 5.6 Evaluator Engine
- Registry-based factory pattern for evaluator discovery
- Two registered evaluators: `fire` (MCQ) and `fire_scene` (Scenario)
- Abstract base class defines the evaluator contract

### 5.7 Result Storage
- Structured JSON output: metrics summary + per-sample detailed results
- Cache layer (JSONL) enables evaluation resume and avoids redundant API calls

---

## 6. Data Flow Overview

```mermaid
sequenceDiagram
    participant User
    participant CLI as CLI Runner
    participant Config as Config Manager
    participant Pipeline as Evaluation Pipeline
    participant Loader as Dataset Loader
    participant Client as Model Client
    participant LLM as LLM API
    participant Evaluator as Evaluator
    participant Judge as Judge Model API
    participant Storage as Result Storage

    User->>CLI: Run evaluation command
    CLI->>Config: Load YAML configuration
    Config-->>CLI: Dataset and model configs
    CLI->>Pipeline: Start evaluation

    loop For each dataset
        Pipeline->>Loader: Load dataset samples
        Loader-->>Pipeline: List of samples
        Pipeline->>Pipeline: Extract prompts and ground truths
        Pipeline->>Pipeline: Check cache for existing responses
        Pipeline->>Client: Send uncached prompts in batch
        Client->>LLM: Async API calls with concurrency control
        LLM-->>Client: Model responses
        Client-->>Pipeline: Predictions
        Pipeline->>Pipeline: Strip think tags from responses
        Pipeline->>Evaluator: Evaluate predictions vs ground truths

        alt MCQ Evaluation
            Evaluator->>Evaluator: Extract answer choices via regex
            Evaluator-->>Pipeline: Accuracy metrics per subtask
        else Scene Evaluation with Rules
            Evaluator->>Evaluator: Parse JSON and route to rule judge
            Evaluator-->>Pipeline: Binary correctness scores
        else Scene Evaluation with Principles
            Evaluator->>Client: Send judge prompts to FIRE_RM
            Client->>Judge: Scoring requests
            Judge-->>Client: Score responses 1 to 5
            Client-->>Evaluator: Judge outputs
            Evaluator->>Evaluator: Parse and normalize scores
            Evaluator-->>Pipeline: Aggregated scene scores
        end

        Pipeline->>Storage: Save metrics summary and details
    end

    Pipeline-->>CLI: Evaluation results
    CLI-->>User: Display results
```

---

## 7. Deployment Topology

```mermaid
graph LR
    subgraph User_Machine[User Machine]
        Python_Runtime[Python 3.x Runtime]
        FIRE_App[FIRE_Bench Application]
        Local_FS[Local Filesystem<br>Datasets and Results]
    end

    subgraph Remote_Services[Remote LLM Services]
        API_1[LLM API Endpoint 1]
        API_2[LLM API Endpoint 2]
        API_N[LLM API Endpoint N]
        Judge_Endpoint[Judge Model Endpoint<br>FIRE_RM]
    end

    FIRE_App -->|HTTPS| API_1
    FIRE_App -->|HTTPS| API_2
    FIRE_App -->|HTTPS| API_N
    FIRE_App -->|HTTPS| Judge_Endpoint
    FIRE_App --> Local_FS
    Python_Runtime --> FIRE_App
```

The system runs as a **local CLI application** on the user's machine. It connects to remote LLM API endpoints (which can be self-hosted or cloud-based) via HTTPS. All datasets, configurations, caches, and results are stored on the local filesystem.

---

## 8. Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Language | Python 3.x | Core application language |
| Async Runtime | asyncio | Concurrent API request handling |
| LLM SDK | openai >= 1.0.0 | OpenAI/Azure API client |
| HTTP | httpx >= 0.24.0 | Underlying HTTP transport |
| Data Processing | pandas, numpy, pyarrow | Dataset loading (CSV, Excel, Parquet) |
| Configuration | PyYAML | YAML config parsing |
| Data Models | pydantic >= 2.0 | Typed configuration and result models |
| Logging | loguru | Structured logging |
| CLI | argparse | Command-line argument parsing |
| Project Layout | pyrootutils | Automatic project root discovery |
| Serialization | jsonlines | JSONL cache read/write |

---

## 9. Quality Attributes

| Attribute | Approach |
|-----------|----------|
| **Performance** | Async concurrency with semaphore control; multi-URL load balancing; batch processing with progress tracking |
| **Reliability** | Caching and resume support; graceful error handling per sample; retry logic in OpenAI client |
| **Extensibility** | Registry-based evaluator pattern; abstract base classes for all major components; plugin-ready architecture |
| **Maintainability** | Clear layer separation; Pydantic models for config validation; centralized path management |
| **Security** | API keys passed via CLI or environment variables; no credential persistence; HTTP logging suppressed |

---

## 10. Constraints and Assumptions

1. **OpenAI-Compatible API**: All LLM endpoints must expose an OpenAI-compatible chat/completions API
2. **Local Execution**: The platform runs on a single machine; no distributed evaluation support
3. **Python Ecosystem**: All components are Python-based; no polyglot deployment
4. **Dataset Locality**: Datasets must be available on the local filesystem before evaluation
5. **Judge Model Availability**: FIRE Scene principle-based evaluation requires a separate judge model (FIRE-RM) accessible via API
