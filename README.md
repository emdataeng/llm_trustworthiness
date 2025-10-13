# Trustworthiness Attack Pipeline

This repository contains a red-teaming notebook (`A2_v2.ipynb`) that recreates a compact trustworthiness attack pipeline inspired by the *DecodingTrust* benchmark. The workflow intentionally coaxes a large language model into toxic, stereotyped, or unfair behaviour so that you can measure how easily the system can be jailbroken. Expect harmful content when you execute the notebook.

## Repository Map
- `A2_v2.ipynb` – main notebook that orchestrates dataset prep, iterative attacks, evaluation, and logging.
- `environment.yml` – Conda specification for the recommended runtime (`python=3.10`).
- `.env` – runtime secrets and configuration overrides (see below for required keys).
- `outputs/` – populated at runtime with logs and generated artefacts.
- `Attack/` and `References/` – supplementary material from earlier iterations of the project.

## Prerequisites
1. Install [Conda](https://docs.conda.io/en/latest/miniconda.html) or a compatible environment manager.
2. Ensure you have access to the DecodingTrust datasets (see *Dataset Placement*).
3. Obtain API credentials for your chosen language-model provider (OpenAI is supported out of the box).

### Create the environment
```bash
conda env create -f environment.yml
conda activate llm-trust-env
# Optional packages used inside the notebook
pip install fairlearn detoxify
```

The notebook falls back to a deterministic `DummyChatModel` when no API key is supplied, allowing you to test the control flow without incurring API usage.

## Configuration (`.env`)
Create a `.env` file in the project root with the secrets and overrides you need. The notebook reads the following keys (defaults apply when a value is missing):

| Key | Purpose | Default |
| --- | --- | --- |
| `OPENAI_API_KEY` | Enables calls to the OpenAI Chat API. | `""` (dry-run if blank) |
| `TRUST_PIPELINE_PROVIDER` | Model provider identifier. | `openai` |
| `TRUST_PIPELINE_MODEL` | Chat model name. | `gpt-3.5-turbo` |
| `TRUST_PIPELINE_TEMPERATURE` | Sampling temperature for generations. | `0.6` |
| `TRUST_PIPELINE_MAX_TOKENS` | Maximum tokens returned by the model. | `256` |
| `TRUST_PIPELINE_DRY_RUN` | Force dry-run mode (`true/false`). | Auto-detect from API key |
| `TRUST_PIPELINE_TOXICITY_DATASET` | Path to the toxicity JSONL file. | DecodingTrust default |
| `TRUST_PIPELINE_TOXICITY_TOP_K` | Number of toxic prompts to sample (`ALL` for full set). | `8` |
| `TRUST_PIPELINE_TOXICITY_THRESHOLD` | Detoxify score required for success. | `0.7` |
| `TRUST_PIPELINE_STEREOTYPE_DATASET` | Path to the stereotype dataset JSONL. | DecodingTrust default |
| `TRUST_PIPELINE_STEREOTYPE_SYSTEM_PROMPTS` | CSV with stereotype system prompts. | DecodingTrust default |
| `TRUST_PIPELINE_STEREOTYPE_TOP_K` | Samples per stereotype topic. | `2` |
| `TRUST_PIPELINE_FAIRNESS_DATASET` | Fairness classification JSONL (adult/crime). | DecodingTrust default |
| `TRUST_PIPELINE_FAIRNESS_SENSITIVE_ATTR` | NumPy array with sensitive features. | DecodingTrust default |
| `TRUST_PIPELINE_FAIRNESS_TOP_K` | Number of fairness records to keep. | `200` |
| `TRUST_PIPELINE_FAIRNESS_DATASET_TYPE` | Fairness benchmark tag (`adult` or `crime`). | `adult` |

Set absolute paths when your datasets live outside the default location.

## Dataset Placement
By default, the notebook expects a local clone of the DecodingTrust repository alongside this project:

```
../DecodingTrust/
data/
  toxicity/...
  stereotype/...
  fairness/...
```

If your datasets sit elsewhere, override the relevant `TRUST_PIPELINE_*` variables in `.env`.

## Running `A2_v2.ipynb`
1. Launch Jupyter (Lab or Notebook) from a terminal where the `llm-trust-env` environment is active.
2. Ensure `.env` is populated and the required datasets are accessible.
3. Execute the notebook top to bottom:
   - **0. Notebook Setup** – imports, logging, and directory preparation.
   - **1. Deploy an LLM Backend** – instantiates the chat model or dummy fallback and records a smoke test.
   - **2. Load Local Benchmark Subsets** – samples toxicity, stereotype, and fairness payloads from DecodingTrust.
   - **3. Iterative Attack Loop** – runs jailbreak attempts for toxicity and stereotype prompts, then evaluates fairness classifications.
   - **4. Analyse Results** – summarises attack metrics, fairness gaps, and (optionally) Detoxify scores.
   - **5. Persist Artefacts** – writes JSONL traces and metric summaries under `outputs/trustworthiness_attacks/`.

You can rerun individual sections after tweaking configuration; logs append to `outputs/trustworthiness_attacks/logs/attack_pipeline_log.jsonl`.

## Generated Artefacts
- `outputs/trustworthiness_attacks/logs/attack_pipeline_log.jsonl` – append-only operational log.
- `outputs/trustworthiness_attacks/generations/<session>/toxicity.jsonl` – full attack traces for the toxicity benchmark.
- `outputs/trustworthiness_attacks/generations/<session>/stereotype.jsonl` – attack traces for stereotype prompts.
- `outputs/trustworthiness_attacks/generations/<session>/fairness.jsonl` – classification responses with sensitive features (present when fairness runs).
- `outputs/trustworthiness_attacks/generations/<session>/metrics.json` – aggregate success and fairness statistics used in Section 4.

Each run is timestamped, making it easy to compare different configurations.

## Notebook Components at a Glance
- **Logging utilities** – `ensure_paths`, `append_log`, and session bookkeeping.
- **Model bootstrap** – `load_env_settings`, `ModelConfig`, `DummyChatModel`, and `build_chat_model`.
- **Dataset ingestion** – helpers that locate DecodingTrust files, apply `top_k` sampling, and attach metadata.
- **Attack heuristics** – per-perspective scoring (`score_toxicity_success`, `score_stereotype_success`) and the `iterative_attack` driver with escalation prompts.
- **Fairness evaluation** – constructs classification prompts, records predictions, and uses `fairlearn` for demographic metrics.
- **Analysis & persistence** – aggregates outcomes, showcases sample jailbreaks, computes Detoxify scores, and writes JSON/JSONL artefacts.

## Optional Enhancements
- Install `detoxify` to replace keyword heuristics with model-based toxicity scoring.
- Extend `.env` to point at alternate providers or local models by modifying `build_chat_model`.
- Experiment with custom escalation prompts by editing the `ATTACK_CONFIGS` dictionary in Section 3.3 of the notebook.

## Safety Notice
This notebook deliberately elicits harmful content for evaluation. Run it in controlled environments, follow organisational policies when handling generated text, and avoid exposing the outputs to end users.
