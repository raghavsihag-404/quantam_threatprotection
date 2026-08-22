# QSENTINEL: A Protocol-Aware Threat Detection Layer for Teleportation-Based Quantum Digital Signatures
## Final Consolidated Architecture & Experimental Validation

---

## 1. Executive Summary

QSENTINEL is an independent, protocol-aware security observability and threat-detection layer for teleportation-based Quantum Digital Signature (QDS) systems. It does not provide or replace the underlying QDS protocol's cryptographic guarantees — it watches a live protocol execution against a mathematically derived honest-execution baseline, fuses quantum-statistical evidence with protocol-state evidence, and produces evidence-backed decisions (ACCEPT / REJECT / QUARANTINE / INVESTIGATE / MODEL_INVALID) instead of a black-box verdict. Its architecture went through three design revisions and a hostile review before being subjected to actual Monte Carlo experimentation, and the resulting claims are narrower and more defensible than the initial design: **the architecture's measurable value over a simple protocol-invariants-plus-threshold baseline is concentrated in two specific places — detecting unmodeled/structural anomalies and detecting persistent low-level drift across sessions — not in the architecture broadly.** That narrowed, evidence-backed claim is the actual deliverable, not a marketing overstatement.

---

## 2. Problem Interpretation

**Asked for:** a detection/monitoring framework — not a new QDS protocol, not a quantum network platform, not an AI system — using quantum-native observables (Pauli eigenstates, Bell correlations, projective measurements) and classical statistics to detect forgery, impersonation, replay, channel manipulation, and unauthorized verification against a teleportation-based QDS system, while preserving its information-theoretic security guarantees.

**Not asked for:** a production QDS cryptosystem, a full QKD network, an ML classifier, a general SIEM, or a blockchain product.

---

## 3. Product Definition

**What it is:** an independent, protocol-aware security observability and threat-detection layer, external to the protected QDS protocol, consuming classical telemetry produced by that protocol's normal operation (never claiming to observe undisturbed quantum states) and protocol-event telemetry, producing decisions with evidence.

**What it is not:** the QDS protocol itself (it protects one — Section 4); a quantum simulator (used only as a test harness); a general IDS/SIEM (protocol-specific by design); an AI system; a source of cryptographic security (that comes from the QDS protocol; QSENTINEL adds detection value on top of it, explicitly not instead of it).

---

## 4. The Protected Protocol — Explicitly Grounded, Not Invented

The verification logic is inherited from the published **QS-L family** of Lamport-inspired, entanglement/raw-key-based QDS protocols: a three-party structure (Signer + two Recipients who cross-verify), per-message raw key pairs, recipient-side key symmetrization, and — the mechanism that actually provides non-repudiation — an **asymmetric mismatch-threshold verification rule**: the threshold `s_a` used when a recipient verifies a signature received directly from the Signer is stricter (lower) than the threshold `s_v` used when verifying a signature forwarded by the other recipient, which is what prevents the Signer from successfully repudiating a valid signature.

**Distribution mechanism (the one deliberate substitution, stated explicitly):** QS-L literature typically distributes raw keys via direct QKD-style transmission. Since the SIH brief specifically requires **teleportation-based** distribution, that step is replaced: the Signer prepares Pauli-eigenstate qubits, teleports them to each Recipient via pre-shared EPR pairs (Bell-basis measurement + classical correction bits over an authenticated channel), and each Recipient applies the Pauli correction and measures immediately (no long-term quantum memory required — BB84-style basis sifting).

**Explicit, load-bearing scope statement:**

> This is a monitoring-oriented protocol abstraction. Its verification logic (three-party structure, symmetrization, asymmetric thresholds) is inherited from the published QS-L family. Its distribution mechanism (teleportation via pre-shared Bell pairs) is an original substitution made to satisfy the SIH brief's teleportation requirement, and this substitution has not been independently security-proven. This framework does not claim to prove the security of the resulting composite protocol — it claims to detect deviations from its expected, specified behavior. Proving the security of the teleportation-substituted QS-L variant is future research, not an accomplished result.

**Protocol stages (the FSM the monitor watches):**
`ENTANGLEMENT_DISTRIBUTED → TELEPORTATION_INITIATED → BELL_MEASUREMENT → CLASSICAL_CORRECTION → KEY_SYMMETRIZATION → SIGNATURE_COMMITMENT → VERIFICATION_REQUEST → THRESHOLD_CHECK → ACCEPT/REJECT`

---

