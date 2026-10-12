# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Ahmed El-Dasoky
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Eldasoky1/flyrank-ml-internship
- **Date:** 2026

## 0. Abstract

Which existing pages should a content team review for refresh, in what order, when the team can
only read ~50 per pass? Working on the bundled 30,000-page anonymized 90-day snapshot, I frame the
lane as a ranking/scoring task, build a transparent hand-rule baseline first, then train logistic
regression and random-forest models on a **client-grouped holdout**. A random forest reaches
ROC-AUC 0.750 and Precision@50 0.74 (baseline 0.627 / 0.24), and the winning features are all
backward-looking visibility/position/freshness signals. The output is a ranked, human-reviewable
refresh queue with machine-readable reason codes — decision support for triage, not an
automated refresh.

## 1. Problem framing

- **Decision:** which content page to review for refresh next.
- **Unit of analysis:** one content page (client × content item).
- **Output:** a ranked queue plus `action` and `reason codes` per row.
- **Action a human takes:** click into the top pages, confirm demand, then refresh/expand if the
  signals align.
- **Cost of a wrong call:** a human's review time spent on the wrong page (cheap) vs. a genuinely
  declining, high-demand page left to rot (lost visits). Triage worth ranking because the base
  pool of declining pages is large (16,262 / 30,000 = 54%).

## 2. Data safety

- **Used:** bundled `data/raw/content_refresh_anonymized.csv` (30,000 pages × 44 columns).
- **Excluded and why:** `provider_used`, `model_used` (business-internal provenance, not
  outcome); `trend_direction` (defines the label); raw IDs as grouping only.
- **Leakage risks considered:** no label-derived features, no future windows, no product flags;
  re-checked in `w03_feature_leakage_check.ipynb` and `w06_validation_audit.ipynb`.
- Nothing client-identifying appears anywhere in `work/` (verified by construction — no text,
  no queries, only aggregate signals).

## 3. Baseline

Transparent hand-rule action score (see `w04_baseline_score.ipynb`): +4 if declining-with-demand,
+2 low CTR (visible), +2 low engagement (visible), −1 stale (>90d no update); scaled 0–100, ranked
descending. Reason codes: `declining_with_demand`, `low_ctr_visible_page`,
`low_engagement_visible_page`, `stale`, `healthy`.

**Baseline numbers (client holdout, same as the model):** ROC-AUC 0.627 · avg precision 0.468 ·
Precision@20 0.15 · Precision@50 0.24 · Precision@100 0.36 · recall 0.19 · F1 0.27.
Fair comparison: same data, same metric, same split.

## 4. Model / analysis

- **Why:** trees handle the heavy-tailed, interaction-heavy signals and return readable feature
  importances; logistic regression kept as the auditable baseline model.
- **Features (final set):** 16 backward-looking numeric + static metadata features (impressions /
  clicks / sessions / users / engaged / scroll 90d, days_with_*, search_volume, word/char count,
  content_age_days, days_since_last_update, ctr, avg_position, engagement_rate, scroll_rate,
  ai_traffic_pct). Heavy-tailed counts log-transformed; categoricals bucketed by the pipeline.
- **Target (proxy), one sentence:** `is_declining_label = (trend_direction == "down")`, an
  observed 90-day trend rule shipped with the release.

## 5. Evaluation

- **Split:** client-grouped holdout (`GroupShuffleSplit`, 20% of *clients* held out, seed 42) —
  pages of the same client never straddle the split, mirroring production where a queue mixes many
  clients.
- **Metrics (same split, same data):**

| Model | ROC-AUC | Avg precision | Precision@20 | Precision@50 | Precision@100 | Recall | F1 |
|---|---:|---:|---:|---:|---:|---:|---:|
| baseline_rules | 0.627 | 0.468 | 0.15 | 0.24 | 0.36 | 0.19 | 0.27 |
| logistic_regression | 0.700 | 0.522 | 0.35 | 0.40 | 0.44 | 0.57 | 0.57 |
| decision_tree | 0.742 | 0.575 | 0.55 | 0.62 | 0.60 | 0.72 | 0.63 |
| **random_forest** | **0.750** | **0.618** | **0.65** | **0.74** | **0.72** | **0.74** | **0.64** |

- Base rate of the target: 0.542 — all models beat it; AUC/lift over baseline are the honest
  discrimination numbers.
- **Errors:** false positives are recently-updated, high-impression pages whose labels flipped
  `down` for demand reasons; the model reads "visible + engaged" as healthy even when momentum is
  fading. For triage this is acceptable (the human confirms demand), but it's the top limitation.

## 6. Interpretation

Top feature importances (random forest): `days_with_impressions` (0.16), `log_impressions_90d`
(0.13), `avg_position` (0.11), `content_age_days` (0.10), `char_count` (0.04), `word_count` (0.04),
`log_clicks_90d` (0.03), `ctr` (0.03), `scroll_rate` (0.03), `days_with_sessions` (0.03).

Plain words: **persistence of visibility** (days with impressions), **absolute demand**
(log impressions), and **position** dominate. A page that is consistently seen but aging and
under-clicking is the highest-leverage refresh. Negative/surprise result: `search_volume` and
`competition` rank low once impressions/position are already in the model — raw keyword demand is
already folded into observed visibility.

## 7. Recommendation

Ranked actions (see `work/outputs/action_playbook.csv` and `w07_action_playbook.ipynb`):
- **refresh / refresh_and_review_ctr** — declining, visible, low CTR pages: rework copy/CTR.
- **refresh_and_review_engagement** — declining, visible, low engagement: improve on-page depth.
- **monitor** — healthy or tiny: leave alone, re-score next cycle.

How a FlyRank editor uses it tomorrow: read the top 20 of the queue, confirm demand still exists
(`search_volume`, competition) and the page is actually underperforming on inspection, then act.
Confidence: high for *ranking triage*; **none for "will recover"** — this work scores decline-risk
and routes the review, it does not promise improvement.

## 8. Reproducibility

- Re-run everything from a fresh clone:
  `pip install -r requirements.txt` then `python scripts/run_all.py` (reference pipeline) and open
  `work/notebooks/*.ipynb` top to bottom (or `jupyter nbconvert --execute`).
- Seeds: all models and splits `random_state=42`; same `GroupShuffleSplit` logic everywhere.
- Receipts committed: `outputs/model_results.json`, `outputs/model_report.md`,
  `work/outputs/baseline_action_score.csv`, `work/outputs/action_playbook.csv`.
- Notebook 03 (`notebooks/03_working_with_the_full_release.ipynb`) requires the gated HuggingFace
  warehouse token (setup step, Week 3) — the assignable notebooks here run entirely on the bundled
  data.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. [https://flyrank.ai](https://flyrank.ai) — anonymized,
observable-signals release (fixed 90-day snapshot); data governed by `DATA_USE.md`. No
client-identifying information is reproduced here.

---

*Claims checklist: observed / measured / directional / decision-support. Metrics reported beside
the base rate. No causal claims. Numbers match a fresh re-run (`outputs/model_results.json`).*