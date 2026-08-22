# Quantum-Inspired Cyber Threat Detection for Teleportation-Based QDS
## Flagship Architecture Document — SIH Problem Statement Response

---

## 1. Executive Summary

Most teams entering this problem statement will build a Qiskit teleportation demo with a pass/fail signature checker and a dashboard that flashes "Attack Detected" when a fixed threshold is crossed. That solution technically satisfies the brief but demonstrates no security engineering depth — it is a simulator with a UI, not a threat-detection *architecture*.

The proposal below — **QSENTINEL** (Quantum Signature Entinel: protocol-Native Threat Intelligence, Ledger and Attribution) — reframes the problem from "verify a signature" to "continuously validate an entire protocol execution against its own mathematically-derived expected trajectory, convert any deviation into an evidence-backed threat fingerprint, and attribute the most statistically likely attack — all without machine learning." This produces something evaluators haven't seen ten versions of already: a protocol-state-integrity engine plus a forensic attribution layer sitting on top of the physics simulation, rather than the physics simulation *being* the product.

---

## 2. What Most Teams Will Build (Hostile Analysis)

| Obvious Approach | Why It's Generic | Why It's Easy to Copy | Where It Fails Under Questioning |
|---|---|---|---|
| Basic Qiskit BB84/teleportation demo | Tutorial-level; exists in every Qiskit textbook | Copy-paste from Qiskit docs in an hour | "What's novel here?" has no answer |
| Signature check via measurement comparison | Single binary check, no depth | Trivial `if a==b` logic | Can't explain *why* mismatch occurred |
| Fixed-threshold attack detection | Arbitrary constant, no statistical basis | One `if` statement | Judges ask "why 5%? why not 3%?" — no defensible answer |
| Simple timestamp-based replay protection | Classical trick with no quantum tie-in | Standard nonce/timestamp pattern | Doesn't use any of the required quantum principles |
| Circuit visualization only | Cosmetic, not a security feature | Any Qiskit visualization call | Adds no detection or forensic value |
| "Attack Detected" banner | Zero explainability | Trivial boolean flag | Judges ask "detected how, with what confidence, which attack?" — silence |

**What we must NOT build:** a linear pipeline that ends at a boolean. Any solution whose entire security logic collapses to "compare two numbers, print red/green" will be indistinguishable from every other team's submission and will lose on defensibility the moment a judge asks "how confident are you, and why?"

---

## 3. Innovation Opportunities (Selected Directions)

We pursued Directions A, B, C, D, E, F, G from the brief and evaluated all seven for genuine security payoff versus buzzword risk. Four survived scrutiny as load-bearing; three were folded in as supporting mechanisms rather than standalone modules (to avoid overengineering an SIH prototype).

**Kept as core modules:**
- **A — Threat Fingerprinting**: different attacks genuinely do leave different statistical footprints on Bell correlation, mismatch rate, and entropy — this is real physics/statistics, not decoration.
- **B — Protocol-Aware Zero Trust (state machine)**: catches attacks invisible to measurement statistics alone (reordering, replay, skipped steps) — this is the single most defensible novelty because it doesn't depend on quantum hardware fidelity at all.
- **C — Forensics/attribution reporting**: turns a boolean into an evidence-backed narrative, which is what actually impresses judges in a live demo.
- **E — Adaptive statistical thresholds (SPRT/binomial)**: replaces the "arbitrary constant" every other team will use with a defensible, noise-aware decision boundary.

**Folded in as supporting logic, not separate modules:**
- **D — Multi-layer detection**: absorbed into the pipeline design (each engine below *is* a layer) rather than presented as its own module — avoids padding the architecture with relabeled duplicates.
- **F — Digital Twin**: implemented as the mathematical backbone that Fingerprinting and Attribution both consume (expected-vs-observed), not as a separate UI-facing module — it's infrastructure, not a feature.
- **G — Attack attribution matrix**: merged into the Statistical Attribution Engine (C) as its mathematical core (likelihood ratios), rather than kept as a separate "matrix" gimmick.

---

