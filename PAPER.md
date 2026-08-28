# Prioritizing Content Refresh Decisions with Observable Search Signals

## Abstract
FlyRank clients accumulate large content libraries that outgrow manual review capacity, raising the question of which pages deserve attention first. This paper builds and validates a content-refresh prioritization pipeline on a 30,000-page anonymized slice of FlyRank's search-performance data. After defining a data contract and auditing two product-relevant signal assumptions, a transparent baseline rule and three trained models (logistic regression, decision tree, random forest) were compared under an honest, client-grouped holdout split. The baseline rule (declining trend + real traffic demand) achieved Precision@50 = 0.64, outperforming every trained model (0.58-0.60), a result verified free of leakage before being reported. The output is a ranked, reason-coded action queue intended as decision support for human reviewers, not an automated or causal system.

## Introduction / Problem Statement
FlyRank's product surfaces content health signals to help clients decide where to invest limited content-review capacity. This project explores one specific lane of that problem: Refresh / Content Opportunity Scoring — ranking which pages deserve review first, based on observed decline and demand signals, rather than a client manually scanning their full content inventory. The work below documents the full pipeline: a research question grounded in a real content-operations decision, a transparent data contract, a signal audit testing two product-relevant beliefs, a validated baseline, a model comparison under honest evaluation, and a final ranked action playbook including an explicit account of where the model underperformed a simpler rule.

## Data
This project uses the small anonymized starter dataset shipped with the FlyRank ML Internship starter repo (`content_refresh_anonymized.csv`, ~30,000 pages), plus schema exploration of the larger `FlyRank/internship-warehouse` release on Hugging Face (build id `flyrank_pseudonymized_warehouse_release_v20260703`; fact table spanning 2025-01-27 to 2026-06-30). No client names, domains, URLs, or raw queries are included in either release. All identifiers are pseudonymized hashes. Rows were filtered to `impressions_90d > 0` and `content_age_days >= 90`, and deduplicated by `content_id`. FlyRank's own product decision outputs (`health_score`, `priority_score`, `action_type`) are deliberately excluded from the dataset and were never used as features, to avoid circular results.

## Methodology
**Task type:** binary classification (declining vs. not), used downstream to produce a ranking.
**Label/proxy:** `trend_direction == "down"`, a current-window proxy, not a validated future outcome.
**Features:** `impressions_90d`, `avg_position`, `content_age_days`, `word_count`, `days_since_last_update`, `ctr`, `sessions_90d`, and related observable signals, all confirmed knowable before any refresh decision.
**Baseline:** a two-condition rule (declining AND `impressions_90d >= 100`), weighted by impression volume.
**Validation design:** client-grouped holdout split, preventing pages from the same client appearing in both