## 5. Threat Model

| Actor | Vector | Detection Path |
|---|---|---|
| External attacker | Injects/blocks classical messages | Protocol-state FSM |
| MITM on classical channel | Alters correction bits | Pauli-correction-consistency evidence |
| Quantum channel attacker | Intercept-resend, entanglement degradation | Correlation/entropy deviation |
| Replay attacker | Resubmits valid past transcript | FSM freshness/single-use invariant (zero quantum signature — protocol evidence is the *only* path) |
| Signature forger | No legitimate key material | Mismatch-rate deviation vs. reference model |
| Impersonator | False identity claim | Authorization-token check at FSM layer (explicit limitation: undetectable if the classical-channel authentication assumption itself is broken) |
| Insider | Valid credentials, unauthorized verification | FSM authorization-invariant |
| Compromised quantum node | Corrupts entanglement source | Partially visible via correlation deviation; explicitly acknowledged as the hardest case — if the compromised node also supplies the monitor's telemetry, detection degrades |
| Compromised monitor itself | Suppresses alerts, forges records | Independent, source-signed telemetry (Section 6) + FAIL-SAFE default; full protection requires deployment isolation (Section 12), not delivered by software alone |
| **Adaptive adversary** | Tunes attack strength to stay inside statistical confidence regions, knowing the architecture | Randomized audit sampling (secretly-scheduled stricter thresholds) + cross-session cumulative testing (Section 9); protocol-state invariants remain the strongest defense since they are structurally, not statistically, enforced and gain an adaptive attacker nothing by tuning statistics |
| Unknown/unmodeled adversary | Any manipulation not in the attack library | Goodness-of-fit rejection → INVESTIGATE (Section 8) |

---

## 6. Independent Observation — What "Independent" Actually Means

**Not claimed:** the monitor does not observe live quantum states mid-protocol (this would disturb them — physically incoherent to claim otherwise).

**Actually done:** Recipients necessarily measure their teleported qubits and disclose classical outcomes as part of the protocol's own normal operation (this is not something the monitor adds — QS-L-style verification requires this disclosure to function at all). The Evidence Collector reads this **already-disclosed classical telemetry** over an authenticated channel, signed at the measurement device at the moment of collapse — functionally the same physical basis QBER-based QKD eavesdropping monitoring already relies on.

**Explicit limitation:** a compromised measurement device, compromised *before* signing its own telemetry, cannot be caught by this scheme — that requires hardware root-of-trust, out of scope for a software framework and stated as such rather than hidden.

---

## 7. Architecture

```
              Protected Teleportation-Distributed QS-L Protocol (Section 4)
                                    │
              (read-only, source-signed telemetry — Section 6)
                                    │
                ┌───────────────────┴───────────────────┐
                ▼                                        ▼
      Quantum Evidence Collector                Protocol Evidence Collector
   (mismatch rate, correlation,                (FSM transition validity,
    entropy, Pauli-correction                   freshness, authorization,
    consistency)                                sequencing)
                │                                        │
                ▼                                        ▼
      Honest-Execution Reference Model         Protocol State Integrity Engine
   (closed-form expected values under          (deterministic gate — REJECT on
    a declared, periodically recalibrated       any invariant violation,
    noise model p — Section 10)                 independent of statistics)
                │                                        │
                ▼                                        │
      Stage 1: Model-Fit Check                           │
   (does data fit the iid noise family                   │
    at ANY p? No → MODEL_INVALID/                        │
    RECALIBRATE. Yes → continue)                          │
                │                                        │
                ▼                                        │
      Stage 2: SPRT vs. declared H0/H1                    │
   (sequential — Section 9)                               │
                │                                        │
                ▼                                        │
      Unknown-Anomaly Gate                                │
   (block goodness-of-fit — catches                       │
    structural/clustered deviations                       │
    SPRT's fixed H1 misses)                                │
                │                                        │
                ▼                                        │
      Attack Attribution (Section 8)                      │
   CONFIDENT / AMBIGUOUS / NO-ATTRIBUTION                  │
   (e.g. Pauli-structured vs. depolarization                │
    via regularity/autocorrelation statistic)                │
                └───────────────────┬───────────────────┘
                                    ▼
                     Cross-Session CUSUM (Section 9)
              (catches persistent low-level drift no
               single session can — the architecture's
               single largest measured win)
                                    ▼
                        Threat Response Engine
     ACCEPT / REJECT / QUARANTINE / INVESTIGATE / MODEL_INVALID / FAIL-SAFE
                                    ▼
                  Signed, Append-Only Forensic Evidence Log
                                    ▼
                          SOC / Human Response
```

