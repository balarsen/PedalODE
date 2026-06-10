# PedalODE — Project Map (v2, comprehensive)

> Physics-grounded circuit topology discovery and parameter identification for analog
> audio hardware. Sibling project to PedalDSP. This document is the full conceptual
> record of the design conversation — architecture, primitives, grammar, rung ladder,
> deployment, identifiability, metrics, capture philosophy, commercial framing, and
> the long-range full-chain vision. It supersedes the v1 project map.
>
> Convention carried from the working skill: literature claims are confidence-marked;
> numbers are tagged as measured / derived / assumed-typical. Equations are named,
> not derived — they are well known and look-up-able.

---

## 1. The Central Idea

Every analog pedal is a composition of a small set of circuit primitives, each
described by a known ODE or static nonlinearity from first principles. The transfer
function of each primitive is known analytically. Series composition multiplies
transfer functions; parallel composition adds them; feedback composes by the standard
closed-loop formula.

This means:

- **Primitive physics is not learned — it is initialized exactly from first principles.**
- **What is learned is topology (which primitives, how connected) and component values
  (R, C, Is, n, f0, Q, blend α).**
- **The warm start is not near the answer — for linear primitives it IS the answer.**
  Initialization at the true solution, not merely near it, produces qualitatively
  different convergence behavior.

The fitting problem factorizes:

```
P(pedal behavior) = P(topology) · P(parameters | topology) · P(ODE dynamics | parameters)
```

- Third factor: **known physics** — not learned, just simulated.
- Second factor: **initialized analytically** from transfer-function fitting, refined by ODE training.
- First factor: **the genuine unknown** — this is what the graph search solves.

You only do hard learning on the part of the problem that genuinely requires it.
Everything else is physics. This is gray-box modeling extended to the topology level.

**Detector-physics analogy (the framing that makes this natural):** primitives are
calibrated detector elements with known response; the topology search is response-matrix
unfolding; calibration sweeps → physics-data validation maps to synthetic sweeps →
guitar DI validation.

---

## 2. ODE vs DAE

### Pure ODE
```
dx/dt = f(x, u, t)
```
State evolves explicitly. Examples: RC filter, biquad section, gain stage.
Lumped circuit elements give ODEs, not PDEs — no spatial extent below ~100 kHz.
PDEs only enter for spatially extended systems (speaker cones, room acoustics,
transmission lines), and even those are usually handled as lumped modal approximations.

### DAE (Differential-Algebraic Equation)
```
dx/dt = f(x, z, u, t)     ← differential part (capacitor voltages, inductor currents)
0    = g(x, z, u, t)      ← algebraic constraint (instantaneous; no time derivative)
```

The Klon is a DAE: the antiparallel diode pair in the op-amp feedback path creates an
implicit algebraic constraint (KCL + Shockley diode equation) solved by Newton-Raphson
at every sample. Converges in ~3–5 iterations because the previous sample's solution
is a good initial guess.

**DAE index:** number of differentiations of the algebraic constraint needed to recover
a pure ODE. Audio circuits are essentially always **index-1** — the well-behaved case.
Higher-index DAEs (rigid mechanical constraints) do not appear here.

Mechanics analogy: the DAE is the circuit analog of constrained dynamics — pendulum
rod constraint, Lagrange multipliers. SPICE solving a circuit is mathematically the
same problem class.

### DAE handling options for learned models
1. **Eliminate z analytically** — closed-form substitution (works for tanh-style soft clippers)
2. **Implicit DAE-capable solver** — correct but slow and hard to train through
3. **Newton-Raphson inner loop** — solve constraint per timestep; must be torch-native
   ops only (differentiability requirement; known implementation trap)
4. **Learn the constraint implicitly** — works with enough capacity; loses interpretability

PedalODE uses option 3 inside an explicit DAE node type. Feedback is **only** permitted
inside DAE nodes — never as free graph edges (causality constraint, §6).

---

## 3. The Primitive Library

Each primitive carries: a named ODE (time domain), a named transfer function H(s)
(Laplace domain), and analytical parameter relationships (e.g. cutoff = 1/2πRC).
Equations named, not derived.

### Linear primitives

| Primitive | Parameters | Named form | Notes |
|---|---|---|---|
| Gain stage | A | Flat H(s)=A | Op-amp + resistor feedback |
| RC low-pass | R, C | First-order LP | Single real pole |
| RC high-pass | R, C | First-order HP | Zero + pole |
| Biquad | f0, Q, gain | Second-order section | Covers LP/HP/BP/notch/peak/shelf |
| All-pass | f0, Q | All-pass section | Phase only |

### Nonlinear primitives

