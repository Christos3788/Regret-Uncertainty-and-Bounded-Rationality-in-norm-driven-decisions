# ODD Protocol — Regret–Uncertainty ABM with Social Norm Dynamics (“RegretUncertainty”)
*(Overview–Design concepts–Details; aligned with Grimm et al. 2006/2010 and the 2020 update)*

> This ODD documents the Julia implementation referred to as **RegretUncertainty**. It follows the ODD structure and wording conventions recommended in the second update by Grimm et al. (2020). Where helpful, we include code-oriented names to ease traceability. The textual description is sufficient to re‑implement the model independently of the original code.

---

## 1. Purpose and patterns

**Purpose.** Explain how **anticipated regret**, **uncertainty-dependent trust weighting**, and **social norms** (personal, descriptive, injunctive) jointly shape **seasonal vaccination** decisions that coevolve with a **seasonal event‑driven SIR** epidemic on a two‑layer network. The model is used to study qualitative regimes (stability, coordination, oscillations) and intervention levers (e.g., injunctive vs descriptive nudges).

**Target patterns / evaluation criteria (pattern‑oriented):**
- Seasonal trajectories of **vaccination intention/coverage** and **outbreak size**; convergence or sustained cycles.
- Comparative statics w.r.t. **bounded rationality** (κ), **regret sensitivity** (η₁, η₂), and **information observability** (z).
- Relative influence and timescales of **descriptive** vs **injunctive** vs **personal** norms.
- Robustness across network types (small‑world physical layer; Klimek–Thurner social layer).

**Model use type.** Primarily **explanation/understanding** and **demonstration of mechanisms**; not calibrated to a specific locale.

---

## 2. Entities, state variables, and scales

**Entities.**
- **Agents** \(i = 1 … N\) with:
  - **Action** `aᵢ ∈ {0,1}` (0=don’t vaccinate, 1=vaccinate) decided once per **season**.
  - **Intention** `xᵢ ∈ [0,1]` (Bernoulli parameter for `aᵢ`).
  - **Learning memory** of material payoffs over the last `m` seasons: `π̂ᵛᵃᶜᵢ`, `π̂ᵘⁿᵛᵃᶜᵢ`.
  - **Norm variables** in \([0,1]\): personal attitude `yᵢ`, empirical expectation `x̃ᵢ` (perceived average peer intention), normative expectation `ỹᵢ` (perceived moral approval).
  - **Risk & uncertainty**: estimated infection risk `Eᵢ[P]`, variance `Varᵢ[P]`; **safety** `Sᵢ=1−(risk)`, **fear** `fᵢ=1−Sᵢ`; **uncertainty** `uᵢ = uᵢ^{intr} + uᵢ^{info}`.
  - **Neighborhood stability/consensus**: `qᵢ^{stab}`, `qᵢ^{cons}` from recent neighbor choices.

- **Networks (two layers).**
  - **Physical contact layer**: small‑world (Watts–Strogatz‑like) with mean degree ⟨k⟩ and rewiring probability; used by SIR.
  - **Social layer**: Klimek–Thurner friendship‑like evolution (triadic closure, turnover, overlap with physical ties).

- **Disease states (per SIR season)**: Susceptible S, Infected I, Recovered R (permanent immunity). Vaccinated agents still record exposure attempts.

**Scales.**
- **Time**: discrete **seasons** \(t = 0,1,2,…\). Within each season, run `n_sim` event‑driven SIR realizations to estimate `Eᵢ[P]`, `Sᵢ`, etc.
- **Space**: non‑spatial graphs; no geometric coordinates.

---

## 3. Process overview and scheduling

For each **season** \(t\):
1. **Epidemic sampling (event‑driven SIR)** on the **physical layer** with current vaccination actions; run `n_sim` realizations. For each agent compute:
   - expected infection probability `Eᵢ[P]` (unvaccinated baseline), its variance `Varᵢ[P]`,
   - safety `Sᵢ`, fear `fᵢ=1−Sᵢ`, and neighbor infection summaries (for vaccinated exposure proxy).
2. **Learning & regret (material channel).**
   - Update payoff attractions `π̂ᵘⁿᵛᵃᶜᵢ`, `π̂ᵛᵃᶜᵢ` from observed costs: vaccination cost `c_V`, infection cost `c_I` via last‑season outcomes; apply **regret–rejoice** adjustment `R(·)` with strength η₁ and curvature η₂ to emphasize foregone‑better outcomes.
