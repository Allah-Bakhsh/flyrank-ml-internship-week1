# Capstone Report — <**Refresh / Content Opportunity Scoring**>

- **Author:** **Allah Bakhsh Khuhro**
- **Lane:** **Refresh / Content Opportunity Scoring**
- **Repo:** **https://github.com/Allah-Bakhsh/flyrank-ml-internship-week1**
- **Date:**  **August 2026**


## 0. Abstract
This study asks which content pages are the best candidates for a refresh review, using observable search signals rather than a fixed age-based rule. A random forest model was trained on March 2026 search signals (impressions, clicks, CTR, days observed) to predict whether a page's ranking position would meaningfully worsen by May 2026, deliberately excluding position itself as a feature after an early leakage check. The model was validated on a held-out, time-aware split of pages and compared against a hand-written CTR-vs-position baseline rule on the same split. The model achieved a Precision@20 of 0.950 versus the baseline's 0.800, and stayed ahead of baseline at every cutoff up to 200, both well above the 0.676 base rate. Results are offered as decision-support for content review prioritization, not as proof of causal refresh impact.

## 1. Problem framing

Content review teams can only check a limited number of pages per cycle, so the decision this work supports is: **which pages should a reviewer look at first?** The unit of analysis is one content page. The output is a continuous priority score, sorted into a ranked queue with a reason code and action label. The action a human takes is opening the top-ranked pages and deciding whether to rewrite, update, consolidate, or leave alone, the model orders the queue, it does not act. The cost of a wrong call: a false positive wastes reviewer time; a false negative means a genuinely declining page goes unnoticed until the next cycle. Given limited review capacity, false negatives are the costlier failure mode. A fixed rule (e.g. "flag pages older than 365 days") was tested in earlier work and failed content age didn't reliably predict decline, so a model that can weigh several weak signals together is needed instead of a single threshold.


## 2. Data safety

**Data used:** FlyRank Internship Warehouse (Hugging Face, gated release), table `fact_content_daily_performance`, months `2026-03` (features) and `2026-05` (forward-looking label window). **Excluded:** all `ga4_*` and `sessions_*`/`ai_*` fields, since GA4 connectivity is sparse across clients, including them would bias the model toward GA4-connected clients only. **Leakage risk considered and caught:** an early version of the model included `avg_position` as a feature; since the label is defined directly from `avg_position`'s starting value, this caused a near perfect but circular result (Precision@K = 1.000). The feature was removed and the model retrained honestly (see Section 5). Pseudonymous IDs (`content_hash_id`, `client_hash_id`) were used only for grouping/joining, never as model features. No client names, domains, or private queries appear anywhere in this repo.


## 3. Baseline

The baseline is a CTR-vs-position rule: for each page, compare its actual CTR to the expected CTR for its position bucket (computed from training data only), and flag pages falling meaningfully below expectation, weighted by impression volume. This is a fair, transparent comparison it uses the same underlying signals as the model, just combined with a single hand-written formula rather than learned weights. On the same held-out test set, the baseline scored: Precision@20 = 0.800, @50 = 0.860, @100 = 0.910, @200 = 0.910.


## 4. Model / analysis

A random forest classifier (200 trees, max depth 6, class-weight balanced) was trained on four features: `days_observed`, `avg_impressions`, `avg_clicks`, `ctr` — all computed from March 2026 only, all knowable at the decision moment. `avg_position` was deliberately excluded after the leakage check in Section 2. The target is a proxy: whether the page's average position in May 2026 worsened by 10% or more relative to March not a confirmed "needs refresh" outcome, since no such label exists in the data.


## 5. Evaluation

Pages were split 70/30 into train/test by page, stratified on the label, so no page appears in both sets. This is a time-aware evaluation in the sense that the label itself is drawn from a genuinely future month (May) relative to the features (March) — the model is judged on real forward prediction, not same-month pattern matching. Base rate (majority class, "declined"): 0.676.

| K | Model Precision@K | Baseline Precision@K |
|---|---|---|
| 20 | 0.950 | 0.800 |
| 50 | 0.960 | 0.860 |
| 100 | 0.960 | 0.910 |
| 200 | 0.920 | 0.910 |

An early version of the model (including `avg_position` as a feature) scored a suspicious 1.000 across the board — feature importances showed `avg_position` responsible for 56% of the model's decisions, confirming it was exploiting the label's own construction rather than finding genuine signal. Removing it produced the honest numbers above.


## 6. Interpretation

In the honest model, feature importances were: `avg_impressions` (50.8%), `days_observed` (35.2%), `ctr` (10.4%), `avg_clicks` (3.7%). This suggests that how much a page is *seen* (impressions) and how long it's been actively observed matter more to predicting decline than raw click volume or CTR alone a page getting consistent impressions but not much else may be an early warning sign, more so than low CTR by itself. The clear negative result  that `avg_position` had to be excluded is itself a useful finding: any future work on this lane should treat position-derived features with caution when the label is also position-derived.


## 7. Recommendation

The model's ranked output (`work/outputs/capstone_ranked_recommendations.csv`) gives a content reviewer a prioritized queue: pages with `action = review_for_refresh` and `reason_code = predicted_decline_risk` are the top candidates for the next review cycle, ordered by `model_score`. At K=20, roughly 95% of flagged pages genuinely declined in the following two months, meaningfully above both the baseline rule and the base rate a real, if imperfect, improvement in review efficiency. Confidence: moderate. This is a directional, decision-support signal, not a guarantee a human should still confirm each flagged page before committing review time, and the limits below should be read before acting on the list at scale.


## 8. Reproducibility

To re-run: clone the repo, open `work/notebooks/capstone.ipynb` in Colab, set an `HF_TOKEN` secret with read access to `FlyRank/internship-warehouse`, and run all cells top to bottom. Random seed used throughout: `random_state=42`. Key packages: `duckdb`, `pandas`, `scikit-learn`, `matplotlib` (standard Colab environment, no special installs beyond `duckdb`/`huggingface_hub`). The ranked CSV and precision chart are regenerated fresh on every run; `work/outputs/*.png` is committed, `work/outputs/*.csv` is not (per the repo's leak-guard).



## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — [flyrank.ai](https://flyrank.ai).

---

> **Claims checklist:** all results above are reported as observed, measured, and decision-support — no causal claims about why pages decline or that refreshing them would reverse the trend. Precision@K is reported alongside the 0.676 base rate throughout, per the checklist requirement. No client-identifying details appear anywhere in this report.