| Primitive | Parameters | Named form | Notes |
|---|---|---|---|
| Soft clipper | Is, n (per branch) | Shockley diode equation | Smooth knee; asymmetric if branches differ |
| Hard clipper | threshold, asymmetry | Piecewise linear | Fuzz character |
| Half-wave rectifier | threshold | Piecewise linear | Envelope detection |
| Full-wave rectifier | threshold | Abs + threshold | Compressor side-chain |

Nonlinear primitives also carry a **describing function** N(A) — effective gain vs
input amplitude, derivable analytically from the Shockley equation — used for
frequency-domain initialization of nonlinear parameters.

### Dynamic / modulation primitives (later rungs)

| Primitive | Parameters | Notes |
|---|---|---|
| Envelope follower | attack/release RC | Peak detector + RC decay |
| VCA / OTA | control-voltage law | Tremolo, compressor gain cell |
| Delay line | delay, feedback | Echo, chorus, flange |

### Composition algebra

```
Series:    H_total(s) = H_1(s) · H_2(s) · ...
Parallel:  H_total(s) = α·H_1(s) + β·H_2(s) + ...
Feedback:  H_total(s) = H_fwd(s) / (1 + H_fwd(s)·H_fb(s))
```

The grammar (§4) is simultaneously a circuit grammar and a transfer-function algebra:
every sentence has a corresponding (semi-)closed-form transfer function.

---

## 4. The Circuit Grammar

Context-free grammar over primitives:

```
Circuit → Series(Circuit, Circuit)
Circuit → Parallel(Circuit, Circuit, blend_weight)
Circuit → Feedback(Circuit, DAE_node)
Circuit → Primitive                      ← base case
```

**Key property: closed under composition.** Each parallel branch is itself drawn from
the same grammar at lower complexity. You do not enumerate topologies — primitives plus
composition rules generate the full library recursively. This recursion is also what
makes warm-starting branches from converged lower-complexity fits possible.

The Klon as a sentence:

```
Parallel(
    Gain,                                                       ← clean path
    Series(Gain, RC_HP, Feedback(OpAmp, Diode_pair), RC_LP),   ← dirt path
    blend_weight = α                                            ← Blend knob
)
```

---

## 5. The Graph Representation

### Node types

```
ElementNode:
    node_type     identifier into primitive library
    parameters    learnable dict (R, C, f0, Q, Is, n, ...)
    instance_id   distinguishes repeated elements of same type
    forward()     ODE step or static map

JunctionNode:
    Split   one input → multiple outputs (copies signal)
    Sum     multiple inputs → one output (learnable blend weights)
```

Repeated elements (e.g. seven GE-7 bands) share `node_type` (same ODE, same forward())
with independent parameters — parameter sharing vs independence is an explicit choice.

### Edges

```python
# Known schematic: hardcoded edges, weights fixed at 1.0
# Unknown topology: learnable
edge_weights = sigmoid(edge_logits) * hard_mask   # hard_mask permanently zeros illegal edges
# + L1 sparsity regularization on edge weights during topology search
```

### Forward pass

Topological sort of the feedforward DAG; each node sums weighted predecessor outputs
then applies its forward(). With soft edge weights during search, all paths run
simultaneously gated by their weights; the sort only matters after pruning.

### Dense initialization for topology search

Start with all legal N² connections active at small/prior-informed weights, train
with L1 penalty, threshold near-zero edges. This is **LASSO on graph structure**.
Manage cost via: hard-banned illegal edges, type-compatibility grouping, hierarchical
search (coarse block structure first, parameters second), and the Level-1 frequency
screening (§8) to prune candidates cheaply before ODE training.

---

## 6. Constraint Classes

Three classes with different mathematical character.

### Hard constraints (mask to zero permanently — never gradient-updated)
- **Causality / acyclicity:** feedforward graph must be a DAG. Zero-delay loops are
  physically impossible (every real feedback path has at least one capacitor of delay).
  Feedback only via explicit DAE nodes.
- **Parameter positivity:** time constants, resistances, Q must be positive — which
  also guarantees linear-stage stability analytically.
- **Passivity:** purely passive subgraphs cannot have gain > 1.
- **DC sanity:** flag learned pure differentiators (zero DC gain, unbounded HF gain).

