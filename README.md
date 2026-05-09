# Code release — ICML 2026 Mech Interp Workshop submission

Anonymized code for the experiments in *"Linear Probes for Strategic
Deception Generalize Across Three LLM Families in Multi-Agent
Negotiation."*

## Layout

- `concordia_mini/` — minimal Concordia fork; multi-agent framework
- `negotiation/` — negotiation-specific agent components
- `config/` — Pydantic experiment configuration with model presets
- `interpretability/`
  - `core/` — TransformerLens activation capture, DeepEval LLM-judge
    detector, ground-truth scoring, QC filter
  - `probes/` — mass-mean and ridge probe training, sanity checks,
    mech-interp tools
  - `causal/causal_validation.py` — probe faithfulness, logit-level
    steering, behavioral steering with Spearman + permutation criterion
  - `scenarios/` — three negotiation scenarios (ultimatum bluff,
    alliance betrayal, information withholding)
  - `evaluation.py` — `InterpretabilityRunner` orchestrator
  - `run_deception_experiment.py` — entry point for activation
    collection
- `run_causal.py` — entry point for probe-faithfulness ablation and
  logit-level steering (Tables 2 and 4 in the paper)
- `run_behavioral_steering.py` — entry point for behavioral
  generate-under-steering test (deferred in the paper)

## Install

Tested on Python 3.10+ with PyTorch 2.7 and CUDA 12.4.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install torch --index-url https://download.pytorch.org/whl/cu124
pip install -r requirements.txt
pip install -e .
```

## Reproducing the paper's experiments

### Collect activations (GPU)

```bash
python interpretability/run_deception_experiment.py \
  --model google/gemma-7b-it \
  --scenario-name ultimatum_bluff \
  --trials 50 --max-rounds 5 \
  --device cuda --dtype bfloat16
```

Repeat for the three scenarios `{ultimatum_bluff, alliance_betrayal,
info_withholding}` and the three models
`{google/gemma-7b-it, meta-llama/Llama-3.1-8B-Instruct,
mistralai/Mistral-7B-Instruct-v0.1}`.

### Causal validation (GPU)

```bash
python run_causal.py --scenario ultimatum_bluff
python run_causal.py --scenario alliance_betrayal
python run_causal.py --scenario info_withholding
```

Runs probe-faithfulness ablation (Table 2) and logit-level steering
dose-response (Table 4) per scenario across the three model
families.

### Behavioral steering (GPU; explicitly deferred in the paper)

```bash
python run_behavioral_steering.py \
  --n-samples 100 --evaluator-type llm
```

Uses the GPT-4o-mini judge through DeepEval (set `OPENAI_API_KEY`).
Pass criterion: Spearman rank correlation `|rho| >= 0.5` with
permutation `p < 0.05` over 10,000 shuffles, plus
`control_ratio >= 2x` against a matched random direction.

## Hyperparameters

- Ridge regularization: `alpha = 100`
- PCA components: 30
- Train-test split: 80/20 stratified, seed 42
- Bootstrap: 200 resamples, seeds 42 through 241
- Sample-type filter excludes pre/post-negotiation
  belief-verification probes (round_num -1 and -2)

## Compute

- Activation capture and steering: 1x H100 80GB. Behavioral steering
  with `n_samples=100` and the LLM judge runs ~24h across 9 cells.
- Probes and analysis: CPU-only, ~3h total.

## License

AGPL-3.0. Concordia portions retain Apache 2.0. Activation datasets
referenced in scripts use anonymized HuggingFace namespaces; the real
namespaces will be revealed at de-anonymization.
