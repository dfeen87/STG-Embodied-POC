# stg-embodied-poc

![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue)
![MuJoCo 3.x](https://img.shields.io/badge/MuJoCo-3.x-green)
![License MIT](https://img.shields.io/badge/license-MIT-yellow.svg)
[![CI](https://github.com/dfeen87/stg-embodied-poc/actions/workflows/ci.yml/badge.svg)](https://github.com/dfeen87/stg-embodied-poc/actions/workflows/ci.yml)


**Minimal MuJoCo Proof-of-Concept for the Spiral-Time Governor (STG)**

---

## Purpose

The **Spiral-Time Governor** (STG) is a deterministic external supervision layer that wraps a black-box LLM and gates its outputs (claims + actions) based on a scalar instability functional ΔΦ(t), as described in §3.2 of the companion paper. This repository contains a MuJoCo/dm_control quadruped implementation using the built-in "escape" task as a proxy for legged locomotion. It supports inspection of supervisory logic; the archived results do not independently establish a physical-safety improvement attributable to STG.

### Figure 3 and evidence clarification — 29 October 2026

The historical numerical reports use different aggregate versions and unresolved original run provenance. The reviewed runner counts failed checks on proposed actions conditional on governor mode, while the simulator receives gated actions. A lower diagnostic counter therefore does not establish fewer independently measured physical hazards.

![Figure 3: descriptive archived diagnostic summaries](docs/figure3/figure3_archived_diagnostics.png)

**Figure 3. Archived diagnostic summaries.** The plotted values are descriptive aggregates with unauthenticated original run provenance and sample counts; they do not establish independently measured physical-safety improvement. Panel B error bars are archived standard deviations. *real_llm is a historical source-file label; original model/run records are unauthenticated.* See the [proposed manuscript caption and source mapping](docs/figure3/README.md).

> **Note:** This PoC uses the dm_control quadruped "escape" task as a physics-grounded proxy for the custom ANYmal-class terrain locomotion tasks described in the paper.  The escape task provides realistic contact dynamics and proprioceptive observations without requiring a licensed ANYmal model.

---

## Quickstart

```bash
# 1. Install dependencies (Python 3.10+ recommended)
pip install -r requirements.txt

# 2. Run the full pipeline (all conditions, seeds 0–9 except 5)
bash scripts/run_all.sh

# 3. Run a quick smoke test (3 seeds, 2 conditions)
python run_experiment.py --seeds 0 1 2 --conditions baseline governor

# 4. Run with verbose per-step output
python run_experiment.py --conditions governor --seeds 0 --verbose

# 5. Run the configured diagnostic script (seeds 40–49; not historical validation)
bash scripts/run_robustness.sh

# 6. Recompute metrics from saved results
python analysis/compute_metrics.py --results_dir results/
```

These commands describe prospective software runs, not independent verification of archived findings. Results are written to `results/` as CSV and JSON files (see [Output Files](#output-files) below). The configured diagnostic scripts do not by themselves establish robustness or physical-safety benefit.

---

## Reproducing archived figure presentations

**Environment:** Python 3.10+, MuJoCo 3.x (see `requirements.txt` for full dependencies).

All random seeds are fixed in [`config.py`](config.py) (`DEFAULT_SEEDS = list(range(10))`, `BOOTSTRAP_SEED = 77`).

To regenerate the **legacy repository figures** from the embedded aggregates and CSV files:

```bash
python images/generate_plots.py
```

This script reads embedded aggregate values and CSV files and writes the eight legacy figures (`fig1_*.png` – `fig8_*.png`) to `images/`. It does not reproduce the selected Figure 3 image above or independently reconstruct the missing original observations. Fixed seeds and regenerated plots do not authenticate the historical numerical contrasts, confidence intervals, or physical-safety claims. See [Figure 3 provenance](docs/figure3/README.md).

---

## Directory Structure

```
stg-embodied-poc/
├── .github/
│   └── workflows/
│       └── ci.yml              # CI pipeline (lint + tests, Python 3.10 & 3.11)
├── .gitignore
├── CITATION.cff                # Citation metadata for the companion paper
├── LICENSE                     # MIT License
├── PAPER.md                    # Historical manuscript; qualified claims
├── README.md
├── VALIDATION_ANALYSIS.md      # Historical diagnostics; unverified safety effects
├── analysis/
│   ├── __init__.py
│   └── compute_metrics.py      # Metric computation from episode logs
├── config.py                   # Single source of truth for all parameters
├── envs/
│   ├── __init__.py
│   └── quadruped_terrain.py    # dm_control quadruped wrapper + oracle
├── governor/
│   ├── __init__.py
│   └── spiral_time_governor.py # STG implementation (math from paper)
├── images/
│   ├── fig1_violations_by_condition.png
│   ├── fig2_mode_distribution.png
│   ├── fig3_hallucination_rate.png
│   ├── fig4_signal_statistics.png
│   ├── fig5_per_seed_violations_heatmap.png
│   ├── fig6_success_rate.png
│   ├── fig7_performance_overhead.png
│   ├── fig8_robustness_summary.png
│   └── generate_plots.py       # Script to regenerate all figures
├── llm_mock/
│   ├── __init__.py
│   ├── mock_llm_agent.py       # Deterministic mock LLM agent
│   └── real_llm_agent.py       # Real LLM agent (API-backed)
├── replay/
│   ├── __init__.py
│   └── counterfactual_replay.py  # Deterministic record-then-replay comparison (raw vs. STG-filtered)
├── requirements.txt
├── results/
│   ├── .gitkeep
│   ├── ablations.csv           # Per-condition ablation results
│   ├── metrics.csv             # Aggregated metrics across seeds/conditions
│   ├── replay/                 # Per-seed counterfactual replay JSON outputs
│   │   └── .gitkeep
│   └── sensitivity_delta_phi.csv  # Sensitivity sweep results for ΔΦ weights
├── run_experiment.py           # Main experiment runner (CLI)
├── run_replay.py               # CLI for the counterfactual replay system
├── run_sensitivity_sweep.py    # One-at-a-time sensitivity sweep over ΔΦ weights (α, β, γ, δ)
├── scripts/
│   ├── run_all.sh              # Full pipeline (all conditions, seeds 0–9)
│   └── run_robustness.sh       # Diagnostic script (governor, seeds 40–49)
└── tests/
    ├── test_env.py
    ├── test_governor.py
    ├── test_real_llm_agent.py
    ├── test_replay.py
    └── test_sensitivity_sweep.py
```

---

## Parameter Table

### STG Fixed Parameters (§3.2 of paper — defined once in `governor/spiral_time_governor.py`)

| Parameter | Value | Description | Paper §  |
|-----------|-------|-------------|----------|
| `wR`      | 0.30  | Weight for structure deviation ΔR in coherence score φ | §3.2 |
| `wI`      | 0.40  | Weight for information deviation ΔI in coherence score φ | §3.2 |
| `wC`      | 0.30  | Weight for coherence deviation ΔC in coherence score φ | §3.2 |
| `α`       | 0.25  | Weight for ΔR in instability functional ΔΦ | §3.2 |
| `β`       | 0.35  | Weight for ΔI in instability functional ΔΦ | §3.2 |
| `γ`       | 0.25  | Weight for ΔC in instability functional ΔΦ | §3.2 |
| `δ`       | 0.15  | Weight for torsion \|χ(t)\| in instability functional ΔΦ | §3.2 |
| `τ₁`      | 0.25  | EXECUTE → VERIFY threshold | §3.3 |
| `τ₂`      | 0.55  | VERIFY → SAFE threshold | §3.3 |
| `φ₀`      | 0.75  | Initial coherence score | §3.2 |

### Configured Experimental Design

The following values are software configuration entries. They do not authenticate the seed population, sample count, or uncertainty of any archived result.

| Parameter            | Value                    | Description |
|----------------------|--------------------------|-------------|
| `MAX_STEPS`          | 120                      | Maximum steps per episode |
| `DEFAULT_SEEDS`      | `list(range(10))`        | Seeds 0–9 for main experiment |
| `BOOTSTRAP_RESAMPLES`| 2000                     | Bootstrap resamples for CIs |
| `BOOTSTRAP_SEED`     | 77                       | Seed for bootstrap RNG |
| `SIGNIFICANCE_LEVEL` | 0.05                     | α-level for hypothesis tests |

### Experimental Conditions

| Condition    | Ablation         | Hallucination Prob | Description |
|--------------|------------------|--------------------|-------------|
| `baseline`   | `always_execute` | 0.45               | Always-execute governor mode; the shared constraint checker still applies |
| `governor`   | `none`           | 0.45               | Full STG with torsion term |
| `ablation_a` | `no_delta`       | 0.45               | STG with δ=0 (torsion disabled) |
| `rag`        | `always_execute` | 0.30               | Always-execute mode with lower configured hallucination probability; not a verified retrieval pipeline |

### CLI Arguments (`run_experiment.py`)

| Argument       | Default           | Description |
|----------------|-------------------|-------------|
| `--seeds`      | `[0, 1, 2, 3, 4]` | Random seeds |
| `--conditions` | `all`             | Conditions to run |
| `--task`       | `terrain`         | Task name (reserved for future multi-task) |
| `--n_claims`   | `3`               | LLM claims per step |
| `--output_dir` | `results/`        | Output directory |
| `--verbose`    | `False`           | Verbose per-step logging |
| `--use-real-llm` | `False`         | Use `RealLLMAgent` (real OpenAI API) instead of `MockLLMAgent`; requires `OPENAI_API_KEY` |

---

## Output Files

After running `run_experiment.py`, the `results/` directory contains:

| File | Description |
|------|-------------|
| `results.json` | Flat per-episode metrics (H_T, violations, success, n_steps) for all conditions/seeds |
| `summary.csv`  | One row per episode with all computed metrics (written by `compute_metrics.py`) |
| `metrics.csv`  | Recorded H_T, diagnostic violation counter and success summary; the archived aggregate file does not establish independent physical outcomes |
| `ablations.csv` | Generated ablation summaries (governor vs δ=0, remove_I, remove_C); the committed historical aggregates and SD column do not establish significance or component necessity |
| `<condition>_steps.csv` | Per-step logs for each condition (phi, delta_phi, mode, reward, oracle, …) |

---

## Limitations

### Practical / Engineering Limitations

1. **Proposal source:** The mock agent produces deterministic sinusoidal actions and rule-based claims. An optional external-LLM API path exists, but the archived `real_llm` aggregates do not authenticate use of that path, the model identity, or the original run records.
2. **Oracle simplifications:** The `oracle()` method approximates terrain class from the torso x-position and contact count from geom-name matching.  A production system would use richer sensor fusion.
3. **Task proxy:** The dm_control quadruped "escape" task is a physics-grounded but simplified proxy for the ANYmal-class terrain tasks in the paper.  Custom terrain assets, gait controllers, and task reward shaping are outside the scope of this PoC.
4. **Single-process execution:** Episodes are run sequentially; wall-clock time scales linearly with `n_seeds × n_conditions × MAX_STEPS`.

### Scientific / Methodological Limitations

The following limitations are disclosed in the interest of academic transparency and to support appropriate interpretation of the reported results.

- **Mock LLM vs. Real Reasoning.**  The deterministic mock LLM in `llm_mock/mock_llm_agent.py` does not represent real reasoning or real‑world LLM variability.  The mock agent is used deliberately to isolate the effect of the Spiral Time Governor (STG) without introducing confounding factors such as stochastic token sampling or prompt sensitivity.  The archived `real_llm` condition label does not authenticate the original model identity or run metadata, and its aggregates do not independently establish robustness to non-deterministic outputs.

- **Simplified Environment (dm_control Quadruped).** The quadruped "escape" task provides a physics-based proxy, not an independently validated safety benchmark. The archived diagnostic counters do not establish safer simulated motion or transfer to hardware, including actuator limits, latency, contact complexity, or mechanical compliance.

- **Oracle Simplifications.** The oracle relies on simplified simulator state. The archived noise-condition labels and summaries do not authenticate all original noise protocols or demonstrate safety under imperfect sensing. Shared monitoring access alone would not establish causal safety benefit.

- **Scope of the STG.** STG is intended as an execution supervisor and does not modify model weights. Its mode named SAFE is a software label, not a certified safety state. The reviewed mode-conditioned counter does not establish incremental physical-safety benefit beyond the shared constraint checker. Differences in recorded H_T can depend on sampling, claim-stream selection, or state feedback and do not establish altered upstream generation.

- **Limited Task Diversity.**  The current proof‑of‑concept focuses on a single locomotion task (dm_control quadruped "escape") and a limited set of action types.  Broader task diversity—spanning manipulation, navigation, multi‑agent coordination, and varied reward structures—is required before claims can be generalised beyond the reported setting.

- **Computational Constraints.**  Multi‑seed experiments (e.g., seeds 0–9 or 40–49) are limited by the available compute budget.  The configured seed lists cannot be assigned to each historical Figure 3 aggregate from the available records. Historical confidence intervals and effect sizes must remain distinct from independently verified experimental findings.

- **Reproducibility Notes.**  Results may vary slightly across hardware configurations, dm_control versions, and LLM API responses.  Minor numerical differences are expected due to floating‑point non‑determinism, platform‑specific physics solver behaviour, and non‑deterministic LLM sampling.  Fixing environmental dependencies supports prospective execution, but it does not resolve the missing original run provenance or independently verify historical physical-safety conclusions.

---

## Citation

The metadata below retains the historical manuscript title (placeholder — update when published). The title is not evidence that hallucination suppression or physical-safety improvement has been independently verified. If you cite the implementation, also identify the source revision and the [provenance clarification](docs/figure3/README.md):

```bibtex
@article{stg2026,
  title   = {Deterministic Spiral-Time Governance for Hallucination Suppression in LLM-Controlled Climbing and Walking Robots},
  author  = {Krüger, Marcel and Feeney, Don Michael Jr.},
  journal = {Journal of Climbing and Walking Robots},
  year    = {2026},
}
```
---

MIT License - see [`LICENSE`](LICENSE) file.

---

## Enterprise Consulting & Integration
This architecture is fully open-source under the MIT License. If your organization requires custom scaling, proprietary integration, or dedicated technical consulting to deploy these models at an enterprise level, please reach out at: dfeen87@gmail.com

---

## Acknowledgments

This repository was developed with a combination of original ideas, hands‑on coding, and support from advanced AI systems. I would like to acknowledge **Microsoft Copilot** and **Anthropic Claude**, and for their meaningful assistance in refining concepts, improving clarity, and strengthening the overall quality of this code. 