---

## 8. Attack Attribution — Three Honest Output States

Ranked lists that always name a "winner" were rejected as misleading. Attribution now produces one of three explicit outcomes:

- **CONFIDENT:** exactly one hypothesis exceeds all others by more than a declared separability margin.
- **AMBIGUOUS:** two or more hypotheses (e.g., Pauli-structured manipulation vs. channel depolarization at matched overall mismatch rate) fall within that margin — reported as a tied set, not a forced ranking. This is a **named, expected** case, not a failure mode.
- **NO ATTRIBUTION:** either `H0` (legitimate) wins, or every hypothesis fails goodness-of-fit → routed to the unknown-anomaly path.

---

## 9. Statistical Decision Machinery

- **Stage 1 (Model-Fit):** block-wise chi-square test of whether the current session's mismatch pattern fits an i.i.d. Bernoulli process at *some* re-estimated `p`, independent of the previously calibrated value. Fails → `MODEL_INVALID/RECALIBRATE`, not a security verdict.
- **Stage 2 (SPRT):** sequential log-likelihood-ratio test between `H0` (declared noise model) and a specific `H1` (e.g., elevated mismatch rate consistent with depolarization/forgery), stopped early at configured `α, β` boundaries.
- **Unknown-Anomaly Gate:** block-chi-square test for clustering/structure inconsistent with the fitted i.i.d. family — catches attacks whose average deviation is too mild for SPRT's fixed `H1` to catch (Section 11.5 shows this is not hypothetical).
- **Attribution:** likelihood ratios per known attack model, each either analytically derived from the attack's physical effect or Monte-Carlo calibrated — never hand-assigned.
- **Cross-Session CUSUM:** cumulative sum of per-session mismatch-rate deviation from baseline across a monitoring window, tuned to a declared false-positive budget over that window — the only mechanism in the architecture that catches persistent, per-session-invisible drift.

---

## 10. Noise Modeling and Adaptive Adversaries

Noise parameter `p` is a rolling-MLE-recalibrated configuration input, not a hardcoded constant — absorbing gradual drift into "the model" rather than false alarms, while sudden shifts still trigger Stage 1/Stage 2 (Section 9). Against a strategic adversary who knows the architecture and tunes attack strength to survive typical thresholds: (a) a **secretly-scheduled subset of sessions** receives stricter, unpredictable thresholds (randomized audit sampling), raising the cost of successful evasion since the attacker cannot know which sessions are being scrutinized harder; (b) **cross-session CUSUM** (Section 9) catches an adversary who successfully stays inside any single session's confidence region but cannot sustain that across many sessions without a detectable cumulative bias; (c) **protocol-state invariants remain the strongest defense** against adaptive tuning specifically because they are structural, not statistical — an attacker cannot "tune" their way past a freshness or authorization check the way they can tune a mismatch rate.

---

## 11. Experimental Validation — What Was Actually Measured

A Monte Carlo simulation (4,000 trials/condition, 200 measurements/session, six conditions: legitimate, depolarization, Pauli-structured, low-and-slow, replay, unmodeled burst) compared full QSENTINEL against two audit-proposed simpler baselines (fixed mismatch threshold alone; protocol-invariants + fixed threshold) and six one-component-removed ablations. Full code is reproducible (see project files); results below are exactly what the code produced, including the unflattering ones.

| Detector | legitimate (FP) | depolarization | pauli_structured | low_and_slow | replay | burst_unknown |
|---|---|---|---|---|---|---|
| Baseline: FixedThreshold only | 0.0045 | 0.9920 | 1.0000 | 0.2597 | 0.0053 | 0.5527 |
| Baseline: Protocol+FixedThreshold | 0.0047 | 0.9900 | 1.0000 | 0.2507 | 1.0000 | 0.5367 |
| **Full QSENTINEL** | 0.0150 | 0.9865 | 1.0000 | 0.2552 | 1.0000 | **0.8502** |
| Full − Quantum Evidence | 0.0000 | 0.0000 | 0.0000 | 0.0000 | 1.0000 | 0.0000 |
| Full − Protocol Evidence | 0.0150 | 0.9868 | 1.0000 | 0.2482 | **0.0152** | 0.8450 |
| Full − Sequential Detection (fixed-N) | 0.0107 | 0.9895 | 1.0000 | 0.2502 | 1.0000 | 0.8475 |
| Full − Unknown Anomaly Detection | 0.0077 | 0.9872 | 1.0000 | 0.2655 | 1.0000 | **0.4480** |