### Soft priors (initialization bias from circuit-design convention)
Encoded as a **type compatibility matrix** T[i,j] = prior probability of edge from
node type i to type j; initializes edge logits. Known high-prior sequences:
- Buffer/gain before high-impedance tone networks
- Filter before clipper (pre-clipping harmonic shaping — the Klon's HP→diodes is canonical)
- Clipper before output filter (post-clipping tone shaping)
- Output buffer as final stage

Known degenerate / low-prior combinations:
- Two identical linear stages in series with no nonlinearity between (collapsible —
  redundant parameters, flat loss landscape)
- Clipper with no upstream gain (mV-level signal never reaches the ~0.6 V silicon knee —
  assumed-typical value)
- Parallel branches with identical topology and converged parameters (redundant)
- Feedback around a purely linear chain (represent as a single biquad instead)

### Differentiable stability penalty
Jacobian eigenvalues at the operating point must have negative real parts
(continuous-time) / magnitude < 1 (discrete). ReLU penalty on positive real parts.
Expensive; use sparingly. Discrete-time analog: poles inside the unit circle.

### Bayesian framing (long-range)
The compatibility matrix is a prior over graph structures; spike-and-slab priors on
edge/branch activations + MCMC over the graph posterior gives **uncertainty over the
decomposition** — is the parallel path load-bearing? Hard constraints shrink the
search space enough to make this tractable. Natural fit to Brian's MCMC background.

---

## 7. The Graph Library (G_1 … G_9), Ordered by Complexity

Design rule: **each level adds exactly one topological concept** — clean attribution
when fit improves.

| Level | Topology | ~Params | Captures | New concept |
|---|---|---|---|---|
| G_1 | In → Gain → Out | 1 | Clean boost, buffer | Baseline |
| G_2 | Gain → RC (LP or HP) | 2 | Treble bleed, simple tone | First dynamic element |
| G_3 | Gain → Biquad | 4 | Single EQ band, fixed wah | Resonance (complex pole pair) |
| G_4 | Gain → Biquad×N cascade | 3N+1 | **Boss GE-7 (N=7)**, tone stacks | Repeated element, series composition |
| G_5 | Gain → SoftClipper → LP | ~6 | Simple overdrive | First nonlinearity (Shockley) |
| G_6 | Gain → HP → SoftClipper → LP | ~8 | Tube Screamer topology (Hammerstein-Wiener) | Pre-clip filtering |
| G_7 | Split → {clean Gain ∥ G_6 dirt} → Sum(α) | ~10 | **Klon-class blend pedals** | Parallel paths, learnable blend |
| G_8 | G_7 with DAE feedback clipper in dirt path | ~12 | **Full Klon** | DAE node, Newton-Raphson |
| G_9+ | + envelope follower, VCA, delay line | — | Trem, comp, auto-wah, modulation | Time-varying elements |

GE-7 parameter-recovery test at G_4: do learned f0 values match the spec-sheet band
centers? (Spec values to be pulled from the Boss sheet at implementation time — do not
trust recalled numbers.)

---

## 8. The Fitting Hierarchy

Four levels; each uses the level below as initialization.

```
Level 1 — Transfer Function Screening (frequency domain)
    H_empirical = FFT(wet)/FFT(dry); curve-fit to candidate analytical TF shapes
    Cost: milliseconds. Linear regime only.
    Uses: topology screening; parameter init for Level 2.
    Nonlinearity detector: amplitude dependence of H_empirical
        (H at low sweep amplitude ≠ H at high amplitude → nonlinear element required).

Level 2 — ODE Parameter Estimation (time domain, known topology)
    Nonlinear least squares on ODE output vs wet. Init from Level 1.
    Cost: minutes. Captures nonlinear dynamics the TF view misses.

Level 3 — Graph Search + ODE Training (unknown topology)
    Sparsity-regularized edge learning; Level 1 pre-screens candidates;
    warm starts throughout. Cost: hours, pruned heavily by screening.

Level 4 — GNN Topology Search (corpus-scale, long range)
    Message passing over circuit graphs; learns grammar priors across many devices;
    generalizes to unseen pedals. Cost: days; requires a capture corpus.
    Prior art note: GNN-for-circuit-simulation is an active research area
    (EDA/SPICE acceleration) — medium confidence on specifics; survey before building.
```

Time/Laplace/frequency are three views of the same object: the ODE is the general
view (handles nonlinearity); H(s) is its Laplace transform / linearization; warm
starting lives at exactly this interface.

---

## 9. Training Strategy (when implementation begins)

**Bottom-up curriculum in three phases:**

**Phase 1 — Pretrain primitives independently** on synthetic signals designed to
excite each one (calibration-signal philosophy applied to the vocabulary):
white noise → RC cutoffs; stepped sine sweeps → biquad f0/Q; amplitude-swept sines →
clipper knee and saturation. Each node arrives at topology search knowing roughly
what it is.

**Phase 2 — Greedy topology search with warm starts.** For k = 1…K: build G_k from the
grammar; initialize primitives from Phase 1 and parallel branches from the best G_{k−1}
fit; brief warmup with primitive params frozen (learn topology weights only); then
joint training with sparsity regularization; evaluate fidelity gate; on failure,
**diagnose which branch/node is failing** and let the residual guide G_{k+1} —
complexity is added where the residual demands it, not blindly.

**Phase 3 — Fine-tune the selected topology** with sparsity penalty removed (the
penalty biases parameters slightly off their true values) and a tighter LR.

**Topology-aware loss:** branch-level supervision where parallel paths exist —
approximate the clean component as the low-THD/linear-filtered part of wet, the dirt
component as the harmonic residual, and give each branch an independent loss term in
addition to the full-output loss. This is blind source separation with a far stronger
prior than ICA/NMF: sources must be realizable circuit topologies. For the Klon, the
separation is perceptually verifiable at Blend extremes.

**Why the recursive structure trains well:** gradients reach each sub-circuit along
paths mirroring physical signal flow; parallel branches get independent gradient
signals; failures localize to branches and are diagnosable — versus a flat black box
where a bad fit carries no structural information.

**Carried-over hard rule from PedalDSP:** no >100-epoch run executes automatically;
Brian triggers long runs manually. Coverage fraction and key diagnostics printed at
every run start.

---

## 10. Stopping Criterion and the MDL Framing

The complexity-ordered search with a stop-on-pass gate is **Occam's razor as an
algorithm** — minimum description length: the simplest graph that explains the data.

```
F(θ_k) = fidelity_score − λ · complexity(G_k)     # complexity = node/edge/param count
Stop at the first G_k that passes the fidelity gate.
```

**The over-specification test:** run a too-complex graph (G_7 on the GE-7) — it should
fit, but degenerately: blend α → 0 or 1, clipper learns near-linear behavior. The model
communicates "I don't need these elements" through its parameters. The 2×2 validation
matrix (GE-7 / Klon × G_4 / G_7) makes this a designed experiment, with the lower-left
cell (Klon on G_4) producing an **informative failure**: residual concentrated in the
high-amplitude nonlinear regime, pointing at exactly what is missing.

---

## 11. The Proof-of-Concept Rung Ladder

Each rung proves exactly one new thing and gates the next. (See the working skill —
the rungs are also decision-discipline anchors: revisit a rung's design only when its
data says so.)

### Rung 1 — TF recovery on the Boss GE-7 (linear, known answer)
Log sine sweep at flat setting → H_empirical → fit G_4 biquad cascade → compare f0
to spec sheet. **Pass: band centers within ±10% of spec.** If this fails nothing else
matters — it validates the measurement pipeline. Failure diagnostics: SNR, sweep
amplitude (stay linear), FFT length/windowing. A single afternoon's work.

### Rung 2 — ODE parameter recovery on the GE-7 (warm-start benefit + generalization)
Capture ≥4 documented band configurations. Train G_4 ODE from (a) random init,
(b) Rung-1 warm start. **Pass: warm start converges in fewer steps; ESR below threshold
on a held-out setting.** Proves generalization across knob positions, not single-setting fit.

### Rung 3 — Topology recovery on the GE-7 (structure from data)
Run the search blind from G_1 upward. **Pass: selects series biquad topology; parallel
branches degenerate; recovered f0 match Rungs 1–2.** Includes the over-specification
test (G_7 on GE-7 → degenerate parallel path).

### Rung 4 — Nonlinearity detection on the Notaklon
Multi-amplitude sweeps at moderate Gain. **Pass: H_empirical is amplitude-dependent
(nonlinearity flag fires); G_4 fails the gate at high amplitude with residual
concentrated in the nonlinear regime; G_5/G_6 passes; transition amplitude consistent
with silicon diode forward voltage (~0.6 V, assumed-typical).**

### Rung 5 — Parallel path recovery on the Notaklon (the headline result)
Captures at ≥3 Blend positions. **Pass: search selects parallel topology (G_7/G_8)
unprompted; dirt branch contains the nonlinearity, clean branch ≈ linear; learned α
correlates with the physical Blend knob across captures.** α tracking the knob is the
proof the model recovers a *physically meaningful* parameter, not a curve fit.
Dirt-branch parameters should be consistent with Rung 4.

### Rungs 6–8 (horizon)
6: GNN trained across multiple pedals, generalizing to unseen topology.
7: Bayesian structure learning — MCMC over the graph posterior, uncertainty on which
elements are load-bearing.
8: Knob-conditioned single graph spanning a pedal's full control space.

---

## 12. Capture Philosophy: Tile the Extremes, Interpolate the Middle

**Principle:** if training signals span the convex hull of the operating space, every
real playing condition is an interior point — and interpolation under a physics-grounded
model is reliable because the ODE constrains what can happen between boundary points.
Extrapolation beyond the hull is never validated; design captures so it never has to be.

```
Capture the extremes → physics interpolates the interior → never extrapolate
```

Notaklon boundary survey: max gain × max amplitude (saturated clipping), min × min
(linear, diode invisible), Blend fully wet and fully dry (branches in isolation).
Capture protocol per device:

```
Amplitude axis:   min, max, a few interior points (amplitude ladder)
Frequency axis:   near-DC, near-Nyquist, resonances
Knob positions:   each knob at min / max / noon
Combinations:     corner combinations of knob extremes
Interior points:  held out as VALIDATION, not training —
                  correct interpolation there confirms physics, not curve-fitting
```

This bounds the capture count: corners + a few interior checks. A two-knob pedal is
~a dozen captures; a five-knob amp more, but far below a dense grid.

**The dangerous failure mode:** dense middle coverage with empty corners — looks great
on the training distribution, fails exactly at the musically important extremes (hard
pick attack, volume swell into a cranked pedal).

**Product implication:** because the training hull is designed, a deployed model can
ship with an **operating-range guarantee** — validated for inputs/settings within the
captured hull, physically-extrapolated-but-unvalidated outside it. No commercial
modeler can make that claim today because black boxes don't know where their hull is.

PedalODE captures don't wait on the DI-box guitar setup: sine sweeps and amplitude
ladders are the primary signals; guitar DI is later validation. (Guitar playing
under-samples the hard-clipping regime — known lesson from PedalDSP Big Muff work.)

---

## 13. Identifiability, Uniqueness, and Extrapolation

**Uniqueness is not guaranteed.** The fit is non-convex; multiple decompositions can
match the data (the identifiability problem in system identification). Examples:
overlapping HP→LP vs single bandpass; two weak clippers in series vs one stronger;
a near-degenerate parallel path vs a series correction. Linear devices are much better
behaved — a transfer function has a unique pole/zero decomposition up to section ordering.

**When it doesn't matter:** deployment. Two decompositions that both pass the gate and
both deploy correctly are equivalent for sounding right.

**When it matters:**
- *Comparative library work* — cross-device comparison needs canonical grammar forms
  (fixed ordering conventions: filters before clippers, branches ordered by complexity).
- *Extrapolation* — the sharp issue. Two decompositions identical on the training
  distribution can diverge arbitrarily outside it. The physically correct one (true
  Shockley parameters, real blend structure) extrapolates because its primitives are
  the mechanism; a mathematical-coincidence fit extrapolates wrongly because its
  primitives were fitting data, not physics.

**Mitigation — physics grounding throughout:** hard constraint masks and type priors
push toward realizable topologies; describing-function initialization anchors nonlinear
parameters near physical values; and the amplitude ladder forces any coincidental fit
to match the physics across the full amplitude sweep — a much harder accident.
Uniqueness and extrapolation have the same answer, and it is the whole argument for
PedalODE over a black box.

---

## 14. Metrics: Capturing What the Ear Says

The open problem. ESR/MSE reward sample-level agreement; the ear does feature
extraction. Two signals can share high ESR and sound identical (1 ms shift) or low ESR
and sound different (audible −40 dB harmonic). The ear, roughly: outer/middle-ear
weighting → cochlear filterbank (~24 critical bands, quasi-log axis) → compression and
masking → temporal fine structure / envelope coding. Good metrics approximate stages
of that cascade.

### Composite metric suite (organized by failure mode caught)

```
Static accuracy
    ESR                      waveform fidelity — necessary, not sufficient
    A-weighted ESR           errors weighted where the ear is sensitive
                             (A-weighting / ISO 226; ITU-R BS.1770 the more rigorous
                              loudness standard — high confidence these exist as named;
                              implementation via pyloudnorm, medium confidence on API)

Spectral character
    Mel-scale STFT loss      perceptual frequency warping; standard in neural speech
                             synthesis training (high confidence on the practice)
    Per-harmonic amplitude error   even (warm) vs odd (harsh) harmonic balance —
                             total energy can match while character is wrong
    THD_err                  total nonlinear character (existing PedalDSP metric;
                             the one that exposed the Big Muff linear-filter failure)
    IMD error                intermodulation on multi-tone inputs — a model right on
                             single notes and wrong on chords fails here
    Harmonic phase           relative phase of harmonics affects timbre
                             (medium confidence on perceptual weight; less studied)

Temporal character
    Transient ESR            ~10 ms attack-window ESR; the ear identifies timbre from
                             the first few ms
    Spectral flux error      frame-to-frame spectral change — harmonic bloom on pick
                             attack; static metrics can pass while this fails
                             ("plasticky" steady-state-only models)
    Envelope shape error     amplitude envelope via Hilbert transform / analytic signal

Perceptual reference
    PEAQ / ODG               ITU codec-quality standard — high confidence it exists;
                             designed for codec artifacts, may be insensitive to
                             pedal-modeling failure modes; compute as reference only
```

### Gate structure (carried from the conversation; thresholds are starting points)

```python
passes = (esr < 0.05) and (thd_err < 0.10) and (spectral < 0.02)   # all gates, not a sum
score  = 0.4*esr + 0.4*thd_err + 0.2*spectral                       # for ranking only
```

A model must clear **all** gates — the gate structure prevents the topology search
from stopping at a linear approximation of a nonlinear pedal (the exact PedalDSP TCN
failure). Always ask: *what would this metric hide?*
Known trap: per-column-normalized heatmaps mislead on uniformly under-distorting
model fields — absolute scale matters.

### The metric you actually want (research path)
Speech synthesis is ahead here — learned perceptual metrics trained on human ratings
(UTMOS / NISQA / DNSMOS family — high confidence the family exists; verify names
before citing). No equivalent exists for guitar effects. Path: run blind A/B/X (or
MUSHRA-style) listening tests across the existing model zoo → collect ratings →
train a small neural predictor on (metric vector, rating) pairs → that predictor
becomes the perceptual metric for everything after. This is the already-flagged
PedalDSP perception-correlation study, and it replaces the z-score composite
placeholder in the perceptual metrics suite. It is a contribution in itself.

Also connects to the six-metric perceptual FFT suite already designed for PedalDSP
(mel/bark warping, equal loudness, masking, signal-adaptive weighting, guitar prior,
harmonic structure weighting) — PedalODE consumes that suite rather than rebuilding it.

---

## 15. Deployment: From Decomposition to Digital Version

The PedalODE output deploys **more** directly than the TCN path, not less.

**Linear devices (GE-7):**
```
Learned (f0, Q, gain) per band
  → bilinear transform (s-domain → z-domain; one-time calculation)
  → biquad coefficients (b0,b1,b2,a1,a2) per section
  → Direct Form II
  → runs anywhere: 5 MACs/sample/band; GE-7 = ~35 MACs/sample
```
No network at runtime. The decomposition for a linear device is not an approximation
of the circuit — it *is* the circuit, expressed mathematically.

**Nonlinear devices (Notaklon):**
```
Learned ODE parameters (RCs, Is, n, α)
  → discretize (explicit Euler or trapezoidal) at target sample rate
  → per-sample state update + Newton-Raphson solve for the diode constraint
  → ~20–50 FLOPs/sample (assumed-typical estimate) vs thousands for a meaningful TCN
```

**One-sentence contrast with PedalDSP:** TCN learns a black box that approximates the
behavior and must run that box at inference; PedalODE learns the parameters of a known
physical model and at deployment you just run the physics. The physics is always
cheaper than the network that learned to imitate it — and behaves better outside the
training distribution (§13).

**Complementarity:** PedalODE deploys directly where decomposition succeeds; PedalDSP
catches everything else (too complex, unknown vocabulary). Results also flow
PedalODE → PedalDSP: a discovered topology informs the physics-informed (Tier 8
gray-box) architecture; the Rung 5 Notaklon decomposition should match the
hand-derived schematic used there.

---

## 16. Project Structure

```
PedalODE/
├── README.md                        ← this document condensed
├── primitives/
│   ├── base.py                      # PrimitiveNode: analytical_tf(), describing_fn(), ode_step()
│   ├── rc_lowpass.py  rc_highpass.py  biquad.py  gain_stage.py
│   ├── clipper_soft.py              # Shockley + describing function
│   └── clipper_hard.py
├── grammar/
│   ├── rules.py                     # Series / Parallel / Feedback composition
│   ├── graph.py                     # ElementNode, JunctionNode, edges, topological_sort
│   ├── constraints.py               # hard masks, type compatibility matrix, stability penalty
│   └── library.py                   # G_1 … G_8 definitions
├── fitting/
│   ├── tf_screening.py              # Level 1
│   ├── ode_training.py              # Level 2
│   ├── topology_search.py           # Level 3
│   └── warm_start.py                # analytical initialization
├── validation/
│   ├── metrics.py                   # gates + composite suite (§14)
│   └── parameter_recovery.py        # learned params vs spec sheet / schematic
├── captures/
│   ├── ge7/                         # documented settings, JSON sidecar manifests
│   └── notaklon/                    # documented Gain/Tone/Blend
└── experiments/
    ├── rung1_ge7_tf_recovery.py
    ├── rung2_ge7_ode_warmstart.py
    ├── rung3_ge7_topology_search.py
    ├── rung4_notaklon_nonlinearity.py
    └── rung5_notaklon_parallel_path.py
```

Conventions inherited from PedalDSP: Marimo notebooks at 96 kHz; JSON sidecar manifests
on all generated signals (every parameter, seed, generator version, labeled sample
boundaries); dark-background plot palette (target #00FF88, predicted #FF6B6B,
dry #4FC3F7, error #FFD700); uv + PyTorch stack; key diagnostics printed every run.
Each experiment file is self-contained and independently reproducible.

Implementation traps to carry into every Claude Code prompt: Newton-Raphson must be
torch-native (no scipy); IIR/biquad processing is sequential; recurrent anything needs
contiguous chunks; state assumptions at train and inference must match.

---

## 17. Commercial Framing (honest)

**What doesn't exist today:** commercial modelers (Neural DSP, Kemper, Line 6) are
capture devices, not understanding devices — they fit black boxes and cannot say what
the circuit is doing or how it behaves outside training data. PedalODE would be the
first system producing an **interpretable circuit decomposition from captures alone**.
(Kemper profiling is closer to classical gray-box system ID than deep learning —
medium confidence on internals; their method is proprietary.)

**Where value lives:**
1. **Reverse engineering / documentation** — boutique builders, repair techs,
   authenticity verification of vintage units. Niche, high willingness to pay.
2. **B2B capture front end** — topology discovery as initialization/constraint for
   commercial neural modelers: less capture data, better generalization. Licensing play.
3. **Inverse design** — run the grammar backwards: specify a target transfer function /
   harmonic character, solve for topology + component values. Replaces breadboard
   iteration for builders.
4. **Forensic / valuation** — objective circuit fingerprints for disputes, insurance,
   authentication. Small, high-value.

**Defensibility:** the individual techniques are not novel; the combination + domain is
a methods position, replicable by a funded competitor. The real moats: (a) the curated
primitive library and constraint knowledge (compatibility matrix, known-bad
combinations — years of circuit expertise encoded), and (b) **the data flywheel** —
a corpus of decompositions becomes proprietary training data for the Level-4 GNN:

```
more captures → better GNN priors → faster search on new pedals → lower contribution
barrier → more captures
```

**Honest caveat:** the pedal market is real but niche. The framework generalizing to
*any* analog audio hardware (synth voice cards, vintage compressors, tape electronics,
console strips) is the larger addressable claim. Commercial pursuit is optional — the
research contribution and personal-tool value stand alone.

---

## 18. The Corpus and the Full-Chain Vision

### The library as a living asset
Brian's pedalboard plus friends' boards = a distributed capture network. The protocol
(sweep + amplitude ladder + documented settings) is simple enough for anyone with a DI
and interface. At scale this becomes a community circuit-fingerprint database — like
Discogs for analog hardware, except entries are objective mathematical decompositions:
two people capturing the same model should agree within measurement noise. That
reproducibility, enabled by the common representation language, is what makes it
science rather than a catalog. Comparative questions become answerable: do Klon clones
actually cluster in diode-parameter space? What grammar feature predicts "sounds like
a Tube Screamer"? Germanium vs silicon Fuzz Face in circuit terms?

### Extension to amplifiers (natural, two new primitives)
- **Supply sag node:** current-dependent supply droop with its own RC constant — a
  dynamic feedback from output load to effective rail voltage; a large part of
  power-amp compression character (Fender vs Marshall — medium confidence on the
  attribution magnitude, high on the mechanism).
- **Speaker/cabinet primitive:** coupled electromechanical system (inductance,
  back-EMF, cone resonance, thermal compression) — lumped 2nd/3rd-order mechanical ODE
  coupled to the electrical one; biquad as first approximation. Cone flexure modes are
  technically PDE territory; lumped modal treatment suffices in practice.

### Extension to microphones
Signal chain run backwards: acoustic → electrical. New primitive: the **acoustic
capsule** — a damped harmonic oscillator ODE (resonant frequency, Q, polar pattern) in
the acoustic domain; composition rules unchanged. A condenser ≈
Series(AcousticCapsule, ImpedanceConverter, TransformerStage, OutputBuffer).
"Unified ODE grammar for acoustic-electrical transduction" is publishable on its own;
objective capsule characterization doesn't exist for recording engineers today.

### Extension to the instrument itself ("annoying guitar stuff")
- **String primitive:** the 1D wave equation handled as **modal decomposition** —
  damped sinusoids with fundamental, inharmonicity coefficient (stiffness), per-mode
  decay rates. Standard in physical-modeling synthesis (Karplus-Strong, modal
  synthesis — high confidence).
- **Pickup primitive:** a second-order resonant system set by coil L, R, and load C
  (tone pots + cable). Explains humbucker darkness (higher L → lower resonant peak)
  and cable-capacitance effects from first principles. Plus a **spatial sampling
  primitive**: pickup position weights the string's modal amplitudes (node sampling
  cancels harmonics → bridge bright, neck warm) — not an ODE, a linear modal weighting,
  but mathematically clean in the grammar.
- **Body/tonewood primitive:** boundary-condition modification — body resonant modes
  couple to string decay through bridge/saddle, as a weak feedback path perturbing
  per-mode decay rates. For electrics the coupling is real but small relative to
  pickup/electronics (medium confidence; this is precisely the empirical question).
  **The tonewood debate becomes measurable:** estimate the coupling coefficient in the
  grammar and compare it to perceptual threshold. A number instead of a forum war.

Same treatment dissolves: does expensive cable matter (resonant-peak shift vs
audibility), are boutique pickups different (compare L/R/f_res), vintage vs reissue
(compare full fingerprints), why new strings sound different (inharmonicity + decay
rates).

### The instrument fingerprint
Full-chain decomposition = a complete structured description from string to speaker.
Identical fingerprints → identical sound through the same chain; different fingerprints
→ predictable, quantifiable differences. Applications: valuation, manufacturing QC,
session tone-matching, insurance documentation, quantitative organology.

**Naming:** PedalODE stays the name for the pedal proof-of-concept. The full-chain
framework gets its own name later (ChainODE / SignalDNA / TBD). Not a current decision.

---

## 19. Working Disciplines (from the PedalODE skill — binding on all work)

- **Decision discipline:** decisions are explicit named commitments — state
  alternatives, commit, note what evidence reopens it. The discriminator between
  healthy iteration and perturbation: *did new data arrive?* Probes are legitimate
  when named as probes and read for what they reveal about skipped prerequisites.
- **Skeptical collaborator stance:** ask before assuming; correct real technical
  errors directly; explain reasoning legibly enough to be challenged; don't capitulate
  to pressure, reverse only for reasons.
- **Hallucination control:** confidence-mark all from-memory citations (in chat AND in
  files — this file follows the convention); search when a number is load-bearing and
  verifiable; distinguish measured / derived / assumed-typical; don't invent precision.
- **Claude Code handoff:** Claude holds concepts, Claude Code holds code state; the
  repo model here is always stale — state assumptions at the top of every
  implementation prompt; define the report-back before diagnosing.
- **Honest rung test:** parameter recovery vs ground truth, not just low fit error —
  a model can fit well and recover the wrong circuit.

---

## 20. Status and Entry Point

**Status:** design complete; implementation not started. Blocked on nothing — Rung 1
does not require the DI boxes (sweeps through the GE-7 via the Studio 1824c suffice).

**Entry point when ready:** Rung 1 — sine sweep through the Boss GE-7 at flat setting,
FFT division, biquad-cascade fit, compare to spec sheet. One afternoon. Everything
else builds from there.

**Key open items before Rung 3+:**
- Pull actual GE-7 spec-sheet band centers (do not trust recall)
- Lock the fidelity-gate thresholds as a named decision (current values are starting points)
- Decide canonical grammar ordering conventions (matters for the corpus, not for Rung 1–2)
- Literature survey pass with verified citations: Wright et al. recurrent amp modeling,
  Chowdhury 2020, neural-ODE audio work, GNN circuit-simulation work
  (all currently cited from memory at varying confidence — verify before writing the paper)

**The publishable arc:**
> *Physics-Informed Circuit Topology Discovery for Analog Audio Effect Modeling* —
> grammar-based graph search with analytical warm starts recovers physically correct
> topology and component values from dry/wet captures alone; validated by blind
> recovery of GE-7 band structure and Klon parallel blend path, with learned
> parameters matching known component values and the learned blend coefficient
> tracking the physical knob.

---

*v2, June 2026. Consolidates the full design conversation: factorization and warm-start
core, ODE/DAE foundations, primitives and grammar, graph representation and
constraints, G_1–G_9 library, four-level fitting hierarchy, training strategy, MDL
stopping, five-rung ladder, capture-the-extremes philosophy, identifiability and
extrapolation, perceptual metrics program, deployment path, commercial framing, corpus
and full-chain vision, and the binding working disciplines. Companion project to
PedalDSP at /mnt/d/Projects/PedalDSP.*
