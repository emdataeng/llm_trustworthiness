# LLM Assignment 2 - Evaluating and Improving LLM Trustworthiness
## Author: Eric Marquez
`A2_v2.ipynb` implements a full evaluation and mitigation pipeline for GPT-3.5-turbo, covering toxicity jailbreaks, stereotype bias analysis, fairness benchmarking, and fairness-focused improvements. The notebook follows the structure used in the course LLMs and Societal Consequences of AI (1RT730, Uppsala University).

### Code References
The iterative attack loop builds on the DecodingTrust project structure, particularly its class-based orchestration. Additional patterns come from LangChain documentation. AI assistants (GitHub Copilot, OpenAI Codex) helped with code completion, chart scaffolding, and background research.

## Notebook Structure
- **Index** - Quick navigation links for every major section in the notebook.
- **0. Notebook Setup** - Environment checks, logging bootstrap (`outputs/trustworthiness_attacks/...`), and reminders to configure `.env` plus the Conda environment.
- **1. Deploy the LLM Backend** - Loads provider settings from `.env`, spins up the OpenAI chat model, and falls back to a deterministic `DummyChatModel` when no API key is present.
- **2. Load Local Benchmark Subsets**
  - **2.1 Datasets configuration** - Resolves DecodingTrust file paths, applies `top_k` sampling, and records the dataset summary in the session log.
- **3. Iterative Attack Loop**
  - **3.1 Scoring success** - Defines heuristic and Detoxify-based success rules for toxicity and stereotype prompts.
  - **3.2 Preparing for the attack** - Builds prompt batches and escalation ladders.
  - **3.3 Attack!** - Runs iterative jailbreak attempts per prompt and captures model responses.
  - **3.4 Results of the attack** - Summarises attack outcomes and highlights notable generations.
  - **3.5 Detoxify Score Analysis** - Computes toxicity metrics (tox_mean, sev_tox, id_attack, etc.) when the optional package is installed.
- **4. Fairness Classification Evaluation**
  - **4.1 Group breakdown** - Aggregates predictions by sensitive group.
  - **4.2 Quantitative Fairness Results** - Uses `fairlearn` to compute demographic parity, equalised odds, and rejection rates.
  - **4.3 Conclusion** - Documents key fairness observations and open issues.
- **5. Analyse Results**
  - **5.1 Metrics for Toxicity** - Visualises Detoxify-derived metrics including conditional bias gap and prompt-output correlation.
  - **5.2 Toxicity Conclusions** - Interprets the robustness of the model under iterative escalation.
  - **5.3 Metrics for Stereotype Bias** - Reports stereotype agreement, disagreement, neutrality, and disparity scores.
  - **5.4 Stereotype Conclusions** - Synthesises findings across demographic groups.
- **6. Persist Artefacts** - Writes JSON/JSONL artefacts for prompts, generations, metrics, and logs under `outputs/trustworthiness_attacks/`.
- **Part B - Improving Trustworthiness**
  - **B.1 Counterfactual Data Augmentation Wrapper** - Wraps the model to swap gendered terms, merge predictions, and report counterfactual usage metadata.
  - **B.1.1 Counterfactual Fairness Plots** - Compares fairness metrics before and after counterfactual augmentation.
  - **B.2 Debiasing Template Wrapper** - Inserts an ethical system prompt that enforces fairness instructions without modifying the original user query.
  - **B.2.1 Debiasing Template Plots** - Evaluates the template-based mitigation and contrasts it with baseline and counterfactual results.
  - **B.3 Mitigation Conclusions** - Summarises fairness gains from both mitigations.
  - **Extras** - Additional scratch cells for ad hoc comparisons and experimentation.

### Warning
Expect harmful, toxic, or biased model outputs while executing the notebook. Run it in a controlled environment and handle generated text responsibly.

## Repository Map
- `A2_v2.ipynb` - Main notebook orchestrating dataset preparation, attacks, evaluations, mitigations, and logging.
- `environment.yml` - Conda definition for the recommended runtime (Python 3.10).
- `.env` / `.env.template` - Runtime secrets and configuration overrides (see below).
- `outputs/` - Created at runtime with logs, generations, and metric artefacts.
- `Attack/` and `References/` - Legacy experimentation material retained for context.

