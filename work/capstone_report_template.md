# Capstone Report — Machine Learning Track

- **Author:** Noor ul Ain Zahid
- **Lane:** Machine Learning
- **Repo:** (https://github.com/NoorDataAnalyst/flyrank-ML-Internship.git)
- **Date:** September 6, 2026

---
#### **Abstract & Introduction Excerpt**

> **Context & Problem Framing:** Enterprise search management platforms face a fundamental resource allocation challenge: prioritizing editorial refreshes across portfolios spanning hundreds of thousands of URLs. Using **324,940 active enterprise content URLs** from the FlyRank internship warehouse, this case study addresses how short-term trailing search metrics and AI search interactions can be systematically transformed into decision-support priorities.
>
> **Findings:** Evaluating a **LightGBM regressor** against a **historical momentum baseline** under a domain-held-out `GroupKFold` validation split revealed that a simple **0.5× historical click baseline** achieved a superior test **MAE (0.85 clicks vs. 3.24 clicks for LightGBM)**. This demonstrates that short-term organic search performance across unseen enterprise clients is heavily momentum-driven. The resulting model and baseline signals are operationalized into a **4-archetype Content Action Playbook** (`STRIKING_DISTANCE`, `DECAY_RISK`, `AI_SNIPPET_OPP`, and `MAINTAIN`) to guide human-in-the-loop editorial workflows.
---
## 1. Problem Framing

This work supports resource allocation for monthly enterprise SEO and content refresh workflows. 

* **Unit of Analysis:** Content URL (`content_hash_id`) aggregated over trailing 20-day observation windows.
* **Model Output:** Predicted 10-day forward organic click volume (`model_preds_10d`) mapped to actionable content archetypes (`STRIKING_DISTANCE`, `DECAY_RISK`, `AI_SNIPPET_OPP`, `MAINTAIN`).
* **Human Action:** Editorial teams use the prioritized queue to optimize title tags/snippets, update outdated facts, or format content for generative search snippets.
* **Cost of a Wrong Call:** False positives waste limited editorial hours on low-impact URLs; false negatives allow high-value content with traffic momentum to decay unnoticed.
* **Why ML Helps:** Automated heuristic rules fail to capture complex non-linear interactions across impressions, search positions, and AI search session volume at enterprise scale (~325k URLs).

---

## 2. Data Safety

* **Data Release:** FlyRank Internship Warehouse (`hf://datasets/FlyRank/internship-warehouse`).
* **Tables Used:** `fact_content_daily_performance_sample.parquet` and `dim_clients.parquet`.
* **Deliberately Excluded Columns:** Client names, internal domain URLs, raw search queries, user PII, and client identity markers (`client_hash_id` used strictly for evaluation grouping).
* **Leakage Risks Mitigated:** 
  * Label-derived fields and future-window aggregations were excluded from feature matrices.
  * Temporal separation enforced: Features extracted strictly from June 1–20, 2026; target defined as organic clicks from June 21–30, 2026.
* **Safety Confirmation:** Zero client-identifying strings or raw URLs exist in `work/` or exported artifacts.

---

## 3. Baseline

* **Definition:** Historical Momentum Baseline—scaling 20-day historical click volume down to a 10-day proportional estimate (`feat_gsc_clicks_20d * 0.5`).
* **Fair Comparison:** Evaluated on the exact same 10-day forward target window (`target_clicks_late_june`) and held-out domain splits as the machine learning model.
* **Baseline Metrics:**
  * **Test MAE:** 0.85 clicks
  * **Spearman Rank Correlation:** 0.64

---

## 4. Model / Analysis

* **Method:** LightGBM Regressor (`n_estimators=100`, `learning_rate=0.05`, `max_depth=6`) trained on log-transformed targets (`log1p`) to manage multi-order-of-magnitude volume disparities.
* **Exact Feature List:**
  1. `feat_gsc_clicks_20d` (Trailing 20-day GSC Clicks)
  2. `feat_gsc_impressions_20d` (Trailing 20-day GSC Impressions)
  3. `feat_avg_position_20d` (Trailing 20-day Average Rank Position)
  4. `feat_organic_sessions_20d` (Trailing 20-day GA4 Organic Sessions)
  5. `feat_ai_sessions_20d` (Trailing 20-day GA4 AI Search Sessions)
  6. `feat_ctr_20d` (Derived Click-Through Rate)
  7. `feat_flag_gsc_available` (GSC Source Availability Flag)
  8. `feat_flag_ga4_available` (GA4 Source Availability Flag)
* **Target Definition:** Total accumulated Google Search Console organic clicks over the subsequent 10-day period (`target_clicks_late_june`).

---

## 5. Evaluation

* **Validation Strategy:** 5-fold `GroupKFold` cross-validation grouped strictly by `client_hash_id`. This simulates deployment to unseen enterprise client portfolios and eliminates domain-level authority leakage.

### Metric Evaluation Summary
| Model Variant | Test MAE (Lower is Better) | Spearman Rank Correlation |
| :--- | :---: | :---: |
| Historical Momentum Baseline (`0.5x` 20d Clicks) | **0.85** | **0.64** |
| Honest LightGBM (`GroupKFold` by Client) | **3.24** | **0.32** |

* **Error Analysis:** Out-of-fold evaluation reveals that tree-based gradient boosting models incur cross-domain variance penalties when generalizing to unseen enterprise account scale distributions. Historical momentum remains the single strongest deterministic predictor for short-term 10-day traffic volume.

---

## 6. Interpretation

* **Feature Signals:** Historical 20-day clicks and overall impression volume provided the dominant predictive signals for baseline scale.
* **Generative & Position Dynamics:** `feat_ai_sessions_20d` and top-5 position metrics serve as effective classification triggers for generative overview optimization (`AI_SNIPPET_OPP`).
* **Negative Result / Honest Finding:** Machine learning regressors without domain-specific historical embeddings struggle to outperform proportional historical scaling across held-out client domains. Short-term traffic persistence is highly momentum-driven.

---

## 7. Recommendation

Each URL is categorized into four decision-support archetypes forming the content action playbook:

1. **Striking Distance (`STRIKING_DISTANCE`):** Positions #4–#20 with $\ge 100$ impressions. Prioritize for CTR, title tag, and intent alignment.
2. **Decay Risk (`DECAY_RISK`):** High historical volume ($\ge 50$ clicks) experiencing $>60\%$ predicted drops. Focus on updating facts, dates, and internal links.
3. **Generative Overview Opportunity (`AI_SNIPPET_OPP`):** Top-5 ranking pages exhibiting AI search sessions. Format content into concise Q&A schemas.
4. **Maintain (`MAINTAIN`):** Stable performance requiring no immediate editorial allocation.

### Human-in-the-Loop Guardrails & No-Go Rules
* **No Automated Publishing:** direct CMS automated rewriting or publishing without editorial review is strictly prohibited.
* **YMYL / Compliance Exclusion:** Medical, financial, legal, or privacy policy pages must never be modified automatically.
* **URL Structure Lock:** Permalinks, canonical tags, and URL redirects require manual technical SEO approval.

---

## 8. Reproducibility

* **Execution:** Run `work/notebooks/capstone.ipynb` sequentially from top to bottom.
* **Seeds:** `random_state=42` set for model initialization and data splits.
* **Dependencies:** `python>=3.10`, `duckdb>=0.9.0`, `lightgbm>=4.0.0`, `pandas>=2.0.0`, `numpy>=1.24.0`, `scikit-learn>=1.2.0`, `matplotlib`, `seaborn`.

---

> **Acknowledgments & Data Credit:** Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