3. **Weights (context‑dependent).**
   - Compute **empirical vs normative gate** `ϕᵉᵐᵖᵢ = (fᵢ uᵢ)^{1/2}`.
   - Compute **collective vs individual gate** `ϕᶜᵒˡᵢ = (qᵢ^{stab} qᵢ^{cons} fᵢ uᵢ)^{1/4}` and `θᶜᵒˡᵢ = (qᵢ^{stab} qᵢ^{cons} fᵢ uᵢ^{intr})^{1/4}`.
   - Set normalized **decision weights** that sum to 1:
     - material `A_mat = (1−ϕᵉᵐᵖᵢ)(1−ϕᶜᵒˡᵢ)`,
     - personal norm `A_y = ϕᵉᵐᵖᵢ(1−θᶜᵒˡᵢ)`,
     - descriptive norm `A_{x̃} = (1−ϕᵉᵐᵖᵢ) ϕᶜᵒˡᵢ`,
     - injunctive norm `A_{ỹ} = ϕᵉᵐᵖᵢ θᶜᵒˡᵢ`.
4. **Utilities and intention.**
   - Material payoff: `Πᵢ(a) = a·π̂ᵛᵃᶜᵢ + (1−a)·π̂ᵘⁿᵛᵃᶜᵢ`.
   - Normative payoff: `Vᵢ(a,yᵢ,x̃ᵢ,ỹᵢ) = A_y|a−yᵢ| + k_a|1−a−x̃ᵢ| − k_d|a−x̃ᵢ| + A_{ỹ}(2ỹᵢ−1)(a−0.5)` with `k_a+k_d=A_{x̃}`.
   - **Total utility** `Uᵢ = A_mat Πᵢ + Vᵢ`. Compute difference `ΔUᵢ = Uᵢ(a=1)−Uᵢ(a=0)`.
   - **Intention** (quantal response): `xᵢ = (1 + exp(−ΔUᵢ/κ))^{-1}`; **draw action** `aᵢ ~ Bernoulli(xᵢ)`.
5. **Norm dynamics (DeGroot‑type update on social layer).**
   - Update `yᵢ, ỹᵢ, x̃ᵢ` using linear recurrences that mix own variables vs observed neighbor behavior `Xᵢ` (average neighbor vaccination), with **collective weight** `θᶜᵒˡᵢ` and optional external authority targets `G¹,G²,G³`. Personal/adaptive rates `ξ₁, ξ₂, ξ₃` may differ (typically `ξ₃ > ξ₂ > ξ₁`).
6. **Stopping rule.** If rolling‑window change in mean intention/coverage is below ε for a prescribed number of seasons, terminate; else continue to the next season.

---

## 4. Design concepts

- **Basic principles.** Coevolution of infection and behavior; probabilistic choice with bounded rationality; regret‑augmented learning; norms with distinct channels and timescales.
- **Emergence.** Population vaccination and outbreak patterns emerge from local risk estimates, memory, norms, and network topology.
- **Adaptation.** Agents adapt via payoff learning (EWA‑style) and norm updates; context gates (`ϕᵉᵐᵖ`, `ϕᶜᵒˡ`) reweight channels as fear/uncertainty/consensus change.
- **Objectives.** Avoid infection costs and disutility from norm misalignment; no explicit global objective.
- **Learning.** Finite memory `m`; regret–rejoice adjustment emphasizes foregone better outcomes.
- **Prediction.** No forward look‑ahead; risks are **empirically** estimated each season via Monte‑Carlo SIR.
- **Sensing.** Agents sense neighbors’ recent actions for `Xᵢ`, `q^{stab}`, `q^{cons}`; they infer infection risk from season simulations; observability `z` limits knowledge of neighbors’ infection states.
- **Interaction.** Physical layer (infection); social layer (norm pressure and observations).
- **Stochasticity.** Network generation, initial seed infection each season, event times in SIR, and action sampling from intentions.
- **Collectives.** Ego‑networks; no explicit higher‑level groups.
- **Observation.** Time series of coverage/intention, outbreak size, distributions of `xᵢ, yᵢ, ỹᵢ, x̃ᵢ`, and convergence diagnostics.

---

## 5. Initialization

- **Networks.** Generate small‑world physical layer (⟨k⟩, rewiring β_SW). Create social layer via Klimek–Thurner (triadic closure `r`, turnover `p`) with overlap to physical neighbors.
- **Agents.** Initialize `aᵢ` (e.g., all unvaccinated), seed **norm variables** `yᵢ, ỹᵢ, x̃ᵢ ∼ U(0,1)`. Initialize payoff memories with neutral/low values; set memory length `m`.
- **Disease.** Choose one random initial infected agent per season start; set SIR parameters (β, μ=1 by normalization).
- **Randomness.** Record RNG seed(s) for reproducibility.
- **Parameters.** Provide a run configuration (see Section 6) and output filenames/paths.

