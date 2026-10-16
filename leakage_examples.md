# Leakage Examples Catalog

Concrete leakage patterns in student-success modeling. Each entry: what it looks like, why it's tempting, how to catch it, and the fix. All scenarios are synthetic.

## 1. The cumulative GPA trap

- **Looks like:** `cumulative_gpa` as a top feature with suspiciously high importance.
- **Why it's tempting:** it's the single best predictor of future GPA — because it *contains* the future when computed carelessly.
- **Catch it:** check the feature's as-of date. If the warehouse view is `current_cumulative_gpa` with no term parameter, it's computed over all terms including the outcome term.
- **Fix:** build `cumulative_gpa_as_of_census` — aggregate only terms strictly before the prediction term's census date.

## 2. The "ever withdrew" flag

- **Looks like:** `ever_withdrew = 1` predicts stop-out almost perfectly.
- **Why it's tempting:** withdrawing *is* the outcome process starting.
- **Catch it:** any feature named `ever_*` or `total_*` without a time bound. Ask: does the window include the outcome period?
- **Fix:** time-bound versions only — `withdrew_prior_term`, `cumulative_withdrawals_as_of_census`.

## 3. Post-hoc administrative codes

- **Looks like:** `leave_status = 'Medical Leave'` shows up as highly predictive of non-return.
- **Why it's tempting:** it's in the student record, neatly coded.
- **Catch it:** compare the code's *recorded date* against the prediction date. Leave/withdrawal codes are often entered weeks after the student stopped attending.
- **Fix:** exclude any feature whose recording timestamp can postdate the prediction point, or lag it explicitly.

## 4. Advisor notes and early alerts

- **Looks like:** text-derived "risk mentions" from advising notes predict attrition well.
- **Why it's tempting:** advisors are good at spotting trouble.
- **Catch it:** notes written *after* the outcome (e.g. "student confirmed not returning") contaminate the corpus. Also: if the model's predictions will *trigger* advisor notes, you've built a feedback loop, not a feature.
- **Fix:** timestamp-filter notes to before the prediction date; document the feedback-loop risk in the model card.

## 5. Target-adjacent outcomes

- **Looks like:** predicting "will not graduate in 4 years" with `applied_for_graduation` as a feature.
- **Why it's tempting:** it's strongly correlated — because it's nearly the target itself.
- **Catch it:** the "census morning" test: would this value exist before the outcome is knowable? Application for graduation happens *because* the student expects to finish.
- **Fix:** drop it. Some features are too close to the target to be features at all.

## 6. Leaky imputation and scaling

- **Looks like:** a small but consistent performance boost that's hard to attribute.
- **Why it's tempting:** `SimpleImputer` / `StandardScaler` fit on the whole dataframe is the default in every tutorial.
- **Catch it:** audit the pipeline — are transformers fit before or after the temporal split?
- **Fix:** fit imputers/scalers on the training terms only, serialize them, apply frozen to validation/test/production.

## 7. The "current major" problem

- **Looks like:** `current_major` predicts persistence better than `entry_major`.
- **Why it's tempting:** it's a richer, more up-to-date field.
- **Catch it:** "current" as of *when*? If it's current-as-of-today, it reflects major changes that happened *after* the prediction term — including changes caused by struggling.
- **Fix:** use `major_as_of_census` (prediction term), and consider *change of major* itself as a separate, properly-lagged feature.

## The meta-lesson

Leakage isn't a rookie mistake — it's the **default state** of warehouse data, which is organized for reporting "as of now," not for prediction "as of then." The fix is always the same discipline: **reconstruct the world as it looked on prediction morning**, feature by feature, with timestamps to prove it.