## 4. Innovation Scoring Matrix

Scored 1–10 across Innovation, Feasibility, Cybersecurity Relevance, Demo Impact, Technical Depth, Copy-Resistance.

| Idea | Innovation | Feasibility | Cyber Relevance | Demo Impact | Tech Depth | Copy-Resistance | Include? |
|---|---|---|---|---|---|---|---|
| Threat Fingerprint Vector | 9 | 8 | 9 | 8 | 8 | 9 | ✅ Core |
| Protocol State Integrity Engine (FSM) | 9 | 9 | 10 | 7 | 8 | 9 | ✅ Core |
| Statistical Attribution Engine (likelihood ratios) | 8 | 8 | 9 | 8 | 9 | 9 | ✅ Core |
| Forensic Incident Report Generator | 7 | 9 | 8 | 9 | 6 | 7 | ✅ Core |
| Adaptive Threshold Engine (SPRT) | 8 | 8 | 8 | 6 | 8 | 8 | ✅ Core |
| Protocol Digital Twin (math model) | 8 | 7 | 8 | 6 | 8 | 8 | ✅ Infrastructure |
| Tamper-Evident Security Ledger (hash-chained log) | 6 | 9 | 7 | 6 | 5 | 6 | ✅ Supporting |
| Standalone "layered detection" UI module | 4 | 8 | 5 | 5 | 3 | 3 | ❌ Redundant with pipeline |
| Circuit visualization dashboard | 2 | 9 | 2 | 6 | 2 | 1 | ❌ Cosmetic only |
| ML-based anomaly classifier | 3 | 6 | 6 | 5 | 4 | 3 | ❌ Explicitly disallowed by brief |

---

## 5. Selected Flagship Innovation

**Core innovation statement:**

> Unlike conventional QDS simulators that answer only "valid or invalid," QSENTINEL treats every protocol run as a hypothesis test against a mathematically derived digital twin of the honest protocol. Deviations across quantum-state, protocol-state, and temporal dimensions are compressed into a Threat Fingerprint Vector, scored against per-attack likelihood models via sequential hypothesis testing, and resolved into a ranked, evidence-cited attribution — entirely without machine learning, using only quantum measurement statistics and formal protocol invariants.

---

## 6. Final Solution Name and Acronym

**QSENTINEL** — **Q**uantum **S**ignature **E**ntinel: protocol-**N**ative **T**hreat **I**ntelligence, **N**etwork **E**vidence & **L**edger

(A defensible, memorable name — "sentinel" signals defense/vigilance, and the backronym legitimately maps to the five functional pillars: threat intelligence, protocol-native detection, evidence generation, and a ledger.)

---

## 7. One-Line Innovation Statement

"QSENTINEL doesn't ask 'is this signature valid' — it asks 'does this entire protocol execution match its own mathematical twin,' and when it doesn't, it tells you which attack, at which step, with what confidence, and why."

**30-second pitch:** Every legitimate QDS run has a predictable statistical and procedural shape. QSENTINEL builds that shape mathematically before the run starts, watches the live measurements and protocol transitions against it in real time, and the moment reality diverges from the twin, converts the divergence into a threat fingerprint. That fingerprint is scored against likelihood models for eight known attack classes using sequential hypothesis testing — no AI, no black box, fully explainable — and the system outputs a ranked attribution with the exact evidence a judge (or a real incident responder) needs to trust the verdict.

**Why competitors are unlikely to build it:** it requires combining three disciplines most teams keep separate — quantum information theory (fidelity/correlation math), formal protocol state-machine security (borrowed from classical zero-trust/IDS design), and classical statistical hypothesis testing (SPRT, KL-divergence) — into one coherent pipeline, rather than picking one and stopping.

---

## 8. Problem Being Solved

Teleportation-based QDS protocols are secure *in theory* under ideal quantum mechanics, but any real or simulated deployment is vulnerable at the classical/procedural seams: correction operations can be tampered with, verification requests can be replayed or reordered, and channel noise can be weaponized to hide small persistent manipulations ("low-and-slow" attacks) beneath naive fixed thresholds. QSENTINEL closes this gap by monitoring both the quantum-statistical layer and the protocol-procedural layer simultaneously, and by replacing fixed thresholds with statistically justified, noise-aware decision boundaries.

