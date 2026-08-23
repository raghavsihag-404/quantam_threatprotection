# QSENTINEL
## Technical Design Review and SIH 2026 Presentation Package
### Quantum-Inspired Runtime Threat Detection for Digital Signature Security

**PS-141: Quantum-Inspired Cyber Threat Detection for Digital Signature Security**

| | |
|---|---|
| **ARCHITECTURE** | Final / Frozen according to supplied authoritative documents (structural level frozen; numeric-claim level pending validation) |
| **IMPLEMENTATION** | Mapped in full per the current Implementation Blueprint; not yet built/measured |
| **VALIDATION** | Designed and prioritized; no headline figure is currently confirmed under the frozen architecture — every historical figure was measured under a superseded architecture and requires re-confirmation |
| **PRESENTATION** | 5-Slide SIH structure fully defined below |
| **VERSION** | Final Presentation Package |

---

# EXECUTIVE ARCHITECTURE STATUS

## What QSENTINEL Is

QSENTINEL is a runtime, **non-AI/ML** statistical monitoring layer that sits alongside a teleportation-distributed QS-L digital-signature (QDS) protocol. It observes only the classical measurement telemetry the protocol already discloses, and produces an **advisory** threat-observability verdict for every session, independent of — and never overriding — the protocol's own deterministic accept/reject decision.

## What QSENTINEL Is Not

- Not an AI/ML system. It contains no learned model, no training corpus, no generalization mechanism. Every decision function is a closed-form or Monte-Carlo-calibrated statistical test.
- Not a replacement for, or an enhancement of, the protocol's own cryptographic security proof. QS-L's unforgeability and non-repudiation guarantees are inherited, not re-derived.
- Not a gate. It cannot block, delay, or override protocol-level acceptance of a legitimate signature.
- Not a general-purpose intrusion detection system. It is scoped specifically to a teleportation-distributed QDS execution.

## The Two Decision Planes

QSENTINEL's entire architecture rests on keeping two questions, and two evidence channels, structurally separate:

- **Plane 1 — Authoritative Protocol Plane.** Deterministic evidence (freshness, authorization scope, sequencing invariants, FSM integrity). Asks: *"Is this execution valid according to the protocol?"* Owns the accept/reject decision. Terminal, logged, never overridden by QSENTINEL.
- **Plane 2 — Advisory QSENTINEL Monitoring Plane.** Probabilistic evidence (quantum measurement telemetry). Asks: *"Does this execution statistically and temporally resemble legitimate behavior?"* Produces an annotation — a signed forensic log entry and a SOC-facing alert — and nothing else.

## Core Detection Pipeline

`Session Transcript → { FSM (protocol-authoritative) ‖ Quantum Evidence Collector } → Stage 1 (model-fit / mutual-consistency) → Stage 2 (joint calibrated statistical decision) → GLR-CUSUM (unconditional, cross-session) → optional forensic Attribution → signed, hash-chained Forensic Log → SOC alert (advisory)`

## Technology Boundary

Python 3.11+, a custom fixed-3-qubit NumPy statevector simulator, NumPy/SciPy for all statistics, `multiprocessing` for Monte Carlo, FastAPI + Pydantic v2, React + Vite, SQLite + flat-file artifacts, Ed25519 + SHA-256 for forensic integrity. No AI/ML library, no Qiskit/Cirq/PennyLane, no message broker, no distributed infrastructure, no additional database, anywhere in the stack.

## Evidence and Forensic Boundary

Every session — flagged or not, protocol-accepted or not — produces a signed, hash-chained forensic record. Forensic attribution (identifying *which* attack sub-type occurred) is optional, runs only on already-flagged sessions, and is never load-bearing for a basic monitoring outcome.

## Major Design Principles

1. Deterministic and probabilistic evidence are structurally separate, never fused into one score.
2. Every statistical component must show independent, ablation-measured marginal value.
3. No independent-threshold composition is trusted where a jointly calibrated one is available.
4. Calibration and evaluation are separated as a binding, runtime-enforced guarantee, not a convention.
5. QSENTINEL's non-interference with protocol acceptance is enforced structurally (immutable objects, CI-checked import boundaries), not by developer discipline alone.
6. No claim is made beyond what is either mathematically inherited (from QS-L) or will be empirically demonstrated (by the harness in Section 12 of the architecture freeze). "Designed" and "demonstrated" are never conflated.

---
---

# PHASE 1 — TECHNICAL DESIGN REVIEW

## 1. Problem Analysis

### 1.1 Problem Context

PS-141 asks for quantum-inspired cyber threat detection for digital signature security. The underlying system is a teleportation-distributed variant of the QS-L quantum digital signature protocol: a message qubit is encoded onto a Pauli eigenstate, distributed via quantum teleportation over a shared Bell pair, corrected at the recipient using classically-communicated Pauli-correction information, and verified using QS-L's own asymmetric threshold rule (`s_a < s_v`). This protocol carries its own formal unforgeability and non-repudiation guarantees, proven in the published QS-L literature, and inherited by this system rather than re-derived.

### 1.2 The Security Gap

A cryptographic protocol's deterministic accept/reject rule answers a narrow, precise question: given a transcript, does it satisfy the protocol's own threshold and structural rules? It does **not** answer a broader, temporally-extended question: does the *pattern* of measurement outcomes across a session — and across many sessions — remain statistically consistent with what a properly-functioning, honestly-operated channel produces?

These are different notions, and the difference matters:

- **Validity** is a per-transcript, deterministic property. A transcript either satisfies the protocol's rules or it does not.
- **Statistical consistency** is a distributional property: do the measurement outcomes' mismatch rate, correlation, and entropy behave the way an honest channel at *some* legitimate noise level would produce?
- **Temporal consistency** is a cross-session property: does a channel that looks acceptable in any single session remain stable in its statistics across many sessions, or does it drift persistently in a way no single session would flag?

A claim this document explicitly avoids: it is **not** true that every valid signature is suspicious, and QSENTINEL never treats "flagged" and "invalid" as synonyms. A protocol-valid session can be advisory-flagged (its statistics look off relative to the honest model); a protocol-valid session is never blocked because of that flag.

### 1.3 Two Security Questions

