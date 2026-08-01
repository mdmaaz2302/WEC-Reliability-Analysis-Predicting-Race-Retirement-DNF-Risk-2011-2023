# WEC Reliability Analysis: Predicting DNF vs. Classified Finish (2011–2023)

## Overview
A binary classification project on 12 seasons of FIA World Endurance Championship (WEC) results, predicting whether a car finishes a race as **Classified** or retires (**DNF**), using only information knowable before the race starts. Built with the **PACE** framework (Plan → Analyze → Construct → Execute).

## The question
> Can class, team, circuit, tyre supplier, season, and a team's historical reliability rate predict whether an entry finishes or retires?

## Data
`wec_data.csv` — 3,035 race entries, 2011–2023, 16 circuits, 5 classes (LMP1, LMP2, LMGTE Pro, LMGTE Am, HYPERCAR).

## Methodology
1. **Clean (Pandas):** standardized inconsistent class labels (`LM P1` → `LMP1`, etc.), wrote a regex-based parser to handle mixed time/lap-deficit strings in `total_time`, `gap_first`, `gap_car_ahead`, and built a deliberate 5-way → binary mapping for the `status` field (dropping "Not started" entries and treating "Excluded" as a separate case from mechanical DNFs).
2. **Feature engineer:** separated features into *descriptive* (in-race outcomes like laps completed and pace vs. class best — useful for charts, excluded from the model to avoid leakage) and *pre-race-safe* (class, team, circuit, tyres, season, event duration, class field size, and an expanding team-reliability rate computed only from each team's prior races).
3. **SQL layer (SQLite):** team win rates by season/class, DNF rate by tyre supplier × class, average fastest-lap pace by circuit/year, and team reliability rankings.
4. **Model (scikit-learn):** Random Forest (primary) vs. Logistic Regression (baseline), evaluated on ROC-AUC, confusion matrix, and per-class precision/recall — prioritizing recall on the DNF class, while explicitly reporting precision alongside it as the cost of that choice.
5. **Robustness checks:** 5-fold stratified cross-validation (a single 80/20 split at n≈3,035 with a ~14% minority class is too noisy to trust on its own); a temporal holdout (train 2011–2020, test 2021–2023) across the exact LMP1→Hypercar boundary the "Hypercar is more reliable" finding depends on; permutation importance as a check on impurity-based importance, which is biased toward high-cardinality features like `team`; and a redundancy test dropping raw `team` to see how much signal survives in `team_reliability_rate` alone.

## Key findings
- **Track identity is the single strongest predictor of DNF risk** — confirmed by both impurity and permutation importance, so it isn't a one-hot-encoding artifact. Le Mans's 24-hour format is a fundamentally harder reliability test than a 6-hour sprint round. One honest caveat: the permutation-importance confidence interval on `race` is wide enough to overlap with `event_duration` and `team_reliability_rate` beneath it, so "track identity is #1" is solid but the exact gap to #2 and #3 shouldn't be over-read.
- **A team's reliability *history* matters; its raw identity doesn't, once you control for that history.** `team_reliability_rate` outranks tyre brand in both importance measures, while raw `team` drops to near-zero under permutation importance and can be dropped entirely for a 0.007-AUC cost in cross-validation — worth cutting for a leaner, more generalizable model.
- **The Hypercar era shows a lower DNF rate than the LMP1 era so far** in the top class — visible directly in the data (DNF-by-class-by-season chart), independent of the predictive model. This comes with a real caveat: it's only three Hypercar seasons (2021–2023, swinging from ~5% to ~17% year to year), and the LMP1-era "average" is itself pulled around by a couple of outlier seasons (a rough 2011 debut year, a rough 2018) rather than sitting at one stable level. Treat it as an early, promising signal in the data rather than a settled conclusion — worth re-checking once a few more Hypercar seasons land. The model's own accuracy *drops* across that same boundary in a temporal holdout, mostly because Hypercar and 48 of 80 Hypercar-era teams are entirely absent from pre-2021 training data — a limitation of the model, not a contradiction of the underlying trend.
- Cross-validated **ROC-AUC = 0.73 ± 0.03** (a single 80/20 split reported an optimistic 0.79), with DNF-class recall ≈ 0.72 and precision ≈ 0.34 — `class_weight='balanced'` trades a lot of false DNF flags for that recall, a real cost worth stating alongside the benefit. The ceiling itself is reasonable given that DNF *causes* (mechanical failure, collision, driver error) aren't observable in a results table.

## Limitations
- No qualifying/grid-position data in the source, so a "pole-to-finish delta" feature isn't derivable.
- Team sample sizes vary widely; reliability rates for low-entry teams are noisy — part of why raw team identity adds little beyond the reliability rate.
- The model flags *that* a DNF is likely, not *why*.
- Doesn't transfer cleanly across a regulation-category change (LMP1 → Hypercar), confirmed by the temporal holdout — retraining should track major rule changes, not just elapsed time.
- DNF-class precision (≈0.34) is low: about 2 in 3 "likely DNF" flags don't pan out. Reasonable for a screening tool a strategist reviews, not for an unattended alert.
- The Hypercar-era reliability trend rests on only three seasons (2021–2023) with visible year-to-year swings — directionally interesting, not yet statistically settled.

## Repo contents
- `WEC_Reliability_Analysis.ipynb` — full annotated notebook (cleaning → EDA → SQL → ML → robustness diagnostics)
- `wec_data.csv` — raw source data
- `wec_cleaned.csv` — cleaned, feature-engineered dataset (also BI-dashboard-ready)
- `wec.db` — SQLite database used for the SQL layer
- `WEC_Dashboard.html` — standalone interactive dashboard (open in any browser, no server needed): team × season reliability heatmap, class-evolution timeline with the LMP1→Hypercar boundary marked, tyre × class DNF comparison, lap-pace-by-circuit — built from the SQL layer's exports below
- `team_season_reliability.csv`, `class_evolution_timeline.csv`, `lap_pace_by_circuit.csv`, `tyre_class_dnf.csv` — the four SQL-layer exports powering the dashboard, also ready to import into Power BI / Tableau directly if you want a native `.pbix`/`.twbx` version
- `linkedin_post_draft.md` — draft post leading with the Hypercar-vs-LMP1 finding, framed as a data fact rather than a model claim

## Next steps
- If a native Power BI (`.pbix`) or Tableau (`.twbx`) file matters for the portfolio (e.g. to list the tool by name), rebuild the same four visuals in the desktop app from the four CSV exports above — each is already a clean, pre-aggregated table, so it's a straightforward import-and-chart job rather than a data-prep one.
- Review and publish the LinkedIn post draft.
- Package this notebook with the README into the GitHub portfolio repo alongside the F1 and SpaceX projects.
