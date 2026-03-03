# Low-Level Design (LLD) — FIRE-Bench Evaluation Platform

## 1. Overview

This document provides a detailed technical design of each module in the FIRE-Bench platform, covering class structures, method signatures, data flows, algorithms, and inter-module interactions. It is intended for developers and contributors who need to understand, maintain, or extend the system.

---

## 2. Module Dependency Graph

```mermaid
graph TD
    run_evaluation_py[run_evaluation.py] --> cli[src.utils.cli<br>CLIRunner]
    cli --> config[src.utils.config<br>ConfigManager]
    cli --> pipeline[src.core.pipeline<br>EvaluationPipeline]
    cli --> logging_config[src.utils.logging_config]

    pipeline --> model_client[src.core.model_client<br>OpenAIModelClient]
    pipeline --> dataset_loader[src.core.dataset_loader<br>DatasetLoader]
    pipeline --> evaluator_init[src.core.evaluator<br>evaluator_manager]
    pipeline --> config
    pipeline --> path_manager[src.utils.path_manager]

    evaluator_init --> fire_eval[fire_evaluator<br>FIREMCQEvaluator]
    evaluator_init --> fire_scene_eval[fire_scene_evaluator<br>FireSceneEvaluator]

    fire_scene_eval --> model_client
    fire_scene_eval --> config

    model_client --> base[src.core.base<br>Base Classes]
    dataset_loader --> base
    evaluator_init --> base
    config --> base
    config --> path_manager
```

---

## 3. Base Classes and Data Models

### 3.1 Class Diagram

```mermaid
classDiagram
    class BaseDataset {
        +str name
        +str_or_list path
        +str evaluator
        +bool shuffle
        +str description
        +int repeat_num
    }

    class BaseModelConfig {
        +str name
        +str_or_list urls
        +int per_url_max_workers
        +str api_key
        +str api_type
        +str api_version
        +str model
        +float temperature
        +float top_p
        +int top_k
        +int timeout
        +int max_tokens
        +str system_prompt
        +dict extra_body
        +bool streaming
        +bool use_chat
    }

    class EvaluationResult {
        +str dataset_name
        +str model_name
        +str timestamp
        +dict metrics
        +dict sample_stats
        +dict details
        +accuracy() float
        +total_samples() int
        +correct_samples() int
        +get_primary_metric() tuple
    }

    class BaseDatasetLoader {
        <<abstract>>
        +load(dataset_config) list
        +validate(dataset_config) bool
    }

    class BaseModelClient {
        <<abstract>>
        +generate_batch(prompt) str
        +validate_config(config) bool
    }

    class BaseEvaluator {
        <<abstract>>
        +BaseDataset dataset_config
        +dict kwargs
        +extract_format_prompt(config, sample) str
        +evaluate(predictions, ground_truths) dict
        +_extract_prompt(sample) str
        +extract_ground_truth(sample) str
    }

    BaseDatasetLoader <|-- DatasetLoader
    BaseModelClient <|-- OpenAIModelClient
    BaseEvaluator <|-- FIREMCQEvaluator
    BaseEvaluator <|-- FireSceneEvaluator
```

### 3.2 BaseDataset

A Pydantic model that represents a dataset configuration entry from the YAML file.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `str` | required | Unique identifier for the dataset |
| `path` | `str or List[str]` | required | File or directory path(s) to dataset files |
| `evaluator` | `str` | required | Key to look up in evaluator registry (e.g., `fire`, `fire_scene`) |
| `shuffle` | `bool` | `False` | Whether to randomly shuffle samples before evaluation |
| `repeat_num` | `int` | `1` | Number of times to repeat the dataset (for statistical stability) |

### 3.3 BaseModelConfig

Configuration for connecting to an LLM endpoint. All parameters map to OpenAI API conventions.

Key design decisions:
- `urls` is a **list** to support multi-endpoint load balancing
- `extra_body` allows passing vendor-specific parameters (e.g., reasoning mode)
- `use_chat` toggles between chat completion and text completion APIs

### 3.4 EvaluationResult

A flexible result container supporting both simple accuracy metrics and nested multi-level metrics (e.g., per-subtask, per-scene scores). Backward-compatible properties (`accuracy`, `total_samples`, `correct_samples`) allow legacy consumers to read results without changes.

---

## 4. Dataset Loader

### 4.1 Class Structure

