# Temporal Validation In Higher-Education Retention Modeling

## Plain-Language Definition

Temporal validation tests whether a model trained on earlier cohorts performs well
on a later cohort.

This matters because retention scoring is usually forward-looking. Institutions use
prior student records to support review of a newer or current cohort.

## Simple Example

| Step | Data used |
| --- | --- |
| Train | 2016-2023 cohorts |
| Test | 2024 cohort |
| Score | 2025 cohort |

This structure better reflects operational use than a random split that mixes
students from all years into both training and testing.

## Why Random Splits Are Not Enough

Random train/test splits are useful as a first benchmark, but they can mix students
from the same institutional time period across both training and testing. That can
make the model look stronger than it will be when applied to a future cohort.

Examples of time-related differences:

- policy changes
- grading pattern changes
- advising process changes
- COVID or post-COVID effects
- shifts in student composition
- changes in course modality or enrollment behavior

## Stakeholder Interpretation

Temporal validation does not guarantee future performance. It provides a more
realistic test of whether historical patterns generalize to a newer cohort.

For retention work, this is important because the model should support planning,
review, and outreach before the final retention outcome is known.

## Responsible Use

Temporal validation should be paired with:

- calibration review
- subgroup performance review
- threshold and capacity planning
- intervention-outcome tracking
- data governance review

