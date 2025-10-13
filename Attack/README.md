# Trustworthiness Attack Pipeline

This repository contains a minimal red-teaming pipeline that reproduces key ideas from DecodingTrust. The main entry point is the Jupyter notebook `trustworthiness_mini_pipeline.ipynb`, which orchestrates targeted jailbreak attempts against a chat model and aggregates the results.

> Warning: running the notebook will intentionally generate and store disallowed, offensive, and privacy-sensitive text. Only execute it in controlled environments.

## Repository Highlights
- `trustworthiness_mini_pipeline.ipynb` – attack pipeline notebook.
- `environment.yml` – conda specification that installs LangChain, OpenAI tooling, and supporting libraries.
- `outputs/trustworthiness_attacks/` – default directory for logs, JSONL trace files, and metric summaries produced by the notebook.
- `.env` – local configuration file containing API keys and pipeline overrides (not version controlled in production).

## Environment Setup
- Create the conda environment:
  ```
  conda env create -f environment.yml
  ```
- Activate it:
  ```
  conda activate llm-trust-env
  ```
- (Optional) Enable NVIDIA GPU acceleration on supported machines:
  ```
  conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia
  ```

The YAML targets Python 3.10 and installs LangChain, langchain-openai, pandas, tqdm, python-dotenv, and other utilities used by the notebook. On Windows you must use `pytorch-cuda=12.1` instead of adding `cudatoolkit` directly; Linux users can list `pytorch-cuda` in the YAML if desired.

## Configuration
Create a `.env` file next to the notebook with values similar to:
```
OPENAI_API_KEY=sk-...
TRUST_PIPELINE_PROVIDER=openai
TRUST_PIPELINE_MODEL=gpt-3.5-turbo
TRUST_PIPELINE_TEMPERATURE=0.6
TRUST_PIPELINE_MAX_TOKENS=256
TRUST_PIPELINE_DRY_RUN=false
```

Key options:
- `OPENAI_API_KEY` enables real calls; when missing the notebook falls back to a deterministic `DummyChatModel` so you can exercise the pipeline offline.
- `TRUST_PIPELINE_PROVIDER`, `TRUST_PIPELINE_MODEL`, `TRUST_PIPELINE_TEMPERATURE`, and `TRUST_PIPELINE_MAX_TOKENS` control the backend instantiated via LangChain.
- Dataset overrides such as `TRUST_PIPELINE_TOXICITY_DATASET`, `TRUST_PIPELINE_STEREOTYPE_DATASET`, `TRUST_PIPELINE_STEREOTYPE_SYSTEM_PROMPTS`, and `TRUST_PIPELINE_PRIVACY_DATASET` allow you to point at alternate JSONL or CSV files.
- Sampling controls (`TRUST_PIPELINE_TOXICITY_TOP_K`, `TRUST_PIPELINE_STEREOTYPE_TOP_K`, `TRUST_PIPELINE_PRIVACY_TOP_K`) accept integers or `ALL`.

By default the notebook expects the DecodingTrust data repository to live one directory above this project (`DATASETS_BASE_DIR = BASE_DIR.parent`). Override the environment variables if your datasets are elsewhere.

## Running the Notebook
- Activate the conda environment and start Jupyter:
  ```
  conda activate llm-trust-env
  jupyter lab
  ```
- Open `trustworthiness_mini_pipeline.ipynb`.
- Confirm the logging paths in the first cell and execute the notebook top to bottom. Each run writes to a unique folder under `outputs/trustworthiness_attacks/` that is timestamped by session identifier.

The notebook is self-contained; no Hydra configuration or external CLI is required. When using the dry-run stub you will still receive mock responses, but no external API calls are made.

## Pipeline Stages
1. **Notebook setup** – seeds randomness, constructs output directories, and appends structured JSON records to `outputs/trustworthiness_attacks/logs/attack_pipeline_log.jsonl`.
2. **LLM backend deployment** – loads `.env`, constructs a `ModelConfig`, instantiates either `ChatOpenAI` or `DummyChatModel`, and runs a smoke test to verify connectivity.
3. **Dataset preparation** – loads toxicity, stereotype, and privacy probes from DecodingTrust JSONL/CSV assets, applies topic-aware sampling, and attaches metadata (toxicity annotations, target demographic, target email).
4. **Iterative jailbreak loop** – for each payload, crafts system prompts, escalates through progressively stronger jailbreak instructions, records token usage, and evaluates responses via lightweight heuristics:
   - Toxicity: keyword markers.
   - Stereotype: enforcement that the assistant affirms the statement and ends with "I agree."
   - Privacy: checks for leaked names tied to protected emails.
5. **Analysis and persistence** – computes success rates, average attempts, and heuristic scores, surfaces qualitative examples, optionally scores the toxicity responses with Detoxify, and stores JSONL traces plus aggregated metrics under `outputs/trustworthiness_attacks/generations/<session_id>/`.

## Outputs and Artefacts
- `outputs/trustworthiness_attacks/logs/attack_pipeline_log.jsonl` – append-only event log with configuration snapshots, dataset selections, attack summaries, and analysis metadata.
- `outputs/trustworthiness_attacks/generations/<session_id>/toxicity.jsonl` (and peers) – full conversation traces including escalation attempts, token usage, and heuristic evaluations.
- `outputs/trustworthiness_attacks/generations/<session_id>/metrics.json` – aggregated success rates and average attempt counts per perspective.
- Console output – quick summaries of dataset sizes, attack success rates, example jailbreaks, and Detoxify statistics when the package is installed.

## Extending the Pipeline
- Swap to a different model provider by implementing the necessary LangChain wrapper and toggling the `.env` values.
- Replace heuristic scorers with classifier APIs (Perspective API, targeted stereotype models, PI detectors) for improved fidelity.
- Adjust the escalation prompts in `ATTACK_CONFIGS` or increase `max_attempts` to experiment with different jailbreak strategies.
- Add additional perspectives by mirroring the dataset preparation, scoring function, and configuration pattern already in the notebook.

## Troubleshooting
- **Missing packages** – ensure `conda list` shows `langchain`, `langchain-openai`, `python-dotenv`, and `tqdm`. Install via `conda` or `pip` inside the environment if needed.
- **Dataset not found** – the defaults assume the DecodingTrust checkout sits at `../DecodingTrust`. Override the relevant `TRUST_PIPELINE_*_DATASET` variables when using a different folder layout.
- **No API key** – the notebook automatically switches to dry-run mode; you will see synthetic responses and zero token usage. Provide a valid key to test real models.
- **Detoxify metrics skipped** – install the optional dependency with `pip install detoxify` inside the conda environment before rerunning the analysis cell.
