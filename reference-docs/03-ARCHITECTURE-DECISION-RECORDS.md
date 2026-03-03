# Architecture Decision Records (ADR) — FIRE-Bench

This document captures the key architectural decisions made during the design and development of the FIRE-Bench platform, the context behind them, and their consequences.

---

## ADR-001: Use OpenAI-Compatible API as the Universal Model Interface

**Status**: Accepted

**Context**:  
FIRE-Bench needs to evaluate a wide variety of LLMs (GPT-4, open-source models served via vLLM, Azure-hosted models, custom fine-tuned models). Each provider has different API conventions.

**Decision**:  
Standardize on the OpenAI Chat Completions API as the universal interface. All target models must be accessible through an OpenAI-compatible endpoint. Azure OpenAI is supported as a secondary API type.

**Consequences**:
- Any model served via vLLM, TGI, Ollama, or similar OpenAI-compatible layers can be evaluated without code changes
- Azure-specific deployments are handled via `AsyncAzureOpenAI` client
- Models not exposing an OpenAI-compatible API require a proxy or adapter
- The `extra_body` field allows passing vendor-specific parameters without modifying the core client

---

## ADR-002: Async-First Architecture with Semaphore-Based Concurrency

**Status**: Accepted

**Context**:  
Evaluating 14,000+ questions requires high throughput. Each question involves a network round-trip to an LLM API with typical latencies of 2-30 seconds.

**Decision**:  
Use Python's `asyncio` as the concurrency framework. All API calls are made via `AsyncOpenAI` clients. A global `asyncio.Semaphore` limits the number of concurrent in-flight requests to prevent overwhelming the API endpoint.

**Consequences**:
- Throughput scales linearly with the number of concurrent workers and API endpoints
- Total concurrency = `number_of_URLs x per_url_max_workers` (default: 128 per URL)
- No threading complexity; all concurrency is cooperative via the event loop
- The CLI entry point uses `asyncio.run()` as the bridge from synchronous to async
- All evaluators that need to call external APIs (e.g., `FireSceneEvaluator`) must use async methods

---

## ADR-003: Registry Pattern for Evaluator Extensibility

**Status**: Accepted

**Context**:  
Different datasets require fundamentally different evaluation strategies (multiple-choice accuracy vs. rubric-based scoring). The system must be easily extensible to support new evaluation methods.

**Decision**:  
Implement an `EvaluatorManager` with a decorator-based registration mechanism. Evaluators register themselves with a string key (e.g., `"fire"`, `"fire_scene"`) at import time. The pipeline looks up evaluators by the key specified in the dataset YAML configuration.

```
@evaluator_manager.register("fire")
class FIREMCQEvaluatorAdapter(BaseEvaluator):
    ...
```

**Consequences**:
- Adding a new evaluator requires only creating a new class and decorating it — no changes to pipeline code
- The YAML configuration drives which evaluator is used for which dataset
- Strong decoupling between the pipeline and evaluation logic
- Discovery happens at import time; all evaluators must be imported before the pipeline runs

---

## ADR-004: Dual Evaluation Strategy — Rules + LLM-as-Judge

**Status**: Accepted

**Context**:  
The FIRE Scene dataset contains two types of tasks:
1. **Rule-based tasks** (~1,000 questions): Have deterministic correct answers (e.g., risk approval decisions, classification categories)
2. **Principle-based tasks** (~2,000 questions): Open-ended responses that must be scored on quality using rubric criteria

**Decision**:  
Implement a hybrid evaluation approach within `FireSceneEvaluator`:
- **Rule evaluator**: Deterministic JSON field comparison. Each task type has a specific judge function that knows which fields to compare.
- **Principle evaluator**: Uses a separate LLM (FIRE-RM, a reward model) as a judge. The judge model receives the question, reference answer, evaluation criteria (principle), and the model's response, then outputs a 1-5 score.

**Consequences**:
- Rule-based evaluation is fast, deterministic, and requires no additional API calls
- Principle-based evaluation requires deploying and maintaining a judge model
- Judge model outputs may vary; `repeat_num` averaging mitigates this variance
- A separate cache system (`irm_cache`) prevents re-scoring already-evaluated samples
- Score normalization from [1,5] to [0,100] allows unified reporting across both evaluation types

---

## ADR-005: JSONL-Based Caching and Resume Support

**Status**: Accepted

**Context**:  
Large-scale evaluations can take hours. Network failures, API rate limits, or user interruptions can cause partial completion. Re-running from scratch wastes time and API credits.

