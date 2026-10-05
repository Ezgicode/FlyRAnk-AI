# Capstone Report - Content Opportunity Prioritization

- **Author:** Ezgi Ozturk
- **Lane:** Refresh and Content Opportunity Scoring
- **Repo:** https://github.com/Ezgicode/FlyRAnk-AI
- **Date:** October 2026

## 0. Abstract

This project asks which content pages should be prioritised for human review when editorial capacity is limited. The analysis uses a March 2026 warehouse slice containing 77,540 pseudonymized content pages and search-performance signals measured during the first 15 days of the month. A transparent rule-based baseline, Logistic Regression, and Random Forest were compared on the same grouped client holdout using a later-window decline proxy. Logistic Regression achieved the strongest observed Precision@50 at 0.28, compared with 0.14 for the baseline, while the baseline and Random Forest were stronger at Precision@20 with 0.20. The output is a public-safe decision-support queue for human review; it does not guarantee that a page will decline or recover after an edit.

## 1. Problem framing

This project supports a content-review prioritisation decision: which pages should a content editor or SEO analyst inspect first?

The unit of analysis is one pseudonymized content page. The output is a ranked priority score with an action label and reason code. A reviewer may choose to create a content-refresh brief, review a title and snippet, investigate a technical or intent issue, defer the item, or take no action.

A false positive can waste limited editorial capacity on a page that does not need intervention. A false negative can leave a potentially important page outside the review queue. Data and machine learning help because a large content inventory cannot be reviewed page by page using identical manual criteria. The system supports prioritisation; it does not replace editorial judgement.

## 2. Data safety

The analysis uses the FlyRank ML Internship warehouse release and the `fact_content_daily_performance` table for March 2026. The raw warehouse grain is one pseudonymized client-content-date record.

The feature window is 1–15 March 2026. The later outcome window is 16–31 March 2026 and is used only to construct the historical decline proxy. The page-level feature frame retains pages with at least 100 prior-period impressions.

The final model uses only:

- `prior_15d_impressions`
- `prior_15d_clicks`
- `prior_15d_avg_position`
- `prior_15d_ctr`
- `prior_15d_active_days`

Client and content IDs are used only for grouping, joining, and validation; they are never model features. Later-window impressions, the decline label, and any ratio derived from later data are excluded because they would leak future outcome information. Existing product flags and previous decision scores are also excluded.

No client names, domains, URLs, raw search queries, credentials, or private account information appear in the public project artifacts.

## 3. Baseline

The first comparison method was a transparent hand-written action score. It uses only pre-decision impressions, average position, and CTR.

The score gives additional priority to pages with meaningful visibility, low CTR, and an average position in the 21–50 range. Visibility is log-transformed so that higher-visibility pages receive more impact weight. Position ranges receive hand-written weights, while low CTR increases review priority. Every selected page receives a human-review action and an explanatory reason code.

The baseline is a fair comparison because it uses the same held-out pages, grouped client split, decision-time information, and Precision@K metrics as the learned models. On the grouped client holdout, the baseline measured Precision@20 of 0.20 and Precision@50 of 0.14. The test-set decline-proxy base rate was 0.148.

## 4. Model / analysis

Logistic Regression was selected as the main readable model because the lane requires an explainable ranked queue. Random Forest was included as a conservative non-linear comparison.

The target is a later-window decline proxy. A page receives label 1 when it has at least 100 impressions in the first 15 days of March 2026 and receives at least 20 percent fewer impressions in the later March window.

The final feature list is:

- `prior_15d_impressions`
- `prior_15d_clicks`
- `prior_15d_avg_position`
- `prior_15d_ctr`
- `prior_15d_active_days`

The analysis deliberately excludes client IDs, content IDs, future impressions, future-derived ratios, the label itself, existing decision flags, and prior scores. These exclusions reduce leakage risk and prevent the models from memorising client-specific patterns.

## 5. Evaluation

The evaluation uses `GroupShuffleSplit` by pseudonymized client with `random_state=42`. The training set contains 61,747 rows from 28 clients. The held-out test set contains 15,793 rows from 10 different clients. No client appears in both groups.

The same held-out pages and Precision@K metrics are used for all methods.

