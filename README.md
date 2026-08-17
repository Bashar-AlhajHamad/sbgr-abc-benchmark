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
| `protocol_sweep.py`       | Protocol-choice sweep (dimension, population, budget, seed families), and the independent replication of the published `base` cell. |
| `sbgr_controls.py`        | Controls testing alternative explanations for the surrogate-to-circuit disagreement: dimension, constraint pressure, unmeasurable regions. |
| `requirements.txt`        | Python dependencies.                                                        |

**Section 6 — transistor-level cross-check on SKY130** (see the dedicated section below):

| File                      | Description                                                                 |
|---------------------------|-----------------------------------------------------------------------------|
| `run_spice.py`            | Circuit-level campaign driver — same protocol as `run.py`, ngspice evaluator. |
| `spice/spice_problem.py`  | The ngspice-backed problem: netlist rendering, `.measure` parsing, six specifications, penalized fitness. |
| `spice/ngspice_bridge.py` | Persistent ngspice server (`alterparam`/`reset`), so the PDK is parsed once per worker. |
| `spice/templates/`        | The SKY130 bandgap decks (`bgr_sky130.cir.tmpl` and its higher-dimensional variant). |
| `truba/`                  | Cluster scripts: ngspice built from source, PDK fetched at a pinned commit, SLURM job arrays. |
| `pvt_reference_control.py`| Runs the untouched published reference design through all fifteen process/supply conditions. |
| `pvt_summary_fix.py`      | Writes a corrected PVT summary beside the shipped one — see *Known issues*.  |
| `corner_verify.py`, `verify_*.py`, `preflight_spice.py`, `audit_before_launch.py` | Pre-launch and post-hoc checks: grid independence, threshold freeze, anchor reproduction, population and budget audits. |

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

## Section 6 — transistor-level cross-check on a SKY130 bandgap reference

The surrogate buys statistical power, and that leaves one question open: **does an optimizer ranking obtained on SBGR say anything about the same ranking on a real netlist?** Section 6 of the paper answers it by re-posing the identical optimization problem — same objective, same six specifications, same penalized fitness, same protocol — against a transistor-level Kuijk bandgap reference in the open-source SkyWater SKY130 process, evaluated with `ngspice`.

**The measured answer is no, and it is reported as such.** GWO attains the best average rank in all three circuit cases; ABC ranks third, fifth and second. What does transfer is feasibility attainment (ABC, GWO and ACO all reach 100 % on the one circuit case where that measure discriminates) and the absence of collapsed designs. Separately, **no design produced by any optimizer survives all fifteen process and supply conditions — 0 of 540 — and neither does the published reference design, which passes 5 of 15.**

### Additional requirements

