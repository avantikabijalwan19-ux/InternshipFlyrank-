# Capstone Report — Refresh / Content Opportunity Scoring

* **Author:** Avantika Bijalwan
* **Lane:** Refresh / Content Opportunity Scoring
* **Repo:** https://github.com/avantikabijalwan19-ux/InternshipFlyrank-
* **Date:** August 2026

> This report documents the capstone work for the **Refresh / Content Opportunity Scoring** lane. The analysis is intended as **decision-support** for prioritizing content for human review using anonymized FlyRank search data.

---

## 1. Problem framing

This project supports the decision of **which content pages should be reviewed first for a possible refresh**.

The unit of analysis is an individual **content page**. The model produces a **ranked score** representing the likelihood that a page belongs to the observed `down` trend category. Instead of automatically deciding which pages should be refreshed, the system creates a prioritized review queue for editors.

A FlyRank editor can use this ranked queue to:

* Review pages showing strong signals of declining visibility.
* Inspect content freshness and search intent.
* Decide whether to refresh, monitor, protect, or leave the page unchanged.

The cost of a wrong prediction is primarily **opportunity cost**. Refreshing content unnecessarily consumes editorial time, while missing a genuinely declining page may delay a useful content intervention.

Machine learning helps because reviewing thousands of pages manually is impractical. Combining multiple search, engagement, and content signals creates a repeatable and transparent prioritization process that scales better than manual review alone.

The model is used as a **decision-support system**, not an automated content editing system.

---

## 2. Data safety

The project uses the anonymized **FlyRank ML Internship dataset**.

### Dataset summary

* Approximately **30,000 content pages**.
* **44 columns** containing search, traffic, engagement, freshness, and content metadata.
* Target variable: `trend_direction`.

### Target distribution

| Trend Direction |      Count |
| --------------- | ---------: |
| down            |     16,262 |
| stable          |      5,962 |
| up              |      4,388 |
| new             |      2,236 |
| flat            |      1,152 |
| **Total**       | **30,000** |

The `down` category represents roughly **54.2%** of the dataset and is therefore the majority class.

### Columns deliberately excluded

The following columns were removed before training:

| Column            | Reason                                                       |
| ----------------- | ------------------------------------------------------------ |
| `trend_direction` | Target label; cannot be used as an input feature.            |
| `trend_pct`       | Directly derived from the target trend information.          |
| `content_id`      | Pseudonymous identifier; excluded from model features.       |
| `client_id`       | Used only for grouped validation; never used for prediction. |

### Leakage audit

Potential leakage was reviewed for historical performance fields including:

* `impressions_last_30d`
* `impressions_prev_30d`
* `clicks_last_30d`
* `clicks_prev_30d`
* `sessions_last_30d`
* `sessions_prev_30d`
* `impressions_90d`
* `clicks_90d`
* `sessions_90d`

Target-derived columns and identifiers were excluded before modelling. The leakage audit passed its basic checks.

### Privacy and public-safe handling

This project intentionally excludes:

* Client names.
* Domains or URLs.
* Search queries.
* Credentials.
* Raw client exports.

Everything inside the `work/` directory remains anonymized and safe for public sharing.

---

## 3. Baseline

Before training a machine learning model, a transparent baseline was established.

The baseline represents a simple rule-based approach using available historical performance signals instead of a learned classifier. This provides an honest reference point that the machine learning model must outperform.

### Why the baseline is fair

* Uses the same dataset.
* Uses the same prediction target (`trend_direction`).
* Evaluated using the same metrics wherever applicable.
* Allows comparison between a simple decision rule and a learned model.

### Baseline vs learned models

The Week 5 modelling notebook compared the baseline with Logistic Regression and Random Forest.

Random Forest achieved stronger measured performance than the simpler Logistic Regression model and therefore became the final ranking model used in the capstone.

The baseline remains important because it demonstrates that improved performance comes from combining multiple signals rather than relying on one simple rule.

---

## 4. Model / analysis

### Method choice

The primary model is a **Random Forest classifier**.

