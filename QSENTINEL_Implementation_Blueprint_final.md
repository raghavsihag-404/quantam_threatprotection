# QSENTINEL — Implementation Blueprint
## From Frozen Architecture to Buildable System

This document maps the frozen QSENTINEL architecture (protocol path / monitoring path separation, FSM, two-independent-dimension quantum evidence, Stage 1 mutual-consistency check, Stage 2 joint calibrated decision, unconditional GLR-CUSUM, optional forensic attribution, signed hash-chained log) onto concrete, buildable software. No conceptual component from the frozen architecture is redesigned here; every decision below is a mapping and hardening decision, not an architectural one. This document is self-contained: a new engineering team can build QSENTINEL from this file alone.

---

## 1. Title and Scope

**System:** QSENTINEL — a runtime, non-AI/ML statistical monitoring layer sitting alongside a teleportation-distributed QS-L digital-signature protocol, observing only the classical measurement telemetry the protocol already discloses.

**Scope of this document:** the complete technical implementation — technology stack, repository structure, module boundaries, runtime and offline data flow, statistical pipeline, persistence model, forensic logging, API, frontend, testing, deployment, and build order. The conceptual architecture (protocol/monitor separation, FSM, Stage 1/2, GLR-CUSUM, optional attribution) is treated as frozen input; nothing here proposes an alternative to it.

**Non-negotiable invariant carried through every section:** QSENTINEL's advisory verdicts never override, block, or delay the protocol's own deterministic acceptance of a legitimate signature. This is enforced structurally, not by convention, as detailed in Section 6.

---

## 2. Executive Technical Verdict

The frozen architecture is implementable by a small team within a hackathon timeline if the quantum simulation stays minimal (fixed 3-qubit statevector, no general circuit engine) and the statistical layer stays in NumPy/SciPy rather than a probabilistic-programming framework. Both choices are justified on necessity grounds, not familiarity, in Section 3.

The single highest-risk element in the system is not the quantum simulation — it is Stage 2's calibration/evaluation separation and the protocol/monitor non-interference guarantee. Both are discipline problems, not technology problems, and both are made structural rather than aspirational throughout this document: calibration/evaluation separation via a runtime-enforcing `SeedAllocator` and a content-hashed `CalibrationArtifact` (Section 9), and non-interference via immutable protocol objects plus CI-enforced import boundaries (Section 6). No AI/ML library appears anywhere in this stack.

---

## 3. Technology Stack and Rationale

| Layer | Choice | Why | Why the alternatives were rejected |
|---|---|---|---|
| Core language | **Python 3.11+** | One language covers quantum simulation, statistics, backend, and glue code with no build/FFI boundary; NumPy/SciPy are fast enough at this qubit count (statevectors of size 8) and trial count (thousands, not millions) | A compiled hybrid (Python+C++/Rust) adds a build step and a language boundary for no measured performance need — 3-qubit statevector ops are already sub-millisecond in pure NumPy. Julia has a weaker web/API ecosystem for the demo layer |
| Quantum simulation | **Custom lightweight NumPy statevector simulator**, fixed 3-qubit register (message qubit + 2 halves of a Bell pair) | The protocol never exceeds 3 qubits per session and needs no general circuit compiler. A ~150-line module with explicit Bell-pair preparation, Bell-measurement, Pauli-correction, and projective-measurement functions is fully sufficient, fully transparent (every operation is inspectable, which matters for a hostile-audit-style demo), and orders of magnitude faster per trial than a general SDK for Monte Carlo harnesses running thousands of sessions | Qiskit/Cirq/PennyLane's circuit-compilation and backend-abstraction machinery solves problems this system does not have (variable circuit depth, hardware targeting, transpilation). It would run correctly but slower per trial, and would obscure exactly the operations a judge or auditor most wants to see in plain math |
| Statistical computing | **NumPy + SciPy (`scipy.stats`, `scipy.optimize`) with hand-written likelihood functions** | Profile likelihood, SPRT, GLR-CUSUM, and goodness-of-fit are all closed-form or simulation-calibrated for this problem. `scipy.optimize.minimize_scalar` profiles out nuisance parameters; `scipy.stats.chi2`/custom Monte Carlo supply critical values. Every formula maps directly to an explicit, auditable function | PyMC/Stan are Bayesian inference engines built for problems requiring MCMC — nothing in Stage 1/Stage 2/GLR-CUSUM requires it. `statsmodels` offers convenience wrappers but would still require hand-written GLR-CUSUM and joint-region calibration, adding a dependency without removing custom code |
| Monte Carlo execution | **Python `multiprocessing.Pool`, explicit `multiprocessing.get_context("spawn")`, NumPy `Generator(PCG64)` seeded per trial** | Thousands of independent Monte Carlo trials with no cross-trial dependency is an embarrassingly parallel, single-machine problem — a process pool saturates available cores with zero infrastructure. The `spawn` context is pinned explicitly (rather than left to the OS default, which is `fork` on Linux and `spawn` on macOS/Windows) so trial results are identical across every team member's machine, not just within one OS. Reproducibility comes from explicit per-trial seeds derived from a documented seed-sequence scheme (Section 10), not from a distributed job queue | Celery/Redis and Dask solve distributed/durable-queue problems this system does not have; they add operational surface (broker, workers, serialization) that is pure overhead at this scale and does not improve reproducibility |
| Backend/API | **FastAPI + Pydantic v2** | Automatic OpenAPI docs (useful for a judge inspecting the API) and Pydantic v2 models give the request/response schema the same explicitness the statistical layer needs — a `Stage2Result` Pydantic model is simultaneously documentation and validation. One Server-Sent Events endpoint (Section 19) uses FastAPI's async support for one-directional live-session streaming, so every stated reason for choosing FastAPI corresponds to an implemented endpoint | Flask requires bolting on async, docs, and validation separately. Django's ORM/admin/full-stack conventions solve problems (multi-app routing, auth systems, templating) this single-purpose backend does not have |
| Frontend | **React + Vite + Recharts** | A hackathon demo needs a small number of highly interactive, real-time-updating views (live session flow, CUSUM drift chart, forensic log) — Vite gives fast dev iteration with no SSR/server-component complexity this project has no use for | Next.js's routing/SSR/edge-function machinery solves problems (SEO, server rendering) irrelevant to an internal demo tool. Streamlit is faster to prototype but cannot cleanly express the protocol-decision-vs-advisory-decision visual separation the architecture requires as a first-class UI concept — it is app-per-script, not component-per-concern |
| Relational storage | **SQLite**, with a documented, mechanical PostgreSQL migration path | Single-writer, single-machine, no concurrent-write contention at this stage (one demo session/experiment runner at a time). The schema (Section 17) uses only standard SQL types, so migrating to Postgres later is a connection-string change plus `alembic upgrade`, not a rewrite | Standing up Postgres now is an infrastructure dependency with no matching current need |
| Migrations | **Alembic, `render_as_batch=True`** | SQLite's limited `ALTER TABLE` support (no `DROP COLUMN`, no altering column types/constraints in the general case) means every non-additive future migration needs Alembic's batch mode, which recreates the table under the hood. This is configured from Phase 0, before it is needed | Leaving batch mode unset works until the first non-additive schema change, then fails with an opaque SQLite error at an inconvenient point in the project |
| Calibration/experiment artifacts | **Filesystem: version-keyed JSON (metadata) + NumPy `.npz` (raw arrays), each artifact carrying an embedded SHA-256 content hash verified on load** | Calibration artifacts and Monte Carlo raw output are large, write-once, and read-heavy — flat files under `artifacts/calibration/{version}/` are simpler to version and diff than BLOBs in SQLite. The embedded content hash, not the filename, is what guarantees an artifact hasn't been silently altered | A document store (MongoDB) adds a second database technology for a need the filesystem already meets. Storing hash-chained records as mutable table rows would require extra application-level enforcement that a genuinely append-only file avoids by construction |
| Forensic log — chain | **`hashlib.sha256`, append-only JSONL** | Stdlib-only, zero new dependency, naturally append-only as a flat file where each record includes the previous record's hash | Storing hash-chained records as mutable table rows requires extra application-level enforcement a flat file avoids by construction |
| Forensic log — signature | **Ed25519 via the `cryptography` package** | A mature, minimal-dependency, widely audited signature scheme; one keypair per deployment, private key loaded from a filesystem path set by an environment variable, never committed to source control and never stored in SQLite | No alternative was seriously considered — this is the one place a new cryptographic dependency is genuinely required, named explicitly rather than left implicit |
| Dependency locking | **A single committed lockfile** (`pip-compile`-generated `requirements.lock`, or a Poetry lockfile — one approach, chosen once, not both) | Reproducibility is a headline requirement of this system; an unpinned dependency set silently undermines it | Presenting multiple competing package-management approaches invites drift between what different team members actually have installed |
| Testing | **pytest**, four tiers: unit / integration / statistical / regression | Covers quantum correctness, deterministic reproducibility, protocol/monitor non-interference, statistical validity, and stale-figure prevention without adding categories the project doesn't need | Property-based testing, mutation testing, and performance-regression tiers were considered and rejected — nothing in this system's risk profile is better covered by them than by the four tiers already specified |
| Import boundary enforcement | **import-linter** | A CI-enforced contract cannot rot silently the way a code-review convention can; this is the single most important CI check in the system, protecting the non-interference guarantee | — |