```mermaid
classDiagram
    class DatasetLoader {
        +load(dataset_config) List_Dict
        +validate(dataset_config) bool
        -_load_single_file(file_path) List_Dict
        -_load_directory(dir_path) List_Dict
        -_load_json(file_path) List_Dict
        -_load_jsonl(file_path) List_Dict
        -_load_excel(file_path) List_Dict
        -_load_csv(file_path) List_Dict
        -_load_parquet(file_path) List_Dict
        -_detect_encoding(file_path) str
        -_validate_file(file_path) bool
    }
```

### 4.2 Loading Flow

```mermaid
flowchart TD
    A[Start load] --> B[resolve_dataset_path]
    B --> C{Is file or directory?}
    C -->|File| D[_load_single_file]
    C -->|Directory| E[_load_directory]
    D --> F{File extension?}
    F -->|.json| G[_load_json]
    F -->|.jsonl| H[_load_jsonl]
    F -->|.xlsx .xls| I[_load_excel]
    F -->|.csv| J[_load_csv]
    F -->|.parquet| K[_load_parquet]
    F -->|other| G
    E --> L[rglob for all supported patterns]
    L --> D
    G --> M[Normalize to list of dicts]
    H --> M
    I --> M
    J --> M
    K --> M
    M --> N[Return aggregated data]
```

### 4.3 JSON Normalization Logic

The `_load_json` method handles multiple JSON structures:
1. **Array at root**: Returns directly as list of samples
2. **Object with `data` key**: Extracts and returns `data` array
3. **Object with `examples` key**: Extracts and returns `examples` array
4. **Object with `items` key**: Extracts and returns `items` array
5. **Plain object**: Wraps in a single-element list

### 4.4 Encoding Detection

The `_detect_encoding` method tries encodings in order: `utf-8` → `gbk` → `gb2312` → `latin1` → `cp1252`, reading the first 1000 characters to detect which encoding works.

---

## 5. Model Client

### 5.1 Class Structure

```mermaid
classDiagram
    class OpenAIModelClient {
        +BaseModelConfig config
        +list urls
        +int max_workers
        -list _aclients
        -Lock _cache_lock
        -Lock _usage_lock
        -Semaphore _sem
        -int _total_input_tokens
        -int _total_output_tokens
        +__aenter__() self
        +__aexit__() void
        +validate_config(config) bool
        +generate_batch(prompts, cache_writer) List_str
        -_make_async_request(prompt, callback) str
    }

    class ModelClientFactory {
        +create_client(config) OpenAIModelClient
    }

    ModelClientFactory --> OpenAIModelClient : creates
```

### 5.2 Multi-URL Load Balancing

```mermaid
flowchart TD
    A[generate_batch called with N prompts] --> B[Create N async tasks]
    B --> C{For each task}
    C --> D[Acquire semaphore slot]
    D --> E[random.choice selects a client]
    E --> F[Send request to selected URL]
    F --> G{Streaming?}
    G -->|Yes| H[Collect chunks and track progress]
    G -->|No| I[Read full response]
    H --> J[Write to cache via callback]
    I --> J
    J --> K[Release semaphore slot]
    K --> L[Update token counters]
```

**Concurrency Model**:
- Total workers = `len(urls) x per_url_max_workers`
- An `asyncio.Semaphore` limits concurrent in-flight requests to `max_workers`
- Each request randomly picks one `AsyncOpenAI` client from the pool
- Cache writes are serialized via `asyncio.Lock` to prevent corruption

### 5.3 Streaming Support

When `streaming=True`:
1. The client receives server-sent events (SSE) chunks
2. Each chunk's content delta is accumulated into a list
3. A progress callback is invoked per chunk for real-time UI updates
4. Token usage is extracted from the final chunk's usage field

### 5.4 FIRE-RM Special Handling

When the model name contains `rm` (case-insensitive), a `repetition_penalty` of `1.05` is automatically injected into `extra_body`. This prevents the reward/judge model from generating repetitive output.

---

## 6. Evaluation Pipeline

### 6.1 Class Structure

```mermaid
classDiagram
    class EvaluationPipeline {
        +ModelClientFactory model_client_factory
        +DatasetLoader dataset_loader
        +EvaluatorManager evaluator_manager
        +str config_path
        +Path results_dir
        +Path cache_dir
        +str timestamp
        +Path results_folder
        +run_evaluation(config_manager, model_config, dataset_names, max_samples, results_dir, resume_folder) List_EvaluationResult
        -_evaluate_single_dataset(model_client, model_config, dataset_config, max_samples, config_manager) EvaluationResult
        -_get_model_response_from_cache(prompt, cache_dict_list) Optional_str
        -_load_dataset_configs(config_manager, dataset_names) List_BaseDataset
        -_save_results(results, model_name, config_manager) void
    }
```

