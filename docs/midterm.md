# Midterm Report

Report for **[CS-CP-4997]** capstone midterm evaluation.

*Aarush Kumbhakern*

## 1. Problem Statement

Microfluidics is an interdisciplinary technology (primarily) in the field of applied biochemistry. Microfluidics has been explored for chemical synthesis, diagnostics, materials science, and micro-scale computing (lab-on-chip, organ-on-chip, microreactors). Programmable/modular, dynamically reconfigurable microfluidic chips in particular are a recent (early 2000's) development with recognized scope in the biomedical field. [1-7]

At present, the process of going from functional description (eg. assay specification) $\to$ physical microfluidic chip design is a labor-intensive process involving experts from unrelated fields, and expensive (albeit powerful) general-purpose software. While some work has been done on automating the design of static-control (fixed channel layouts) flow-based microfluidic systems, there remains a gap in automating the generation of schedules for dynamically controlled (time-varying valve/electrode actuation) flow-based microfluidic systems. [8-11]

> **There is no standard compiler that maps a structured assay specification to an executable control schedule (valve waveforms/timelines) on a specific device under physical constraints.** [8-11]

## 2. Aim of the Project

*I aim to propose a novel, extensible bus & register-based dynamically controllable microfluidic architecture, and develop for the same an end-to-end pipeline that takes in an assay specification and generates an execution plan on the platform.*

My primary goals are to:

1. Make a strong case for the **novelty, viability, and value** of a bus & register-based architecture.
2. **Formally prove the scope and limits** of the architecture.
3. Develop **tooling** for the automated generation of schedules on the architecture.
4. Integrate a minimal **physics-in-the-loop verifier** for maximal usability.

## 3. Architecture Proposal

```       
          #   ╭────── Bus        
          │   ▼                  
     ╭────╰───╮                  
     │       ─┼─                 
 #───╮        ╰─│─#              
     │       ─┼─  ▲              
     ╰────╮───╯▲  │              
          │    │  ╰── Register   
          #    │                 
               │                 
               ╰───── Valve      
```

### 3.1. Informal overview & design goals

The device comprises a **closed-loop microfluidic bus** (a ring) to which multiple registers (functional modules) are attached via valved junctions. The bus transports fluid "slugs" between registers; registers perform operations (storage, mixing, incubation, sensing, reagent dispense, waste). The design goals are:

- **Minimality:** a single programmable transport backbone with attachable modules.
- **Extensibility:** new registers can be added without changing bus semantics.
- **Composability:** assays decompose into macro-ops over a small set of primitives.
- **Scheduleability:** timing/resource constraints are explicit to enable compilation.

> **Mental model:** We can think of the bus as a conveyor with addressable side-ports (registers). The compiler decides when to attach/detach fluid slugs at ports and how to space, route, flush, and park them.

### 3.2. Components (structural)

- **Bus:** a closed loop of channels partitioned into segments by valves and T-junctions to registers. Optionally multi-lane; default is one lane with a programmable high-precision peristaltic pump (3-valve loop or on-chip pressure drive).

- **Registers:** a finite set $\{R_i\}$, each with:
  
  - **Type:** Reservoir, Storage, Mixer, Incubator/Heater, Magnet, Detector, Waste, Wash.
  
  - **Ports:** each register attaches to the bus via one or two **access valves** (IN/OUT) and an internal volume.
  
  - **Capabilities:** supported macro-ops (eg. `MIX`, `INCUBATE`, `SENSE`).
  
  - **Capacity:** $[V_i^{\min}, V_i^{\max}]$; **dead volume** $DV_i$.
  
  - **Compatibility classes:** for contamination policy.

- **Valves:** binary actuators with open/close latency $(t_{\text{open}}, t_{\text{close}})$, minimum pulse width $t_{\min}$, maximum safe actuation rate $f_{\max}$.

- **Pumps/Drive:** provides commanded volumetric displacement $\Delta V_{\text{bus}}$ per timeslot $\Delta t$, or equivalently a nominal flow $Q_{\text{bus}}$. Clock-cycle approximation.

- **Sensors/Actuators on registers:** temperature controllers, magnets, optical paths, electrodes, etc., each with activation constraints and warm-up times.

- **Global utilities:** at least one Wash register and one Waste register are assumed.

### 3.3. Why Bus-and-Register over Valve-Array Grids

A fully addressable valve-array grid offers maximal spatial freedom but at high control, scheduling, verification, and manufacturing cost: pathfinding with global timing/valve conflicts is combinatorial; actuation counts and duty cycles are large; contamination risk grows with many shared intersections; and pneumatic/electrical I/O scales poorly (dozens–hundreds of lines). The bus-and-register architecture collapses most spatial choices into temporal ones on a single transport backbone: FIFO routing with a pump-as-clock gives deterministic latencies and simple spacing guarantees; the compiler reduces to binding + list scheduling on a ring; hygiene becomes path-local with predictable `FLUSH` insertion; and the device uses far fewer valves/lines, improving reliability, power, footprint, and cost. Registers encapsulate operations (mix, incubate, sense), so adding capability is modular and does not inflate global routing complexity. For assays that are sequential/branching (common in diagnostics and prep), a ring yields higher utilization per actuator and easier formal checking than a grid. 

Grids still enable extreme parallel point-to-point traffic; when needed, we can mitigate the bus' concurrency limits with multiple coupled rings.


## 4. Assay IR & Semantics

### 4.1. Design goals

The Assay IR (AIR) is a **typed, dependency and resource-aware action graph** that:

- Expresses **what** to do (domain ops like MIX, INCUBATE, SENSE) without committing to **where/when** (left to mapping/scheduling);
- Carries minimal physical facts (volumes, classes, temperatures, time windows) sufficient to derive a correct schedule on the bus-and-register device;
- Supports parallelism and barriers, and encodes hazard/compatibility information for contamination control.

> **Separation of concerns:** AIR names logical **containers**; the compiler later **binds** containers to concrete *registers* and emits device macro-ops (eg. `ATTACH`, `DETACH`, `SHIFT`, `FLUSH`).

### 4.2. Graph Model & Typing

AIR is a **directed acyclic hypergraph**.

### 4.3. Constraints & Annotations (carried to the scheduler)

- **Quality of result (QoR) hints**: `priority`, `minimize_makespan`, `minimize_valve_actuations`, `minimize_flush`
- **Contamination policy**: `allow_contact_with`, `required_flush`
- **Binding hints**: `prefer`, `avoid`, `colocate`, `isolate`

## 5. Mapping & Scheduling

- **Inputs:** (i) Assay IR (AIR) graph, fluids, constraints.
- **Output:** a feasible, discrete-time schedule of device macro-ops.

### 5.1. Problem Sketch

Map logical containers to physical registers and route **fluid tokens** on the bus, such that:

- All AIR data/ timing constraints hold (deadlines, min/max durations).
- Resource constraints hold (single-server registers/valves, bus exclusivity).
- Hygiene constraints hold (compatibility classes, required flush volumes).

### 5.2. Objectives

1. **Feasible schedule**
2. Minimize flush volume
3. Minimize valve actuations

### 5.3. Guarantees & Complexity

Full optimality is NP-hard (resource-constrained scheduling + routing). We provide a feasibility-first heuristic with bounded backtracking; when infeasible, a minimal counterexample is emitted (op, path, resource, time window). Determinism via fixed seeds and tie-break rules.

## 6. Physics-in-the-Loop Verifier

> Discrete-time; laminar $Q$, $\Delta P$ via Hagen–Poiseuille; valve timing/rate limits; volume conservation & register capacity; simple thermal holds; washout proxy $r(V)=r_0 e^{-kV}\le \epsilon$.

We aim to check pressure/flow bounds, timing windows, exclusive resources (registers/valves/bus), spacing/no overtaking, conservation & capacities, required `FLUSH/PURGE` sufficiency, and wash/waste budgets. For every step per $\Delta t$, update valve/pump/slug states; compute $Q$ and $\Delta P$, positions, residue; on first violation emit witness + minimal fix (eg. increase flush, shift window, reduce concurrency, rebind). We operate nder the following limits: 1D laminar, constant $\mu$; conservative (may over-reject). 

If `PASS`, schedule is physics-consistent under stated bounds.

## 7. Benchmarks & Evaluation Plan

We will evaluate the compiler and scheduler on a small, canonical suite of assay workflows — ELISA, magnetic-bead cleanup, serial dilution, and a simple PCR-like incubation chain — implemented in the AIR and compiled to the bus-and-register device. The primary outcome is feasibility (schedulability rate and verifier PASS), with efficiency (makespan/throughput, bus occupancy), and cost (total flush volume, valve actuations, compile time) being a future goal.

## 8. Big Picture: A Standalone Computational Unit

The bus-and-register device is intended as a self-contained computational unit for assays: a minimal loop (transport) with attachable registers (state/operations) that executes a clocked sequence of macro-ops. In this view, the pump is the clock (setting slot length / throughput), while valve openings/closings are the instructions that realize macro ops. The compiler targets this unit with a discrete schedule; larger systems emerge by composing units (multiple rings or rings + specialized modules) via well-defined fluidic ports. This decouples local correctness (each unit guarantees spacing, capacity, and hygiene) from system integration (units exchange only validated slugs), enabling hierarchical designs without re-deriving low-level timing.

As a system building block, a single ring can drive complete small assays; multiple rings — each clocked by its own pump — can be tiled or coupled through buffering registers to scale throughput or isolate chemistries. The result is a programmable, clocked substrate that behaves like a microfluidic processing element, ready to plug into larger lab-on-chip assemblies while staying independently schedulable and verifiable.

## 9. References

1. G.M. Whitesides, "The origins and the future of microfluidics," *Nature* **442** (2006): 368–373.  

2. M.A. Unger *et al.*, "Monolithic microfabricated valves and pumps by multilayer soft lithography," *Science* **288** (2000): 113–116.  

3. T. Thorsen, S.J. Maerkl, S.R. Quake, "Microfluidic large-scale integration," *Science* **298** (2002): 580–584.  

4. K. Choi, A.H.C. Ng, R. Fobel, A.R. Wheeler, "Digital Microfluidics," *Annu. Rev. Anal. Chem.* **5** (2012): 413–440.  

5. M. Prakash, N. Gershenfeld, "Microfluidic Bubble Logic," *Science* **315** (2007): 832–835.  

6. D. Huh *et al.*, "Reconstituting organ-level lung functions on a chip," *Science* **328** (2010): 1662–1668.  

7. J. Melin, S.R. Quake, "Microfluidic large-scale integration: The evolution of design rules for biological automation," *Annu. Rev. Biophys.* **36** (2007): 213–231.  

8. E.E. Tsur, "Computer-Aided Design of Microfluidic Circuits," *Annu. Rev. Biomed. Eng.* **22** (2020): 285–307.  

9. X. Huang *et al.*, "Computer-aided design techniques for flow-based microfluidic lab-on-a-chip systems," *ACM Comput. Surv.* **54** (2022): 1–29.  

10. G. Liu *et al.*, "Design automation for continuous-flow microfluidic biochips: A comprehensive review," *Integration* **82** (2022): 48–66.  

11. I.E. Araci, S.R. Quake, "Recent developments in microfluidic large scale integration," *Curr. Opin. Biotechnol.* **25** (2014): 60–68.