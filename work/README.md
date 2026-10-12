# work/ — Ahmed El-Dasoky · ML Internship (Applied Search Intelligence)

**Lane:** Refresh / Content Opportunity Scoring.
**Approach:** observe-signals-first → transparent baseline → honest model → validation audit →
ranked, human-reviewable action queue. Public-safe language throughout (no client text, no raw
queries, no causal claims).

## Notebooks (one per assignment)

| Notebook | Assignment | What it delivers |
|---|---|---|
| `w01_research_question.ipynb` | ML-02 | Lane + narrow, honest research question; 2–3 real numbers from the bundle |
| `w02_ml_task_framing.ipynb` | ML-03 | Task as ranking/scoring; label = `trend_direction==down`; Precision@50; grain verified |
| `w03_data_contract.ipynb` | ML-04 | Unit of analysis, feature/label/context/excluded buckets, query-backed verification |
| `w03_feature_leakage_check.ipynb` | ML-05 | Feature vector + leakage/privacy audit (no label-derived or future features) |
| `w04_signal_audit.ipynb` | ML-06 | 3 signal tests with verdicts; a product-flag assumption tested on real data |
| `w04_baseline_score.ipynb` | ML-07 | Transparent action score + reason codes; ranked queue CSV; top-20 review |
| `w05_model.ipynb` | ML-08 | Client-holdout model (logreg + gradient boost) vs Week-4 baseline; feature attributions |
| `w06_validation_audit.ipynb` | ML-09 | Paper findings critique; before/after honest-split; claim rewrite |
| `w07_action_playbook.ipynb` | ML-10 | Ranked actions, intended use/limits, human no-go list, retrain triggers, paper exports |
| `capstone.ipynb` | ML-11 / capstone | Mirrors the deployed paper: question → data → method → results → recommendations → artifacts |

## Outputs (receipts the report’s numbers trace to)

`work/outputs/` (gitignored data, committed summary files):
- `baseline_action_score.csv` — ML-07 rule queue
- `action_playbook.csv` — ML-10 playbook queue
- `signal_distributions.png`, `playbook_score_dist.png` — figures reused by the paper

## Reproducibility

- Bundled dataset: `data/raw/content_refresh_anonymized.csv` (fixed 90-day snapshot).
- Reference pipeline: `scripts/run_all.py` (verbatim from the starter) — baseline yardstick.
- Seeds fixed (`random_state=42`); same split logic everywhere (`GroupShuffleSplit` by client).
- To re-run: `pip install -r requirements.txt` then, from repo root, open each notebook
  top-to-bottom (or `python scripts/run_all.py` for the reference outputs).