### 6.2 Single Dataset Evaluation Flow

```mermaid
flowchart TD
    A[Start _evaluate_single_dataset] --> B[Load cache file if exists]
    B --> C[Build evaluator from registry]
    C --> D[Load dataset via DatasetLoader]
    D --> E{Shuffle enabled?}
    E -->|Yes| F[Randomly shuffle samples]
    E -->|No| G[Keep original order]
    F --> H{Repeat num > 1?}
    G --> H
    H -->|Yes| I[Duplicate dataset N times]
    H -->|No| J[Keep original]
    I --> K{Max samples set?}
    J --> K
    K -->|Yes| L[Truncate to max_samples]
    K -->|No| M[Use all samples]
    L --> N[Separate cached vs uncached samples]
    M --> N
    N --> O[For each sample extract prompt and ground_truth]
    O --> P[Check prompt against cache]
    P --> Q{Cached?}
    Q -->|Yes| R[Add to cached list]
    Q -->|No| S[Add to candidate list]
    R --> T[Send candidate prompts to model_client.generate_batch]
    S --> T
    T --> U[Merge cached and new predictions]
    U --> V[Strip think tags via process_model_response]
    V --> W{Evaluator has evaluate_async?}
    W -->|Yes| X[Call evaluate_async]
    W -->|No| Y[Call evaluate]
    X --> Z[Build EvaluationResult]
    Y --> Z
```

### 6.3 Response Processing — Think Tag Removal

The `process_model_response` function strips chain-of-thought reasoning markers before evaluation:

1. Search for `<think>...</think>` tags
2. Fallback to `<seed:think>...</seed:think>` tags
3. Fallback to `<thinking>...</thinking>` tags
4. If found, extract only the content **after** the closing tag
5. Additionally check for `<answer>...</answer>` tags (used by some models like XuanYuan, DianJing)
6. Return the cleaned response for evaluation

### 6.4 Cache Mechanism

- **Cache file**: `{results_folder}/cache/{model_name}_{dataset_name}.json` (JSONL format)
- **Key**: The full prompt string (exact match)
- **Value**: The model's raw response
- On resume, cached responses are loaded and matched against current prompts
- New responses are appended to the cache file in real-time during batched generation
- Cache is write-through: each response is flushed immediately

### 6.5 Result Persistence

Two types of output files per evaluation run:

1. **Metrics Summary** (`metrics_summary.json`): Contains model name, timestamp, per-dataset metrics (excludes raw sample details)
2. **Per-Dataset Details** (`{model}_{dataset}.json`): Contains every sample with its original data, filled prompt, model response, and correctness flag

---

## 7. Evaluator Subsystem

### 7.1 Registry Pattern

```mermaid
flowchart TD
    A[EvaluatorManager] --> B[factory dict]
    B -->|key fire| C[FIREMCQEvaluatorAdapter]
    B -->|key fire_scene| D[FireSceneEvaluatorAdapter]
    C --> E[FIREMCQEvaluator]
    D --> F[FireSceneEvaluator]
    F --> G[SampleTypeDetector]
    F --> H[RuleEvaluator]
    F --> I[SceneAggregator]
```

New evaluators can be added by:
1. Creating a class that extends `BaseEvaluator`
2. Decorating it with `@evaluator_manager.register("key_name")`
3. Implementing `extract_format_prompt`, `extract_ground_truth`, and `evaluate` methods

### 7.2 FIRE MCQ Evaluator

**Purpose**: Evaluates multiple-choice questions from financial certification exams.

**Prompt Construction**:
- Concatenates the base prompt with a format instruction: "Answer directly with option letters, output format: Answer: Options"
- Optionally prepends few-shot demonstrations (`demo_count` up to 5)

**Answer Extraction Algorithm** (`extract_choice`):
1. Normalize response: uppercase, remove separators (commas, spaces, Chinese punctuation)
2. Try regex patterns in priority order:
   - `答：ABC` format (Chinese answer marker)
   - `答案是ABC` format (Chinese "the answer is" marker)
   - `答案选项ABC` format
   - Standalone `ABC` at line start
3. Fallback: collect all `[A-E]` characters found anywhere in the response
4. Last resort: random choice from A-E
5. Return deduplicated, sorted option string (e.g., `"ACD"`)

