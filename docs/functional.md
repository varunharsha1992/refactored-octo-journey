# Functional Specification — B2B SaaS Churn Prediction

> **Status: HOLDING DOCUMENT.** This captures the problem as handed over, before any
> implementation or design decisions are locked. Nothing here is a commitment to a
> particular model, library, or data source. `APPROACH.md` will supersede the
> "open questions" section once decisions are made.

## 1. Context

A mid-size B2B SaaS company is losing customers fast enough that the board has raised it.
The retention team today works **reactively**: outreach begins only after a cancellation
notice arrives, by which point the decision to leave has usually already been made.

The ask is a **churn-prediction capability** that surfaces at-risk accounts early enough
to intervene while the outcome is still changeable.

## 2. Who uses this and what they do with it

**Primary user:** the retention team (non-technical, works a queue).

**Interventions available to them, in rough order of cost:**

| Intervention | Cost to company | Appropriate for |
|---|---|---|
| Feature walkthrough / enablement | Low (CSM time) | Accounts with low feature adoption |
| Check-in call | Low–medium (CSM time) | Accounts with recent friction signals |
| Discount / commercial concession | High (margin) | High-value accounts, late-stage risk |

This matters functionally: the output cannot be a single yes/no flag. The team needs
**ranked risk plus enough context to choose an intervention**.

## 3. Functional requirements

### 3.1 Required deliverables

- **R1 — Pipeline.** An end-to-end, runnable pipeline: raw data → feature engineering →
  model training → evaluation → churn-risk score per customer.
- **R2 — Data.** Either a self-generated synthetic dataset or the provided synthetic
  dataset pack. **The choice must be documented** (either is acceptable).
- **R3 — `APPROACH.md`.** Committed before or alongside the implementation. Must cover:
  - churn framed in business terms — what churn means *for this product*, what the
    prediction is *for*, and what "good enough to be useful" means;
  - pipeline decomposed into distinct stages, with each stage's responsibility stated;
  - model and metric choices justified **against the retention use case specifically** —
    including the recall/precision asymmetry (a missed churner costs a customer; a false
    alarm costs an outreach email) and how that shapes threshold and metric selection;
  - **at least one concrete risk or trade-off** — label leakage, synthetic-data realism,
    class imbalance, or similar — and whether it was mitigated or knowingly accepted.
- **R4 — `README`.** Setup and reproduction from a **clean checkout**: install
  dependencies, regenerate or load data, retrain, and reproduce the reported evaluation
  numbers.
- **R5 — AI collaboration transcript.** The exchange with the AI coding agent is itself
  graded. Expectation is deliberate use: challenge or correct the agent's framing where
  it is wrong, supply domain reasoning the agent did not introduce, and state constraints
  and verification targets rather than accepting the first plausible answer.

### 3.2 Data domains in scope

The raw inputs span four domains:

1. **Usage events** — logins, feature invocations, seat activity, session depth.
2. **Subscription / billing history** — plan, seats, MRR, upgrades/downgrades,
   renewal dates, payment failures.
3. **Support interactions** — ticket volume, severity, response/resolution latency,
   sentiment or escalation markers.
4. **Account metadata** — industry, size, contract terms, tenure, acquisition channel.

### 3.3 Pipeline stages (responsibility sketch)

Stated as responsibilities, not as an implementation plan:

| Stage | Responsible for |
|---|---|
| Data generation / ingestion | Producing or loading the raw four-domain dataset; reproducibly (seeded). |
| Label definition | Turning subscription history into a churn label with an explicit observation window and prediction horizon. |
| Feature engineering | Aggregating event-level data to account-level features **as of the prediction cutoff only**. |
| Training | Fitting the model on a split that respects time ordering. |
| Evaluation | Reporting metrics chosen for the retention use case, not just accuracy. |
| Scoring / output | Emitting a per-customer risk score the retention team can work as a queue. |

### 3.4 Output contract (indicative)

One row per account, at minimum:

- account identifier
- churn risk score (calibrated probability preferred over a raw margin)
- risk tier or rank, so the queue can be worked top-down
- the dominant contributing signals, so an intervention can be chosen (see §2)

## 4. Non-functional expectations

- **Reproducible.** Clean checkout → documented commands → same numbers. Seeds fixed.
- **Runs on a laptop.** No cluster, no GPU, no external service dependency.
- **Readable over clever.** This is a capability the retention team's engineers will
  inherit.

## 5. Known risks to address in `APPROACH.md`

Flagged here so none is silently dropped later:

- **Label leakage.** The highest-signal fields — cancellation-notice flags, downgrade
  events, final-invoice state — are often *consequences* of the churn decision, not
  predictors of it. Any feature computed after the prediction cutoff is leakage.
- **Synthetic-data realism.** If churn is generated from a known rule, a model can
  recover that rule and post excellent metrics that mean nothing. Synthetic data bounds
  how much the headline numbers can be believed.
- **Class imbalance.** Churn is the minority class. Accuracy is uninformative here;
  AUC can also flatter a model that ranks poorly in the region the team actually works —
  the top of the queue.
- **Threshold is a business decision, not a modelling one.** It follows from the
  asymmetry in §2 and from retention-team capacity (how many accounts can they actually
  call per week?), not from maximising F1.
- **B2B account-level churn is low-volume.** Few accounts, few churn events — small
  effective sample size, wide confidence intervals on every metric reported.

## 6. Out of scope (unless raised later)

- Production deployment, scheduling, or serving infrastructure.
- A UI for the retention team.
- Causal estimation of *intervention effectiveness* — this predicts risk; it does not
  claim which treatment works.
- Integration with real customer data.

## 7. Open questions (to be resolved, not asked)

Per instruction, these are resolved by documented judgement rather than escalated:

- Self-generated synthetic data vs. the provided pack.
- Churn definition: hard cancellation only, or also non-renewal and material downgrade.
- Prediction horizon length.
- Model family, and whether a baseline is reported alongside it.
