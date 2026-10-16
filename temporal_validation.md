# Temporal Validation for Student-Success Models

*How to split, what leaks, and how to pick a threshold when the prediction decides who gets help.*

## 1. Why time matters more than usual

Student data is **longitudinal and autocorrelated**: a student's Fall GPA predicts their Spring GPA, cohorts experience shared shocks (a pandemic, a curriculum change), and institutional policies drift. Random k-fold cross-validation treats terms as exchangeable — they aren't. A model validated with random folds will look better than it is, because the future leaks into training through shared students, shared terms, and shared policies.

**Rule: always validate forward in time.**

## 2. The split scheme

For a model predicting, say, "will not return next fall" for the Fall 2024 cohort:

| Split | Terms | Purpose |
|---|---|---|
| Train | Fall 2019 – Spring 2023 | Learn patterns |
| Validation | Fall 2023 | Tune hyperparameters **and** the decision threshold |
| Test | Fall 2024 | One honest, untouched estimate of deployment performance |

Notes:

- **Embargo gap.** Leave at least one full term between train and validation if features aggregate over prior terms (e.g. "cumulative credits"). Without a gap, a student's Spring record in train and Fall record in validation share most of their feature history.
- **Never re-tune on test.** The test term is spent the moment you look at it. If Fall 2024 disappoints, your next honest estimate needs Fall 2025.
- **Cohort integrity.** Keep whole cohorts together — don't split a cohort's students across train and test. Shared instructors, courses, and campus events make them non-independent.

## 3. Leakage: the usual suspects

Leakage = the model sees information at prediction time that wouldn't exist yet. In student-success work it hides in plain sight:

1. **Cumulative features computed "as of now" instead of "as of prediction date."** `cumulative_gpa` pulled from the warehouse today includes the very term you're predicting. Fix: reconstruct features with a strict as-of cutoff (e.g. census date of the prediction term).
2. **"Ever" flags.** `ever_on_probation`, `ever_withdrew` — if the flag's window includes the outcome period, it's leakage. Version them: `ever_on_probation_as_of_census`.
3. **Post-outcome administrative codes.** A "withdrawal reason" or "leave status" recorded *after* the student left, used to predict leaving. Check every feature's *recording timestamp*, not just its label.
4. **Target-adjacent outcomes.** Predicting "stop-out" while including "applied for re-admit" as a feature. If the feature is caused by the outcome process, it's leakage.
5. **Imputation statistics from the full dataset.** Means/medians computed over train+test leak distributional info. Compute imputers on train only, then freeze.

**Detection habit:** for each feature, ask *"Could I have known this value on the morning of census day for the prediction term?"* If the answer is no or maybe, quarantine it.

## 4. Precision, recall, and the threshold as policy

A risk score is not a decision. The **threshold** converts scores into an outreach list — and that list consumes finite advisor hours. This makes threshold selection a resource-allocation decision, not a purely statistical one.

- **High precision, low recall:** almost everyone flagged truly needs help, but you miss many at-risk students. Choose when outreach capacity is tiny or false positives carry stigma.
- **High recall, low precision:** you catch most at-risk students, but advisors chase many false alarms. Choose when the intervention is cheap and non-stigmatizing (e.g. a nudge email).
- **Report both, plus the base rate.** "Precision 0.62 at threshold 0.35" is meaningless without "base stop-out rate is 0.18" — the lift over random matters more than the raw number.

### Threshold selection procedure

1. On the **validation** term, sweep thresholds and plot precision vs. recall.
2. Overlay **capacity constraints**: "advising can handle ~200 outreach cases/term" → find the threshold whose predicted-positive count ≈ 200.
3. Check **subgroup parity**: precision and recall by first-gen / Pell / URM status at the candidate threshold. A threshold that's accurate overall but recalls half as well for one group is a fairness failure, not a tuning detail.
4. Lock the threshold. Evaluate once on **test**. Report the locked-threshold metrics — not the best threshold found on test.

## 5. Beyond the split: deployment validation

A model that validated well can still fail in production:

- **Population Stability Index (PSI).** Compare the score distribution this term vs. validation. PSI > 0.25 → investigate before acting on scores.
- **Calibration drift.** Is "predicted 30% risk" still ~30% actual? Recalibrate annually; curricula and populations change.
- **Subgroup monitoring.** Track precision/recall gaps every term, not once at launch.
- **Outcome feedback loops.** If flagged students get help and succeed, the model "looks wrong" — that's success, not failure. Don't retrain naively on post-intervention outcomes without accounting for the intervention.

## 6. Minimum viable validation checklist

- [ ] Forward-in-time split with an embargo gap; cohorts kept intact
- [ ] Every feature passes the "known on census morning?" test
- [ ] Imputation/scaling fit on train only
- [ ] Threshold chosen on validation, locked, evaluated once on test
- [ ] Precision/recall reported with base rate and subgroup breakdowns
- [ ] PSI + calibration monitoring plan for deployment
- [ ] Synthetic-data rehearsal: the full pipeline runs end-to-end on synthetic data before touching production extracts
