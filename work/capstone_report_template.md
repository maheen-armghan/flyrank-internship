# Capstone Report — <your lane>

* Author: MaheenSaqib
* Lane: Refresh / Content Opportunity Scoring
* Repo: https://github.com/maheen-armghan/flyrank-internship
* Date: 2026-10-08

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract
Which pages should a content reviewer check first when a library has grown too large to review manually? Using a 30,000-page anonymized slice of FlyRank's search-performance data, I built and validated a content-refresh prioritization pipeline combining a transparent baseline rule with three trained models (logistic regression, decision tree, random forest). Under an honest, leakage-free, client-grouped holdout split, the baseline rule (declining trend + real traffic demand) achieved Precision@50 = 0.64, outperforming every trained model (0.58-0.60). The output is a ranked, reason-coded action queue intended as decision support for human reviewers, not an automated or causal system.

## 1. Problem Framing
**Decision supported:** which pages a content reviewer with limited weekly capacity should check first. **Unit of analysis:** one page (`content_id`), aggregated over a 90-day window. **Output:** a ranked score, a one-word reason code, and a suggested action per page. **Action a human takes:** manually review the flagged page and decide whether to refresh, rewrite metadata, or leave it. **Cost of a wrong call:** a false positive wastes limited reviewer time on a healthy page; a false negative lets a real decline go unnoticed and compound. Data/ML helps here because decline is driven by multiple interacting signals (position, CTR, impressions, age) that a human can't easily weigh consistently across thousands of pages by eye.

## 2. Data Safety
**Data used:** the anonymized starter dataset (`content_refresh_anonymized.csv`, ~30,000 pages), filtered to `impressions_90d > 0` and `content_age_days >= 90`, deduplicated by `content_id`. Schema of the larger `FlyRank/internship-warehouse` release (Hugging Face) was also explored for the data contract exercise (build id `flyrank_pseudonymized_warehouse_release_v20260703`).
**Deliberately excluded:** any FlyRank product decision output (`health_score`, `priority_score`, `action_type`) — not shipped in the dataset by design, and never reconstructed, to avoid circular results.
**Leakage risks considered:** `trend_direction` and `trend_pct` are label-derived — used only as the target, never as a feature. Pseudonymous IDs (`content_id`, `client_hash_id`) are used only for grouping/deduplication, never as model features. Two leakage incidents were caught and fixed during this project (see Section 5).
**Confirmation:** no client names, domains, URLs, or raw queries appear anywhere in `work/` — only pseudonymized identifiers and aggregated metrics are used throughout.

## 3. Baseline
**Rule:** `baseline_score = impressions_90d` when `trend_direction == "down"` AND `impressions_90d >= 100`, else 0. **Reason code:** `declining_with_demand`. **Why it's a fair comparison:** it uses only observable, pre-decision signals (decline status and traffic volume) — no information not available at decision time. **Numbers:** Precision@50 = 0.64 on the same client-grouped holdout split used for all models. (Note: an earlier version of this baseline accidentally included the label itself in the score formula, producing an invalid Precision@50 = 1.00 — caught and corrected before this result was reported; see Section 5.)

## 4. Model / Analysis
**Method:** Random Forest classifier (compared against Logistic Regression and a single Decision Tree). **Why it fits:** the lane is a classification problem (declining vs. not) whose output feeds a ranking; a random forest can combine multiple interacting signals in a way a single rule cannot. **Features used:** `search_volume`, `competition`, `cpc`, `word_count`, `content_age_days`, `impressions_90d`, `sessions_90d`, `avg_position`, `ctr`, `days_since_last_update`, `engagement_rate`, `scroll_rate` — all confirmed knowable before any refresh decision. **Left out on purpose:** any FlyRank product decision score (see Section 2). **Target/proxy, one sentence:** `is_declining_label = (trend_direction == "down")`, a current-window proxy rather than a validated future outcome.

## 5. Evaluation
**Split:** client-grouped holdout (`GroupShuffleSplit` on `client_id`, 70/30), chosen because pages from the same client may share patterns a model could memorize — a plain random split would overstate generalization. **Base rate:** 54.2% of pages are labeled declining after filtering — reported here since a high Precision@K can otherwise just reflect a high base rate.

| Method | Precision@50 |
|---|---|
| Baseline (fair, no leakage) | 0.64 |
| Logistic Regression | 0.60 |
| Decision Tree | 0.60 |
| Random Forest | 0.58 |

**Error analysis:** 21 of the random forest's top 50 predictions were false positives (predicted high-decline-risk but actually labeled 'up'/'stable'). These misses shared low CTR, weak position, and comparatively low impressions relative to genuinely declining top-ranked pages — suggesting the model partly conflates "chronically weak-performing" pages with "actively declining" ones, a known weakness of the current-window proxy label.

## 6. Interpretation
Random forest feature importances: `impressions_90d` (31%), `avg_position` (21%), and `content_age_days` (18%) together account for ~70% of the model's decisions; CTR (4%) and staleness/`days_since_last_update` (5%) contributed far less than expected. **Surprise / negative result:** a dedicated signal check found that staleness alone does not cleanly separate declining from non-declining pages (decline rate ranged non-monotonically from 46.7% to 61.1% across staleness buckets) — this was independently confirmed by the trained model assigning staleness very low importance. This is a valid, useful "no effect" finding: FlyRank's staleness-based refresh-flag intuition is not well-supported by this dataset alone.

## 7. Recommendation
The ranked action queue (13,152 non-zero-scoring candidates) uses the `declining_with_demand` rule, since it is both well-populated and matches the validated baseline. A FlyRank editor would use this tomorrow by taking the top N pages (matching their weekly review capacity) from `work/outputs/action_playbook_queue.csv`, reviewing each flagged reason code, and deciding on the suggested action (refresh, rewrite metadata, or monitor). **Confidence:** decision-support only — every recommendation requires human review before action; no recommendation should be automated (see the no-go list in `w07_action_playbook.ipynb`). **Limits:** built on a 30,000-row sample, not the full warehouse; label is a current-window proxy; thresholds are dataset-specific policy choices that would need re-validation on other data.

## 8. Reproducibility
**Fresh clone + run:**
```bash
git clone https://github.com/maheen-armghan/flyrank-internship.git
cd flyrank-internship
pip install -r requirements.txt
python scripts/run_all.py
```
For the notebook-based capstone pipeline specifically, open and run top-to-bottom, in order: `work/notebooks/w01_research_question.ipynb` through `w07_action_playbook.ipynb`, then `work/notebooks/capstone.ipynb`.
**Random seeds:** `random_state=42` used consistently across `GroupShuffleSplit`, `DecisionTreeClassifier`, and `RandomForestClassifier`.
**Environment:** Google Colab default Python 3 runtime; key packages: `pandas`, `numpy`, `scikit-learn`, `duckdb`, `huggingface_hub` (see `requirements.txt` for full pinned versions).
**Sealed/holdout evaluation:** the client-grouped test split (30%) was held out and evaluated once per model configuration; the split-building cell and the resulting `results_table_fixed` comparison are both committed in `work/notebooks/w05_model.ipynb` and `w06_validation_audit.ipynb`.

## 9. Acknowledgments & Data Credit