Mean samples-to-decision, SPRT vs. fixed-N (matched detection rate): **68–89 vs. 200** across conditions — SPRT's real benefit is ~3x faster decisions, not higher accuracy (rates are statistically indistinguishable between the two).

Cross-session CUSUM for low-and-slow, tuned to a 1% false-positive budget over a 50-session window: **100% detection at a mean of 5.7 sessions**, versus ~25–27% for any single-session detector, simple or complex.

Attribution (Pauli-structured vs. depolarization via a lag-10 autocorrelation statistic): **100% accuracy** in this simulation — reported as a best-case bound (the simulated attacker uses a maximally regular, easiest-to-detect period-10 pattern), not a general guarantee.

Model-misspecification handling: pure environmental drift (still i.i.d., wrong `p`, no attack) is correctly routed to `MODEL_INVALID/RECALIBRATE` rather than false attack attribution in **99.7%** of cases, even at a drift magnitude matching the depolarization attack's rate exactly — with the explicit, stated limitation that at that exact magnitude, the framework **cannot** distinguish "channel got worse" from "attacker made it look worse the same way," and correctly declines to guess.

### Honest interpretation

- **For clean, strong, single-session statistical attacks (depolarization, Pauli manipulation), the full architecture provides no measured advantage over a simple protocol-invariants-plus-threshold baseline.** This is stated plainly rather than obscured.
- **Protocol evidence is unambiguously necessary:** removing it collapses replay detection from 100% to 1.5% (indistinguishable from noise) — quantum evidence alone cannot see a replay attack at all.
- **The unknown-anomaly gate is the architecture's clearest positive result:** 85.0% vs. ~54% for either simple baseline on an unmodeled attack shape — and removing it drops detection to 44.8%, *below* even the naive baseline, because SPRT's fixed alternative hypothesis doesn't match this attack's actual deviation magnitude. This is a measured weakness of SPRT alone, not a hypothetical one, and it is exactly why the gate is load-bearing rather than decorative.
- **Cross-session CUSUM is the single largest win in the study**, for an attack class (low-and-slow) that no single-session detector — simple or complex — can catch at all on its own.

**The narrowed, evidence-backed novelty claim:** QSENTINEL's measurable value over the simplest defensible baseline is concentrated in unmodeled/structural-anomaly detection and cross-session persistent-drift detection, plus a secondary, bounded latency benefit from sequential testing — not in the architecture generally.

---

## 12. Trust Boundaries and Deployment

The monitor trusts the entanglement-distribution precondition (out of scope to independently validate) and the classical-channel authentication assumption inherited from Section 4; it does **not** trust the QDS implementation's own self-reported success/failure, independently recomputing acceptance from raw telemetry instead. Evidence records are signed at collection time using a key the QDS operational path does not have, narrowing (not eliminating) the compromised-monitor risk in Section 5. Production deployment would place collector agents at lower operational privilege than the signing/verification service, on a separate authenticated telemetry channel from the QDS protocol's own classical channel, with SOC integration via standard alerting rather than a bespoke dashboard as the primary interface — explicitly deferred from the SIH prototype, which runs single-process with simulation-only telemetry.

---

## 13. Mathematical Model (Reference)

- Correlation: `C = (1/N) Σ a_i b_i`; expected under symmetric depolarizing noise `p`: `C_exp = 1 - 2p`
- Mismatch rate: `m = mismatches/N`, `E[m] ≈ p`
- Entropy: `H = -Σ P(x) log2 P(x)` over matching-basis outcomes
- Stage 1 goodness-of-fit: block-wise Pearson chi-square, `χ² = Σ(O_k - E_k)²/E_k`, `df = n_blocks - 1`
- SPRT: `Λ_n = Σ log[f_1(x_i)/f_0(x_i)]`; decide `H1` at `Λ_n ≥ log[(1-β)/α]`, decide `H0` at `Λ_n ≤ log[β/(1-α)]`
- Attribution likelihood ratio: `LR_j = P(evidence|H1_j)/P(evidence|H0)`, reported per Section 8's three-state contract
- CUSUM: `S_n = max(0, S_{n-1} + (m_n - p_base - k))`, flag at `S_n > h`; `k, h` tuned to a declared false-positive budget over a stated monitoring horizon

