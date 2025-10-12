# Physics-Aware Synthesis and Impossibility Conditions for Mapping Assay Graphs onto Valve-Grid Flow-Based Lab-on-Chips with Periphery I/O

## Problem Statement

Valve-based, continuous-flow microfluidic biochips (FBMBs/mLSI) are increasingly designed using EDA-style flows that schedule, place, and route assay “sequencing graphs” onto physical architectures with pneumatic microvalves. Much prior art targets generic continuous-flow architectures, distributed channel storage (DCSA), and control-logic minimization; however, practical labs frequently fabricate **square valve-grid** chips with **inlets/outlets restricted to the periphery** for packaging simplicity. Mapping arbitrary assay graphs onto such grids is constrained by (i) degree limits of grid nodes, (ii) single-layer, planar routing, (iii) valve co-actuation and contamination rules, and (iv) fluid-physics limits (pressure drop, mixing/dispersion, timing). 

There is no open, end-to-end toolchain that:

- **(A)** maps assay graphs onto pre-fabricated valve grids under periphery-I/O and control constraints;

- **(B)** co-simulates physics fast enough to sit in the loop; and

- **(C)** provides formal impossibility certificates when mapping cannot succeed—together with a quantified argument for **vertical vias/out-of-plane ports** as a minimal relaxation.

**Goal:** Build a physics-aware synthesis + simulation framework for valve-grid LoCs with periphery I/O, and derive graph-theoretic **necessary (and where possible sufficient) conditions** for non-embeddability on single-layer grids; demonstrate how limited vertical connectivity eliminates specific obstruction classes.

## Research Questions

1. **Mapping:** Given a sequencing graph \(G\) and a valve-grid \( \mathcal{L} \) with periphery I/O, when does there exist a contamination-safe, time-feasible mapping respecting valve/control constraints?

2. **Physics-aware feasibility:** How do hydraulic limits (pressure/flow headroom, dispersion, valve timing) prune otherwise “logical” mappings?

3. **Impossibility:** What graph/embedding properties (planarity variants, boundary-pin order, max-degree, edge-disjoint path demands) imply **no single-layer mapping with periphery I/O**?

4. **Minimal relaxation:** What is the minimum set of vertical vias/out-of-plane ports that turns an impossible instance feasible?

## Formalization

### Inputs

- **Assay:** sequencing graph $G=(V,E)$ with operation types (mix/incubate/split), volumes, and precedence/timing windows.  

- **Architecture:** grid graph $\mathcal{L} = (U,F)$ (orthogonal edges), valve states, periphery I/O set $T \subseteq \partial \mathcal{L}$, control-line constraints (sharing, multiplexing).  

- **Physics:** fluid properties, channel cross-sections, allowed pressure bounds, valve actuation latency.

### Outputs

- **Mapping:** $\phi: V \rightarrow U$ 

- **Routes/Schedules:** $\rho: E \rightarrow \text{paths}(\mathcal{L}) \times \mathbb{T}$  

- **Control Signals** and a **Feasibility Certificate** (or Impossibility Certificate).  

- **Simulation Report:** pressures/flows over time, dispersion/mixing surrogates, wash/contamination checks.

### Constraints

- Grid node degree $\leq$ 4  

- Single-layer planarity  

- Periphery-only terminals  

- No cross-contamination  

- Valve conflicts minimized  

- Pressure & timing limits respected  

## Approach & Methods

### A. Assay-to-Grid Synthesis (AGS)

- **Baseline ILP/MIP:** joint scheduling, placement, and path-routing with valve-conflict and wash constraints; warm-start from list-scheduling + multicommodity-flow relaxation.  

- **Heuristics:** multi-source A*/negotiation-based routing with rip-up-and-reroute; distributed storage awareness (DCSA); control-pin sharing with logic minimization post-pass.  

- **Boundary-I/O ordering:** circular-order pin assignment to minimize crossings; reuse VLSI “pin assignment + global routing” ideas adapted to fluidic constraints.

### B. Physics-in-the-Loop Simulation (PILS)

- **1-D network solver:** Hagen–Poiseuille resistances; valve on/off; optional compliance (PDMS wall & trapped gas) as capacitors; solve via modified nodal analysis with implicit time-stepping.  

- **Mixing/dispersion surrogates:** Taylor–Aris effective diffusivity along paths; simple reactor models for mixers/incubators.  

- **Calibration & speed:** cross-check selected cases against recent open 1-D simulators and literature; keep runtime sub-second for inner-loop pruning.

### C. Impossibility Conditions (BIG-E)

- **Necessary conditions:**
 
  - Max-degree ≤ 4 after module expansion  
 
  - **Fixed boundary order test:** chord crossings in the induced demand graph imply no planar single-layer routing (connects to outerplanar/book-embedding criteria).  
 
  - **Edge-disjoint path lower bounds** on grid cuts vs required concurrent flows (min-cut obstructions).  

- **Certificates:** (i) pin-order crossing witness; (ii) cut-set capacity violation; (iii) forbidden minor after boundary fixation.  

