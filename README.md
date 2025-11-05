# Regret–Uncertainty ABM with Social Norm Dynamics — README

This repository contains the Julia implementation of the **Regret–Uncertainty behavioral epidemic model** where individual **vaccination decisions** coevolve with **social norms** and an **event‑driven SIR** process on a physical contact network. The implementation corresponds to the notebook/code named **`RegretUncertainty`** and is documented via a full ODD file.

- 👉 See the full ODD: `ODD_RegretUncertainty.md` (included in this repo).  
- 👉 Scientific article draft: use the notebook’s equations/sections; the ODD mirrors the code as ground truth.



---

## Files

- **`RegretUncertainty.ipynb`** — Main implementation (Julia-in-notebook). Contains the seasonal loop, event-driven SIR sampler, regret‑augmented learning, normative gates/weights, and DeGroot-style norm updates.
- **`ODD_RegretUncertainty.md`** — Complete ODD (Overview–Design–Details) aligned with Grimm et al. updates; includes a code–ODD traceability table.
- **`/results/`** — (Created at runtime) CSV summaries of time series and parameter configurations per run.
- **`/figures/`** — (Optional) Saved plots, if plotting is enabled in the notebook.

---

## Quick Start

1. **Install Julia** (≥ 1.9 recommended) and the dependencies used in the notebook (e.g., `LightGraphs`, `Random`, `StatsBase`, `Distributions`, `DataFrames`, `CSV`, `SparseArrays`, `LinearAlgebra`, `Arpack`).  
2. Open `RegretUncertaintyGood.ipynb` and run **all cells**. Default parameters will execute a full set of **seasons** with **Monte‑Carlo epidemic sampling** each season.  
3. Outputs (CSV files) will be written under `./results/` with run‑specific suffixes (e.g., RNG seed/timestamp).

---

## Model at a Glance

- **Population:** `N` agents on **two layers**:
  - **Physical (SIR spread)**: small‑world (Watts–Strogatz‑like) / ER as configured.
  - **Social (norm influence)**: Klimek–Thurner‑like evolution (triadic closure `r`, turnover `p`, overlap to physical).
- **Epidemics (per season):** event‑driven **SIR** with transmission `β` and recovery `μ` (usually `μ=1` for scaling). Each season runs `Nsim` realizations to estimate individual infection risk and uncertainty.
- **Decision making (per season):**
  - **Material channel:** recency‑weighted payoffs (infection cost `C_I`, vaccination cost `C_V`) plus **regret–rejoice** adjustment with strength/curvature `(η₁, η₂)`.
  - **Normative channels:** personal attitude `y`, descriptive proxy `x̃`, injunctive expectation `ỹ`, blended by **context gates** from **fear/safety**, **uncertainty** and **local consensus/stability**.
  - **Action sampling:** intentions via **quantal response** with rationality `κ`; vaccination action is Bernoulli-drawn from intention.
- **Outputs:** time series of vaccination coverage, outbreak size, distributions of `y, ỹ, x̃`, stability diagnostics, and run configuration logs.

For a structured description of entities, schedule, submodels, and parameters, refer to `ODD_RegretUncertainty.md`.

---

## Running and Reproducibility

- **RNG Seeds:** Every run records seeds/configuration in results CSVs for replication.
- **Batch Runs:** Duplicate the notebook, or parameterize via top‑level cells; keep a **one‑run‑per‑folder** convention (`results/`, `figures/`) for clarity.
- **Performance Tips:** Reduce `Nsim` (per‑season epidemic samples) and shorten `vacCycles` to explore behavior quickly; increase them for stable estimates.

---

## Outputs

By default, runs persist:
- **`results/Params_*.csv`** — complete parameter vector (including RNG seed).
- **`results/Series_*.csv`** — optional full time series (coverage, outbreak size, norm variables).
- **`results/Summary_*.csv`** — compact “info matrix” style summary of the final state and selected averages.

The exact filenames may include an RNG/timestamp suffix to avoid overwriting previous runs.

---

## Parameter Highlights

| Block | Key parameters | Notes |
|---|---|---|
| **Disease** | `β`, `μ` | Transmission and recovery (usually rescaled `μ=1`). |
| **Economics** | `C_I`, `C_V` | Infection vs vaccination costs (typically `C_V < C_I`). |
| **Regret** | `η₁`, `η₂` | Strength/curvature of regret–rejoice transformation. |
| **Choice** | `κ` | Rationality (logit slope) for intentions. |
| **Uncertainty/Obs.** | `z` | Fraction of neighbors’ states observed (affects perceived risk/variance). |
| **Memory** | `m` | Payoff/behavior memory (in seasons). |
| **Norms** | `ξ₁, ξ₂, ξ₃`; `γ¹,γ²,γ³`; `G¹,G²,G³` | Personal/descriptive/injunctive update rates; authority weights/targets. |
| **Topology** | `N`, `⟨k⟩`, `β_SW`; `r`, `p`, `overlap` | Small‑world/ER physical layer; KT‑style social layer. |

Exact parameter names and defaults are defined in the notebook’s parameter cells; the ODD lists roles and typical ranges.

---

## Code Map (Notebook Sections)

1. **Parameters & Setup** — initialize seeds, network sizes, epidemic and behavioral parameters.  
2. **Network Construction** — build physical (WS/ER) and social (KT‑like) layers; ensure connectivity where required.  
3. **Event‑Driven SIR** — per‑season Monte‑Carlo sampling on the physical layer; compute per‑agent risk and uncertainty.  
4. **Learning & Regret** — update attractions/payoffs with finite memory and regret–rejoice correction.  
5. **Norm Weights & Utilities** — compute context gates and normative weights; build utilities and intentions (quantal response).  
6. **Norm Dynamics** — DeGroot‑style updates for `y, ỹ, x̃` with optional external signal targets.  
7. **Loop & Stopping** — iterate seasons until max cycles or stability; log CSV outputs and (optionally) plots.

---

## Troubleshooting

- **Long runtimes:** Decrease `Nsim` (epidemic repetitions per season) or `N` (agents).  
- **No convergence:** Increase seasons or relax stability thresholds; check consensus/stability gates not suppressing material payoffs excessively.  
- **Different results across runs:** Confirm fixed RNG seeds and identical network generation options.

---

## Citation

If you use this code or ODD, please cite the associated manuscript (once available) and the ODD markdown (`ODD_RegretUncertainty.md`). For ODD structure, follow Grimm et al. (2020 update). 

---

*Maintained by the authors of the Regret–Uncertainty ABM. Issues and feature requests are welcome via your typical channels.*