---

## 14. Delivery Table

| Deliverable | Implementation | Validation | Status |
|---|---|---|---|
| Teleportation-distributed QS-L protocol simulator | Qiskit Aer | Deterministic acceptance on noiseless runs | Core prototype |
| Quantum Evidence Collector | Reads statevector/measurement data | Cross-check vs. Qiskit ground truth | Core prototype |
| Protocol State Integrity Engine (FSM) | Custom validator | Injected-violation test set (100% target) | Core prototype |
| Honest-Execution Reference Model | Closed-form NumPy/SciPy | Model-vs-empirical agreement | Core prototype |
| Stage 1 Model-Fit Check | Block chi-square | Misspecification sweep (Section 11) | **Validated: 99.7% correct routing** |
| SPRT Decision Engine | Custom sequential test | Ablation vs. fixed-N | **Validated: 3x sample reduction, matched accuracy** |
| Unknown-Anomaly Gate | Block chi-square (structure) | Ablation on burst_unknown | **Validated: 85.0% vs. 44.8%–55.3% alternatives** |
| Attribution Engine | Likelihood ratios, 3-state output | Regularity-statistic separation test | **Validated (best-case): 100% on clean structured attacks** |
| Cross-Session CUSUM | Rolling cumulative sum | Tuned false-positive budget test | **Validated: 100% detection, 5.7-session mean latency** |
| Forensic Evidence Log | Signed, hash-chained SQLite | Tamper-injection test | Core prototype |
| Monte Carlo Evaluation Suite | Parameterized simulation grid | This document, Section 11 | **Delivered** |

---

## 15. What Other Teams Will Build vs. What This Delivers

| Other Teams | QSENTINEL |
|---|---|
| Generic teleportation demo as the whole deliverable | Teleportation as the distribution layer of a literature-grounded QS-L protocol variant, with the substitution explicitly flagged as unproven |
| Undefined "use Bell states for QDS" | Fully specified protocol (Section 4), citable verification logic |
| Fixed-threshold pass/fail | SPRT + explicit two-stage model-fit/attack separation |
| Single confident attack label | Three-state attribution (confident/ambiguous/no-attribution) |
| Claims of architectural superiority without evidence | Actual Monte Carlo ablation, including results unfavorable to the architecture, reported anyway |
| "Digital twin" as marketing | Honestly named, narrowly scoped closed-form reference model |
| No handling of unmodeled attacks | Goodness-of-fit gate, empirically shown to matter |
| No handling of adaptive adversaries | Randomized audit sampling + cross-session cumulative testing |
| Assumes noise model is always correct | Explicit MODEL_INVALID state, empirically validated |

---

## 16. Stated Limitations (Not Hidden)

- Attack models were built and evaluated by the same design process — the circularity risk is real and only partially mitigated by using physically-derived models where possible; no adaptive, strategically-optimized attacker was tested.
- The Monte Carlo study operates at the abstract observable level, not on a full Qiskit statevector simulation of the actual protocol; the attack→observable mapping is itself a modeling assumption.
- CUSUM's false-positive rate depends on the chosen monitoring horizon and requires periodic re-tuning or resetting in continuous deployment.
- At noise-drift magnitudes matching a real attack's effect exactly, the model-fit/attack distinction cannot be resolved by statistics alone — this is stated as a hard boundary, not solved.
- Full independent-observation and deployment-isolation guarantees require hardware root-of-trust and production infrastructure outside a software prototype's scope.

---

## 17. Final Innovation Statement

**One-line:** A protocol-aware QDS threat-detection layer, grounded in a citable QDS protocol family rather than an invented one, that reads only already-disclosed classical telemetry, separates "attack" from "my model is wrong," reports attribution honestly (including when it doesn't know), and backs its narrowed novelty claim with actual Monte Carlo evidence rather than architectural argument alone.

**What is claimed:** measurable, demonstrated value in detecting unmodeled/structural anomalies and persistent cross-session drift — attacks that a simple protocol-invariants-plus-threshold baseline measurably misses.

**What is not claimed:** superior detection of clean, strong, single-session statistical attacks over a simple baseline (the data doesn't support it); guaranteed separability of statistically similar attacks (Pauli manipulation vs. depolarization is a named ambiguous case); resistance to a fully adaptive, strategically-optimized adversary (only partially addressed, not solved); protection against a compromised measurement device at the hardware level; or that this is the final word on the underlying protocol's security (the teleportation substitution is explicitly unproven).