- **Minimal relaxation:** compute smallest set of **vertical vias** to eliminate crossings, show feasibility via AGS + PILS.

## Evaluation Plan

- **Benchmarks:** PCR/qPCR/ELISA workflows from literature + synthetic DAGs with controllable width/parallelism.  

- **Metrics:** success rate, valve count, control pins, total assay time, wash volume/time, max ΔP and Q, dispersion proxy, solver runtime.  

- **Ablations:** PILS pruning, DCSA, boundary-order heuristics, with/without minimal vias.  

- **Baselines:** existing HLS/control-synthesis and physical-design flows for FBMB/CFMB where available.

## Expected Contributions (Novelty)

1. **Grid/periphery-I/O-specific mapper** with physics-aware pruning (tight coupling of AGS and PILS).  

2. **Formal impossibility tests** (boundary-order chord crossings, cut capacities, degree bounds) packaged as certificates.  

3. **Minimal-via synthesis** that quantifies exactly how few out-of-plane connections resolve obstructions.  

4. **Open suite** (benchmarks + solver + sims) reproducible and extensible.

# Plan of Action (Fast-Track)

## Week 1 — Foundations & Spec

- Lock target grid geometries and periphery I/O models; define contamination & valve-timing rules.  

- Implement **PILS v0**: steady-state resistor network; validate against closed-form Hagen–Poiseuille calculations.  

- Build assay-graph schema + loader; create 3–5 seed assays (PCR, ELISA).  
**Artifacts:** `specs/constraints.md`, `pils/solver_v0.py`, `benchmarks/assays/`

## Week 2 — Mapping v0 (Feasibility First)

- ILP “skeleton” (placement+routing minimal) with big-M contamination constraints; greedy list-scheduler for timing.  

- Boundary-order module: derive terminal circular order, detect chord crossings → early **impossibility flag**.  
**Artifacts:** `synthesis/ags_ilp_v0.py`, `synthesis/boundary_test.py`

## Week 3 — Physics-Aware Pruning

- Integrate PILS into ILP/heuristic loop: reject mappings that violate ΔP/Q or dispersion thresholds.  

- Add valve latency & control-sharing feasibility check.  
**Artifacts:** `synthesis/pils_hook/`, `reports/run_{assay}_{arch}.md`

## Week 4 — Heuristics & DCSA

- Negotiation-based routing with rip-up-and-reroute; simple distributed-storage awareness; wash optimization pass.  
**Artifacts:** `synthesis/heuristics/`, `experiments/ablation_wash/`

## Week 5 — Impossibility & Minimal Vias

- Implement cut-set capacity tests and degree-expansion checks; formulate **minimal-via** ILP (fewest vias s.t. feasible).  

- Show previously impossible cases become feasible with $k$ vias; quantify benefit vs added complexity.  
**Artifacts:** `proofs/certificates/`, `synthesis/min_via_ilp.py`

## Week 6 — Control Synthesis & Packaging

- Generate control-layer signals/pin sharing from routes; compare to recent control-synthesis flows.  

- CLI + docs; reproducible runs; figures/tables for report.  
**Artifacts:** `cli/`, `docs/`, `results/plots/`

*(Stretch Weeks 7–8: refine simulator with compliance & simple mixing; compare to a 1-D open simulator on shared cases.)*

## Methodological Details

- **Hydraulic modeling:** $R = \frac{12\mu L}{w h^3(1 - 0.63h/w)}$ for rectangular channels; series/parallel as Ohmic network; optional capacitive elements for compliance.  

- **Routing objective:** minimize crossings, valve toggles, and pressure drop; penalties for co-actuation conflicts.  

- **Contamination model:** forbid opposite-direction reuse without wash; encode with edge-time disjointness + wash arcs.  

- **Impossibility tests:**  

  - **Chord-crossing:** under fixed boundary order ⇒ no single-layer planar realization.  

  - **Cut capacity:** if concurrent required flows exceed cut of grid edges at any time window → impossible without extra capacity/layer.  

  - **Max-degree:** expanded functional vertices must respect grid degree; otherwise require splitter/merger staging or vertical via.

## Risks & Mitigations

- **ILP runtime:** backstop with heuristics and staged solving.  

- **Physics fidelity vs speed:** keep 1-D models in the loop; validate a few cases; note scope (not a COMSOL replacement).  

- **Fabrication mismatch:** parameterize channel sizes/valve delays; expose config files.

## Related & Grounding Work

High-level/physical/control synthesis for FBMB/CFMB and DCSA; mLSI background; 1-D simulators; electrical analogy for networks; recent control-synthesis advances.

## Deliverables

1. **Paper-style report** with formal problem definitions, algorithms, impossibility theorems (with certificates), and evaluation.  

2. **Open repository**: `benchmarks/`, `synthesis/`, `pils/`, `proofs/`, `docs/`.  

3. **Reproducible artifact:** one-command run to map/simulate/report for a chosen assay & grid.