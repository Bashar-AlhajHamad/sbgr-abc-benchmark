# SBGR: A Surrogate Bandgap-Reference Benchmark for Constrained Metaheuristic Optimization

This repository contains the benchmark, optimizers, and experiment scripts for the study:

> **Artificial Bee Colony for Constrained Optimization in 6G-Motivated Analog Integrated Circuit Design via Surrogate Evaluation**
> (under review, *Applied Soft Computing*).

It reproduces all experiments in the paper: the main algorithm comparison and the penalty-scaling sensitivity analysis.

---

## What is SBGR?

**SBGR (Surrogate Bandgap-Reference)** is a deterministic, closed-form benchmark that mirrors the *optimization structure* of bandgap-reference (BGR) analog transistor sizing: maximizing power-supply rejection ratio (PSRR) subject to several coupled specifications (reference voltage, temperature coefficient, loop gain, phase margin, gain margin, and power). It is designed for fair, reproducible, simulator-independent comparison of constrained metaheuristics.

Given a design vector, SBGR computes six BGR-inspired metrics and PSRR from fixed closed-form expressions, then a penalty-based fitness

```
f(x) = -PSRR(x) + lambda * penalty(x)      # lambda = 1000 by default
```

where `penalty(x)` counts violated specifications (a solution is feasible when `penalty = 0`). Three cases of increasing difficulty are provided:

| Case      | Dimension | Description                                  |
|-----------|:---------:|----------------------------------------------|
| `base`    | 12        | Nominal specifications                       |
| `hard`    | 12        | Tightened specifications                     |
| `highdim` | 30        | Higher-dimensional variant (scalability)     |

> **Important — scope.** SBGR metrics are **dimensionless surrogate quantities on internally consistent scales**, calibrated to reproduce the optimization difficulty of BGR sizing (coupled metrics, a narrow feasible region, multimodality). They are **not** physical units (volts, ppm/°C, dB, µW) and SBGR is **not** a circuit simulator. It targets *relative* algorithm comparison, not absolute circuit-performance prediction. See the paper for the full metric definitions and the fidelity/scope discussion.

---

## Repository contents

| File                      | Description                                                                 |
|---------------------------|-----------------------------------------------------------------------------|
| `problems.py`             | The SBGR benchmark: metric model, constraints, penalty, and fitness.        |
| `algorithms.py`           | Six optimizers (ABC, GWO, FA, PSO, GA, ACO/ACOR) + a shared evaluation wrapper. |
| `run.py`                  | Main comparison campaign; writes per-case and combined results and figures. |
| `penalty_sensitivity.py`  | Penalty-scaling sensitivity study (sweeps `lambda`).                        |
| `requirements.txt`        | Python dependencies.                                                        |

---

## Requirements

- Python 3.10+
- `numpy`, `pandas`, `scipy`, `matplotlib` (and `Jinja2` for optional LaTeX table export)

Install:

```bash
pip install -r requirements.txt
```

---

## Reproducing the paper's experiments

All runs are deterministic given the seed. Each `(case, run)` uses a fixed seed shared by every algorithm (paired design), so comparisons are fair.

### 1. Main comparison campaign (540 runs)

Six algorithms × three cases × 30 runs, population 40, evaluation budgets 150,000 (base/hard) and 220,000 (highdim):

```bash
python run.py \
  --cases base hard highdim \
  --pop 40 --runs 30 --seed 42 \
  --max-evals 150000 --highdim-max-evals 220000 \
  --outdir results_main
```

> Note: `run.py`'s built-in defaults are smaller (pop 30, 30,000 evaluations, 20 runs, `base` only) for quick tests. The command above reproduces the published configuration.

### 2. Penalty-scaling sensitivity (2,160 runs)

The same protocol repeated for `lambda ∈ {10, 100, 1000, 10000}`:

```bash
python penalty_sensitivity.py \
  --runs 30 --pop 40 --seed 42 \
  --max-evals 150000 --highdim-max-evals 220000 \
  --lambdas 10,100,1000,10000 \
  --outdir results_penalty
```

Both scripts print progress and, when finished, write CSV summaries (and, for `run.py`, figures) into the chosen `--outdir`. The full campaigns take on the order of hours on a desktop CPU; reduce `--runs` and `--max-evals` for a faster smoke test.

---

## Outputs

`run.py` writes, per case, a `case_<name>/` folder containing:

- `summary_<case>.csv` — one row per run per algorithm (final fitness, feasibility, metrics, runtime);
- statistics CSVs (descriptive stats, violation counts, ranks, Friedman, pairwise Wilcoxon + Holm);
- figures: `convergence_sbgr-<case>.png/.pdf`, `boxplot_fitness_sbgr-<case>.png/.pdf`, `violations_bar_sbgr-<case>.png/.pdf`.

Combined files (`summary.csv`, overall ranks, overall Friedman) are written at the top level of `--outdir`.

`penalty_sensitivity.py` writes `per_run_records.csv`, `summary_by_lambda.csv`, and a plain-text `verdict.txt`.

---

## Citation

If you use this benchmark or code, please cite the paper (details to be updated on publication):

```bibtex
@article{sbgr2026,
  title   = {Artificial Bee Colony for Constrained Optimization in 6G-Motivated
             Analog Integrated Circuit Design via Surrogate Evaluation},
  author  = {Alhaj Hamada, Bashar Aqel Younis and Y{\i}ld{\i}z, Do{\u{g}}an and
             {\c{S}}ahin, Durmu{\c{s}} {\"O}zkan and Demirci, Sercan and Aslan, Sel{\c{c}}uk},
  journal = {Applied Soft Computing},
  year    = {2026},
  note    = {Under review}
}
```

> Please confirm the final author order and update the entry (volume, pages, DOI) once the paper is published.

---

## License

Released under the MIT License — add a `LICENSE` file to the repository (MIT is a common, permissive choice for research code; you may select a different license if preferred).

---

## Contact

For questions about the benchmark or code, please contact the corresponding author (see the paper).