**Explicitly not used, and not to be added without a genuine implementation impossibility, internal contradiction, correctness flaw, or security flaw:** Qiskit, Cirq, PennyLane, Celery, Redis, RabbitMQ, Dask, PyMC, Stan, scikit-learn, TensorFlow, PyTorch, PostgreSQL (at this stage), MongoDB, any authentication/multi-tenancy system, generic admin/CRUD screens, a general N-qubit circuit engine.

---

## 4. Complete Implementation Architecture

```
┌─────────────────────────────── qds/  (PROTOCOL — owns authoritative decisions) ───────────────────────────────┐
│  bell_pair.py · teleportation.py · pauli.py · measurement.py · noise.py · protocol.py (QS-L verify()) ·        │
│  transcript.py  →  SessionTranscript (frozen) · ProtocolDecision (frozen)                                       │
└────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────┘
                                                   │ frozen session transcript + measurement telemetry
                                                   ▼
┌─────────────────────── qsentinel_monitor/  (MONITOR — owns advisory decisions only) ────────────────────────────┐
│                                                                                                                   │
│  protocol_evidence/          quantum_evidence/                                                                   │
│    fsm.py                      collector.py (m, C, H, Pauli-consistency)                                         │
│    freshness.py                stage1.py (profile-likelihood mutual-consistency)                                 │
│    authorization.py            stage2.py (joint SPRT + goodness-of-fit, calibrated region)                       │
│                                 glr_cusum.py (unconditional ingestion, closed-form sup_θ, SQLite-backed)          │
│                                 attribution.py (optional, forensic-only)                                          │
│                                                                                                                   │
│  orchestrator.py  ← qsentinel_monitor.orchestrator.analyze(): combines FSM + quantum-evidence result into one    │
│                       MonitoringDecision (advisory)                                                              │
│  forensic_log.py  ← signed (Ed25519), hash-chained (SHA-256) append-only writer                                  │
└────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
┌──────────────────────────── attacks/  (test-time only — never imported by qds/ or qsentinel_monitor/ core) ─────┐
│  base.py (AttackStrategy)  · forgery.py · replay.py · impersonation.py · unauthorized_verification.py            │
│  channel_manipulation.py · low_and_slow.py                                                                        │
└────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
┌───────────────────────────────────────── experiments/  (offline) ────────────────────────────────────────────┐
│  calibration.py (Stage 2 offline calibration, held-out enforced)  · harness.py (7-condition Monte Carlo)         │
│  ablation.py · verification_accuracy.py · complexity_benchmark.py                                                │
└────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
┌────────────────────────────────────────── api/ (FastAPI) ───────────────────────────────── frontend/ (React) ──┐
│  routes for session run, experiment run, forensic query, CUSUM history, session stream (SSE)  live demo dash    │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Repository root is `qsentinel-system/`** — deliberately distinct from both `qds/` and `qsentinel_monitor/`, so no reader or tool ever has to disambiguate the checkout directory name from a Python package name.

---

## 5. Component-by-Component Implementation Plan

| # | Component | Responsibility | Inputs | Outputs | Runtime/Offline | Deterministic/Statistical | Key classes/functions |
|---|---|---|---|---|---|---|---|
| 1 | QDS Protocol Simulator | Orchestrates one full session: setup → teleport → correct → measure → verify | Session config | `SessionTranscript`, `ProtocolDecision` (both frozen) | Runtime | Deterministic (given inputs) | `qds.protocol.run_session()` |
| 2 | Quantum State / Teleportation Engine | Statevector ops for a fixed 3-qubit register | Basis choices, noise params | Post-teleportation statevector | Runtime | Deterministic given RNG seed | `qds.teleportation.teleport()` |
| 3 | Bell-Pair Module | Prepares `\|Φ⁺⟩` | — | 2-qubit entangled statevector | Runtime | Deterministic | `qds.bell_pair.prepare()` |
| 4 | Pauli Encoding Module | Encodes raw key bit as Pauli eigenstate | `(k_i, b_i)` | 1-qubit state | Runtime | Deterministic | `qds.pauli.encode_eigenstate()` |
| 5 | Pauli Correction Module | Applies `I/X/Z/XZ` per Bell-measurement outcome | Bell outcome | Corrected state | Runtime | Deterministic | `qds.pauli.correct()` |
| 6 | Projective Measurement Module | Random-basis BB84-style measurement | State, basis | Classical outcome bit | Runtime | Deterministic given seed (statistical outcome distribution) | `qds.measurement.project()` |
| 7 | Noise Model | Injects symmetric depolarizing noise at parameter `p` | State, `p` | Noisy state | Runtime | Statistical | `qds.noise.depolarize()` |
| 8 | Attack Injection Framework | Applies an `AttackStrategy` to a session before/during measurement | `AttackConfig` | Modified transcript | Runtime (test-time only) | Deterministic per-strategy, statistical in aggregate | `attacks.base.AttackStrategy` |
| 9 | Session Transcript Generator | Assembles the full signed transcript both paths consume, then freezes it | All module outputs | `SessionTranscript` (frozen) | Runtime | Deterministic | `qds.transcript.build()` |
| 10 | Protocol State Integrity Engine | FSM invariant checking | Transcript metadata | PASS / protocol REJECT | Runtime | Deterministic | `qsentinel_monitor.protocol_evidence.fsm.FSM` |
| 11 | Authorization/Freshness Manager | Token validity, single-use, sequencing | Transcript metadata | PASS/FAIL per check | Runtime | Deterministic | `qsentinel_monitor.protocol_evidence.authorization.check()` |
| 12 | Quantum Evidence Collector | Computes `m, C, H`, Pauli-correction consistency | Measurement telemetry | Observable vector | Runtime | Statistical (estimators) | `qsentinel_monitor.quantum_evidence.collector.extract()` |
| 13 | Stage 1 Detector | Profile-likelihood mutual-consistency test | Observable vector | MODEL_VALID/INVALID + statistic | Runtime | Statistical | `qsentinel_monitor.quantum_evidence.stage1.evaluate()` |
| 14 | Stage 2 Decision Engine | Joint SPRT + goodness-of-fit against calibrated region | Stage-1-routed vector, `CalibrationArtifact` | ACCEPT/FLAG(REJECT)/FLAG(INVESTIGATE) | Runtime | Statistical | `qsentinel_monitor.quantum_evidence.stage2.evaluate()` |
| 15 | Offline Calibration Engine | Monte Carlo joint-null simulation, region search | Honest-execution reference model, held-out calibration trials | `CalibrationArtifact` (versioned, content-hashed) | **Offline only** | Statistical | `experiments.calibration.calibrate()` |
| 16 | GLR-CUSUM Detector | Cross-session change-point statistic, SQLite-backed persistence | Per-session summary, persisted rolling window state | Flag / not flagged | Runtime | Statistical | `qsentinel_monitor.quantum_evidence.glr_cusum.CusumState.update()` |
| 17 | Attribution Engine (optional) | Sub-type likelihood ratios on flagged sessions only | Flagged session's observable stream | CONFIDENT/AMBIGUOUS/NO ATTRIBUTION | Runtime, conditional | Statistical | `qsentinel_monitor.quantum_evidence.attribution.classify()` |
| 18 | Threat Orchestrator | Combines FSM + Stage1/2 + CUSUM into one `MonitoringDecision` | All monitor-side outputs | `MonitoringDecision` (advisory) | Runtime | Deterministic combination of statistical inputs | `qsentinel_monitor.orchestrator.analyze()` |
| 19 | Forensic Evidence Logger | Writes signed (Ed25519), hash-chained (SHA-256) append-only record | `ProtocolDecision`, `MonitoringDecision`, evidence | Log entry | Runtime | Deterministic | `qsentinel_monitor.forensic_log.append()` |
| 20 | Hash Chain / Integrity Engine | Computes `hash(record_i) = SHA256(record_i \|\| hash(record_{i-1}))`; verification tolerates a torn trailing write | Log entries | Chain, verifiable | Runtime + offline (verify) | Deterministic | `qsentinel_monitor.forensic_log.HashChain` |
| 21 | Experiment Runner | Executes a configured experiment | `ExperimentConfig` | `ExperimentRun` record + raw results | Offline | Mixed | `experiments.runner.run()` |
| 22 | Monte Carlo Harness | Runs the 7-condition attack suite at N trials/condition, spawn-context process pool | Attack configs, trial count, seed scheme | Per-condition detection-rate table | Offline | Statistical | `experiments.harness.run_all_conditions()` |
| 23 | Ablation Runner | Re-runs harness with each component disabled in turn | Component-disable flags | Per-ablation detection-rate table | Offline | Statistical | `experiments.ablation.run()` |
| 24 | Verification Accuracy Evaluator | Legitimate-session acceptance sweep across noise levels | `p` range | Acceptance-rate-vs-`p` table | Offline | Statistical | `experiments.verification_accuracy.sweep()` |
| 25 | Performance Benchmark Runner | Wall-clock + complexity-class confirmation per module | Module, input size range | Timing table | Offline | Empirical | `experiments.complexity_benchmark.run()` |
| 26 | API Layer | Exposes protocol/monitor/experiment operations, including a live-session SSE stream | HTTP requests | JSON / SSE responses | Runtime | — | `api/routes/*.py` |
| 27 | Frontend Dashboard | Visualizes session flow, evidence, decisions, forensic log | API responses | Rendered UI | Runtime | — | `frontend/src/pages/*.tsx` |

Every row above has at least one corresponding test class, specified per tier in Section 21.

---

## 6. Critical Protocol/Monitor Separation

The protocol/monitor separation is the guarantee the entire system's credibility depends on. It is enforced through three independent, structural mechanisms — not through code-review convention.

**Mechanism 1 — immutable protocol objects.** `ProtocolDecision` and `SessionTranscript` are defined as frozen types (`@dataclass(frozen=True)` or Pydantic `model_config = ConfigDict(frozen=True)`, chosen consistently with the rest of the type stack). Once `qds.transcript.build()` produces a `SessionTranscript` and `qds.protocol.run_session()` sets its `ProtocolDecision`, no code anywhere — including `qsentinel_monitor/` — can mutate either object. An attempted write raises immediately, at the point of the attempted mutation, rather than being merely absent from the observed behavior of a test suite.

```text
qds/ owns authoritative protocol objects
        ↓
objects are finalized and frozen
        ↓
qsentinel_monitor/ consumes them as read-only input
        ↓
mutation is impossible by construction, not merely undesired by convention
```

**Mechanism 2 — CI-enforced import direction.** `qds/` has zero imports from `qsentinel_monitor/`, `api/`, `experiments/`, `attacks/`, or `frontend/`. This is enforced by an `import-linter` contract that fails the build on violation:

```ini
[importlinter]
root_package = qsentinel_monitor

[contract: protocol-purity]
name = qds must not depend on qsentinel_monitor or anything downstream of it
type = forbidden
source_modules =
    qds
forbidden_modules =
    qsentinel_monitor
    api
    experiments
    attacks
    frontend
```

**Mechanism 3 — a dedicated non-interference test.** `test_non_interference.py` asserts, for every attack condition in the harness, that `protocol_decision` is bit-for-bit identical whether or not `qsentinel_monitor.orchestrator.analyze()` is called at all — i.e., calling the monitor is provably a no-op on the protocol's own output.

```python
# api/routes/session.py

def run_session(config: SessionConfig) -> SessionResult:
    transcript = qds.protocol.run_session(config)          # builds transcript, includes protocol's own verify()
    protocol_decision = transcript.protocol_decision        # frozen — authoritative, immutable from this point on

    monitoring_decision = qsentinel_monitor.orchestrator.analyze(
        transcript=transcript,                               # frozen, read-only
        protocol_decision=protocol_decision,                 # frozen, read-only — passed in for logging context ONLY,
    )                                                         # never used as an input to any monitor detection rule

    forensic_log.append(protocol_decision, monitoring_decision, transcript.evidence)

    return SessionResult(protocol_decision, monitoring_decision)  # stored and returned SEPARATELY, never merged
```

`qsentinel_monitor.orchestrator.analyze()` has no code path that can raise, block, or mutate `protocol_decision` — this is structurally impossible (Mechanism 1), not merely unobserved (Mechanism 3 confirms it; Mechanism 2 prevents the import that would be needed to attempt it).

The only exception, explicitly scoped and unchanged from the frozen architecture: FSM violations are a **protocol-level** rejection, computed inside `qds/` as part of `run_session()`, not inside `qsentinel_monitor/` — the FSM's authoritative role is a property of the protocol's own transcript validity, not of the advisory monitor.

---

## 7. Dependency Direction and Import Boundaries

```text
api/
 ├── qds                          (calls protocol.run_session())
 ├── qsentinel_monitor            (calls orchestrator.analyze(), read-only inputs)
 └── db / forensic_store          (persistence)

qsentinel_monitor/
 ├── may consume the necessary frozen qds domain objects (ProtocolDecision, SessionTranscript)
 │   as read-only, immutable types
 └── must never invoke qds protocol decision logic, mutate protocol-owned objects, or create any
     feedback path into authoritative protocol acceptance

qds/
 └── must never import qsentinel_monitor, api, experiments, attacks, or frontend

experiments/
 ├── may consume qds and qsentinel_monitor
 └── must request seeds only from its own SeedAllocator range — CALIBRATION / VALIDATION /
     EVALUATION ranges are disjoint and enforced at runtime, not only by directory convention

attacks/
 ├── may consume qds's transcript-builder interface
 └── must never be imported by qds/ or qsentinel_monitor/ core (test-time only, one-directional)
```

The import-linter configuration in Section 6 is the mechanical enforcement of this rule set and is validated in CI on every commit, alongside a dedicated check that no file under `qds/` imports anything from `qsentinel_monitor/`.

---

## 8. Runtime Data Flow

```
Session Transcript (frozen once built)
    → { FSM check (qds-authoritative), Quantum Evidence Collector }
    → Stage 1  — evaluates against the FROZEN CalibrationArtifact's noise-family parameters,
                 loaded once into memory at API process startup (Section 11), never re-read
                 or re-hashed per session
    → Stage 2  — evaluates against the FROZEN CalibrationArtifact's rejection region, never
                 recomputed at runtime
    → GLR-CUSUM — reads and updates the persisted CusumState row in SQLite (Section 13), the
                 sole store for this state
    → MonitoringDecision (advisory) → Forensic Log (Section 18)
```

Stage 1/2 never fit or calibrate anything at runtime — they only *evaluate* against an already-frozen, versioned `CalibrationArtifact` loaded from disk once per process lifetime.

---

## 9. Offline Calibration/Validation/Evaluation Data Flow

```
Seed range [0, 50_000)  →  CALIBRATION trials  →  honest-execution reference model fit
                                                  →  Monte Carlo joint-null simulation of (S_SPRT, S_gate)
                                                  →  rejection region search (maximize power s.t. α_system=0.01)
                                                  →  CalibrationArtifact (versioned, SHA-256 content-hashed)

Seed range [50_000, 100_000)  →  EVALUATION trials (7 conditions)  →  detection-rate tables
                                                                    →  MUST load CalibrationArtifact,
                                                                       MUST NOT regenerate it
```

Calibration, validation, and evaluation code are separated by directory convention (`experiments/calibration.py` vs. the rest) **and** by a runtime-enforcing `SeedAllocator` that raises if a requested seed falls outside its declared purpose's range — the separation is a binding, checkable guarantee, not an aspiration. `CalibrationArtifact` objects carry a SHA-256 hash of the calibration trial set they were derived from; the evaluation harness logs this hash into every `ExperimentRun` record, so any future audit can mechanically confirm no evaluation run used a calibration artifact whose hash doesn't match a properly separated calibration run.

---

## 10. Seed Allocation and Reproducibility

`experiments/seed_allocator.py` defines a `SeedAllocator` that:

- assigns non-overlapping seed ranges by declared purpose (`CALIBRATION`, `VALIDATION`, `EVALUATION`);
- raises at request time if a caller asks for a seed outside its declared range;
- is asserted non-overlapping by a dedicated CI test, independent of any individual experiment script honoring the convention correctly.

Every Monte Carlo trial derives its RNG state from `numpy.random.Generator(numpy.random.PCG64(seed))`, with the seed itself drawn from the allocator, never hand-picked inline.

Every process pool used for Monte Carlo execution is created with an explicit start-method context:

```python
import multiprocessing

ctx = multiprocessing.get_context("spawn")
with ctx.Pool(processes=n_workers) as pool:
    ...
```

The explicit `spawn` context — rather than the platform default, which is `fork` on Linux and `spawn` on macOS/Windows — is part of the cross-platform reproducibility guarantee: it ensures Monte Carlo results are identical across every team member's machine regardless of OS, not merely within one OS's default behavior. This is documented alongside the `SeedAllocator` as a single reproducibility contract, and every dependency (Section 3) is pinned via a single committed lockfile so the software environment itself is reproducible, not only the RNG state.

Every `ExperimentRun` record stores `architecture_version`, `code_version` (git SHA), `config`, `seed_range_used`, and `calibration_artifact_hash` (if applicable) — enough to mechanically detect a figure quoted without its architecture-version caveat.

---

## 11. Stage 1 Implementation

```
Input:  observable vector (m, C, H) for one session
Step 1: profile out nuisance parameter p̂ = argmax_p L(m, C, H | p)   [scipy.optimize.minimize_scalar, bounded 0≤p≤0.5]
Step 2: compute joint mutual-consistency statistic
        T = -2 log[ L(data | p̂) / L(data | saturated/unconstrained model) ]
Step 3: compare T against a critical value obtained by SIMULATION at the actual operating n≈200
        (not a chi-square table lookup)
Output: MODEL_VALID / MODEL_INVALID, plus {p̂, T, calibrated_p_value, n}
```

This computes joint consistency of `m`, `C`, `H` against the single-parameter depolarizing relationship — never treating the three as independent likelihood contributions to be simply multiplied, which would silently reintroduce the redundancy the frozen architecture's evidence-reframing corrected. The forensic log records `{p̂, T, calibrated_p_value}` for every session, MODEL_VALID or not.

**Optimizer-failure handling is an explicit, first-class outcome, not an implicit assumption:**

```text
if optimizer.result.success is False:
    route to MODEL_INVALID
    reason = "optimizer_non_convergence"
    log the full optimizer result for later inspection

elif p̂ lands exactly at a bound (p̂ == 0 or p̂ == 0.5):
    this is a legitimate boundary optimum, not a failure
    log it distinctly, continue normal Stage 1/2 routing
```

A boundary optimum and a non-convergent optimizer failure are distinguishable outcomes with distinct log entries; neither is silently treated as the other.

---

## 12. Stage 2 Implementation

```
Input:  Stage-1-routed observable stream
S_SPRT  = sequential log-likelihood-ratio statistic, evaluated up to its own stopping point or a sample cap
S_gate  = block-wise Pearson chi-square structural statistic
Decision = (S_SPRT, S_gate) ∈ R,  R loaded from the frozen CalibrationArtifact (never computed at runtime)
Output: ACCEPT / FLAG(REJECT) / FLAG(INVESTIGATE), plus the fast provisional S_SPRT-only signal (non-authoritative,
        used only for demo/operational responsiveness — logged as provisional, never overwritten as final)
```

`R` is never recomputed inside the runtime path. The `CalibrationArtifact` loader raises a hard error if its content hash doesn't match the hash recorded at calibration time, preventing an accidentally stale or hand-edited artifact from being used silently. The artifact is loaded and verified exactly once, at API process startup (Section 11's runtime-flow description); re-verification happens only when explicitly deploying a new artifact version, never on a per-session basis.

---

## 13. GLR-CUSUM Implementation and Persistence

```
State (persisted, keyed by monitored channel/session-stream):
    window: deque[SessionSummaryStat], maxlen=w   — an in-memory cache, reconstructed at process
                                                     start from the persisted database state
    g_t, flagged, updated_at                       — persisted fields

Update, on EVERY session regardless of Stage 2 outcome:
    window.append(new_session_summary)
    G_t = max over window start-points s of:
              sup_θ  Σ_{r=s+1}^{t} log[ f_θ(X_r) / f_θ0(X_r) ]
          # for this Bernoulli/binomial-type family, sup_θ is closed-form: θ̂ = sample proportion over [s+1, t]
    flagged = G_t > cusum_threshold   (threshold calibrated offline, same discipline as Stage 2)

Complexity: O(w) elementary operations per update — closed-form MLE, no numerical optimization loop.
```

**Persistence — exactly one source of truth.** CUSUM state (`window`, `g_t`, `flagged`, `updated_at`) is persisted in a single SQLite table, `CusumState`, keyed by `stream_id`, updated transactionally every session. There is no separate JSON/NPZ flat-file store for this state. The in-memory `deque` is a cache reconstructed from the `CusumState` row at process start; SQLite's transactional row updates give crash-safe, atomic read-modify-write semantics for exactly this access pattern with no hand-rolled serialization. This is what makes low-and-slow detection latency reproducible across a service restart: there is only one place the state can be, so there is no ambiguity about which value is current after a crash.

```
Low-and-slow simulation: attacks.low_and_slow.LowAndSlowAttack biases the noise parameter by a small,
             sub-Stage-2-threshold amount, applied consistently across a configurable number of consecutive
             sessions — the harness runs this condition at multiple magnitudes to trace the detection-latency curve.
```

---

## 14. Attack Simulation Framework

Every attack is a pure transform on the session-construction pipeline via a shared interface, never a special case inside `qds/` or `qsentinel_monitor/` core logic:

```
attacks/
  base.py                          AttackStrategy(ABC): apply(builder) -> builder
  forgery.py                       CleanForgeryAttack, SubThresholdForgeryAttack
  replay.py                        ReplayAttack
  impersonation.py                 ImpersonationAttack   (invalid/expired/missing token — in-scope only)
  unauthorized_verification.py     UnauthorizedVerificationAttack
  channel_manipulation.py          InterceptResendAttack, PauliStructuredAttack, StructuralBurstAttack
  low_and_slow.py                  LowAndSlowAttack(magnitude, n_sessions)
```

`UnauthorizedVerificationAttack` constructs a verification request carrying an authorization-scope token that is syntactically valid but not within the requesting party's granted scope; the test asserts `FSM` produces a deterministic protocol-level REJECT. This is the seventh Monte Carlo condition and a first-class, named module.

Each attack strategy is independently unit-testable against a clean session fixture, and independently selectable in the Monte Carlo harness config — no attack strategy has special-cased handling anywhere in `qds/` or `qsentinel_monitor/` core.

---

## 15. Experimentation Architecture

```
experiments/
  config.py                ExperimentConfig (Pydantic v2): seed_range, architecture_version,
                            code_version (git SHA), noise_params, attack_params, n_trials,
                            calibration_artifact_hash
  seed_allocator.py         SeedAllocator — disjoint CALIBRATION / VALIDATION / EVALUATION ranges,
                            raises on out-of-range seed use
  calibration.py            → produces CalibrationArtifact                    [CALIBRATION]
  harness.py                → 7-condition Monte Carlo, spawn-context pool     [EVALUATION]
  verification_accuracy.py  → legitimate-session sweep                        [EVALUATION]
  stage1_small_sample.py    → critical-value simulation                      [VALIDATION]
  ablation.py                → full-architecture-minus-one                    [EVALUATION]
  naive_baseline.py          → independently-thresholded OR comparator        [EVALUATION, comparator]
  alpha_spending_baseline.py → group-sequential comparison                    [EVALUATION, comparator]
  complexity_benchmark.py    → wall-clock per module                          [VALIDATION]
  latency_benchmark.py       → end-to-end session latency                     [VALIDATION]
```

CALIBRATION / VALIDATION / EVALUATION separation is enforced by directory convention plus the `SeedAllocator` (Section 10): calibration code can only request seeds from the calibration range; validation and evaluation code can only request seeds from disjoint, separately-tracked ranges. Every `ExperimentRun` record stores enough metadata (Section 10) to mechanically detect a figure quoted without its architecture-version caveat.

Experiments are plain library functions invoked by CLI scripts, executed via `multiprocessing.Pool` with the explicit `spawn` context (Section 10) — no generic experiment framework, no task-queue abstraction, no API-triggered async job system. This is proportionate to a single-machine, single-operator workload with no durability requirement across process restarts beyond what raw result files on disk already provide.

---

## 16. Detailed Repository Structure

```
qsentinel-system/                       # repository root — distinct from both qds/ and
│                                        # qsentinel_monitor/; never ambiguous with a package name
├── qds/                                 # PROTOCOL — authoritative, owns ProtocolDecision
│   ├── __init__.py
│   ├── bell_pair.py
│   ├── pauli.py
│   ├── teleportation.py
│   ├── measurement.py
│   ├── noise.py
│   ├── protocol.py                     # QS-L verify(), s_a < s_v threshold rule
│   └── transcript.py                   # SessionTranscript, ProtocolDecision — both frozen
│
├── qsentinel_monitor/                  # MONITOR — advisory only, imports qds/ read-only,
│   │                                    # never imported by it
│   ├── __init__.py
│   ├── protocol_evidence/
│   │   ├── fsm.py
│   │   ├── freshness.py
│   │   └── authorization.py
│   ├── quantum_evidence/
│   │   ├── collector.py
│   │   ├── stage1.py                   # explicit optimizer-success handling
│   │   ├── stage2.py
│   │   ├── glr_cusum.py                # reads/writes CusumState via db/, sole persistence store
│   │   └── attribution.py
│   ├── orchestrator.py
│   └── forensic_log.py                 # Ed25519 signing, torn-write-tolerant verification
│
├── attacks/
│   ├── base.py
│   ├── forgery.py
│   ├── replay.py
│   ├── impersonation.py
│   ├── unauthorized_verification.py
│   ├── channel_manipulation.py
│   └── low_and_slow.py
│
├── experiments/
│   ├── config.py
│   ├── seed_allocator.py
│   ├── calibration.py
│   ├── harness.py                      # multiprocessing.get_context("spawn") pinned
│   ├── verification_accuracy.py
│   ├── stage1_small_sample.py
│   ├── ablation.py
│   ├── naive_baseline.py
│   ├── alpha_spending_baseline.py
│   ├── complexity_benchmark.py
│   └── latency_benchmark.py
│
├── artifacts/                          # generated, gitignored except .gitkeep
│   ├── calibration/{version}/artifact.json + region.npz   # version-keyed directories,
│   │                                                        # embedded content hash, hash
│   │                                                        # verified on load
│   └── experiment_runs/{run_id}/results.npz + metadata.json
│
├── db/
│   ├── models.py                       # SQLAlchemy models; CusumState is the sole CUSUM store
│   ├── migrations/                     # Alembic, render_as_batch=True set for SQLite
│   └── session.py
│
├── forensic_store/                     # append-only hash-chained log, flat files, never edited
│   └── chain_{date}.jsonl              # the authoritative forensic record
│
├── api/
│   ├── main.py                         # loads and verifies CalibrationArtifact once at startup
│   ├── routes/
│   │   ├── session.py                  # includes GET /sessions/{id}/stream (SSE)
│   │   ├── experiment.py
│   │   ├── forensic.py
│   │   └── cusum.py
│   └── schemas.py                      # Pydantic v2 request/response models
│
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   │   ├── LiveSession.tsx
│   │   │   ├── AttackLab.tsx
│   │   │   ├── ForensicLog.tsx
│   │   │   └── ExperimentDashboard.tsx
│   │   └── components/
│   │       ├── ProtocolDecisionPanel.tsx    # visually distinct from —
│   │       ├── MonitoringDecisionPanel.tsx  # — this, on purpose
│   │       ├── QuantumEvidenceView.tsx
│   │       └── CusumDriftChart.tsx
│   └── package.json
│
├── tests/
│   ├── unit/          (mirrors qds/, qsentinel_monitor/, attacks/ structure)
│   ├── integration/
│   ├── statistical/
│   └── regression/
│
├── docs/
│   └── QSENTINEL_Final_Architecture_Freeze.md   # the governing conceptual spec, checked into the repo
│
├── .import-linter.cfg                  # enforces qds ⇏ qsentinel_monitor (Section 6)
├── pyproject.toml
├── requirements.lock                   # (or poetry.lock — one lockfile mechanism, chosen once)
└── README.md
```

**Monorepo justified** because protocol, monitor, attacks, experiments, API, and frontend are developed by one small team, deployed together, and versioned together for the demo; nothing here has an independent release cadence that would justify repo-splitting overhead.

---

## 17. Persistence Model

**Runtime session data (SQLite):**

| Model | Key fields | Relationships |
|---|---|---|
| `Session` | id, created_at, config_id | 1—1 `SessionTranscript`, 1—1 `ProtocolDecision`, 1—1 `MonitoringDecision` |
| `SessionTranscript` | session_id, raw_measurement_data (blob ref), fsm_check_log | belongs to `Session` |
| `ProtocolDecision` | session_id, accepted (bool), reason | belongs to `Session` — **never** references `MonitoringDecision` |
| `MonitoringDecision` | session_id, verdict (ACCEPT/FLAG_REJECT/FLAG_INVESTIGATE/MODEL_INVALID), advisory (always True) | belongs to `Session`; references `Stage1Result`, `Stage2Result`, `CusumState` snapshot |
| `QuantumEvidence` | session_id, m, C, H, pauli_consistency | belongs to `Session` |
| `Stage1Result` | session_id, p_hat, statistic, p_value, model_valid (bool), failure_reason (nullable) | belongs to `Session` |
| `Stage2Result` | session_id, s_sprt, s_gate, in_rejection_region (bool), calibration_artifact_id | belongs to `Session`, references `CalibrationArtifact` |
| `CusumState` | stream_id, window (serialized), g_t, flagged (bool), updated_at | **sole persistence store for CUSUM state**; one row per monitored stream, updated every session |
| `ForensicRecord` | id, session_id, record_hash, prev_hash, payload (json), signature | queryable index/metadata only — **not** the authoritative chain (Section 18) |

**Offline experimental data (SQLite):**

| Model | Key fields | Relationships |
|---|---|---|
| `AttackConfiguration` | id, attack_type, params (json) | referenced by `ExperimentRun` |
| `Experiment` | id, name, category (CALIBRATION/VALIDATION/EVALUATION) | 1—many `ExperimentRun` |
| `ExperimentRun` | id, experiment_id, architecture_version, code_version, seed_range, config, calibration_artifact_hash, metrics (json) | belongs to `Experiment`; references `CalibrationArtifact` (nullable) |
| `CalibrationArtifact` | id, version, content_hash, rejection_region (npz ref), created_from_seed_range | referenced by `Stage2Result`, `ExperimentRun` |

The runtime/offline separation is structural: runtime tables are written by the API path during a live demo session; offline tables are written only by `experiments/` scripts, never by `api/`. `Stage2Result.calibration_artifact_id` is the only foreign key crossing the boundary, and it is read-only from the runtime side.

**Filesystem (large, write-once, content-addressed via an embedded hash — not a hash-based filename):**

```text
artifacts/calibration/{version}/artifact.json + region.npz
artifacts/experiment_runs/{run_id}/results.npz + metadata.json
```

Each `artifact.json` embeds a `content_hash` field, verified against the actual file contents on load; the directory is keyed by version for human navigability, and integrity comes from the embedded hash, not from the path.

---

## 18. Forensic Logging and Integrity Model

**Authoritative source of truth:** the append-only, hash-chained JSONL file under `forensic_store/`. SQLite `ForensicRecord` rows are a queryable index and metadata layer over this file — never a replacement for it, and never treated as authoritative on their own.

**Hash chain:** `hashlib.sha256`, computed as `hash(record_i) = SHA256(record_i || hash(record_{i-1}))`. Stdlib only, no new dependency.

**Signatures:** Ed25519, via the `cryptography` package. One keypair per deployment. The private key is loaded from a filesystem path configured through an environment variable, is never committed to source control, and is never stored in SQLite.

**Stated security scope:** the signed hash chain protects the forensic record against external, post-hoc tampering — someone editing the log file after the fact without access to the signing key cannot produce a valid chain. It does not protect against an attacker who has already compromised the API process itself and can therefore use the signing key to sign a rewritten chain. This is a named limitation, not an implied guarantee.

**Torn-write recovery.** A process crash or power loss mid-write can leave a partially-written final line. Verification distinguishes two cases:

```text
Malformed trailing final line
    → treat as an incomplete, in-flight write
    → log a warning
    → ignore the trailing fragment
    → verify the hash chain up to the last complete, hash-linked record

Malformed non-trailing record
    → genuine corruption
    → fail verification
```

This is what makes the forensic log the file most trustworthy to inspect after an incident, rather than the most fragile: a torn write on the last line — the only line a crash can plausibly corrupt — never invalidates the rest of the chain.

---

## 19. API Architecture

| Endpoint | Method | Purpose |
|---|---|---|
| `/sessions` | POST | Run a new session (optionally with an attack config attached); returns `ProtocolDecision` + `MonitoringDecision` separately |
| `/sessions/{id}` | GET | Retrieve full session detail: transcript, protocol decision, monitoring decision, evidence |
| `/sessions/{id}/protocol-decision` | GET | Protocol decision only |
| `/sessions/{id}/monitoring-decision` | GET | Monitoring decision only |
| `/sessions/{id}/quantum-evidence` | GET | Observable vector, Stage 1/2 detail |
| `/sessions/{id}/stream` | GET (SSE) | One-directional live stream of session-execution stages (Bell-pair distribution → teleportation → correction → measurement → verification) for the Live Session demo screen |
| `/attacks` | GET | List available attack strategies and their configurable parameters |
| `/cusum/{stream_id}` | GET | Current CUSUM window state and history, for the drift chart |
| `/experiments` | POST | Launch an experiment run (category, config) |
| `/experiments/{id}` | GET | Experiment run status/progress |
| `/experiments/{id}/results` | GET | Metrics, once complete |
| `/forensic/{session_id}` | GET | Forensic chain entries for a session |
| `/forensic/verify` | GET | Verify hash-chain integrity end-to-end (torn-trailing-write tolerant) |

Kept intentionally small and demo-oriented — no auth/multi-tenant endpoints, no generic CRUD beyond what a live demo and an experiment operator actually need. The `/sessions/{id}/stream` SSE endpoint is the one place FastAPI's async capability is actually exercised, matching the technology's justification in Section 3 exactly — no unused capability is claimed.

`CalibrationArtifact` is loaded and content-hash-verified exactly once, in `api/main.py` at process startup, and kept in memory as a verified singleton for the process lifetime; it is never re-read or re-hashed per request.

---

## 20. Frontend / Demo Design

**Recommended screens, in the order they tell the demo narrative:**

1. **Live Session** — runs one legitimate session end-to-end, animating Bell-pair distribution → teleportation → Pauli correction → projective measurement → protocol verification (driven by the `/sessions/{id}/stream` SSE endpoint), ending in a clearly labeled **PROTOCOL: ACCEPTED** panel.
2. **Attack Lab** — lets the presenter pick an attack strategy and parameters, re-runs the session, and shows two panels side-by-side, deliberately styled differently (a solid-bordered "Protocol Decision" card vs. a dashed-bordered "QSENTINEL Advisory" card) so the authoritative/advisory distinction is visually unmistakable, not just textually stated.
3. **Quantum Evidence View** — shows `m, C, H`, Pauli-correction consistency, and Stage 1's mutual-consistency statistic, with a plain-language annotation of why `C` and `H` are shown as *consistency checks*, not independent features.
4. **CUSUM Drift Chart** — runs a low-and-slow attack across many sessions live, plotting `G_t` climbing toward the threshold over session count, to make the cross-session detection story visible rather than asserted.
5. **Forensic Log** — shows the hash-chained record for the demo session, with a one-click "verify chain integrity" action that surfaces the torn-write-tolerant verification result.

No component beyond these five is built — a settings/admin screen, user-management UI, or generic experiment-config builder would not improve the specific narrative this system needs to tell.

---

## 21. Testing Strategy

**Unit tests** (`tests/unit/`, mirrors package structure): Bell-pair fidelity, teleportation correctness (fixed random seed → expected corrected state), each Pauli correction case, measurement statistics under zero noise, FSM transition table (every valid/invalid transition), freshness/token expiry edge cases, closed-form statistic formulas (`m`, `C`, `H` against hand-computed values), profile-likelihood optimizer convergence and non-convergence handling, hash-chain append/verify including a simulated torn trailing write, immutability of `ProtocolDecision`/`SessionTranscript` (an attempted mutation must raise).

**Integration tests** (`tests/integration/`): one full legitimate session end-to-end (protocol ACCEPT + monitor ACCEPT); one integration test per attack strategy (asserts the *correct* detector fires — FSM for replay/impersonation/unauthorized-verification, Stage 2 for channel manipulation, GLR-CUSUM for low-and-slow); full detector pipeline order (FSM → Stage1 → Stage2 → CUSUM → optional attribution); forensic pipeline writes a valid, verifiable chain entry for every outcome including protocol-accepted-but-monitor-flagged sessions; CUSUM state correctly reconstructed from `CusumState` after a simulated process restart.

**Statistical validation tests** (`tests/statistical/`): Type I error rate of Stage 2 empirically matches `α_system` within Monte Carlo tolerance on a held-out sample; detection rate on each of the 7 attack conditions; calibration/evaluation seed-range non-overlap, asserted programmatically via `SeedAllocator`; confidence intervals computed and non-empty on every headline metric.

**Regression tests** (`tests/regression/`): any change to `architecture_version` invalidates cached `CalibrationArtifact` usage (loading an artifact whose version doesn't match the running code's declared architecture version raises, not warns); historical `ExperimentRun` records remain queryable and correctly tagged by the architecture version they were produced under, so a stale figure can never be silently presented as current.

**CI enforcement:** import-linting (Section 6), `pytest` across all four tiers, a dedicated check that no file under `qds/` imports anything from `qsentinel_monitor/`, and a Monte Carlo reproducibility check confirming identical results across the pinned `spawn` context regardless of the CI runner's OS.

---

## 22. Deployment Model

Single-machine deployment: FastAPI served via `uvicorn`, SQLite file alongside the application, React build served as static files via the same FastAPI process using `StaticFiles`, avoiding a second server for the demo. No containerization is required for a judged demo running on presenter hardware, but a minimal `Dockerfile` + `docker-compose.yml` (API + static frontend, SQLite volume-mounted) is included as a portability convenience between team members' laptops — not for production scaling, which is explicitly out of scope. Dependency reproducibility across environments comes from the committed lockfile (Section 3/10), not from containerization.

---

## 23. Phased Implementation Roadmap

| Phase | Objective | Deliverables | Dependencies | Tests required | Completion criteria |
|---|---|---|---|---|---|
| 0 | Repo + reproducibility foundation | Repo skeleton (`qsentinel-system/` root), `pyproject.toml`, lockfile, `SeedAllocator`, CI skeleton with import-linter contract targeting `qsentinel_monitor` | — | Import-linter passes on empty skeleton | `pip install -e .` + `pytest` (no tests yet) run clean; import-linter contract references correct package names |
| 1 | Minimal teleportation-QDS protocol | `qds/` complete: Bell pair, teleportation, Pauli correction, measurement, noise, `protocol.py` verify(); `ProtocolDecision`/`SessionTranscript` defined as frozen types from the start | Phase 0 | Unit tests for all `qds/` modules | Deterministic acceptance on noiseless legitimate sessions; attempted mutation of a frozen object raises |
| 2 | Deterministic FSM protection | `fsm.py`, `freshness.py`, `authorization.py` | Phase 1 | FSM transition-table unit tests | Replay/impersonation/unauthorized-verification transcripts deterministically rejected by FSM in isolation |
| 3 | Quantum evidence extraction | `collector.py` | Phase 1 | Closed-form statistic unit tests | `m, C, H`, Pauli-consistency computed and match hand-derived values on fixtures |
| 4 | Attack injection framework | `attacks/` complete, all 7 strategies | Phases 1–3 | Unit test per strategy on a clean-session fixture | Each attack strategy demonstrably alters the transcript in its intended, isolated way |
| 5 | Stage 1 | `stage1.py` with explicit optimizer-success/failure handling | Phase 3 | Unit + statistical validation (small-sample simulation) | MODEL_VALID/INVALID routing correct on synthetic drift and synthetic attack fixtures; optimizer non-convergence is a distinct, logged outcome |
| 6 | Stage 2 + offline calibration | `stage2.py`, `experiments/calibration.py`, `CalibrationArtifact` | Phases 4, 5 | Seed-range separation test, Type I error validation | Calibration artifact produced from a properly held-out sample; Type I error empirically within tolerance of `α_system` on a disjoint evaluation sample |
| 7 | GLR-CUSUM | `glr_cusum.py`, `CusumState` as the sole persistence store | Phase 4 | Low-and-slow integration test, closed-form-`sup_θ` unit test, restart-reconstruction test | Detects a simulated low-and-slow condition within a bounded session-count latency, unconditionally ingesting every session; state survives a process restart with no second, disagreeing store |
| 8 | Forensics + non-interference guarantee | `forensic_log.py` (Ed25519 signing, torn-write-tolerant verify), hash chain, `test_non_interference.py`, `orchestrator.py` | Phases 2, 5–7 | Hash-chain verify test (including a simulated torn trailing write), non-interference test across all 7 attack conditions | Protocol decision provably identical whether or not the monitor runs; forensic chain verifiable, tolerating an in-flight torn write on the last line |
| 9 | Experiment harness | `harness.py` (spawn context pinned), `ablation.py`, `verification_accuracy.py`, `naive_baseline.py` | Phase 8 | Full 7-condition run, ablation run, verification-accuracy sweep | Every experiment category produces a stored, versioned `ExperimentRun`, reproducible across machines |
| 10 | API | `api/` complete, `CalibrationArtifact` loaded once at startup, `/sessions/{id}/stream` SSE endpoint | Phase 9 | API integration tests | All endpoints functional against the full pipeline |
| 11 | Frontend | `frontend/` five screens | Phase 10 | Manual demo walkthrough | Full narrative runnable live, protocol/advisory visually distinct |
| 12 | Validation and optimization | Alpha-spending baseline, complexity/latency benchmarks, confidence intervals on all headline figures, one full Alembic batch-mode migration exercised end-to-end | Phase 9 | Full statistical test tier | All outstanding validation tasks closed or explicitly re-confirmed as still-open with a stated reason |

**Ordering rationale:** the demo-critical path (Phases 0–11) never requires an attack to exist before its detector does — Phase 4 (attacks) is placed after Phases 1–3 build the clean pipeline those attacks target, and detector phases (5–8) precede the harness phase (9) that exercises them at scale. Phase 12 (naive baseline, alpha-spending comparison, wall-clock benchmarking) is placed last because it is comparator/validation work that strengthens claims but is not required for the pipeline itself to function or demo correctly. Phases 0–9 alone already satisfy the full detection/monitoring pipeline as a runnable, testable, non-visual system; frontend and comparator-baseline work are the lowest-risk phases to cut or defer if time is short.

---

## 24. Final "DO NOT BUILD" List

- No message queue / task broker (Celery, Redis, RabbitMQ) — the Monte Carlo harness is a single-machine, embarrassingly parallel workload.
- No Qiskit, Cirq, or PennyLane — the protocol never exceeds 3 qubits; a general circuit SDK adds overhead and opacity, not capability.
- No PyMC, Stan, or any Bayesian inference engine — nothing in Stage 1/2/CUSUM requires MCMC.
- No PostgreSQL, MongoDB, or other database beyond SQLite + flat files, until an actual multi-writer or multi-machine need is demonstrated.
- No authentication/user-management system for the demo API — out of scope for a judged hackathon deliverable.
- No generic admin/CRUD screens beyond the five demo screens in Section 20.
- No AI/ML library of any kind, anywhere.
- No general N-qubit circuit engine — the fixed 3-qubit statevector is sufficient and faster.
- No second persistence mechanism for GLR-CUSUM state — `CusumState` in SQLite is the sole store.
- No WebSocket bidirectional channel — the one live-streaming need (session animation) is one-directional and served by Server-Sent Events, not a heavier bidirectional protocol.
- No structural change to the frozen pipeline shape (FSM → Stage 1 → Stage 2 → CUSUM → optional Attribution) — no implementation detail encountered during this planning pass constitutes a genuine implementation impossibility or internal contradiction that would justify revisiting it.

---

## 25. Final Implementation Invariants / Acceptance Criteria

The following must hold for the implementation to be considered complete and correct, independent of which phase produced them:

1. **Non-interference is structural.** `ProtocolDecision` and `SessionTranscript` are frozen types; `qds/` has zero imports from `qsentinel_monitor/`, `api/`, `experiments/`, `attacks/`, or `frontend/`, enforced by a CI-failing `import-linter` contract; `test_non_interference.py` passes across all 7 attack conditions.
2. **Calibration never leaks into evaluation.** `SeedAllocator` raises on any out-of-range seed request; every `ExperimentRun` logs its `calibration_artifact_hash`; the `CalibrationArtifact` loader raises on a content-hash mismatch.
3. **Runtime never calibrates.** Stage 1's critical value and Stage 2's rejection region are loaded from a frozen artifact, verified once at process startup, and never recomputed or refit on the runtime path.
4. **CUSUM state has exactly one persisted home.** `CusumState` in SQLite; the in-memory `deque` is a cache, not a second store; state is correctly reconstructed after a process restart.
5. **The forensic chain is authoritative and crash-tolerant.** `forensic_store/chain_{date}.jsonl` is the source of truth; SQLite `ForensicRecord` rows are an index only; verification tolerates a torn trailing write without failing the whole chain, and fails loudly on any non-trailing corruption.
6. **Reproducibility is mechanical, not aspirational.** All Monte Carlo pools use an explicit `spawn` context; all dependencies are pinned via a single committed lockfile; every `ExperimentRun` records enough metadata (architecture version, code version, seed range, calibration artifact hash) to detect a stale figure quoted out of context.
7. **Statistical failure modes are explicit outcomes.** Stage 1 optimizer non-convergence routes to a distinctly-reasoned MODEL_INVALID; a boundary optimum is logged but is not treated as a failure.
8. **No technology justification is dangling.** Every capability named as a reason for choosing a technology (e.g., FastAPI's async support for `/sessions/{id}/stream`) corresponds to an implemented, exercised code path.
9. **Nothing on the DO-NOT-BUILD list (Section 24) is present**, and nothing outside the technology stack in Section 3 is introduced without a genuine implementation impossibility, internal contradiction, correctness flaw, or security flaw discovered during the phases in Section 23.

The technical implementation described in this document is frozen. No architectural or technology-stack changes should be made unless a genuine implementation contradiction, a measured performance bottleneck, a security issue, or a validated statistical flaw is discovered during the build described in Section 23.