Random Forest fits this lane because the objective is to rank content pages according to the likelihood that they belong to the observed declining trend category. Tree-based models can learn nonlinear relationships between traffic history, engagement, freshness, search position, and content characteristics.

A Logistic Regression model was trained first as an interpretable comparison model.

### Target definition

The prediction target is:

`trend_direction`

The model predicts one of five observed trend categories:

* down
* stable
* up
* new
* flat

The final ranking queue uses the Random Forest probability for the **down** class.

### Features used

The model includes search, engagement, traffic, freshness, and content characteristics.

**Search features**

* search_volume
* competition
* competition_level
* cpc

**Content features**

* content_type
* main_intent
* word_count
* char_count
* provider_used
* model_used

**Traffic features**

* impressions_90d
* clicks_90d
* pageviews_90d
* sessions_90d
* users_90d
* engaged_sessions_90d
* ai_sessions_90d
* scroll_events_90d

**Historical performance**

* impressions_last_30d
* clicks_last_30d
* sessions_last_30d
* impressions_prev_30d
* clicks_prev_30d
* sessions_prev_30d

**Freshness and age**

* content_age_days
* age_tier
* age_tier_order
* days_since_last_update
* freshness_tier

**Engagement metrics**

* ctr
* avg_position
* engagement_rate
* scroll_rate
* ai_traffic_pct

**Categorical tiers**

* word_count_tier
* char_count_tier
* impression_tier
* position_tier

### Features intentionally excluded

* content_id
* client_id
* trend_direction
* trend_pct

These fields were removed to avoid identifier leakage or target leakage.

The Random Forest probabilities were later converted into ranked editorial recommendations.

---

## 5. Evaluation

### Validation design

Two evaluation strategies were compared.

### Week 5 random split

The original model used a standard random train/test split.

**Results**

| Metric      | Value      |
| ----------- | ---------- |
| Accuracy    | **78.75%** |
| Weighted F1 | **76.82%** |

This split produced stronger headline performance but allows pages from the same client to appear across training and testing.

### Week 6 client-grouped split

A stricter evaluation was introduced using **GroupShuffleSplit** grouped by `client_id`.

**Validation setup**

| Item             | Value |
| ---------------- | ----- |
| Training clients | 25    |
| Testing clients  | 7     |
| Client overlap   | 0     |

### Grouped validation results

| Metric      | Value      |
| ----------- | ---------- |
| Accuracy    | **72.40%** |
| Weighted F1 | **69.35%** |

### Before vs After validation

| Validation Design    |   Accuracy | Weighted F1 |
| -------------------- | ---------: | ----------: |
| Random Split         | **78.75%** |  **76.82%** |
| Client-Grouped Split | **72.40%** |  **69.35%** |

The grouped split provides a more conservative estimate of model performance on unseen clients and is treated as the primary validation result in this capstone.

### Error analysis

The model performs best on pages with strong historical visibility signals.

More difficult cases include:

* Pages with limited historical impressions.
* Newly published pages.
* Pages whose performance signals overlap between trend categories.
* Stable pages with mixed engagement behaviour.

These errors reinforce that the ranked output should be reviewed by a human rather than executed automatically.

### Top-K evaluation

The model was also evaluated as a prioritization system.

**Observed ranking metrics**

| Metric       | Value     |
| ------------ | --------- |
| Precision@20 | **1.00**  |
| Precision@50 | **1.00**  |
| Base rate    | **0.511** |

These metrics indicate that the highest-ranked review candidates contained a much higher concentration of observed `down` pages than the overall dataset.

---

## 6. Interpretation

The Random Forest identified historical visibility and search-performance features as the strongest contributors to the fitted model.

### Top feature importances

| Feature               | Importance |
| --------------------- | ---------: |
| impressions_prev_30d  |      0.146 |
| impressions_last_30d  |      0.124 |
| impressions_90d       |      0.062 |
| avg_position          |      0.061 |
| days_with_impressions |      0.055 |
| content_age_days      |      0.033 |
| sessions_last_30d     |      0.028 |
| sessions_prev_30d     |      0.025 |
| ctr                   |      0.025 |
| pageviews_90d         |      0.024 |

### Interpretation

Historical impression activity appears to be the strongest measured signal within the fitted model.