---

## 9. System Architecture

```
                    Users / Applications
                            |
                            v
                     QDS Gateway (API)
                            |
                            v
         Teleportation & Bell-State Simulation Layer
             (Qiskit Aer — entanglement, teleport,
                 Pauli correction, measurement)
                            |
              +-------------+-------------+
              |                           |
              v                           v
   Quantum Protocol Digital Twin   Protocol State Integrity Engine
   (expected correlation, mismatch  (FSM: validates transition
    rate, entropy — derived from    sequence, timing, duplication,
    protocol math, not observed)    unauthorized transitions)
              |                           |
              +-------------+-------------+
                            |
                            v
              Deviation Engine (Observed vs Expected)
                            |
                            v
              Threat Fingerprint Generator
       [mismatch, correlation dev, entropy shift,
         Pauli anomaly, temporal inconsistency,
                verification mismatch rate]
                            |
                            v
         Adaptive Statistical Detection Engine
        (SPRT / binomial CI — noise vs adversary)
                            |
                            v
          Statistical Attack Attribution Engine
      (likelihood ratios across 8 attack models)
                            |
                            v
      Decision Engine: ACCEPT / REJECT / QUARANTINE / INVESTIGATE
                            |
                            v
        Forensic Incident Report + Tamper-Evident Ledger
                            |
                            v
              Real-Time Monitoring Dashboard
```

---

## 10. Detailed Module Explanation

**Teleportation & Bell-State Simulation Layer** — Input: signer/verifier key material, message hash. Output: entangled pair, teleported qubit state, classical correction bits, measurement outcomes. Responsibility: physically faithful simulation of the honest protocol using Qiskit Aer, providing ground-truth data for every downstream engine.

**Quantum Protocol Digital Twin** — Input: protocol parameters (expected fidelity, channel noise model). Output: expected correlation C_exp, expected mismatch rate m_exp, expected entropy H_exp — all derived analytically from Bell-state math and a declared noise model, *not* from a second simulation run. Responsibility: gives every other engine a principled "what should have happened" baseline instead of a magic number.

**Protocol State Integrity Engine** — Input: sequence and timing of protocol events. Output: pass/fail per transition, plus which invariant failed. Responsibility: models the QDS run as a finite-state machine (ENTANGLEMENT_CREATED → TELEPORTATION_INITIATED → BELL_MEASUREMENT → CLASSICAL_CORRECTION → SIGNATURE_COMMITMENT → VERIFICATION_REQUEST → MEASUREMENT_VALIDATION → ACCEPT/REJECT) and rejects any reorder, skip, duplication, or out-of-window replay — catching attacks that leave the measurement statistics untouched.

**Threat Fingerprint Generator** — Input: (observed − expected) across all measured quantities. Output: a 6-dimensional Threat Fingerprint Vector. Responsibility: compresses raw deviation into a compact, comparable signature.

**Adaptive Statistical Detection Engine** — Input: fingerprint vector, sample size, declared noise floor. Output: statistical significance verdict (noise-explainable vs adversarial) with confidence interval. Responsibility: replaces fixed thresholds with SPRT/binomial-CI-based decisions that adapt to sample size and expected noise, specifically to catch low-and-slow attacks that hide under a static threshold.

**Statistical Attack Attribution Engine** — Input: fingerprint vector + significance verdict. Output: ranked list of attack hypotheses with likelihood scores. Responsibility: computes P(evidence | attack model) for each of eight attack classes and ranks them — this is the "which attack, how confident" answer competitors can't produce.

**Decision Engine** — Combines integrity-engine verdict and attribution-engine verdict into one of four actions: ACCEPT, REJECT, QUARANTINE (suspicious, hold for review), INVESTIGATE (statistically ambiguous, log and monitor).

**Forensic Incident Report Generator** — Assembles a human-readable report: what happened, where, which invariant failed, which evidence supports it, which attacks are consistent with it, recommended response.