---

## 6. Input data

- No external datasets are required. All inputs (networks, seeds) are **synthetic** and generated at runtime.

---

## 7. Submodels

### 7.1 Event‑driven SIR (seasonal sampler)
- **Process.** Continuous‑time infection and recovery using exponential clocks on the **physical layer**. Vaccinated nodes record exposure attempts but do not infect with the same probability as unvaccinated; immunity is permanent within/after season.
- **Outputs per agent.** `Eᵢ[P]`, `Varᵢ[P]`, neighbor infection summary `Îᵢ,neighbors` based on `n_sim` realizations.

### 7.2 Material learning with regret
- **Attractions.** `π̂ᵘⁿᵛᵃᶜᵢ`, `π̂ᵛᵃᶜᵢ` updated from realized costs (`c_I` if infected and unvaccinated; `c_V` if vaccinated). Apply **regret–rejoice** `R(x)=η₁·x^{η₂}` for positive foregone advantages.

### 7.3 Norm‑weighted utility and intention
- **Weights.** `A_mat, A_y, A_{x̃}, A_{ỹ}` derived from gates `ϕᵉᵐᵖ`, `ϕᶜᵒˡ`, `θᶜᵒˡ`; they sum to 1.
- **Utility difference.** `ΔUᵢ` combines material and normative parts; intention uses a Fermi/logit function with rationality `κ`.

### 7.4 Norm dynamics (DeGroot‑type)
- **Updates.** Linear recurrences for `yᵢ, ỹᵢ, x̃ᵢ` combining own variables with neighbor behavior `Xᵢ`, paced by `ξ₁, ξ₂, ξ₃`, optionally nudged by external authority signals `G¹,G²,G³` (weights `γ¹,γ²,γ³`). Collective weight is `θᶜᵒˡ`.

### 7.5 Uncertainty decomposition and observability
- **Observability.** A fraction `z ∈ [0,1]` of neighbors’ infection states is visible. Unobserved neighbor infections are modeled binomially, yielding closed‑form `Eᵢ[P]` and `Varᵢ[P]` used in `Sᵢ, uᵢ`.

---

## 8. Parameters (indicative ranges and roles)

| Symbol | Meaning | Typical range / note |
|---|---|---|
| `β`, `μ` | SIR transmission and recovery rates | `μ=1` (scale), `β` in `[0.1,1.0]` |
| `c_V`, `c_I` | Vaccination and infection costs | `c_V < c_I`, e.g. `c_V∈[0.1,1], c_I=1` |
| `m` | Memory length (seasons) | `1–8` |
| `η₁`, `η₂` | Regret strength & curvature | `η₁≈1`, `η₂∈[0.25,2]` |
| `κ` | Rationality (logit slope) | `0.1–1.0` (typical) |
| `z` | Observability fraction of neighbors | `0–1` |
| `ξ₁, ξ₂, ξ₃` | Norm update rates | `ξ₃>ξ₂>ξ₁`, e.g. `1.0, 0.1, 0.01` |
| `γ¹,γ²,γ³`/`G¹,G²,G³` | External authority weights/targets | `[0,1]` (optional) |
| Network | Size & structure | `N≈10²–10³`, small‑world ⟨k⟩≈6; social KT with triadic closure/turnover |

---

## 9. Initialization, outputs, and experiments

**Key outputs.** Seasonal time series of average intention ⟨x⟩, coverage, outbreak size; distributions of norm variables; stability metrics.

**Experiments.** Sweep over `(κ, η₁, η₂, z, c_V/c_I, ξ’s)` and network options; compare descriptive vs injunctive nudges (vary `γ`/`G`); test robustness to memory `m` and observability `z`.

---

## 10. Code–ODD traceability (guide)

- **Season loop & SIR sampler** → implements Section 3.1 and Submodel 7.1.
- **Regret/EWA update** → Section 7.2; parameters `η₁, η₂, m, c_V, c_I`.
- **Gates & weights** → Section 3.3; computes `ϕᵉᵐᵖ`, `ϕᶜᵒˡ`, `θᶜᵒˡ` and `A_*`.
- **Utility & intention** → Section 3.4; parameters `κ`, `k_a,k_d` with `k_a+k_d=A_{x̃}`.
- **Norm update** → Section 3.5 and Submodel 7.4; parameters `ξ_q`, `γ^q`, `G^q`.
- **Uncertainty module** → Submodel 7.5; parameter `z`.

> **Reproducibility note.** Record RNG seeds, network seeds, and configuration JSON/CSV with every run.