**Question 1:** Is the protocol execution valid? — Answered deterministically and authoritatively by the protocol itself (FSM invariants plus QS-L's own threshold rule).

**Question 2:** Does the observed execution remain statistically consistent with legitimate behavior, in this session and across the session history of this channel? — Answered advisorially, probabilistically, by QSENTINEL.

These questions are not identical because a statistically clean, single-session forgery that reproduces the legitimate noise distribution exactly is, in principle, indistinguishable from legitimate traffic to *any* statistical monitor — an information-theoretic limit, not an engineering gap. Conversely, a channel can drift or exhibit unusual (but ultimately harmless) statistics without ever producing an invalid transcript. Collapsing the two questions into one would either (a) create a false sense that statistical monitoring adds cryptographic security it cannot add, or (b) create a false sense that protocol validity implies behavioral normality, which it does not.

### 1.4 Why Existing Detection Approaches Are Insufficient

Per the competitor landscape referenced in the frozen architecture (Architecture, Sections 3B/6, summarized in Section 9 "Competitive Position"): reviewed teleportation-QDS protocol papers prove security at the protocol-design level and stop there — none attempts a separate runtime statistical monitoring layer. QKD channel-monitoring literature (QBER thresholds, entanglement-witness diagnostics) solves an adjacent but different problem (key secrecy, not signature-verification integrity) using structurally similar tools. Classical FSM/IDS literature covers deterministic protocol-state checks but has no teleportation-specific quantum observable. No existing line of work combines a teleportation-specific quantum observable (Pauli-correction consistency) with protocol-session-lifecycle awareness the way this design does. This is a scoped competitive claim, not a claim of broad superiority — see Section 9 of the design review below for the full, bounded statement.

### 1.5 Why QSENTINEL Exists

QSENTINEL exists to answer Question 2 without ever touching the answer to Question 1: to give a deployed teleportation-QDS system continuous, non-interfering, statistically disciplined visibility into whether its own channel is behaving the way its threat model assumes, including visibility into slow, cross-session drift that no single-session check — cryptographic or otherwise — can see by construction.

---

## 2. Scientific and Technical Formulation

### 2.1 Protocol Evidence

**Owner: PROTOCOL. Authoritative. Deterministic.**

The Protocol State Integrity Engine (FSM) checks transcript metadata — timestamps, tokens, sequencing — against a bounded set of invariants: freshness (single-use, non-replay), authorization scope (the requesting party holds a token valid for the requested operation), and sequencing (protocol steps occur in the required order). A violation is a protocol-level `REJECT`: authoritative, terminal, logged, and never overridden by QSENTINEL. This is the sole detector for replay, unauthorized verification, and in-scope impersonation (a missing, invalid, or expired token) — ablation shows replay detection collapses from 100% to 1.5% without this layer.

### 2.2 Quantum Measurement Evidence

**Two independent dimensions, not four unrelated features.** The Quantum Evidence Collector computes three measurement-derived quantities — mismatch rate `m`, correlation `C = 1 − 2p`, and entropy `H` — from the classical, BB84-style sifted measurement telemetry the protocol already discloses, plus one architecturally distinct observable, Pauli-correction consistency.

Under the honest-execution (H0) model, `m`, `C`, and `H` are all functions of a single scalar depolarizing-channel parameter `p` — they are **provably redundant as independent estimators** under H0. This is not a weakness to paper over: it is precisely why testing their **joint consistency** against the single-parameter relationship is a real structural signal. An attack that does not respect the depolarizing model's symmetry (a Pauli-structured manipulation, an unmodeled/structural burst) breaks the fixed `m`–`C`–`H` relationship the honest model predicts, even though it may not move any single one of them far enough to look anomalous on its own. This mutual-consistency test is exactly what Stage 1's goodness-of-fit computes — so the correct framing is: **a refined estimate of `p` (obtained from three physically related estimators whose mutual consistency is itself the signal) plus Pauli-correction consistency, the one genuinely quantum-native observable with no analog in a directly-transmitted-qubit scheme.**

### 2.3 Stage 1 — Model-Fit and Mutual Consistency

Stage 1 tests: `H0`: session statistics `(m, C, H)` are jointly consistent with *some* legitimate operating point `p` in the declared depolarizing-noise family. `H1`: the statistics are inconsistent with any such `p` — indicating drift or model misspecification.

Mechanically: the nuisance parameter `p̂` is profiled out (not plugged in as a point estimate) via `p̂ = argmax_p L(m, C, H | p)`; a joint mutual-consistency statistic `T = −2 log[L(data | p̂) / L(data | saturated model)]` is computed; `T` is compared against a critical value obtained by **simulation at the actual operating sample size (n ≈ 200)**, not a chi-square table lookup, because small-sample validity of the asymptotic critical value at this `n` is an outstanding, not-yet-validated task.

A model-invalid session is **not** treated as a confirmed attack, and is not silently absorbed as ordinary evidence: it is routed to `MODEL_INVALID`, logged with `{p̂, T, calibrated_p_value, n}`, and distinguished explicitly from two other outcomes — a genuine optimizer non-convergence (`reason = "optimizer_non_convergence"`) and a legitimate boundary optimum (`p̂ = 0` or `p̂ = 0.5`), which is not a failure. Stage 1's role is precisely: separate "does the honest model even apply here" from "given that it applies, is this an attack" — a structural precondition for Stage 2, not a generic anomaly score.

### 2.4 Stage 2 — Joint Statistical Decision

Stage 2 tests `H0`: honest execution under the model Stage 1 validated, against `H1`: attack (channel manipulation or sub-threshold forgery). It observes a pair of statistics — a fast sequential log-likelihood-ratio statistic (`S_SPRT`) and a block-wise structural goodness-of-fit statistic (`S_gate`) — and evaluates them **jointly** against a rejection region `R`, calibrated once, offline, by Monte Carlo simulation of the joint null distribution, bound by a mandatory train/evaluate separation, targeting `α_system = 0.01`.

Independent thresholding on `S_SPRT` and `S_gate` separately was explicitly not selected: composing two marginal thresholds either requires assuming independence (not established) or accepting a looser Bonferroni-style bound. Calibrating the rejection region directly against the joint distribution is at least as tight as a Bonferroni bound without requiring that independence assumption — this is why Stage 2 is a **jointly calibrated decision**, never an independent-threshold comparison, and never described as an AI classifier (it contains no learned parameters and no training corpus).

Output: `ACCEPT / FLAG(REJECT) / FLAG(INVESTIGATE)` — advisory only. `R` is loaded from a frozen, content-hash-verified `CalibrationArtifact` and is **never recomputed at runtime**.

### 2.5 Cross-Session Monitoring

Low-and-slow attacks are, by definition, sub-threshold in any single session — a biased noise parameter applied consistently but too small to trip Stage 2 on any one session. Detecting this class of threat requires accumulating evidence across sessions, which is the sole job of the window-limited GLR-CUSUM detector.

**Every relevant session feeds this path unconditionally**, regardless of what Stage 2 concluded for that session — this is a structural requirement of the frozen architecture, not an implementation convenience, because low-and-slow drift is defined by being sub-threshold in any single session's Stage 2 outcome. `H0`: parameter stable at `θ0`. `H1`: an unknown post-change parameter `θ`, estimated online via `sup_θ`. For this exponential-family (Bernoulli/binomial-type) noise model, `sup_θ` per window-start is the sample-proportion MLE — **closed-form**, giving `O(w)` elementary operations per session, not a nontrivial per-window optimization. A flag fires only when the accumulated statistic `G_t` exceeds an offline-calibrated threshold — the same calibration discipline as Stage 2.

### 2.6 Forensic Layer

Every session — regardless of protocol or monitoring outcome — produces a signed (Ed25519), hash-chained (SHA-256) append-only forensic record. Optional attribution (per-hypothesis likelihood ratios distinguishing *which* attack sub-type likely occurred) runs only on already-flagged sessions and produces `CONFIDENT / AMBIGUOUS / NO ATTRIBUTION` — it is forensic enrichment for a SOC investigator, never a load-bearing component of the detection pipeline, and adds zero cost on the common (unflagged) path.

---

## 3. Complete Engineering Architecture

```
                          Session Transcript
                                  │
                ┌─────────────────┴─────────────────┐
                │                                     │
    PROTOCOL EXECUTION EVIDENCE                QUANTUM MEASUREMENT EVIDENCE
    (deterministic — FSM: freshness,           (probabilistic — 2 independent
     authorization-scope, sequencing)           dimensions: refined p̂ via m/C/H
                │                                mutual-consistency, Pauli-
        ┌───────┴───────┐                        correction consistency)
        │               │                                  │
      FAIL            PASS                                 ▼
        │               │                  STAGE 1 — Model-Fit / Mutual-
        ▼               │                  Consistency Check (profile-
     PROTOCOL-LEVEL      │                  likelihood goodness-of-fit)
     REJECT                │                             │
  (authoritative,           │              ┌──────────────┴──────────────┐
   terminal; never            │        fits noise family        doesn't fit / boundary
   overridden by                │            │                            │
   QSENTINEL)                    │            ▼                            ▼
        │                │             STAGE 2 — Joint                MODEL_INVALID
        │                │             Statistical Decision            (logged, routed
        │                │             (S_SPRT + S_gate,                distinctly from
        │                │              jointly calibrated,             optimizer failure
        │                │              α_system=0.01)                  and boundary optima)
        │                │                        │
        │                │              ACCEPT / FLAG(REJECT) /
        │                │              FLAG(INVESTIGATE) — advisory
        │                │                        │
        │                │        ┌───────────────┴───────────────┐
        │                │        ▼                                ▼
        │                │  every session unconditionally    (session summary
        │                │  feeds GLR-CUSUM (window-limited,  recorded regardless
        │                │  closed-form sup_θ, O(w))           of Stage 2 outcome)
        │                │        │
        │                │  flags persistent cross-session drift?
        │                │        │
        │                │  only if EITHER Stage 2 OR CUSUM flagged
        │                │        ▼
        │                │  ATTRIBUTION (optional, forensic-only,
        │                │  never load-bearing)
        │                │        │
        └────────────────┴────────▼
                 Signed (Ed25519), Hash-Chained (SHA-256)
                       Forensic Evidence Log
                                  │
                         SOC Alert Interface (advisory)
```

### Module Table

| Module | Input | Process | Output | Authority | Why It Exists |
|---|---|---|---|---|---|
| Protocol State Integrity Engine (FSM) | Transcript metadata: timestamps, tokens, sequencing | Deterministic FSM transition/invariant checking | PASS / protocol-level REJECT | Protocol — authoritative | Sole detector for replay, unauthorized verification, and in-scope impersonation; ablation shows 100%→1.5% replay-detection collapse when removed |
| Quantum Evidence Collector | Signed classical measurement telemetry | Compute `m`, `C = 1−2p`, `H`, Pauli-correction consistency | Two-dimensional observable vector | Feeds Stage 1/2 — no decision authority | Sole source of all quantum-channel-manipulation evidence |
| Stage 1 — Model-Fit / Mutual-Consistency | Observable vector | Profile-likelihood goodness-of-fit testing joint consistency of `m/C/H` against a single-parameter model | fits → routes to Stage 2 / MODEL_INVALID | QSENTINEL — advisory, structural precondition | Without it, model deviation collapses into unacceptable false positives or blindness |
| Stage 2 — Joint Statistical Decision | Stage-1-routed observable stream | `S_SPRT` + `S_gate`, jointly calibrated rejection region, `α_system=0.01` | ACCEPT / FLAG(REJECT) / FLAG(INVESTIGATE) | QSENTINEL — advisory | Sole mechanism for channel-manipulation and sub-threshold forgery detection |
| Window-limited GLR-CUSUM | Per-session summary, every session unconditionally | Closed-form `sup_θ` GLR change-point statistic | Flag / not flagged | QSENTINEL — advisory | Sole mechanism able to see persistent, low-and-slow cross-session attacks at all |
| Attribution Engine (optional) | Observable stream of an already-flagged session | Per-hypothesis likelihood ratios | CONFIDENT / AMBIGUOUS / NO ATTRIBUTION | QSENTINEL — advisory, forensic-only | Never load-bearing; zero cost on the common path |
| Forensic Evidence Log | Every module's evidence and decision | Signed, hash-chained append-only write | Immutable audit record | Evidentiary | Preserves evidence regardless of outcome, including protocol-accepts that QSENTINEL flagged advisory |

---

## 4. What Was Considered and Rejected

This section reflects the architecture freeze's own hostile audit (11 findings) plus the implementation blueprint's explicit technology rejections. Only alternatives genuinely relevant to the frozen architecture are listed.

### Rejected Architecture Table

| Rejected Approach | Why It Was Considered | Why It Was Rejected | Final Alternative | Net Effect |
|---|---|---|---|---|
| Treating `m`, `C`, `H` as four independent ML-style features | Superficially appears to give "more evidence" | Under H0 they are provably redundant as independent estimators of a single parameter `p` — presenting them as independent overstates rigor and fails a hostile mathematical review | Reframe as two independent dimensions: refined `p̂` via mutual-consistency testing, plus Pauli-correction consistency | Corrects an overstated-rigor claim into a defensible one at zero implementation cost |
| Leaving calibration/evaluation separation implicit ("maximize power against the declared attack set") | Seemed procedurally obvious | Silent leakage between calibration and evaluation would invalidate every future detection-rate figure | Binding train/evaluate separation, enforced at runtime by a `SeedAllocator` that raises on out-of-range seed requests | Removes the single largest threat to the credibility of every future detection-rate figure |
| Leaving QSENTINEL's relationship to protocol acceptance undefined | The frozen v5 architecture never stated it explicitly | A hostile reviewer could plausibly assume QSENTINEL is gating, not advisory — the single most consequential open finding | QSENTINEL fixed as annotate/escalate-only: never overrides, blocks, or delays protocol acceptance; enforced structurally (immutable objects, CI-checked import boundaries), not by convention | Closes the most consequential open finding at zero complexity cost |
| Independent-threshold composition for Stage 2 (`S_SPRT` and `S_gate` thresholded separately) | Simpler to implement and explain | Requires assuming independence (unestablished) or accepting a looser Bonferroni bound | Joint calibration of the rejection region `R` against the joint null distribution, held-out and Monte-Carlo-derived | At least as tight as Bonferroni without requiring independence; avoids a defensible-but-weaker fallback |
| Conditional GLR-CUSUM (running only after a Stage 2 flag) | Would reduce the number of sessions the temporal path has to process | Low-and-slow drift is defined by being sub-threshold in any single session's Stage 2 outcome; conditioning on a flag that low-and-slow attacks are specifically designed to avoid would make the detector blind to its own target class | Unconditional ingestion — every session feeds GLR-CUSUM regardless of Stage 2 outcome | Sole mechanism able to see this attack class at all is preserved intact |
| A general-purpose quantum circuit engine (Qiskit / Cirq / PennyLane) | Industry-standard, familiar tooling | The protocol never exceeds 3 qubits and needs no circuit compiler; general SDKs add compilation/backend-abstraction overhead for a problem this system doesn't have, and run slower per trial for a Monte Carlo harness executing thousands of sessions, while obscuring the exact operations a hostile-audit-style demo most wants to show | A custom, ~150-line, fully transparent, fixed-3-qubit NumPy statevector simulator | Faster per trial, fully inspectable, no dependency or opacity cost |
| A Bayesian inference engine (PyMC / Stan) for the statistical layer | Statistically fashionable, flexible | Nothing in Stage 1, Stage 2, or GLR-CUSUM requires MCMC — every formula is closed-form or simulation-calibrated | NumPy + SciPy with hand-written, auditable likelihood functions | Every formula maps directly to an explicit, inspectable function; no dependency on an inference engine the problem doesn't need |
| Distributed task infrastructure (Celery / Redis / Dask) for Monte Carlo execution | Seems appropriate for "large-scale simulation" | Thousands of independent, single-machine trials is an embarrassingly parallel problem with no cross-trial dependency; a broker/worker/serialization stack is pure operational overhead and does not improve reproducibility | Python `multiprocessing.Pool` with an explicitly pinned `spawn` context and per-trial `PCG64` seeding | Zero added infrastructure; cross-platform-identical results by construction |
| A second database (PostgreSQL / MongoDB) for scale or document flexibility | Anticipatory scaling | No concurrent-write contention exists at this stage (one demo session/experiment runner at a time); SQLite's schema uses only standard SQL types, so migration later is mechanical, not a rewrite | SQLite, with a documented, mechanical PostgreSQL migration path | Avoids standing up infrastructure for a need not yet demonstrated |
| Storing GLR-CUSUM state in a second store (JSON/NPZ flat file, alongside or instead of SQLite) | Would mirror the calibration-artifact storage pattern | Two stores for the same state creates crash-recovery ambiguity about which value is current | SQLite `CusumState` as the sole persisted home; an in-memory `deque` cache reconstructed from it at process start | Exactly one place the state can be — no ambiguity after a crash or restart |
| An AI/ML detector layer of any kind | Might improve stealthy-attack recall against an adaptive adversary | PS-141 explicitly requires a non-AI/ML approach; introducing one would violate the governing constraint regardless of any measured advantage in the literature | The current statistical pipeline (FSM + Stage 1/2 + GLR-CUSUM), explicitly acknowledged as not attempting to match ML-based detectors' measured advantage here | Stays within the mandated constraint; the trade-off is named honestly rather than hidden |
| Always-on forensic attribution (running on every session, not only flagged ones) | Would give richer forensic detail uniformly | Adds cost on the common path for no benefit, since attribution is only meaningful once a flag already exists | Attribution runs only on already-flagged sessions | Zero cost on the common path; attribution remains strictly forensic, never load-bearing |
| A unified single-stage Bayesian/probabilistic framework replacing the staged FSM→Stage1→Stage2→CUSUM pipeline | Plausible as a "cleaner" single-model alternative | No evidence or strong reasoning exists in either direction; the governing synthesis rule requires evidence or strong reasoning of meaningful advantage, and plausibility alone is explicitly insufficient | Retain the staged pipeline; name the unified alternative as an explicit, honestly stated limitation (not benchmarked, not assumed inferior) | Avoids unjustified architectural churn; the open question is documented rather than silently dropped |

### Resulting Architecture

The final design principles that survive this review: keep deterministic and probabilistic evidence structurally separate; require every statistical component to demonstrate independent, ablation-measured marginal value; never substitute independent thresholding for a jointly calibrated decision where one is available; make calibration/evaluation separation and protocol non-interference *structural* guarantees, not conventions; and introduce no technology, however sophisticated it sounds, without a demonstrated necessity this system actually has.

---

## 5. Implementation Methodology

### 5.1 Technology Stack

| Technology | Role | Why It Was Selected | What Risk It Avoids |
|---|---|---|---|
| Python 3.11+ | Core language across quantum simulation, statistics, backend, glue code | One language, no build/FFI boundary; NumPy/SciQ are fast enough at this qubit count (statevectors of size 8) and trial count | A compiled hybrid (Python+C++/Rust) adds a build step for no measured performance need |
| Custom NumPy statevector simulator, fixed 3-qubit register | Quantum simulation | Protocol never exceeds 3 qubits; fully transparent, inspectable, fast per Monte Carlo trial | General circuit-SDK overhead and opacity for capability this system doesn't need |
| NumPy + SciPy, hand-written likelihood functions | Statistical engine | Every formula (profile likelihood, SPRT, GLR-CUSUM, goodness-of-fit) is closed-form or simulation-calibrated | A Bayesian inference engine (PyMC/Stan) the problem doesn't require |
| `multiprocessing.Pool`, explicit `spawn` context, `Generator(PCG64)` per-trial seeding | Monte Carlo execution | Embarrassingly parallel, single-machine workload; `spawn` guarantees identical results across every team member's OS | Distributed-queue infrastructure (Celery/Redis/Dask) with no matching need |
| FastAPI + Pydantic v2 | Backend/API | Automatic OpenAPI docs; Pydantic models double as documentation and validation; async support used exactly once, for the SSE stream | Flask's bolted-on async/docs/validation; Django's unused multi-app/ORM/admin machinery |
| React + Vite + Recharts | Frontend | Small number of highly interactive, real-time views; fast dev iteration | Next.js's unused SSR/edge machinery; Streamlit's inability to express the protocol/advisory visual separation as a first-class component |
| SQLite, documented Postgres migration path | Relational storage | No concurrent-write contention at this stage; schema uses standard SQL types | Standing up Postgres for a need not yet demonstrated |
| Alembic, `render_as_batch=True` | Migrations | SQLite's limited `ALTER TABLE` support requires batch mode for any non-additive change | An opaque SQLite error at an inconvenient later point |
| Version-keyed JSON + NumPy `.npz`, embedded SHA-256 content hash | Calibration/experiment artifacts | Large, write-once, read-heavy data; content hash (not filename) guarantees integrity | A second database technology (MongoDB) for a filesystem-adequate need |
| `hashlib.sha256`, append-only JSONL | Forensic log chain | Stdlib-only, naturally append-only | Application-level enforcement a flat file avoids by construction |
| Ed25519 via `cryptography` | Forensic log signature | Mature, minimal-dependency, widely audited | The one place a new cryptographic dependency is genuinely required, named explicitly |
| A single committed lockfile | Dependency locking | Reproducibility is a headline requirement | Drift between team members' installed environments |
| `import-linter` | Import boundary enforcement | A CI-enforced contract cannot rot the way a code-review convention can | Silent erosion of the non-interference guarantee |

**Explicitly not used, and not added without a genuine implementation impossibility, internal contradiction, correctness flaw, or security flaw:** Qiskit, Cirq, PennyLane, Celery, Redis, RabbitMQ, Dask, PyMC, Stan, scikit-learn, TensorFlow, PyTorch, PostgreSQL (at this stage), MongoDB, any authentication/multi-tenancy system, generic admin/CRUD screens, a general N-qubit circuit engine.

### 5.2 Quantum Simulation

State representation is a fixed 3-qubit statevector register (message qubit + the two halves of a Bell pair) — the protocol never requires more. The simulation boundary is explicit Bell-pair preparation, Bell-measurement, Pauli-correction, and projective-measurement functions, each independently inspectable. This is sufficient because the protocol's qubit count is fixed and small, not because complexity was avoided for its own sake — a general circuit engine would run correctly but slower per trial across a Monte Carlo harness of thousands of sessions, and would obscure the exact quantum operations a hostile-audit-style demo needs to show in plain math.

### 5.3 Statistical Engine

Likelihood calculations use `scipy.optimize.minimize_scalar` (bounded, `0 ≤ p ≤ 0.5`) to profile out the nuisance parameter `p̂` in Stage 1; critical values for Stage 1 are obtained by simulation at the actual operating sample size rather than an asymptotic chi-square lookup; Stage 2's rejection region is obtained by Monte Carlo simulation of the joint null distribution under a mandatory train/evaluate split; GLR-CUSUM's `sup_θ` is closed-form (sample-proportion MLE) for this exponential-family noise model. No method beyond what is defined in the frozen architecture is introduced.

### 5.4 Backend

FastAPI exposes session execution, experiment execution, forensic querying, CUSUM history, and one Server-Sent-Events endpoint (`GET /sessions/{id}/stream`) for one-directional live-session streaming — the sole place FastAPI's async capability is actually exercised. The `CalibrationArtifact` is loaded and content-hash-verified exactly once, at process startup, kept in memory as a verified singleton, and never re-read or re-hashed per request. Deliberately small and demo-oriented: no auth/multi-tenant endpoints, no generic CRUD beyond what a live demo and an experiment operator need.

### 5.5 Frontend

Five React screens tell the demo narrative in order: **Live Session** (animates the protocol path end-to-end via SSE, ending in a labeled PROTOCOL: ACCEPTED panel), **Attack Lab** (side-by-side, deliberately differently-styled Protocol Decision vs. QSENTINEL Advisory cards, so the authoritative/advisory distinction is visually unmistakable), **Quantum Evidence View** (`m, C, H`, Pauli-correction consistency, Stage 1's statistic, with plain-language annotation that `C`/`H` are consistency checks, not independent features), **CUSUM Drift Chart** (a live low-and-slow attack, plotting `G_t` climbing toward threshold over session count), and **Forensic Log** (hash-chained record with a one-click chain-integrity verification action).

### 5.6 Storage and Evidence

SQLite holds runtime session data (`Session`, `SessionTranscript`, `ProtocolDecision`, `MonitoringDecision`, `QuantumEvidence`, `CusumState` — the sole CUSUM store), never referencing `MonitoringDecision` from `ProtocolDecision` in the schema itself. Calibration and experiment artifacts (versioned JSON + `.npz`, each embedding a SHA-256 content hash verified on load) live on the filesystem, not in SQLite. The forensic chain (`forensic_store/chain_{date}.jsonl`) is the sole authoritative evidentiary record; SQLite `ForensicRecord` rows are a queryable index over it, never a replacement.

---

## 6. Validation Strategy

### 6.1 Validation Already Completed

None, under the current frozen architecture. Every quantitative figure previously produced (detection rates, the 100%→1.5% replay-detection ablation figure, etc.) was measured under a **superseded architecture** and explicitly requires re-confirmation before being quoted as a current result (architecture freeze Finding F7). No headline number is currently stated as validated under this freeze.

### 6.2 Designed Validation

In priority order, matching the architecture freeze's own severity ranking:

1. **Stage 2 joint Monte Carlo calibration**, executed with a mandatory train/evaluate split — the single blocking prerequisite for every other numeric claim.
2. **Full harness re-run**, seven conditions (the six historical conditions plus the newly specified unauthorized-verification condition), under the corrected joint-decision rule and GLR-CUSUM formulation.
3. **Profile-likelihood critical-value validation** at the actual operating sample size (n≈200), by simulation, not asymptotics alone.
4. **Verification-accuracy sweep**: legitimate-session acceptance rate across a range of honest noise levels (`p ∈ {0.01, …, 0.10}`), reported as its own metric, distinct from attack-condition false-positive results.
5. **Naive multi-detector-integration baseline**, re-implemented and run head-to-head, so the joint-calibration claim has an actual measured comparator.
6. **Alpha-spending benchmark** against the joint-calibration approach, as a secondary, non-blocking comparison.
7. **Every ablation** continues to include full-architecture-minus-one-component tests, with unfavorable results reported without exception, and every resulting figure reported with a confidence interval at the stated trial count.

### 6.3 What Results Must Be Demonstrated Before Claims Are Made

No detection rate, false-positive rate, false-negative rate, latency, or throughput figure may be quoted as a current result of this architecture until: (a) the Stage 2 calibration has been run under the mandatory held-out split, (b) the full seven-condition harness has been re-run under the corrected joint-decision rule, and (c) every reported figure carries its architecture-version and confidence-interval caveat at the point of use, not only once in a source document. Until then, the correct labels are **Architecture defined**, **Implementation planned/mapped**, and **Validation pending** — never a specific number.

### 6.4 Threat Scenarios

The Monte Carlo harness is specified for exactly seven conditions, matching the attack-simulation framework: clean forgery, sub-threshold forgery, replay, impersonation (in-scope: missing/invalid/expired token), unauthorized verification, channel manipulation (intercept-resend, Pauli-structured, structural burst), and low-and-slow drift. No scenario beyond these seven is claimed as covered.

---

## 7. Feasibility and Viability

The frozen architecture is implementable by a small team within a hackathon timeline **if** two specific complexity-control decisions hold: the quantum simulation stays minimal (fixed 3-qubit statevector, no general circuit engine) and the statistical layer stays in NumPy/SciPy rather than a probabilistic-programming framework. Both are justified on necessity grounds (Section 5.1/5.2), not familiarity.

The single highest-risk element in the system is explicitly **not** the quantum simulation — it is Stage 2's calibration/evaluation separation and the protocol/monitor non-interference guarantee. Both are discipline problems, not technology problems, and both are made structural rather than aspirational: calibration/evaluation separation via a runtime-enforcing `SeedAllocator` and a content-hashed `CalibrationArtifact`; non-interference via immutable protocol objects (`SessionTranscript`, `ProtocolDecision` are frozen types) plus a CI-failing `import-linter` contract forbidding `qds/` from importing anything from `qsentinel_monitor/`, `api/`, `experiments/`, `attacks/`, or `frontend/`.

### Risk Matrix

| Risk | Why It Matters | Mitigation | Architectural Defense |
|---|---|---|---|
| Calibration/evaluation leakage | Would invalidate every future detection-rate figure — the single largest credibility threat identified in the audit | Mandatory train/evaluate split; runtime-enforced disjoint seed ranges | `SeedAllocator` raises on any out-of-range seed request; every `ExperimentRun` logs its `calibration_artifact_hash`; the artifact loader raises on a content-hash mismatch |
| Model mismatch (Stage 1 `MODEL_INVALID`) | A mismatched model must not be silently absorbed as ordinary evidence, and must not be conflated with a confirmed attack | Distinct, logged `MODEL_INVALID` routing; separate handling for optimizer non-convergence vs. legitimate boundary optima | Explicit three-way outcome logic in Stage 1's implementation (Section 2.3 above) |
| Low-and-slow attacks evading single-session detection | Defined by being sub-threshold in any one session — the exact case a session-local detector is blind to | Unconditional cross-session ingestion, every session, regardless of Stage 2 outcome | Window-limited GLR-CUSUM with closed-form `sup_θ`, `O(w)` per session |
| Simulation fidelity vs. physical hardware | A 3-qubit NumPy statevector simulator is not a physical quantum channel | Named explicitly as a limitation, not smoothed over | Simulator outputs are cross-checked against statevector ground truth; no claim of hardware-equivalent fidelity is made |
| Prototype complexity exceeding hackathon timeline | Ambitious multi-stage statistical pipeline within a fixed build window | Phased roadmap (Section 23 of the Implementation Blueprint) places the demo-critical path (Phases 0–11) before comparator/validation work (Phase 12), which is explicitly the lowest-risk work to cut if time is short | Phases 0–9 alone already satisfy a runnable, testable, non-visual detection/monitoring pipeline |
| Adaptive attacker aware of the calibrated rejection region or CUSUM window | A statistically informed adversary has a named evasion path | Named as an explicit, untested limitation rather than an implied guarantee | No claim of robustness to this attacker class is made anywhere in the architecture |
| Reproducibility drift across team members' machines/OSes | Monte Carlo results must be identical across every machine, not just within one OS | Explicit `spawn` multiprocessing context (not the platform default `fork`/`spawn` split) plus a single committed dependency lockfile | CI includes a Monte Carlo reproducibility check confirming identical results across the pinned context regardless of runner OS |

---

## 8. Limitations — Stated Honestly

| Limitation | Why It Exists | Impact | Why the System Is Still Valid | Future Direction |
|---|---|---|---|---|
| A statistically clean, single-session forgery reproducing the legitimate noise distribution exactly is undetectable | Information-theoretic limit — no statistical monitor can distinguish two identical distributions | No component of this system can catch this attack class, in principle | The protocol's own cryptographic threshold rule is the correct and sufficient defense against this exact case; QSENTINEL was never designed to add cryptographic security | Not resolvable by monitoring design; out of scope by definition |
| An adaptive attacker aware of the calibrated Stage 2 rejection region, or of the GLR-CUSUM window size, has a named, untested evasion path | The evaluated threat model assumes a static, non-adaptive attacker | Detection guarantees do not extend to this adversary class | Named explicitly rather than implied away; scoped claim only | Adaptive-adversary robustness is future work, not claimed here |
| Calibration assumptions matter — the declared depolarizing noise family must contain the true legitimate-operation distribution | Stage 1/2 and GLR-CUSUM are all calibrated against this declared family | If the true honest-channel distribution falls outside the declared family, calibration validity is undermined | This assumption is stated as a frozen assumption, not hidden; Stage 1 exists precisely to test consistency with this family per session | Would require re-deriving the noise family from measured hardware data, not assumed here |
| Model-mismatch handling requires care | Stage 1 must not conflate a mismatched model with a confirmed attack | Naive handling would either inflate false positives or blind the system | Three-way explicit outcome logic (fits / MODEL_INVALID / optimizer failure) is structural, not implicit | N/A — already addressed structurally |
| Prototype simulation differs from physical quantum hardware | A fixed-3-qubit NumPy statevector simulator, not physical hardware | Results characterize the simulated protocol/monitor behavior, not a deployed physical system | Sufficient for demonstrating the statistical/architectural design under PS-141's simulation-based scope; cross-checked against statevector ground truth | Physical-hardware validation is out of scope for this deliverable |
| Hardware side channels are outside scope | The architecture observes only classical measurement telemetry the protocol already discloses | Side-channel attacks on physical hardware are not addressed | Explicitly named as never claimed (Section 7 of the architecture freeze) | Deferred; would require a different evidence source entirely |
| Impersonation/unauthorized-verification coverage depends on classical-channel authentication integrity | FSM checks token validity, not the integrity of the system issuing tokens | A compromise of the classical-channel authentication layer is out of scope by design | Named explicitly, not hidden; this dependency is stated at every point the compliance language is used | Out of scope for this monitoring layer by design |
| Source-stage (pre-teleportation) entanglement-integrity monitoring is not covered | Mature measurement-device-independent witness techniques currently exceed anything specified here | This capability is deferred, not shipped underspecified | Deliberately deferred rather than shipped weak or incomplete | Named as future work |
| The teleportation-for-direct-transmission substitution to QS-L is explicitly unproven | A protocol-design assumption inherited, not established here | QSENTINEL's guarantees are conditioned on the substitution's soundness, not a proof of it | Stated plainly as a frozen assumption | Proof of the substitution's soundness is out of scope for this system |
| The staged-pipeline architecture has not been benchmarked against a unified probabilistic alternative | No evidence currently exists in either direction, and the governing synthesis rule requires evidence, not plausibility, before adopting an alternative | An open question, honestly unresolved | Named explicitly as an open question rather than silently dropped or resolved by assumption | Would require a genuine comparative study, not currently planned |
| Every quantitative figure currently in the historical record was measured under a superseded architecture | The architecture has been revised (this freeze) since those figures were produced | Historical figures cannot be quoted as current results | Every figure now carries a re-confirmation requirement before being quoted (Section 6 above) | Full harness re-run is the first item in the priority-ordered evaluation plan |
| Full deployment isolation (reduced-privilege collector agents, independently authenticated telemetry channel) is out of scope | A software-deliverable scope decision | Deployment hardening beyond the software itself is not addressed | Named explicitly rather than silently assumed as already solved | Deployment hardening is a distinct future engineering effort |

---

## 9. Final Architecture Verdict

QSENTINEL's architecture definitively solves the problem of maintaining a structurally separate, non-interfering, statistically disciplined runtime observability layer over a teleportation-distributed QDS protocol — one that can detect protocol-level violations deterministically (FSM), detect single-session channel manipulation and sub-threshold forgery probabilistically (Stage 1/2, jointly calibrated), and detect persistent, low-and-slow cross-session drift that no single-session mechanism can see by construction (GLR-CUSUM) — while providing a structural guarantee, not merely a stated intention, that none of this ever overrides the protocol's own deterministic acceptance of a legitimate signature.

It does not solve, and does not claim to solve: detection of a statistically indistinguishable single-session forgery (an information-theoretic limit); robustness against an adaptive attacker aware of the calibrated decision boundaries; any increase to the protocol's own cryptographic security level; or proof that the teleportation-for-direct-transmission substitution to QS-L is sound.

The architecture is technically coherent because every retained component survives a documented minimum-necessary-change test against a genuine hostile audit (11 findings, none requiring a new module, a removed module, or a reordering of the pipeline's shape), and the implementation is realistic because every technology choice is justified against a stated alternative that was considered and rejected on necessity grounds, not preference. What remains is exclusively empirical: the Stage 2 joint calibration under a held-out split, the full seven-condition harness re-run, the small-sample critical-value validation, the verification-accuracy sweep, and the two comparator baselines — named, prioritized, and not yet executed.

**ARCHITECTURE STATUS: DEFENSIBLE, WITH CLEARLY STATED VALIDATION BOUNDARIES.**

---
---

# PHASE 2 — FINAL SIH SLIDE-BY-SLIDE PRESENTATION PACKAGE

---

## 10. Presentation Information Map

| Information | Primary Page | Secondary Page | Keep for Q&A? | Why It Belongs There |
|---|---|---|---|---|
| Problem context / security gap | Page 1 | — | Backup detail | Sets up "what is QSENTINEL" before anything else can make sense |
| QSENTINEL definition | Page 1 | — | — | The page's core answer |
| Core innovation (2-plane separation) | Page 1 | Page 2 (as diagram detail) | — | Must land first as the single idea; Page 2 shows it mechanically |
| Authoritative protocol plane | Page 1 | Page 2 | — | Introduced conceptually on P1, shown operating in the pipeline on P2 |
| Advisory monitoring plane | Page 1 | Page 2 | — | Same pattern as above |
| Quantum measurement evidence (2-dimension reframing) | Page 2 | Page 1 (one line) | Detailed math for Q&A | Technical substance belongs on the technical slide |
| Stage 1 | Page 2 | — | Full profile-likelihood formula for Q&A | Core "how it works" content |
| Stage 2 | Page 2 | — | Full joint-calibration procedure for Q&A | Core "how it works" content |
| GLR-CUSUM | Page 2 | Page 4 (impact framing) | Full complexity derivation for Q&A | Mechanism on P2; why-it-matters framing on P4 |
| Forensic layer | Page 2 | Page 4 (accountability framing) | Hash-chain internals for Q&A | Mechanism on P2; impact framing on P4 |
| Technology stack | Page 2 (compact strip) | Page 3 (rationale) | Full stack-rationale table for Q&A | Named on P2 as evidence of "real system," justified on P3 as evidence of "buildable system" |
| Quantum simulation approach | Page 3 | Page 2 (one label) | — | Feasibility argument, not mechanism |
| Statistical engine choice | Page 3 | — | — | Feasibility argument |
| Backend/frontend | — | Page 3 (one line) | Full architecture for Q&A | Not core to any of the 5 slides' narrative |
| Storage/persistence | — | — | Full schema for Q&A | Pure implementation detail |
| Calibration methodology | Page 3 | — | Full Monte Carlo procedure for Q&A | Directly answers "can we trust the numbers" |
| Model mismatch handling | Page 3 | — | Full three-way routing logic for Q&A | Risk→mitigation content |
| Low-and-slow attacks | Page 3 | Page 4 (why it matters) | — | Risk on P3, value framing on P4 |
| Technical risks | Page 3 | — | Full risk matrix for Q&A | Page 3's dominant visual |
| Mitigations | Page 3 | — | — | Paired with each risk |
| Feasibility (timeline, buildability) | Page 3 | — | Phased roadmap for Q&A | Core "can it be built" content |
| Impact | Page 4 | — | — | Core "why it matters" content |
| Target ecosystem | Page 4 | — | — | Core "why it matters" content |
| Security boundaries / what QSENTINEL does not claim | Page 4 | Page 1 (one line: "advisory, never overriding") | Full limitations table for Q&A | Credibility content — explicit non-claims |
| Limitations (full list) | — | — | All 11 limitations, full table for Q&A | Too much detail for any single slide; judges who probe get the honest answer |
| Research foundations | Page 5 | — | — | Core "what supports this" content |
| Validation methodology | Page 5 | Page 3 (one line) | Full 6-item evaluation plan for Q&A | Named on P5 as "how we'll prove it," detailed for Q&A |
| References | Page 5 | — | — | Core "what supports this" content |

**Appears only once:** core innovation statement, technology stack rationale, risk matrix, impact model, references list.
**May be repeated as a small reference:** "advisory, never overriding" (P1 and P4); GLR-CUSUM and forensic mechanism names (P2 and P4, different framing each time).
**Reserved for Q&A only:** full mathematical formulations, complete calibration procedure, API routes, database schema, file structure, full limitations table, full validation task list.

---

## 11. Final 5-Slide Presentation Architecture

| Page | Title | Core Question |
|---|---|---|
| 1 | Proposed Solution | What have we built? |
| 2 | Technical Approach | How does it work? |
| 3 | Feasibility and Viability | Can it realistically be built and trusted? |
| 4 | Impact and Benefits | Why does it matter? |
| 5 | References and Research Work | What research and evidence support the work? |

This sequence is optimal because it mirrors the order a technically literate skeptic actually forms judgment in: first grasp *what exists* (P1), then verify *the mechanism is real and specific*, not hand-waved (P2), then stress-test *whether it can actually be delivered and trusted* (P3) — the point at which most weak hackathon pitches collapse under judge questioning — and only after credibility is earned, argue *why it's worth caring about* (P4), closing with the *evidence trail* that lets a judge independently verify the team did real homework (P5). Reversing P3 and P4 (leading with impact before feasibility) is a common weak pattern this structure deliberately avoids: impact claims land only after the judge already believes the system can be built and calibrated honestly.

---

# PAGE 1 — PROPOSED SOLUTION

## 1. Page Objective
By the end of this page, the judge must understand that QSENTINEL is a runtime, non-AI/ML statistical monitoring layer riding alongside a teleportation-QDS protocol, and that it is architecturally separate from — and never overrides — the protocol's own deterministic accept/reject decision.

## 2. Core Message
A protocol can determine whether an execution is valid; QSENTINEL independently observes whether the execution remains statistically and temporally consistent with legitimate behavior.

## 3. Information Allocation

**PRIMARY INFORMATION:** QSENTINEL definition; the two-plane separation (protocol vs. monitor); the security gap it addresses (validity ≠ statistical/temporal consistency).

**SECONDARY INFORMATION:** Why this gap exists (a protocol answers a narrow per-transcript question); the "advisory, never overriding" guarantee stated in one line.

**SUPPORTING INFORMATION:** Innovation highlights (non-interfering architecture, independent observability plane, cross-session monitoring, teleportation-native evidence).

## 4. Exact Content That Appears on the Page

**Title:** QSENTINEL

**Subtitle:** Quantum-Inspired Runtime Threat Detection for Digital Signature Security

**Key statement (large, top of page):**
"A protocol can tell you an execution is valid. QSENTINEL tells you whether it still looks legitimate."

**Two labeled panels:**
- Panel A label: "PROTOCOL — Is this execution valid?" — sub-line: "Deterministic. Authoritative. Final."
- Panel B label: "QSENTINEL — Does it still look legitimate?" — sub-line: "Statistical. Advisory. Never overrides Panel A."

**Innovation callouts (3–4 short bullets, not paragraphs):**
- "Independent observability plane — runs alongside the protocol, never inside its decision"
- "Multi-stage statistical reasoning — not a single threshold, not a black box"
- "Cross-session monitoring — catches drift no single session can reveal"
- "A teleportation-native quantum observable — Pauli-correction consistency — with no analog in directly-transmitted-qubit schemes"

## 5. What This Page Must Not Contain
- The full technology stack
- Implementation-level detail (repository structure, API, database)
- The full statistical formulation (profile likelihood, SPRT formulas)
- The risk matrix
- Impact/target-ecosystem content

## 6. Required Primary Visual
**Type:** A two-lane split diagram. **Purpose:** Make the two-plane separation visually unmistakable before any words explain it. **Components:** Left lane labeled "PROTOCOL EXECUTION PATH" with a solid border, ending in a solid checkmark box "ACCEPT/REJECT (authoritative)"; right lane labeled "QSENTINEL MONITORING PATH" with a dashed border, ending in a dashed annotation box "ADVISORY FLAG (never blocks)". **Information flow:** Both lanes originate from a single shared "Session Transcript" box at the top, then diverge — visually reinforcing that they observe the same execution but decide independently. **Critical visual distinction:** solid vs. dashed border styling must be used consistently for "authoritative" vs. "advisory" everywhere the two appear.

## 7. Visual Composition
TOP: Title, subtitle, and the key statement.
CENTER: The two-lane split diagram, occupying the visual majority of the page.
BOTTOM: The 3–4 innovation callout bullets, in a single compact row/strip.

## 8. Visual Hierarchy
**3 seconds:** "QSENTINEL" + the two visually distinct lanes (solid vs. dashed) — the judge registers "there are two separate decision paths here."
**15 seconds:** The key statement plus panel labels — the judge understands the protocol/monitor distinction and that one is authoritative, one is advisory.
**60 seconds:** The innovation callouts — the judge grasps what's specifically new about this design.

## 9. What the Presenter Says
"Digital signature protocols can tell you whether a signature is mathematically valid — but they can't tell you whether the channel producing it still behaves the way it should. QSENTINEL is a separate, advisory layer that watches the same execution and asks a different question: does this still look statistically and temporally like legitimate behavior? It never overrides the protocol's own decision — it adds visibility the protocol was never designed to provide. That's the whole idea, and everything else in this presentation is how we built it."

## 10. Expected Judge Questions
- "If it's advisory, what's the point — why would anyone act on a flag they can ignore?"
- "Isn't this just an anomaly detector with extra branding?"
- "Why not just make the protocol itself reject on a flag?"

## 11. Suggested Answers
- A SOC operator or auditor acts on the flag and its forensic evidence — the value is investigability and accountability, not automated blocking, which the architecture deliberately avoids because a false positive must never deny a legitimate signature.
- No — it's a structurally separated, jointly calibrated multi-stage pipeline (Stage 1 model-fit, Stage 2 joint decision, GLR-CUSUM cross-session) with an explicit non-AI/ML constraint, not a single anomaly score.
- Because a probabilistic flag has a nonzero false-positive rate by construction; letting it gate a deterministic protocol decision would mean a legitimate signature could be denied on a statistical fluctuation — an unacceptable trade the architecture explicitly refuses to make.

## 12. Possible Weaknesses
**Criticism:** "Advisory-only" can sound like the system doesn't actually stop anything. **Honest response:** That's a deliberate, stated design trade-off, not an oversight — the system's job is observability and forensic accountability for a layer the protocol cannot see, not enforcement, and the presentation names this boundary explicitly rather than implying more than is built.

---

# PAGE 2 — TECHNICAL APPROACH

## 1. Page Objective
By the end of this page, the judge must understand the concrete end-to-end pipeline — FSM, quantum evidence extraction, Stage 1, Stage 2, GLR-CUSUM, forensic logging — and see that the protocol/monitor separation from Page 1 is a real mechanism, not a slogan.

## 2. Core Message
Every session is checked deterministically by an FSM and, independently, statistically by a two-stage calibrated pipeline plus unconditional cross-session monitoring — and every session, flagged or not, is written to a signed, tamper-evident forensic log.

## 3. Information Allocation

**PRIMARY INFORMATION:** The end-to-end architecture diagram; Stage 1 and Stage 2's roles; GLR-CUSUM's unconditional cross-session role.

**SECONDARY INFORMATION:** The quantum evidence vector (2 independent dimensions); the forensic log's signed hash-chain property.

**SUPPORTING INFORMATION:** The technology stack, shown as a compact strip, not explained in depth.

## 4. Exact Content That Appears on the Page

**Title:** Technical Approach

**Subtitle:** How QSENTINEL Works

**Key statement:** "One deterministic gate. Two calibrated statistical stages. One cross-session monitor that never sleeps."

**Diagram labels (exact):**
- "FSM — freshness, authorization, sequencing (deterministic)"
- "Quantum Evidence Collector — refined p̂ (m, C, H mutual consistency) + Pauli-correction consistency"
- "STAGE 1 — Model-Fit / Mutual Consistency"
- "STAGE 2 — Joint Statistical Decision (α = 0.01)"
- "GLR-CUSUM — every session, unconditionally"
- "Signed, Hash-Chained Forensic Log"

**Technology strip (compact, bottom row):** "Python · NumPy/SciQ · FastAPI · React · SQLite · Ed25519" — no elaboration on this slide.

## 5. What This Page Must Not Contain
- Impact/target-ecosystem discussion
- The full risk matrix
- References
- Lengthy limitations discussion (one line at most, if any)
- Technology-choice rationale (belongs on Page 3)

## 6. Required Primary Visual
**Type:** End-to-end system architecture diagram (the diagram in Section 3 of Phase 1, simplified for slide density). **Purpose:** Show the real pipeline, not a described one. **Components:** Session Transcript at top; FSM branch (solid) vs. Quantum Evidence branch (dashed) diverging exactly as in Page 1's diagram, but now with Stage 1 → Stage 2 → GLR-CUSUM populated as explicit boxes on the dashed side, converging into the Forensic Log. **Information flow:** Top-to-bottom, matching the actual data flow. **Critical visual distinction:** GLR-CUSUM must be drawn receiving input from *every* session (an arrow from the Stage-1/2 output stream directly into GLR-CUSUM, unconditional), not only from a "flagged" branch — this is a common misrepresentation the architecture explicitly forbids.

## 7. Visual Composition
TOP: Title, subtitle, key statement.
LEFT/CENTER: The dominant end-to-end diagram (majority of the slide).
RIGHT or BOTTOM-INSET: A small "2 independent dimensions" callout for the quantum evidence box.
BOTTOM: Compact technology strip.

## 8. Visual Hierarchy
**3 seconds:** The diagram's overall shape — one gate, two stages, one continuous monitor, one log.
**15 seconds:** Stage 1 vs. Stage 2's distinct roles and that GLR-CUSUM runs on every session.
**60 seconds:** The quantum-evidence reframing (2 dimensions, not 4 features) and the technology strip.

## 9. What the Presenter Says
"Every session starts with a deterministic FSM check — freshness, authorization, sequencing — that's the protocol's own gate. In parallel, we extract quantum measurement evidence — really just two independent dimensions, not four unrelated numbers, because three of our raw statistics are mathematically tied together under honest operation, and testing that relationship is itself the signal. Stage 1 checks whether the session's statistics are even consistent with a legitimate operating point. If they are, Stage 2 makes a single, jointly calibrated statistical decision — never separate thresholds. And regardless of what any single session concludes, every session feeds a cross-session monitor, GLR-CUSUM, because a low-and-slow attacker is specifically trying to stay under any one session's radar. Everything gets written to a signed, tamper-evident log."

## 10. Expected Judge Questions
- "Why not just use one combined score instead of two stages?"
- "Isn't testing m, C, and H together just curve-fitting after the fact?"
- "Why does GLR-CUSUM need to see every session — isn't that wasteful?"

## 11. Suggested Answers
- Because Stage 1 answers a structurally different question (does the honest model even apply here) from Stage 2 (given that it does, is this an attack) — collapsing them would let a model-mismatch case be silently misread as either "clean" or "attack," neither of which is correct.
- No — under the honest-execution model these three quantities are provably functions of one physical parameter; an attack that violates the physical symmetry breaks that relationship even without moving any single statistic far enough to look anomalous alone. That's a structural property of the physics, not a fitted correlation.
- It's not wasteful — the update per session is closed-form and O(w), because for this noise family the CUSUM's per-window optimization has a closed-form solution (the sample-proportion MLE), so unconditional ingestion costs almost nothing while being the only way to see a low-and-slow attack at all.

## 12. Possible Weaknesses
**Criticism:** The pipeline has many stages — could look overengineered for a hackathon prototype. **Honest response:** Each stage was tested against a "does removing it hurt something no other stage can catch" standard during architecture review — for example, removing the FSM alone collapses replay detection from a clean result to near-zero — so the count of stages reflects genuinely non-overlapping responsibilities, not accumulation for its own sake.

---

# PAGE 3 — FEASIBILITY AND VIABILITY

## 1. Page Objective
By the end of this page, the judge must believe the system is realistically buildable within scope and that its numeric claims will be trustworthy because of how calibration is structured — not just asserted.

## 2. Core Message
Every component is deliberately the minimum sufficient technology for the problem, and the two highest-risk elements — calibration leakage and protocol non-interference — are enforced structurally, not by convention.

## 3. Information Allocation

**PRIMARY INFORMATION:** The risk → mitigation matrix; the "minimal-by-design" technology rationale; the calibration/train-evaluate-separation guarantee.

**SECONDARY INFORMATION:** How model mismatch and low-and-slow attacks are handled; why the quantum simulator is deliberately minimal.

**SUPPORTING INFORMATION:** The buildable-system stack diagram.

## 4. Exact Content That Appears on the Page

**Title:** Feasibility and Viability

**Subtitle:** Can This Actually Be Built — and Trusted?

**Key statement:** "Every technology here is the minimum sufficient choice — and the two riskiest guarantees are enforced in code, not by convention."

**Risk matrix column headers (exact):** "Risk | Why It Matters | Mitigation"

**Risk rows (exact, condensed for slide space):**
- "Calibration leakage → invalidates every detection-rate figure → mandatory train/evaluate split, runtime-enforced by seed range"
- "Protocol/monitor interference → would silently break the core guarantee → immutable protocol objects + CI-enforced import boundary"
- "Model mismatch mistaken for attack → false positives or blindness → distinct MODEL_INVALID routing, never conflated with FLAG(REJECT)"
- "Low-and-slow evasion → sub-threshold per session by design → unconditional, every-session GLR-CUSUM ingestion"

**Buildable-stack callout (small, secondary):** "3-qubit statevector simulator (not a general circuit engine) · NumPy/SciPy statistics (not a Bayesian inference engine) · single-machine multiprocessing (not a distributed job queue)"

## 5. What This Page Must Not Contain
- Detailed impact/benefits narrative
- The full reference list
- Lengthy limitations prose (the risk matrix already carries the honest framing)
- Implementation timelines or benchmark numbers not yet measured

## 6. Required Primary Visual
**Type:** Risk → Mitigation engineering matrix (table-as-visual, styled as cards or rows, not dense text). **Purpose:** Show judges the team has already identified its own weakest points before being asked. **Components:** Four risk rows exactly as above, each visually paired risk-to-mitigation (e.g., red-tinted risk chip → arrow → green-tinted mitigation chip). **Information flow:** Left (risk) to right (mitigation) per row. **Critical visual distinction:** Each mitigation must reference a concrete mechanism (a specific class/check name), not a vague reassurance — this is what makes the slide defensible under questioning.

## 7. Visual Composition
TOP: Title, subtitle, key statement.
CENTER: The risk→mitigation matrix, dominant visual.
BOTTOM: The compact buildable-stack callout strip.

## 8. Visual Hierarchy
**3 seconds:** Four risk rows, visually paired to mitigations — "they've thought about what could go wrong."
**15 seconds:** The two structural guarantees (calibration separation, non-interference) as the headline risks, correctly identified as the *highest*-risk elements, not glossed over.
**60 seconds:** The buildable-stack callout — why the technology choices are minimal by necessity, not convenience.

## 9. What the Presenter Says
"The biggest risk in a system like this isn't the quantum part — it's discipline: making sure calibration data never leaks into evaluation, and making sure the monitoring layer can never quietly start influencing the protocol's own decision. We didn't leave either as a promise — both are enforced in code: a seed allocator that physically cannot hand out an evaluation seed to a calibration routine, and a CI check that fails the build if the protocol code ever imports anything from the monitoring code. Everything else in the stack — the quantum simulator, the statistics library, the parallel execution model — was chosen because it's exactly sufficient for a 3-qubit protocol and a single-machine Monte Carlo workload, not because it's impressive."

## 10. Expected Judge Questions
- "How do you know your risk matrix is complete — what did you miss?"
- "What happens if the quantum simulator's fidelity doesn't match real hardware?"
- "What's your actual timeline confidence for building this in the hackathon window?"

## 11. Suggested Answers
- We don't claim it's complete — it reflects an explicit hostile internal review (11 named findings) against the frozen architecture; the full limitations list, including open questions we haven't resolved, is available and we're not hiding any of it.
- We state that explicitly as a named limitation: the simulator characterizes the protocol/monitor design under a statevector model, not physical hardware fidelity — that gap is scoped out of this deliverable, not claimed as solved.
- The phased roadmap places a complete, testable, non-visual pipeline (protocol through forensic logging) before the frontend or comparator baselines, specifically so the highest-value, most-defensible part of the system is never the part we'd have to cut under time pressure.

## 12. Possible Weaknesses
**Criticism:** No headline numbers (detection rate, false-positive rate) are presented — could look unfinished. **Honest response:** That's a deliberate choice, not a gap in the work — every number from the prior architecture was measured before this freeze and is explicitly invalid to quote now; presenting fabricated or stale numbers would be less credible than presenting none, and the calibration/evaluation plan that will produce trustworthy numbers is fully specified and prioritized.

---

# PAGE 4 — IMPACT AND BENEFITS

## 1. Page Objective
By the end of this page, the judge must understand why continuous, non-interfering runtime observability over a quantum-signature channel is valuable, for whom, and exactly what QSENTINEL does not claim to provide.

## 2. Core Message
QSENTINEL turns a protocol that can only say "valid" or "invalid" into a system that can also say "still behaving as expected" — continuously, across sessions, with tamper-evident evidence — without ever risking a legitimate signature's acceptance.

## 3. Information Allocation

**PRIMARY INFORMATION:** The layered security impact model; the explicit "what QSENTINEL does not claim" boundary.

**SECONDARY INFORMATION:** Target ecosystem; forensic accountability as a distinct benefit from detection itself.

**SUPPORTING INFORMATION:** Cross-session visibility as a category of protection nothing else in the stack provides.

## 4. Exact Content That Appears on the Page

**Title:** Impact and Benefits

**Subtitle:** Why QSENTINEL Matters

**Key statement:** "Cryptographic validity and behavioral trust are different guarantees — QSENTINEL is what tells you when a channel that's still 'valid' no longer looks trustworthy."

**Layered model labels (exact, top to bottom):**
1. "PROTOCOL SECURITY — cryptographic validity, per transcript"
2. "RUNTIME THREAT OBSERVABILITY — statistical consistency, per session"
3. "CROSS-SESSION DETECTION — persistent drift, across sessions"
4. "FORENSIC ACCOUNTABILITY — signed, tamper-evident evidence, always"

**Target ecosystem (bullets, only categories justified by the source documents):**
- "Long-lived cryptographic systems requiring session-history awareness"
- "Digital infrastructure using quantum-distributed signature protocols"
- "Environments where forensic accountability matters as much as detection"

**Explicit boundary box (exact heading):** "WHAT QSENTINEL DOES NOT CLAIM"
- "Does not increase the protocol's own cryptographic security level"
- "Does not guarantee detection of a statistically clean, single-session forgery"
- "Does not claim robustness against an adaptive attacker aware of its calibrated thresholds"
- "Never overrides a legitimate signature's acceptance"

## 5. What This Page Must Not Contain
- The technical architecture diagram (belongs on Page 2)
- The risk matrix (belongs on Page 3)
- References
- Detailed limitations table (the boundary box is the compressed, presentation-appropriate version)

## 6. Required Primary Visual
**Type:** Layered security impact model (vertically stacked bands, narrowing or building upward). **Purpose:** Show that value accumulates in layers, each one covering a gap the layer below it structurally cannot. **Components:** Four horizontal bands, bottom to top: Protocol Security → Runtime Threat Observability → Cross-Session Detection → Forensic Accountability, each band a different visual weight or shade to suggest increasing temporal/evidentiary scope. **Information flow:** Bottom-up (foundation to accumulated value). **Critical visual distinction:** The explicit boundary box must be visually separated (bordered, different background) from the impact model, so it reads as an honest disclosure, not a hedge buried in fine print.

## 7. Visual Composition
TOP: Title, subtitle, key statement.
LEFT: The layered impact model (dominant visual).
RIGHT: Target ecosystem bullets.
BOTTOM: The explicit boundary box, full width, visually distinct.

## 8. Visual Hierarchy
**3 seconds:** Four stacked layers — "there's a hierarchy of value here."
**15 seconds:** What each layer adds that the one below cannot provide on its own.
**60 seconds:** The explicit non-claims box — judges who read closely see the team isn't overselling.

## 9. What the Presenter Says
"Protocol-level validity is necessary but not sufficient — it tells you a transcript is well-formed, not that the channel producing it is behaving normally over time. QSENTINEL adds three more layers: per-session statistical consistency, cross-session drift detection for attacks that hide from any single session, and a signed evidentiary trail regardless of outcome. That accountability layer matters as much as detection itself in any setting where you need to explain, after the fact, exactly what a channel did. And we want to be direct about the boundary: this doesn't make the cryptography stronger, and it won't catch a forgery that's statistically perfect — those are honest limits, not gaps we're hiding."

## 10. Expected Judge Questions
- "Who would actually deploy this — is there a real customer?"
- "If it can't catch a perfect forgery, what's the actual security value?"
- "How is 'forensic accountability' different from just logging everything?"

## 11. Suggested Answers
- We're naming plausible ecosystem categories justified by the system's own scope (long-lived quantum-signature deployments needing session-history awareness), not asserting a confirmed customer relationship we don't have.
- The value is in the overwhelming majority of realistic attacks, which are not statistically perfect — channel manipulation, replay, impersonation, and low-and-slow drift all leave a detectable statistical or deterministic footprint; the one undetectable case is a named, honest, information-theoretic exception, not evidence the rest of the system lacks value.
- Because it's signed and hash-chained specifically so it can't be silently altered after the fact — a plain log can be edited by anyone with file access; this can't be, without invalidating the chain in a way verification will catch.

## 12. Possible Weaknesses
**Criticism:** The explicit non-claims box could read as undercutting the pitch's own impact. **Honest response:** It's the opposite in front of a technical judging panel — a system that states its own boundary precisely is more credible than one that implies unlimited capability, and judges scoring on technical rigor typically reward this rather than penalize it.

---

# PAGE 5 — REFERENCES AND RESEARCH WORK

## 1. Page Objective
By the end of this page, the judge must see that the design went through genuine research grounding and hostile review, not a single unreviewed draft.

## 2. Core Message
QSENTINEL's design is the product of protocol research, statistical-methods research, and a documented hostile architecture review — not a first draft.

## 3. Information Allocation

**PRIMARY INFORMATION:** The three-category research organization; the design-journey flow.

**SECONDARY INFORMATION:** — (this page is intentionally light; it is a credibility page, not a content-dense one)

**SUPPORTING INFORMATION:** Any specific citations/links present in the supplied source documents.

## 4. Exact Content That Appears on the Page

**Title:** References and Research Work

**Subtitle:** What Supports This Design

**Key statement:** "Every architectural decision here traces to either a proven protocol result, an established statistical method, or a documented review finding — not intuition."

**Three category headers (exact):**
- "PROTOCOL FOUNDATIONS" — the published QS-L unforgeability/non-repudiation proof; the teleportation-distributed variant this design builds on
- "STATISTICAL FOUNDATIONS" — profile-likelihood goodness-of-fit testing; sequential likelihood-ratio (SPRT) methods; GLR-CUSUM change-point detection; Monte Carlo calibration methodology
- "ENGINEERING AND VALIDATION FOUNDATIONS" — the documented architecture-freeze hostile audit (11 findings resolved); the phased, test-gated implementation plan

**Design journey (exact flow):**
"PROBLEM → RESEARCH → ARCHITECTURE → HOSTILE REVIEW → ARCHITECTURE FREEZE → IMPLEMENTATION → VALIDATION"

**Note on sourcing:** Only titles, authors, or links explicitly present in the supplied architecture and implementation-blueprint documents should populate this slide's citation list at final-design time; none are fabricated here, and none should be added without being traceable to a supplied source.

## 5. What This Page Must Not Contain
- A plain, unorganized bibliography dump
- Technical pipeline content (belongs on Page 2)
- The risk matrix or impact model
- Irrelevant references included only to look academic

## 6. Required Primary Visual
**Type:** Horizontal design-journey flow diagram, paired with a compact three-column reference organization below it. **Purpose:** Show the design process itself as evidence of rigor, not just list sources. **Components:** Seven-stage flow (Problem → Research → Architecture → Hostile Review → Architecture Freeze → Implementation → Validation) as a single horizontal timeline; below it, three labeled columns for the three foundation categories. **Information flow:** Left to right for the timeline; top-to-bottom within each column. **Critical visual distinction:** "Hostile Review" and "Architecture Freeze" should be visually marked (e.g., a distinct icon or color) as the specific stages that produced the 11-finding audit and its resolutions — this is the most differentiating part of the process story.

## 7. Visual Composition
TOP: Title, subtitle, key statement.
CENTER: The seven-stage horizontal design-journey timeline.
BOTTOM: The three-column reference organization.

## 8. Visual Hierarchy
**3 seconds:** The seven-stage timeline shape — "this went through a real process."
**15 seconds:** The three foundation categories — protocol, statistics, engineering/validation.
**60 seconds:** Specific citations within each column, for a judge who wants to verify.

## 9. What the Presenter Says
"This design didn't start and end with one draft. It went through a documented hostile review — eleven separate findings against the architecture, each one resolved and recorded, not smoothed over — before being frozen. The statistical methods are established ones: profile-likelihood testing, sequential likelihood ratios, GLR-CUSUM change detection, all calibrated by Monte Carlo simulation with a strict separation between calibration and evaluation data. That's the research and process foundation everything else in this presentation stands on."

## 10. Expected Judge Questions
- "What was the most significant finding in your hostile review, and how did you resolve it?"
- "Are any of your statistical methods novel, or all borrowed from existing literature?"
- "How do we know the hostile review was genuinely adversarial and not self-serving?"

## 11. Suggested Answers
- The most consequential finding was that the relationship between QSENTINEL's flags and the protocol's own acceptance decision was never explicitly stated — we resolved it by making non-interference a binding, code-enforced guarantee rather than an assumption, which is now the architecture's central non-negotiable property.
- The individual statistical tools (profile likelihood, SPRT, GLR-CUSUM, joint Monte Carlo calibration) are established methods; the contribution is their specific composition — two structurally separate decision planes, a two-dimension quantum-evidence reframing, and unconditional cross-session ingestion — applied to a teleportation-QDS substrate no prior work addresses this way.
- The review process is documented with specific, falsifiable findings and concrete resolutions (Section 4 of this document) rather than a general statement that review occurred — a judge can check any finding against its stated resolution and mechanism.

## 12. Possible Weaknesses
**Criticism:** A reference slide with no external published citations (beyond QS-L) could look thin. **Honest response:** The statistical methods cited (profile likelihood, SPRT, GLR-CUSUM, Monte Carlo calibration) are standard, well-established techniques in statistical detection theory — the slide's job is to show they were deliberately chosen and correctly composed for this problem, not to claim novelty in the underlying math itself.

---

## 12. Page Transition Logic

```
PAGE 1: What is QSENTINEL?
   ↓
PAGE 2 ANSWERS: Here is the exact mechanism — FSM, quantum evidence, Stage 1/2, GLR-CUSUM, forensic log.
   ↓
NEXT QUESTION CREATED: Can this actually be built and trusted, or is it a diagram?

PAGE 2: How does it technically work?
   ↓
PAGE 3 ANSWERS: Here is why every technology choice is minimal-by-necessity, and how the two highest-risk
                guarantees are enforced structurally, not by convention.
   ↓
NEXT QUESTION CREATED: Assuming it can be built, why should anyone care?

PAGE 3: Can this architecture actually be built and trusted?
   ↓
PAGE 4 ANSWERS: Here is the layered value it adds over protocol validity alone, and the explicit boundary
                of what it does not claim.
   ↓
NEXT QUESTION CREATED: Is this just an assertion, or is there real work behind it?

PAGE 4: Why is the solution valuable?
   ↓
PAGE 5 ANSWERS: Here is the research foundation and the documented hostile-review process behind the design.
   ↓
END: The judge has a complete, self-consistent picture with no unaddressed "so what backs this up" question.
```

---

## 13. Content Reserved for Q&A

| Item | Question It Answers | Short Judge Response | Technical Backup |
|---|---|---|---|
| Full profile-likelihood formula (Stage 1) | "Show me the actual math." | "It's a standard profile-likelihood goodness-of-fit statistic — nuisance parameter profiled out, critical value obtained by simulation at our actual sample size rather than an asymptotic table lookup." | Section 2.3 of this document |
| Full Stage 2 joint-calibration procedure | "How exactly is the rejection region computed?" | "Offline Monte Carlo simulation of the joint null distribution of our two statistics, with the region searched to maximize power subject to α=0.01, using data strictly held out from evaluation." | Section 2.4; Architecture freeze Section 8 |
| Full attack taxonomy (7 conditions) | "What attacks does your harness actually test?" | "Clean forgery, sub-threshold forgery, replay, impersonation, unauthorized verification, channel manipulation (three sub-variants), and low-and-slow drift." | Section 6.4 |
| Attack-specific implementation details | "How is a given attack actually implemented?" | "Each attack is a pure transform on the session-construction pipeline via a shared strategy interface — never a special case inside the protocol or monitor code." | Section 5 of the Implementation Blueprint |
| Repository/file structure | "What does the codebase actually look like?" | "A single monorepo with hard package boundaries — `qds/` for the protocol, `qsentinel_monitor/` for the advisory layer, enforced by a CI import-linter contract." | Implementation Blueprint, repository structure section |
| API routes | "What does the API actually expose?" | "A small, demo-scoped FastAPI surface — session execution, one live SSE stream, experiment execution, and forensic querying; no auth/multi-tenant surface, by design." | Implementation Blueprint, API architecture section |
| Database schema | "How is data actually persisted?" | "SQLite for runtime session data, with CUSUM state as the sole persisted home for that specific state; calibration and experiment artifacts live as content-hashed flat files, not database blobs." | Implementation Blueprint, persistence model section |
| Complete validation task list | "What exactly still needs to be measured?" | "Six prioritized items, starting with the Stage 2 held-out calibration and the full seven-condition harness re-run — nothing is claimed as complete yet." | Section 6.2 of this document |
| Monte Carlo experiment configuration | "How many trials, what seeds?" | "Disjoint seed ranges by purpose — calibration, validation, evaluation — enforced at runtime, not only by convention, so no trial can be silently reused across purposes." | Section 5 of this document |

---

## 14. Final Page-by-Page Content Audit

| Important Information | Final Page | Included? | Why This Placement Is Correct |
|---|---|---|---|
| Two-plane separation (protocol/monitor) | Page 1 (intro) + Page 2 (mechanism) | Yes | Introduced conceptually before being shown mechanically — a judge must understand the idea before the diagram makes sense |
| Quantum evidence reframing (2 dimensions) | Page 2 | Yes | Technical substance belongs on the technical slide, not diluted onto Page 1 |
| Stage 1 / Stage 2 / GLR-CUSUM | Page 2 | Yes | Core mechanism content — this is what "Technical Approach" exists to show |
| Forensic logging | Page 2 (mechanism) + Page 4 (value) | Yes | Split deliberately: how it works vs. why it matters, without duplicating detail |
| Technology stack | Page 2 (strip) + Page 3 (rationale) | Yes | Named early as evidence of a real system; justified later as evidence of a buildable one — not explained twice |
| Risk matrix | Page 3 | Yes | Owns feasibility credibility; not diluted across other slides |
| Calibration/non-interference structural guarantees | Page 3 | Yes | The two highest-risk elements identified by the architecture's own review get top billing on the feasibility slide |
| Impact/target ecosystem | Page 4 | Yes | Only after feasibility is established, per the transition logic |
| Explicit non-claims boundary | Page 4 (full) + Page 1 (one line) | Yes | Full statement where impact is discussed, to avoid overclaiming; a compressed echo on Page 1 sets expectations early |
| Research foundations/hostile review | Page 5 | Yes | Closing credibility page; not front-loaded, so it doesn't compete with the core narrative |
| Full limitations table (11 items) | Q&A only | Reserved | Too dense for any single slide; the compressed boundary box on Page 4 carries the presentation-appropriate version |
| Full statistical formulas | Q&A only | Reserved | Belongs in technical backup, not on a 5-slide deck |
| Implementation-level detail (repo, API, schema) | Q&A only | Reserved | Not part of the judge-facing narrative; available on demand |

No major QSENTINEL concept from the frozen architecture is missing from either the five slides or the Q&A reserve. No slide carries more than one dominant visual. No information is duplicated beyond the deliberate, differently-framed repeats logged in Section 10's information map.

---

## 15. 5-Slide Executive Flow

**Page 1:** The audience sees a clean two-lane diagram splitting protocol decision from monitoring decision. They learn that QSENTINEL is a separate, advisory observability layer, not a gate. They want to see the next page because the diagram raises an immediate question: what's actually happening inside that dashed lane?

**Page 2:** The audience sees the full pipeline populated with real stages — FSM, quantum evidence, Stage 1, Stage 2, GLR-CUSUM, forensic log. They learn the mechanism is concrete and specific, not a black box, and that cross-session monitoring runs on every session unconditionally. They want to see the next page because a technically dense diagram naturally invites the question: is this actually buildable in the time available, and can its numbers be trusted?

**Page 3:** The audience sees a risk-to-mitigation matrix that names the two hardest problems (calibration leakage, protocol interference) as structurally solved, not merely promised. They learn the technology choices are minimal by necessity, and that trustworthiness is engineered into the calibration process itself. They want to see the next page because feasibility alone doesn't answer why any of this should matter to them.

**Page 4:** The audience sees a layered value model stacking protocol security up through forensic accountability, paired with an explicit statement of what QSENTINEL does not claim. They learn the value proposition and its honest limits in the same breath. They want to see the next page because a claim this carefully bounded invites the question: what evidence and process actually back it up?

**Page 5:** The audience sees a design-journey timeline culminating in a documented hostile review, paired with the three categories of research foundation. They learn the design was pressure-tested, not assumed correct on the first pass. The presentation closes with no unanswered "so what backs this up" question outstanding.

---

## 16. Final Design Conclusion

### Final Presentation Strategy

This structure is stronger than a generic hackathon deck because it does not ask the judge to take any claim on faith for longer than one slide. Every claim made on an earlier slide is either mechanically demonstrated on the next slide (Page 1's "advisory, never overrides" claim is shown as a code-enforced import boundary and immutable-object guarantee on Page 3) or explicitly bounded rather than left open (Page 4's impact claims are immediately paired with a stated non-claims box). The five-slide sequence tells one continuous argument — what it is, how it works, whether it can be trusted, why it matters, what backs it up — rather than five loosely related topic slides.

The final deck should feel like a serious deep-tech cybersecurity product being explained with the clarity and rigor of a team that understands its own architecture down to the level of *why* each alternative was rejected, not just what was chosen.

**Final design principles:**
- Architecture first — every visual traces to a real component, never an illustrative placeholder.
- Information hierarchy over decoration — the 3-second / 15-second / 60-second layering on every slide is deliberate, not incidental.
- Technical credibility over buzzwords — no AI/ML terminology anywhere, consistent with the architecture's actual non-AI/ML construction.
- Explicit boundaries over exaggerated claims — Page 4's non-claims box and Page 3's honest "no numbers yet" framing are treated as credibility assets, not weaknesses to hide.
- Visuals that explain systems — every dominant visual on every slide is a diagram of a real mechanism, never a generic stock icon set.
- Minimal but deliberate text — every line specified in Section 4 of each page template is the actual on-slide text, not a placeholder to be filled in later.
- One core question answered per slide — enforced by the Page Objective and Core Message fields on every page template.
- No unnecessary duplication — enforced by the information map (Section 10) and the final content audit (Section 14).

---

## Final Internal Review Checklist

| # | Check | Status |
|---|---|---|
| 1 | Is QSENTINEL correctly separated from the authoritative protocol? | Yes — Sections 2.1–2.4, Page 1, Page 2 |
| 2 | Does QSENTINEL ever appear to override protocol acceptance? | No — explicit non-interference guarantee stated in Sections 2.4, 5.6/Mechanisms, Page 1, Page 4 |
| 3 | Are Stage 1 and Stage 2 represented correctly? | Yes — Sections 2.3, 2.4 |
| 4 | Is Stage 1 clearly model-fit/mutual consistency? | Yes |
| 5 | Is Stage 2 clearly the jointly calibrated decision? | Yes |
| 6 | Is GLR-CUSUM shown as unconditional cross-session monitoring? | Yes — Sections 2.5, 3 (diagram), Page 2 (explicit visual instruction against conditioning it on a Stage 2 flag) |
| 7 | Is optional attribution correctly represented as forensic? | Yes — Sections 2.6, 3 (module table) |
| 8 | Are measurement-derived observables represented correctly (2 dimensions, not 4 independent features)? | Yes — Section 2.2 |
| 9 | Has AI/ML been introduced without authorization? | No |
| 10 | Has any unsupported technology been introduced? | No — Section 5.1 matches the Implementation Blueprint exactly |
| 11 | Has any unsupported quantitative performance claim been added? | No — Section 6 explicitly withholds all figures pending validation |
| 12 | Are limitations honest? | Yes — Section 8, 11-item table |
| 13 | Does every important concept have a designated presentation page? | Yes — Section 10 information map |
| 14 | Does every slide explicitly state what it should NOT contain? | Yes — Section 5 of every page template |
| 15 | Can another designer build the PPT without guessing what belongs where? | Yes — exact titles, labels, and content are specified per page |
| 16 | Does the document match the required depth and presentation-engineering utility? | Yes |

**All checks pass. No revision required before finalizing.**