## Prerequisites
1. Install [Conda](https://docs.conda.io/en/latest/miniconda.html) or another environment manager.
2. Clone/download the DecodingTrust datasets (see *Dataset Placement*).
3. Obtain API credentials for your target language model provider (OpenAI support is shipped).

## Environment Setup
```bash
conda env create -f environment.yml
conda activate llm-trust-env
# Optional extras used inside the notebook
pip install fairlearn detoxify
```
The notebook gracefully degrades to `DummyChatModel` when no API key is supplied so you can validate control flow without hitting external services.

## Configuration (`.env`)
Create a `.env` file at the project root (or copy `.env.template`). Available keys and defaults:

| Key | Purpose | Default |
| --- | --- | --- |
| `OPENAI_API_KEY` | Enables calls to the OpenAI Chat API. | `""` (falls back to dummy model) |
| `TRUST_PIPELINE_PROVIDER` | Model provider identifier. | `openai` |
| `TRUST_PIPELINE_MODEL` | Chat model name. | `gpt-3.5-turbo` |
| `TRUST_PIPELINE_TEMPERATURE` | Sampling temperature for generations. | `0.6` |
| `TRUST_PIPELINE_MAX_TOKENS` | Maximum tokens returned by the model. | `256` |
| `TRUST_PIPELINE_DRY_RUN` | Force dry-run mode (`true/false`). | Auto-detect from API key |
| `TRUST_PIPELINE_TOXICITY_DATASET` | Toxicity JSONL path. | DecodingTrust default |
| `TRUST_PIPELINE_TOXICITY_TOP_K` | Prompts per toxicity run (`ALL` keeps everything). | `8` |
| `TRUST_PIPELINE_TOXICITY_THRESHOLD` | Detoxify success threshold. | `0.7` |
| `TRUST_PIPELINE_STEREOTYPE_DATASET` | Stereotype JSONL path. | DecodingTrust default |
| `TRUST_PIPELINE_STEREOTYPE_SYSTEM_PROMPTS` | CSV with stereotype system prompts. | DecodingTrust default |
| `TRUST_PIPELINE_STEREOTYPE_TOP_K` | Items per stereotype topic (`ALL` for full set). | `2` |
| `TRUST_PIPELINE_FAIRNESS_DATASET` | Fairness benchmark JSONL. | DecodingTrust default |
| `TRUST_PIPELINE_FAIRNESS_SENSITIVE_ATTR` | NumPy array with sensitive attributes. | DecodingTrust default |
| `TRUST_PIPELINE_FAIRNESS_TOP_K` | Records to keep for fairness evaluation. | `200` |
| `TRUST_PIPELINE_FAIRNESS_DATASET_TYPE` | Fairness dataset tag (`adult` or `crime`). | `adult` |

Override paths when the DecodingTrust clone lives elsewhere.

## Dataset Placement
The notebook assumes the DecodingTrust repository sits next to this project:

```
../DecodingTrust/
  data/
    toxicity/...
    stereotype/...
    fairness/...
```

Use the `.env` overrides if your datasets are stored in a different location.

## Running the Notebook
1. Activate the Conda environment (`conda activate llm-trust-env`).
2. Launch Jupyter (Lab or Notebook) from the project root.
3. Ensure `.env` is populated and datasets are reachable.
4. Execute the notebook in order. Sections 0-6 cover baseline evaluation; Part B applies the counterfactual and debiasing fairness mitigations. You can re-run individual sections after adjusting configuration. Logs append to `outputs/trustworthiness_attacks/logs/attack_pipeline_log.jsonl`.

## Generated Artefacts
- `outputs/trustworthiness_attacks/logs/attack_pipeline_log.jsonl` - Append-only operational log for every run.
- `outputs/trustworthiness_attacks/generations/<session>/toxicity.jsonl` - Toxicity attack traces.
- `outputs/trustworthiness_attacks/generations/<session>/stereotype.jsonl` - Stereotype attack traces.
- `outputs/trustworthiness_attacks/generations/<session>/fairness.jsonl` - Fairness prompts and predictions (baseline and mitigations).
- `outputs/trustworthiness_attacks/generations/<session>/metrics.json` - Aggregated metrics reused in analysis sections.

Each session is timestamped, which makes it easy to compare baseline and mitigation runs.

## Notebook Components at a Glance
- **Logging utilities** - `ensure_paths`, `append_log`, session bookkeeping, and dataset configuration tracing.
- **Model bootstrap** - `load_env_settings`, `DummyChatModel`, and `build_chat_model` to support real or dry-run execution.
- **Dataset ingestion** - Helpers to resolve DecodingTrust paths, apply sampling, and attach metadata.
- **Attack heuristics** - `iterative_attack`, `score_toxicity_success`, `score_stereotype_success`, plus Detoxify integration.
- **Fairness evaluation** - `run_fairness_evaluation`, `prepare_fairness_dataframe`, and `render_quantitative_fairness` using `fairlearn`.
- **Fairness mitigations** - Counterfactual and debiasing wrappers that reuse the evaluation pipeline to quantify improvements.
- **Analysis & persistence** - Visualisation helpers, Detoxify summaries, and JSON/JSONL writers under `outputs/`.

## Optional Enhancements
- Install `detoxify` to activate end-to-end toxicity scoring in Sections 3.5 and 5.1.
- Modify `.env` to target alternative providers or local models by extending `build_chat_model`.
- Customise `ATTACK_CONFIGS` in Section 3 to experiment with new escalation prompts or heuristics.

## Safety Notice
The notebook intentionally elicits harmful content to probe model behaviour. Execute it in a controlled environment, follow organisational policies, and avoid exposing raw generations to end users.