**Decision**:  
Implement a cache-first strategy:
1. Before sending prompts to the LLM, check if a response for the exact prompt already exists in the JSONL cache file
2. Only send uncached prompts to the API
3. Write new responses to the cache file immediately (append mode)
4. Support a `--resume-folder` flag to continue from a previous run's cache

**Consequences**:
- Evaluations are idempotent and resumable
- Cache key is the full prompt string (exact match) — even minor prompt changes cause cache misses
- Cache files grow monotonically (append-only); no garbage collection
- The datasets.yaml is copied to the results folder on first run, ensuring reproducibility on resume
- Cache-lock (`asyncio.Lock`) prevents corruption during concurrent writes

---

## ADR-006: Multi-URL Load Balancing via Random Selection

**Status**: Accepted

**Context**:  
For self-hosted models, organizations often deploy multiple serving instances behind different URLs. The evaluation framework needs to distribute load across these instances.

**Decision**:  
Accept a list of API URLs in the configuration. Create one `AsyncOpenAI` client per URL. For each request, randomly select one client from the pool.

**Consequences**:
- Simple implementation with no external dependencies (no load balancer needed)
- Statistical load distribution approximates uniform distribution over large batches
- No health checking or failover — a failed URL wastes one request slot
- Suitable for homogeneous deployment topologies where all URLs serve the same model

---

## ADR-007: Think-Tag Stripping for Chain-of-Thought Models

**Status**: Accepted

**Context**:  
Many reasoning-focused LLMs (DeepSeek-R1, QwQ, etc.) wrap their internal reasoning in tags like `<think>...</think>`. The evaluation should only assess the final answer, not the reasoning process.

**Decision**:  
Implement a `process_model_response` function that strips various think-tag formats before evaluation:
- `<think>...</think>`
- `<seed:think>...</seed:think>`
- `<thinking>...</thinking>`
- `<answer>...</answer>` (extracts content within)

**Consequences**:
- Models using chain-of-thought are evaluated fairly against their final answers
- The stripping is format-agnostic and handles multiple tag conventions
- If a model uses an unrecognized tag format, the full response (including reasoning) will be evaluated — potentially lowering scores
- The raw (unstripped) response is always preserved in the cache and result files for debugging

---

## ADR-008: Pydantic for Configuration Validation

**Status**: Accepted

**Context**:  
Configuration errors (wrong types, missing fields) should be caught early before expensive API calls are made.

**Decision**:  
Use Pydantic v2 `BaseModel` for all configuration and result data classes (`BaseDataset`, `BaseModelConfig`, `EvaluationResult`). Pydantic provides automatic type validation, default values, and serialization.

**Consequences**:
- Invalid configurations fail fast with clear error messages
- Type coercion handles minor mismatches (e.g., string numbers to int)
- `.dict()` and `.json()` methods simplify serialization for result storage
- Requires Pydantic v2+ which has different semantics from v1

---

## ADR-009: pyrootutils for Project Root Discovery

**Status**: Accepted

**Context**:  
The project uses relative paths for datasets, configs, and results. A reliable way to find the project root is needed regardless of where the script is invoked from.

**Decision**:  
Use `pyrootutils` to search upward from the current file for project root indicators (`.gitignore`, `pyproject.toml`, `setup.py`, `requirements.txt`). Implement as a Singleton (`ProjectPathManager`) to ensure all modules use the same root.

**Consequences**:
- The project can be run from any directory (e.g., `python path/to/run_evaluation.py`)
- Fallback logic uses the file's own location if no indicator is found
- All path resolution goes through the singleton, ensuring consistency
- Adding a new root indicator requires a one-line change

---

## ADR-010: Configuration-Driven Dataset Design

**Status**: Accepted

**Context**:  
Different datasets have different file formats, directory structures, evaluator requirements, and evaluation parameters. Hard-coding these would create maintenance burden.

**Decision**:  
Centralize all dataset definitions in `config/datasets.yaml`. Each dataset entry specifies its path, evaluator type, and any evaluator-specific parameters (judge model, prompt template, etc.). The pipeline reads this configuration and dynamically constructs the appropriate components.

**Consequences**:
- Adding a new dataset requires only a YAML entry and (possibly) a new evaluator class
- All dataset parameters are version-controlled and inspectable
- Complex evaluator configurations (judge model URLs, temperature, etc.) are externalized
- The configuration file is copied to each result folder for reproducibility