**Scoring**: Exact set match between extracted prediction and gold answer. Per-subtask (benchmark) accuracy is also computed.

### 7.3 FIRE Scene Evaluator

**Purpose**: Evaluates real-world financial scenario tasks using two complementary methods.

```mermaid
classDiagram
    class FireSceneEvaluator {
        -ConfigManager _config_manager
        -_FirePrincipleConfig _settings
        -ModelClientFactory _model_client_factory
        -SampleTypeDetector _type_detector
        -RuleEvaluator _rule_evaluator
        -SceneAggregator _scene_aggregator
        -Path irm_cache_dir
        +extract_format_prompt(config, sample) str
        +extract_ground_truth(sample) str
        +evaluate_async(predictions, ground_truths, data_samples) dict
        +evaluate(predictions, ground_truths) dict
        -_format_judge_prompt(sample, pred, gt) str
        -_parse_scores(judge_outputs) tuple
        -_extract_score(response) float
        -_build_model_config() BaseModelConfig
    }

    class SampleTypeDetector {
        +determine_sample_type(sample) str
    }

    class RuleEvaluator {
        +extract_answer_from_response(data_source, response) Any
        +judge_subtask_router(data_source, pred, gt) tuple
        -_judge_risk(pred, gt) bool
        -_judge_dianxiao(pred, gt) bool
        -_judge_qiwei(pred, gt) bool
        -_judge_cuishou(pred, gt) bool
        -_judge_content_safe(pred, gt) bool
        -_judge_credit_talk_recommendation(pred, gt) bool
        -_judge_dialogue_state_classification(pred, gt) bool
        -_judge_dialogue_state_classification_1(pred, gt) bool
        -_judge_feedback_attribution(pred, gt) bool
        -_judge_complaint_type_classification_gen(pred, gt) bool
        -_judge_complaint_type_classification_extra(pred, gt) bool
        -_judge_push_content_compliance_qc(pred, gt) bool
        -_judge_risk_behavior_prediction(pred, gt) bool
    }

    class SceneAggregator {
        +aggregate_by_scene(scores, data_samples) dict
    }

    FireSceneEvaluator --> SampleTypeDetector
    FireSceneEvaluator --> RuleEvaluator
    FireSceneEvaluator --> SceneAggregator
```

#### 7.3.1 Sample Type Detection

The `SampleTypeDetector` classifies each sample:
- **Principle type**: Sample contains a `准则` (principle) or `principle` field — evaluated by LLM-as-Judge
- **Rule type**: Sample contains a `data_source` field — evaluated by deterministic rule matching
- **Type field**: Falls back to reading a `type` field
- **Unknown**: Treated as invalid (score = None)

#### 7.3.2 Rule-Based Evaluation

The `RuleEvaluator` routes samples to specialized judge functions based on `data_source` keyword matching:

| Data Source Keyword | Judge Function | Business Domain |
|---------------------|---------------|-----------------|
| `risk` | `_judge_risk` | Credit risk approval decisions |
| `dianxiao` | `_judge_dianxiao` | Telesales emotion state classification |
| `cuishou` | `_judge_cuishou` | Collections compliance violation detection |
| `企业微信` | `_judge_qiwei` | Enterprise WeChat routing |
| `金融内容安全拦截` | `_judge_content_safe` | Financial content safety filtering |
| `客户对话状态判断_1` | `_judge_dialogue_state_classification_1` | Dialogue state classification (variant 1) |
| `客户对话状态判断_0` | `_judge_dialogue_state_classification` | Bargaining stage identification |
| `客户反馈归因分析` | `_judge_feedback_attribution` | Customer feedback attribution |
| `客户风险行为预测` | `_judge_risk_behavior_prediction` | Risk behavior prediction |
| `客户投诉类型判断_生成` | `_judge_complaint_type_classification_gen` | Complaint type classification (generative) |
| `客户投诉类型判断_抽取` | `_judge_complaint_type_classification_extra` | Complaint type classification (extractive) |
| `推送内容合规` | `_judge_push_content_compliance_qc` | Push content compliance check |
| `增信话术推荐` | `_judge_credit_talk_recommendation` | Credit enhancement script recommendation |

Each judge function:
1. Parses the prediction (expected JSON format)
2. Parses the ground truth
3. Compares specific fields for exact match
4. Returns `True/False` and a subtask label

#### 7.3.3 Principle-Based Evaluation (LLM-as-Judge)

