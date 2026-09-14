# Capstone Report — <your lane>

- **Author:**
- **Lane:**
- **Repo:**
- **Date:**

## 1. Problem framing

**Decision Supported:** This project provides directional decision-support for SEO editors to prioritize which pages require meta title and description reviews.
Unit of Analysis: Daily content performance (one row = one piece of content on one day).

**Output:** A ranked probability queue and action playbook.

**Action:** A human editor reviews the live SERP for flagged pages and adjusts meta tags if the page is under-capturing clicks relative to its visibility.

**Cost of a Wrong Call:** Wasting editorial time attempting to optimize a page that suffers from unfixable low intent (e.g., zero-click calculator searches or navigational brand queries).

**Why ML Helps:** A daily data warehouse produces millions of rows. ML reduces this noise into a sorted triage list, replacing random audits with an observed, probability-ranked queue.


## 2. Data safety

**Data Used:** A 10,000-row sample from February 2025 using the `fact_content_daily_performance` table from the public `FlyRank/internship-warehouse` dataset.

**Exclusions & Leakage:** I deliberately excluded label-derived fields (like `gsc_avg_position`, `gsc_clicks`, and `ctr`) from my feature set to prevent taxonomy leakage. I also excluded downstream product flags (like `trend_direction` or `is_declining_label`).

**Privacy:** No client domains, URLs, or raw queries exist in this dataset cut. The `client_hash_id` was used strictly as a grouping variable for validation splits and was never fed to the model as a feature.

## 3. Baseline

**The Rule:** A transparent, arithmetic score that flagged a page if it ranked on Page 1 (`gsc_avg_position` <= 10), had minimum daily visibility (`gsc_impressions` >= 50), and showed poor click capture (`ctr` <= 2.0%).

**Why it is fair:** It acts as an honest heuristic that a human can read and trust. It was evaluated on the exact same dataset, using the exact same split and Precision@K metric as the final ML model.

## 4. Model / analysis

**Method:** Random Forest Classifier.

**Why it fits:** The task is a "which first?" ranking problem. A Random Forest provides the continuous probability scores (`.predict_proba()`) required to sort the queue by confidence (Precision@K) without requiring extensive feature scaling.

**Features:** `gsc_impressions`, `ga4_sessions`, `sessions_organic`, `sessions_direct`.

**Target Definition:** A binary label (`label_needs_fix`) set to 1 if a page ranks on Page 1 (average position <= 10) but yields a CTR of <= 2.0%.

## 5. Evaluation

**Validation Split:** I used a `GroupShuffleSplit` grouped by `client_hash_id` (80/20).

**Why:** A standard random split would be fundamentally dishonest because it would allow the model to memorize specific client domains across different days. Grouping ensures the model is tested on its ability to generalize to completely unseen clients.

**Metrics & Base Rate:** Evaluated using Precision@50.

**Error Analysis:** The model's primary failure mode is "intent blindness". It heavily flags high-impression, low-CTR pages that are actually zero-click searches or navigational brand queries—pages that look broken numerically but are functioning normally in reality.

## 6. Interpretation

**Feature Importances:** The model leaned 100% on `gsc_impressions`.

**Surprises & Negative Results:** While a single feature dominating usually signals data leakage, an audit revealed this was an artifact of data availability. In the observed February 2025 cohort, the `ga4_data_available` flag was FALSE, meaning all GA4 session metrics were zero-filled. The model leaned on impressions because it was the only feature with variance. This is a valid negative result: without GA4 variance, the model essentially functions as a smoothed version of the heuristic baseline.

## 7. Recommendation

**Ranked Actions:** The model output generates an action playbook mapped to the "Leaky Bucket" archetype.

**Editor Workflow:** Editors pull the top 50 flagged items daily. Before touching the CMS, the editor must manually review the live SERP to confirm the low CTR is due to a weak meta title, not an ad-heavy layout or zero-click snippet.

**Limits & Confidence:** This workflow provides directional guidance only. It does not support automated meta-tag rewriting, nor does it guarantee causal traffic increases.

## 8. Reproducibility

To reproduce these exact numbers from a fresh clone:  

**Dependencies:** `pandas`, `datasets`, `scikit-learn`.

**Data Access:** Provide a valid Hugging Face Read Token to stream the `FlyRank/internship-warehouse` dataset.

**Execution:** Run `w05_model.ipynb` and `w07_action_playbook.ipynb`.

**Seeds:** All splits and Random Forest initializations are locked to `random_state=42`.