Average search position also contributes meaningfully, suggesting that ranking position is associated with observed trend categories in this dataset.

Older content (`content_age_days`) also contributes to predictions, indicating that page age and freshness may help distinguish observed performance patterns.

### Important finding

The largest methodological finding was **not** simply model accuracy.

Performance dropped from **78.75%** under a random split to **72.40%** under client-grouped validation.

This shows that evaluation design materially affects measured performance and that grouped validation provides a more realistic estimate of generalization.

### What this does NOT mean

Feature importance does **not** imply causality.

For example:

* High importance for `impressions_prev_30d` does not mean increasing impressions will improve trend direction.
* The model identifies associations in observed data rather than causal effects.

This is treated as a directional finding rather than proof of cause-and-effect.

---

## 7. Recommendation

The model output is converted into a practical editorial action playbook.

### Priority 1 — Refresh declining, high-visibility content

Pages with a high Random Forest probability of belonging to the `down` category should be reviewed first.

**Reason code**

`DECLINING_HIGH_VISIBILITY`

Editors should inspect:

* Content freshness.
* Search intent alignment.
* Recent impression movement.
* Ranking position.
* Engagement signals.

### Priority 2 — Review declining lower-visibility pages

Pages predicted as declining but with smaller visibility should be reviewed after higher-priority candidates.

These pages may still represent opportunities, but available evidence suggests lower editorial value than Priority 1 pages.

### Priority 3 — Monitor stable, new, or uncertain pages

Stable, flat, and new pages are generally better suited for monitoring unless additional evidence justifies intervention.

### Editorial workflow

```text
Model ranking
        ↓
Human review
        ↓
Inspect page context
        ↓
Refresh / Protect / Monitor / No action
        ↓
Measure future performance
```

### Confidence

The strongest supported claim is that the model can prioritize observed content pages for review on this dataset.

### Explicit limits

This project does **not** claim that:

* Refreshing content will cause improved search performance.
* The model predicts Google's ranking algorithm.
* Model probability guarantees future performance.
* Feature importance represents causal influence.

The final editorial decision always remains with a human reviewer.

---

## 8. Reproducibility

The complete project is available in the GitHub repository:

**Repository**

https://github.com/avantikabijalwan19-ux/InternshipFlyrank-

### Notebook workflow

The analysis is organized into the internship notebooks inside:

`work/notebooks/`

Workflow order:

```text
Data preparation
      ↓
Signal audit
      ↓
Baseline
      ↓
Model training
      ↓
Validation audit
      ↓
Leakage audit
      ↓
Action playbook
      ↓
Capstone report
```

### Random seed

The experiments use a fixed random seed:

`42`

This ensures reproducible train/test splits and Random Forest results across notebook executions.

### Environment

Primary Python libraries:

* pandas
* NumPy
* scikit-learn
* matplotlib

### Figures and outputs

Reusable figures are stored in:

`work/figures/`

The recommendation queue is generated by the notebook inside `work/outputs/` rather than committed manually.

### Re-running the project

1. Clone the repository.
2. Install the required Python packages.
3. Run the notebooks in order from `work/notebooks/`.
4. Execute the validation notebook before publishing metrics.
5. Execute the action playbook notebook to regenerate outputs.
6. Open the deployed paper using the URL stored in `submission/paper_url.txt`.

---

### Claims checklist before submitting

* **Observed:** Results describe measured behaviour on the internship dataset.
* **Measured:** Accuracy, Weighted F1, Precision@K, and feature importance come from executed notebooks.
* **Directional:** Findings describe associations rather than causal relationships.
* **Decision-support:** The model prioritizes content for review; it does not automate editorial decisions.

### Metrics vs base rate

The majority class (`down`) represents **16,262 of 30,000 pages (54.2%)**. Accuracy and Precision@K should therefore always be interpreted alongside this base rate.

### Public-safe reporting

* No client-identifying details are included.
* No domains, URLs, or search queries are published.
* No causal claims are made without experimental evidence.
* No claim is made that the model predicts Google's ranking algorithm.
* The reported metrics correspond to the executed validation notebooks.