For open-ended questions, a separate judge model (FIRE-RM) scores responses on a 1-5 scale:

```mermaid
flowchart TD
    A[Collect principle_type samples] --> B[Format judge prompts]
    B --> C[Repeat prompts N times for stability]
    C --> D[Check IRM cache for prior scores]
    D --> E[Send uncached prompts to judge model]
    E --> F[Parse JSON score from judge response]
    F --> G[Average scores across repeats]
    G --> H[Normalize 1_5 to 0_100 scale]
    H --> I[Assign to original sample indices]
```

**Score Extraction** (`_extract_score`):
1. Try to extract a JSON block `{...}` from the response
2. Parse the `score` field from the JSON
3. Fallback: regex for `"score": <number>`
4. Last resort: find any standalone number 1-5 in the response

**Score Normalization**: Raw score in [1, 5] is mapped to [0, 100] via: $\text{normalized} = \frac{(\text{score} - 1)}{4} \times 100$

#### 7.3.4 Scene Aggregation

The `SceneAggregator` groups scores by hierarchical scene labels:
- **Primary scene** (`top_name`): e.g., "Banking", "Insurance"
- **Secondary scene** (`task_name`): e.g., "Corporate Finance Risk Control"
- Separate averages for principle-type vs rule-type samples
- Error counters for samples that failed evaluation

---

## 8. Utility Modules

### 8.1 Path Manager

Uses `pyrootutils` to locate the project root by searching for marker files (`.gitignore`, `pyproject.toml`, `setup.py`, `requirements.txt`). Implements the **Singleton pattern** to ensure consistent path resolution throughout the application.

Key paths managed:
- `get_project_root()` — Absolute project root
- `get_config_path()` — Config YAML location
- `get_results_path()` — Results output directory (auto-created)
- `get_cache_path()` — Per-run cache directory (auto-created)
- `get_irm_cacahe_path()` — Judge model cache directory (auto-created)
- `resolve_dataset_path()` — Resolves relative paths to absolute; supports both single and list paths

### 8.2 Configuration Manager

Responsibilities:
1. Load and parse `datasets.yaml`
2. Create `BaseDataset` objects with resolved paths
3. Create `BaseModelConfig` objects by merging CLI args with YAML defaults
4. Provide dataset listing and validation utilities

### 8.3 Logging Configuration

Uses `loguru` with the following setup:
- Default level: `INFO` (switchable to `DEBUG` with `--verbose`)
- HTTP client logging suppressed (httpx, urllib3, openai, azure)
- Colored console output with structured formatting

---

## 9. Error Handling Strategy

| Scenario | Handling |
|----------|----------|
| API request failure | Returns empty string; logs error; evaluation continues |
| Invalid sample (missing fields) | Logs warning; skips sample; evaluation continues |
| JSON parse error in response | Returns `False` (incorrect); logs warning |
| Cache file missing | Creates new cache; starts fresh |
| Dataset file not found | Raises `FileNotFoundError` |
| Score extraction failure | Returns `None`; counted in error statistics |
| All predictions empty | Returns zero-score result with all-invalid stats |

---

## 10. Configuration Schema

### datasets.yaml Structure

```yaml
datasets:
  DATASET_NAME:
    name: string                    # Dataset identifier
    description: string             # Human readable description
    path: string_or_list            # File or directory path
    evaluator: string               # Registry key: fire or fire_scene
    shuffle: boolean                # Random sample ordering
    repeat_num: integer             # Dataset repetition count
    # Scene-specific fields
    prompt_path: string             # Path to prompt template Python file
    prompt_name: string             # Variable name in prompt file
    judge_model: string             # Judge model name
    judge_model_urls: list          # Judge API endpoints
    judge_model_api_key: string     # Judge API key
    judge_model_api_type: string    # Judge API type
    judge_max_tokens: integer       # Judge response length limit
    judge_repeat_num: integer       # Number of judge repeats
    judge_temperature: float        # Judge sampling temperature
    judge_top_p: float              # Judge nucleus sampling
    judge_timeout: integer          # Judge request timeout
    judge_per_url_max_workers: int  # Judge concurrency per URL
    judge_system_prompt: string     # Judge system prompt

defaults:
  temperature: float               # Default sampling temperature
  top_p: float                     # Default nucleus sampling
  max_tokens: integer              # Default max response tokens
  timeout: integer                 # Default request timeout
  system_prompt: string            # Default system prompt
  extra_body: dict                 # Default extra request parameters
```