- **`ngspice` 46**, built from source (see `truba/00_setup_native.sh`).
- **SKY130 PDK**, `sky130A` variant, fetched with [`volare`](https://github.com/efabless/volare) at a pinned commit. The PDK is **not vendored here** — it is 216 MB and the setup script retrieves exactly the right build.

### PDK provenance — read this before reproducing

The kit ships **more than one model library** under the same commit, and the results depend on which one is used. Every number in Section 6 comes from:

| | |
|---|---|
| library | `sky130A/libs.tech/combined/sky130.lib.spice` — the standard **binned** set, **not** `combined/continuous/` |
| MD5 | `365ab743568de364c2214767735a89c6` |
| `open_pdks` commit | `c6d73a35f524070e85faff4a6a9eef49553ebc2b` (via `volare`) |
| simulator | `ngspice` 46, built with `gcc (GCC) 11.3.1` |

`env.sh`, generated by the setup script on the cluster before the campaign, records all of this and is sourced by every job. **Check its MD5 line matches the value above before comparing your numbers to ours.**

### Running it

```bash
python run_spice.py --lib /path/to/sky130A/libs.tech/combined/sky130.lib.spice --case base --evals 150000 --pop 40 --runs 30 --outdir results/base_150k
```

`--case hard` uses the tightened thresholds; `--case highdim` runs the 16-dimensional variant at 220,000 evaluations. On a cluster, use `truba/runcase.sh`, which chunks the 180 `(algorithm, run)` tasks into SLURM arrays. A full case is roughly 7 hours on 40 cores at 5 workers per node — the evaluator is memory-bandwidth-bound, so more workers per node makes it *slower*, not faster.

> **Before submitting jobs**, edit two placeholders in `truba/*.slurm`: `YOUR_ACCOUNT` (the SLURM allocation) and `YOUR_USER` in the log paths. SLURM parses `#SBATCH` directives before shell expansion, so these cannot be variables; everything else derives from `$USER`, or from `TRUBA_USER` if you set it.

### Data layout

```
results/base_150k/      per_run_records.csv   180 rows = 6 algorithms x 30 runs  <- the authoritative file
results/hard_150k/      rows.zip              the 360 per-task shards it was merged from
results/highdim_220k/   pvt/                  the 15-condition sweep (2,700 rows per campaign)
results/penalty_sensitivity/                  Section 5, lambda sweep; the lambda=1000 slice is the published campaign
results/protocol_sweep/                       protocol-choice sweep; see the note in that folder
env.sh                                        the cluster provenance record: PDK build, model library, simulator
```

Each campaign directory also carries `pvt/pvt_summary_corrected.csv` — use that rather than
`pvt_summary.csv`, for the reason given below. `rows.zip` is provided so the merge can be audited;
`per_run_records.csv` is what every table in the paper is computed from, and the two agree by
construction (`run_spice.py --merge` refuses a wrong row count or a duplicated `(algo, run)`).

### Known issues in the released data

Everything below was found by our own checks, not by a reviewer, and each is discussed in the paper.

1. **`pvt/pvt_summary.csv` is unusable as shipped.** Its `nominal_feasible` column contradicts `per_run_records.csv` for 173 of 540 designs at the very condition the optimizer ran at, because simulation failures are scored identically to specification violations. Use **`pvt_summary_corrected.csv`** in the same folder, regenerated by `pvt_summary_fix.py`, which carries both scoring conventions plus the failure counts. The `pvt_robust` column (0 everywhere) is unaffected.
2. **Per-algorithm PVT figures cannot rank optimizers.** Between 24.8 % and 29.7 % of the corner re-simulations returned no measurement within the solver time limit, and the failure rate tracks the algorithm (95.3 % for ABC on `hard` against 0.0 % for PSO). The only PVT figure invariant to the scoring convention is the 0-of-540 result.
3. **`highdim` is not an equal-effective-budget comparison.** 21.68 % of its evaluations returned no measurement, with usable fractions from 57.8 % (GWO) to 99.6 % (FA). It does not manufacture the ordering — the winner holds the *smallest* effective budget — but it is unequal.
4. **The three Section 6 cases share one seed offset.** `run_spice.py` applies a single `CASE_OFFSET`, so all three draw the same thirty run seeds, and `base`/`hard` additionally share dimension and bounds, hence identical initial populations. Within-case pairing — which the Friedman and Wilcoxon tests require — is intact; the cases are variants, not independent replicates.
5. **`results/protocol_sweep/per_run.csv` contains 540 duplicate rows**, confined to two rehearsal cells at `budget = 2500`. See `README_duplicate_rows.md` in that folder before computing anything from it. The published cell is clean.
6. **The campaign CSVs do not record their own provenance** — no PDK commit, no library path, no simulator version. That is what `env.sh` is for.

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

This code is released under the MIT License; see `LICENSE`.

Two third-party components are used but **not** redistributed here. The SkyWater SKY130 PDK is Apache-2.0 and is fetched at a pinned commit by `truba/00_setup_native.sh`. `ngspice` is distributed under the BSD 3-Clause licence and is built from source by the same script.

---

## Contact

For questions about the benchmark or code, please contact the corresponding author (see the paper).