**Tamper-Evident Security Ledger** — Hash-chains every decision + evidence record so the audit trail itself cannot be silently altered after the fact (classical, cheap, high demo value).

---

## 11. Core Detection Pipeline

1. Simulate honest protocol run → 2. Compute twin (expected values) → 3. Inject or allow real execution (or attack simulation) → 4. Collect observed values → 5. Run FSM validation in parallel → 6. Compute deviation vector → 7. Run adaptive significance test → 8. If significant, run attribution → 9. Decision engine issues verdict → 10. Forensic report + ledger entry → 11. Dashboard update.

---

## 12. Mathematical Model

- **Measurement mismatch rate:** `m = mismatches / total_measurements`
- **Bell correlation deviation:** `D = |C_observed − C_expected|`, where `C = E[⟨A⊗B⟩]` over Bell-basis measurement pairs
- **Entropy shift:** `ΔH = |H_observed − H_expected|`, Shannon entropy over the measurement-outcome distribution
- **Statistical divergence:** Jensen-Shannon divergence (chosen over KL because it's symmetric and bounded [0,1], making cross-attack comparison meaningful): `JSD(P‖Q) = ½KL(P‖M) + ½KL(Q‖M)`, `M = ½(P+Q)`
- **Sequential detection:** Sequential Probability Ratio Test (SPRT) — appropriate because it minimizes expected sample size to reach a decision at fixed false-positive/false-negative rates, which directly defeats low-and-slow attacks that a fixed-N test would miss: accept H1 (attack) when `Λ_n = Π P(x_i|H1)/P(x_i|H0) ≥ (1−β)/α`
- **Attack likelihood score:** `L_attack = P(observed fingerprint | attack model)`, modeled per attack as a multivariate Gaussian or Beta-Binomial over the fingerprint dimensions, parameters derived from the attack's expected physical/procedural effect (e.g., Pauli-X manipulation predicts a specific anticorrelation flip in the Z-basis)
- **Attribution ranking:** normalize via `P(attack_i | evidence) ∝ L_i × Prior_i`, report top-ranked with likelihood ratio vs. legitimate-noise baseline
- **Composite decision:** no arbitrary weighted sum — decision is the SPRT verdict (statistically derived), not a hand-tuned score

---

## 13. Threat Model

| Attacker | Capability | Vector | Observable Evidence | Detection | Response |
|---|---|---|---|---|---|
| External attacker | Network access only | Injects malformed messages | Protocol transition violation | State Integrity Engine | REJECT |
| MITM | Intercepts classical channel | Alters correction bits | Correction-consistency failure | Fingerprint (Pauli anomaly dim) | QUARANTINE |
| Quantum channel attacker | Partial qubit access | Intercept-resend | Correlation collapse, entropy rise | Fingerprint + SPRT | REJECT |
| Insider | Legitimate credentials, malicious intent | Unauthorized verification request | Unauthorized transition attempt | State Integrity Engine | INVESTIGATE + ledger flag |
| Replay attacker | Captured prior valid transcript | Resends old signature | Timestamp/nonce + state-continuity violation | State Integrity Engine (temporal) | REJECT |
| Signature forger | No legitimate key material | Attempts to fabricate signature | High mismatch rate vs. twin | Attribution Engine (forgery model) | REJECT |
| Unauthorized verifier | No verification rights | Triggers verification without authorization | Authorization-invariant failure | State Integrity Engine | REJECT + ledger flag |
| Malicious quantum node | Controls entanglement source | Corrupts Bell-pair distribution | Bell correlation deviation from creation | Fingerprint (correlation dim) | QUARANTINE |

---

## 14. Attack Fingerprint Framework

`ThreatFingerprint = [D_correlation, m_mismatch, ΔH_entropy, PauliAnomaly, TemporalInconsistency, VerificationMismatchRate]`

Each attack class produces a characteristic region in this 6-D space (e.g., Pauli-X manipulation: high `D_correlation` + specific `PauliAnomaly` sign, low `TemporalInconsistency`; Replay: low `D_correlation` + high `TemporalInconsistency`; Low-and-slow: small per-round deviation but persistent bias detected only by SPRT's cumulative likelihood ratio, not a single-round threshold).

---

## 15. Statistical Attack Attribution

For each candidate attack `i`, compute `L_i = P(fingerprint | attack_i)` using attack-specific expected-effect models built from the physics of that attack (e.g., a Pauli-Z flip predicts a specific sign change in Z-basis correlation, not just "some deviation"). Rank by likelihood ratio against the legitimate-noise model; report the top hypothesis with its confidence and the specific fingerprint dimensions that drove the ranking — this is what makes the output "explainable" rather than a score with no justification.

---

## 16. Protocol State Integrity

FSM: `ENTANGLEMENT_CREATED → TELEPORTATION_INITIATED → BELL_MEASUREMENT → CLASSICAL_CORRECTION → SIGNATURE_COMMITMENT → VERIFICATION_REQUEST → MEASUREMENT_VALIDATION → ACCEPT/REJECT`. Each edge carries a cryptographic/timing invariant (valid predecessor state, freshness window, single-use token). Any skip, duplication, reorder, or replay fails the specific edge invariant — this catches attacks (replay, unauthorized verification, protocol reordering) that produce *no* measurement-statistics anomaly at all, which is the layer most competitor solutions entirely omit.

---

## 17. Quantum Protocol Digital Twin

The twin is not a second simulation — it is the closed-form expected value of correlation, mismatch rate, and entropy computed directly from Bell-state math and a declared depolarizing-noise model. This distinction matters for defensibility: judges will ask "is the twin just running the same simulation twice?" — the answer must be no, it's derived analytically, so a compromised simulator can't fool its own twin.

---

## 18. Adaptive Detection Without AI

SPRT accumulates evidence round-by-round and stops as soon as the likelihood ratio crosses a boundary set by chosen false-positive (α) and false-negative (β) rates — this is what lets QSENTINEL catch a low-and-slow attacker whose per-round deviation is individually invisible under any fixed threshold, while still bounding false positives from ordinary channel noise, entirely through classical statistics.

---

## 19. Attack Simulation Plan (summary — 10 attacks per brief)

Each of the ten attacks (forgery, replay, impersonation, Pauli manipulation, Bell-state manipulation, intercept-resend, measurement tampering, protocol reordering, unauthorized verification, low-and-slow) is implemented as a parameterized perturbation injected into the simulation layer, with its expected fingerprint region, triggering detection layer, and demo visualization pre-mapped — low-and-slow is the flagship demo attack because it's the one fixed-threshold competitor solutions will visibly fail to catch.

---

## 20. Security Incident Forensics

Report structure: (1) What happened — attack type + confidence; (2) Where — protocol stage; (3) Which invariant failed; (4) Supporting evidence — exact fingerprint values vs. expected; (5) Consistent alternative hypotheses ranked; (6) Recommended response (reject/quarantine/investigate) — all populated from the attribution and integrity engines, no manual narrative writing.

---

## 21. Demo Flow

Normal run (correlation ~0.97, mismatch <3%, STATUS: SECURE) → live Pauli-X injection → correlation collapses to ~0.48, mismatch 39%, JSD 0.62, state invariant intact but statistical divergence flagged → attribution engine ranks Pauli-X Manipulation at ~0.9 likelihood → decision: QUARANTINE → forensic report auto-generated → repeat with low-and-slow attack to show fixed-threshold failure vs. SPRT success — this contrast is the single most persuasive live moment in the demo.

---

## 22. Tech Stack

- **Quantum simulation:** Qiskit Aer, Qiskit Quantum Information, NumPy
- **Security/statistics engine:** Python, SciPy (stats, hypothesis testing), custom SPRT implementation
- **Backend:** FastAPI
- **Data/ledger:** SQLite (hackathon scope) with hash-chained records for tamper evidence
- **Frontend:** React, with charting for correlation/entropy over time, fingerprint radar chart, protocol-state timeline, attack attribution bar chart

---

## 23. Delivery Table

| Deliverable | Description | Technology | Validation Metric | Demo Evidence |
|---|---|---|---|---|
| Teleportation simulator | Full teleportation circuit | Qiskit Aer | Fidelity vs. theoretical | Circuit + statevector output |
| Bell-state entanglement simulator | Entangled pair generation | Qiskit | Correlation ≈ theoretical max | Correlation plot |
| Pauli correction engine | Applies X/Z corrections | Qiskit gates | Correction accuracy | Before/after state |
| QDS sign/verify module | Full signature lifecycle | Custom protocol logic | Deterministic accept on legit sig | Accept/reject log |
| Protocol Digital Twin | Analytic expected-value model | NumPy/SciPy | Twin vs. simulated-honest-run agreement | Overlay chart |
| Threat Fingerprint Engine | 6-D deviation vector | NumPy | Separation between attack classes | Radar chart |
| Protocol State Integrity Engine | FSM validator | Custom | Detects 100% of reorder/replay/skip in test set | Timeline diagram |
| Adaptive Detection Engine | SPRT/CI-based decisions | SciPy stats | ROC vs. fixed threshold | Comparison chart |
| Attack Attribution Engine | Likelihood-ratio ranking | SciPy | Correct top-1 attribution rate | Ranked bar chart |
| Replay detection | Temporal/state continuity check | FSM logic | 100% detection in test set | Log entry |
| Unauthorized verification detection | Authorization-invariant check | FSM logic | 100% detection | Log entry |
| Attack simulation framework | 10 parameterized attack injectors | Python | Coverage of all 10 attacks | Simulation console |
| Forensic report generator | Structured incident report | Python templating | Report completeness checklist | Sample PDF/JSON report |
| Real-time dashboard | Live protocol + threat view | React | UX walkthrough | Live demo |
| Mathematical security analysis | Written derivations | Markdown/LaTeX | Peer-reviewable | Appendix document |
| Performance evaluation suite | Latency/throughput benchmarks | Python | ms/verification | Benchmark table |
| Test datasets | Labeled attack scenarios | Python fixtures | Coverage % | Test report |
| Deployment-ready framework | Packaged repo | Docker/FastAPI | Runs end-to-end | Live run |

---

## 24. Competitor Differentiation

| Generic Team Solution | Our Solution (QSENTINEL) |
|---|---|
| Signature validation only | Protocol-aware behavioral security |
| Fixed thresholds | Adaptive statistical evidence (SPRT) |
| "Attack detected" | Attack fingerprint + ranked attribution |
| Quantum circuit simulator | Full defense architecture around the simulator |
| Static verification | Protocol digital twin (expected vs. observed) |
| Generic logs | Tamper-evident forensic evidence |
| Pass/fail decision | Explainable, evidence-cited reasoning |
| Single-layer detection | Multi-layer (quantum + protocol + temporal) |
| No noise/adversary distinction | Explicit noise-vs-adversary hypothesis test |
| Can't catch low-and-slow attacks | SPRT explicitly designed to catch them |

---

## 25. Performance Metrics

- Deterministic acceptance rate of legitimate signatures (target: 100% in noiseless simulation)
- False-positive rate at declared α (e.g., ≤5%)
- False-negative rate at declared β (e.g., ≤5%)
- Mean detection latency (rounds-to-decision under SPRT vs. fixed-N)
- Top-1 attribution accuracy across the 10 simulated attacks
- End-to-end verification latency (ms)

---

## 26. Implementation Roadmap (hackathon timeline)

Day 1: Teleportation + Bell-state simulator, QDS sign/verify core. Day 2: Digital Twin math + Protocol State Integrity Engine (FSM). Day 3: Fingerprint Generator + Adaptive Detection Engine (SPRT). Day 4: Attribution Engine + Forensic Report Generator + Ledger. Day 5: Attack simulation suite (all 10), dashboard, performance benchmarking. Day 6: Demo rehearsal, hostile-review fixes, documentation.

---

## 27. Hostile Review

**Is this actually quantum-inspired or classical cybersecurity with quantum words?** Partially fair criticism — the Protocol State Integrity Engine is genuinely classical (a security FSM), and we should say so plainly rather than dress it up. Its justification is that it catches attacks the quantum-statistics layer structurally cannot see, which is a legitimate reason to include a classical component in a "quantum-inspired" system, not a weakness to hide.

**Is information-theoretic security being misrepresented?** Yes, risk exists — we must be precise that QSENTINEL is a *detection/monitoring* layer around the QDS protocol; it does not itself provide or degrade the protocol's information-theoretic guarantees, and the document must say this explicitly to avoid an indefensible claim.

**Does the digital twin actually add security value, or is it just "compute the expected value" relabeled?** Legitimate concern. Its real value is forcing an explicit, auditable statement of what "honest" looks like *before* observing data, which prevents post-hoc threshold tuning — this is the honest justification, not "it's a twin so it's advanced."

**Can fingerprints truly distinguish attacks, or do overlapping regions make attribution unreliable?** Some attack pairs (e.g., strong intercept-resend vs. strong Pauli manipulation) will produce overlapping fingerprints — the system must report attribution *uncertainty* honestly (ranked list with confidence, not a single confident guess) rather than overclaim precision.

**Is attribution mathematically defensible?** Yes, if likelihood models are derived from each attack's actual physical effect and validated against simulated ground truth — this must be shown in the demo (known-attack-in, correct-attribution-out) rather than asserted.

**Can this be implemented within SIH constraints?** Yes — everything here runs on Qiskit Aer simulation and classical statistics; no real quantum hardware or exotic libraries required.

**What if quantum hardware is unavailable?** Not a problem — the entire system is designed simulation-first; this should be stated as a feature (portability), not excused as a limitation.

**How do we distinguish legitimate noise from malicious interference?** This is the explicit job of the Adaptive Detection Engine — answer this directly and confidently in the pitch, since it's the question judges are most likely to ask.

---

## 28. Final Refined Architecture

Same as Section 9, with two additions made after hostile review: (1) an explicit "Assumptions & Non-Claims" panel in the dashboard stating the system does not alter the protocol's information-theoretic security, it monitors it; (2) attribution outputs always shown as a ranked list with confidence intervals, never a single unqualified verdict.

---

## 29. Top 5 SIH Judge Questions and Answers

1. **"Isn't this just classical anomaly detection with quantum vocabulary?"** — Partly, by design: the Protocol State Integrity layer is intentionally classical because certain attacks (replay, reordering) are invisible to quantum measurement statistics; the quantum-statistical layer (fingerprinting, twin, attribution) is where the physics genuinely does the work.
2. **"Why not just use ML for attack classification?"** — The brief explicitly excludes AI/ML as the primary mechanism; more importantly, likelihood-ratio attribution from physically-derived attack models is *more* explainable and auditable than a trained classifier, which matters for a security system.
3. **"How do you pick your thresholds?"** — We don't pick fixed thresholds; SPRT derives decision boundaries from chosen false-positive/false-negative rates, which is a statistically principled substitute.
4. **"What happens with real quantum noise instead of your simulated noise model?"** — The twin's noise model is a parameter, not a hardcoded constant, so it can be recalibrated to any declared channel noise profile without redesigning the pipeline.
5. **"Does this weaken or alter the QDS protocol's own security guarantees?"** — No — QSENTINEL is a monitoring/detection layer external to the protocol; it observes and reasons about protocol execution, it does not modify the cryptographic guarantees of the underlying QDS scheme.

---

## 30. Final Innovation Pitch

"Every team here will show you a signature that passes or fails. We're showing you a system that watches an entire quantum protocol execution against a mathematically derived twin of its honest self, turns any deviation into a six-dimensional threat fingerprint, and tells you — with statistical confidence, not a guess — which of eight known attacks most likely occurred, at which protocol stage, and why. No AI, no black box, no arbitrary threshold. Just physics, protocol invariants, and hypothesis testing, doing what a real security operations center would demand: not a verdict, but evidence."