| Method | Precision@20 | Precision@50 | Test base rate |
|---|---:|---:|---:|
| Hand-written baseline | 0.20 | 0.14 | 0.148 |
| Logistic Regression | 0.15 | 0.28 | 0.148 |
| Random Forest | 0.20 | 0.26 | 0.148 |

The result is mixed rather than uniformly positive. The baseline and Random Forest were strongest for a short top-20 queue. Logistic Regression was strongest for a 50-page review queue and is easier to interpret than Random Forest.

Error review shows that some high-score false positives had high visibility, very low CTR, and positions around 24–34 but did not meet the later decline proxy. Some low-score false negatives had strong positions or comparatively high CTR but still met the proxy. These examples show that the selected signals do not capture every relevant condition.

## 6. Interpretation

The model results show that a small set of pre-decision search-performance signals can directionally support a human review queue, but no single signal is sufficient.

A useful negative result came from the visibility audit. Across four visibility buckets, observed later decline rates ranged from approximately 26.7 to 29.3 percent. This means visibility alone did not strongly distinguish later decline, although it still matters when estimating the potential impact of reviewing a page.

Average position was more directionally useful. The positions 21–50 group had the highest observed later decline rate at approximately 34.9 percent, compared with approximately 23.9 percent for the top-three group. This finding supports additional review attention for visible pages that are outside the strongest ranking range.

The models describe measured associations in this historical slice. They do not show that changing CTR, position, clicks, or impressions causes a future decline.

## 7. Recommendation

The output is a ranked decision-support queue for content editors, SEO analysts, and content strategists.

The top-20 historical queue contains:

- 10 `content_refresh_brief` recommendations
- 9 `title_snippet_review` recommendations
- 1 `human_refresh_review` recommendation

A `content_refresh_brief` is recommended for visible pages in positions 21–50. The reviewer should inspect freshness, topic coverage, structure, internal links, and search intent. A `title_snippet_review` is recommended for visible pages nearer page one or two with weak CTR, where a smaller title or snippet intervention may be more appropriate than a full content rewrite.

Every recommendation requires human review. Before acting, a reviewer checks search intent, the current SERP, page quality, estimated editorial effort, business value, seasonality, and possible technical issues.

The following actions must not be automated:

- Publishing or rewriting content
- Changing titles, metadata, or internal links
- Deleting, merging, or canonicalising pages
- Treating a score as proof of future decline
- Using future-window or label-derived fields as features

The grouped-holdout result provides directional evidence that Logistic Regression can support a 50-page review queue. It does not prove that an edit will improve performance or that the result will apply to every future client.

## 8. Reproducibility

The project repository is:

https://github.com/Ezgicode/FlyRAnk-AI

The main notebooks should be run in this order:

1. `work/notebooks/w01_research_question.ipynb`
2. `work/notebooks/w02_ml_task_framing.ipynb`
3. `work/notebooks/w03_data_contract.ipynb`
4. `work/notebooks/w04_baseline_score.ipynb`
5. `work/notebooks/w05_model.ipynb`
6. `work/notebooks/w06_validation_audit.ipynb`
7. `work/notebooks/w07_action_playbook.ipynb`
8. `work/notebooks/capstone.ipynb`

A fresh local environment can be prepared with:

```bash
git clone https://github.com/Ezgicode/FlyRAnk-AI.git
cd FlyRAnk-AI
python -m venv .venv


After activating the environment:
pip install duckdb huggingface_hub pandas numpy scikit-learn matplotlib jupyter
Warehouse notebooks require approved Hugging Face dataset access and a read token stored outside the public repository. In Colab, the token is stored as the HF_TOKEN Secret.
The grouped validation uses random_state=42. The capstone notebook sorts the page-level frame by pseudonymized client and content ID before splitting, so the held-out evaluation is reproducible. The notebook writes work/outputs/capstone_metrics.json as a public-safe metrics receipt and saves reusable figures in work/figures/. Row-level queues are regenerated by the notebook and are intentionally not committed to the public repository.
9. Acknowledgments & data credit
Built on the FlyRank ML Internship dataset.
The project uses anonymized and pseudonymized internship data for educational machine-learning analysis. It does not publish client-identifying information.

GitHub’da oluşturma adımı:

1. Repo ana sayfasında **Add file → Create new file**.
2. Dosya adı:

```text
work/capstone_report.md
