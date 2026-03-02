# Evaluation Metrics by Task Type

## Classification

### Binary Classification

**Primary metric candidates** (choose based on business objective):

- **Precision**: When false positives are costly (spam filtering, fraud alerts that trigger manual review). "Of the things I flagged, how many were correct?"
- **Recall**: When false negatives are costly (disease screening, security threats). "Of the actual positives, how many did I catch?"
- **F1 Score**: Harmonic mean of precision and recall. Use when both types of errors matter roughly equally.
- **F-beta**: When precision and recall have unequal importance. F2 weighs recall higher; F0.5 weighs precision higher.
- **AUROC**: Measures ranking quality across all thresholds. Good for model comparison, but misleading on heavily imbalanced data.
- **AUPRC (Average Precision)**: More informative than AUROC for imbalanced problems. Always prefer this when positive class is <10% of data.
- **Log Loss**: Measures calibration quality — useful when predicted probabilities drive downstream decisions.

**Threshold selection**: Models output probabilities; the threshold converts them to decisions. Don't default to 0.5. Choose the threshold that optimizes the business-relevant metric on the validation set. Visualize precision-recall and ROC curves to understand the trade-off.

**Calibration**: If predicted probabilities are used directly (not just thresholds), verify calibration with reliability diagrams. Apply Platt scaling or isotonic regression if needed.

### Multi-class Classification

- **Macro-averaged F1**: Treats all classes equally regardless of size. Use when rare classes matter.
- **Weighted-averaged F1**: Weights by class frequency. Use when overall accuracy matters and class distribution is meaningful.
- **Per-class metrics**: Always report per-class precision/recall. Aggregate metrics hide class-level failures.
- **Confusion matrix**: Essential for identifying systematic misclassifications between specific classes.
- **Top-k accuracy**: When the model's job is to narrow possibilities (e.g., top-3 predictions), not pick one answer.

### Multi-label Classification

- **Subset accuracy**: Exact match across all labels. Very strict; rarely the right primary metric.
- **Hamming loss**: Fraction of incorrect labels. Better than subset accuracy for many labels.
- **Per-label AUPRC**: Evaluate each label independently. Most informative approach.
- **Micro-averaged F1**: Pools all labels together. Dominated by frequent labels.
- **Macro-averaged F1**: Averages per-label F1. Gives equal weight to rare labels.

## Regression

- **MAE (Mean Absolute Error)**: Robust to outliers, easy to interpret ("average error in original units").
- **RMSE (Root Mean Squared Error)**: Penalizes large errors more. Use when big errors are disproportionately bad.
- **MAPE (Mean Absolute Percentage Error)**: Useful for business communication ("average X% off"). Breaks when true values are near zero.
- **R² (Coefficient of Determination)**: Proportion of variance explained. Context-dependent — 0.7 can be great or terrible depending on the problem.
- **Quantile losses**: When predicting intervals or risk, evaluate at relevant quantiles (e.g., 10th and 90th percentile coverage).

**Always check residuals**: Plot residuals vs. predicted values and vs. key features. Systematic patterns indicate model deficiencies.

## Ranking

- **NDCG@k (Normalized Discounted Cumulative Gain)**: Best general-purpose ranking metric. Accounts for position and graded relevance.
- **MAP@k (Mean Average Precision)**: Binary relevance only. Good when items are clearly relevant or not.
- **MRR (Mean Reciprocal Rank)**: "How far down is the first correct result?" Good for search/QA.
- **Precision@k / Recall@k**: Simple and interpretable. Choose k based on what users actually see.
- **Hit Rate@k**: "Did at least one relevant item appear in top k?" Coarse but practical.

Choose k values that match the UI/UX: if users see 10 results, evaluate at k=10.

## Time-Series Forecasting

- **MAE / RMSE**: Standard, but compute per-step and aggregate thoughtfully.
- **MASE (Mean Absolute Scaled Error)**: Normalizes by naive forecast performance. Best for comparing across series with different scales.
- **MAPE / sMAPE**: Business-friendly but problematic near zero. Use sMAPE (symmetric) to reduce bias.
- **Coverage (Prediction Intervals)**: If forecasting intervals, check that 90% prediction intervals actually contain 90% of observations.
- **Temporal evaluation**: Report metrics at different horizons (1-step, 7-step, 30-step). Models that look good at 1-step may fail at longer horizons.

**Backtesting**: Use expanding or sliding window validation, never random splits. The test set must always be strictly in the future relative to training.

## Anomaly Detection

- **Precision@k**: Of the top k flagged anomalies, how many are real? Practical when analysts review a fixed number of alerts.
- **Recall@k**: Of the real anomalies, how many appeared in the top k flags?
- **F1 at chosen threshold**: Standard, but threshold selection is harder without balanced classes.
- **AUPRC**: Preferred over AUROC because anomalies are typically rare.
- **Time-to-detect**: For streaming anomaly detection, how quickly after an anomaly starts is it flagged?
- **False positive rate at target recall**: "To catch 95% of anomalies, what fraction of normal samples get falsely flagged?"

## Recommendation Systems

- **Offline metrics**: NDCG, MAP, Recall@k, Hit Rate@k — same as ranking metrics, applied to recommendation lists.
- **Beyond-accuracy metrics**:
  - **Coverage**: What fraction of the item catalog gets recommended to at least one user?
  - **Diversity**: How different are the items within a recommendation list? (intra-list distance)
  - **Novelty**: Are recommendations surfacing items the user hasn't seen?
  - **Serendipity**: Are recommendations surprising AND relevant?
- **Online metrics (A/B test)**: CTR, conversion rate, engagement time, revenue per user. These are the ground truth — offline metrics are proxies.

## Cross-Cutting Concerns

### Statistical Significance

Don't compare models based on single-point metric estimates. Use:
- **Confidence intervals**: Bootstrap the test set predictions (1000+ resamples) to get confidence intervals on metrics.
- **Paired tests**: McNemar's test for classification, paired t-test or Wilcoxon signed-rank test for continuous metrics.
- **Effect size**: A statistically significant difference can be practically meaningless. Report the actual metric difference alongside the p-value.

### Sliced Evaluation

Aggregate metrics lie by omission. Always compute metrics on meaningful slices:
- Demographic groups (for fairness)
- Data source or collection period
- Difficulty buckets (e.g., short vs. long texts, simple vs. complex images)
- Feature value ranges (e.g., new vs. returning users)

A model with 95% overall accuracy that has 60% accuracy on a critical subgroup is not a good model.

### Metric Pitfalls

- **Accuracy on imbalanced data**: 99% accuracy is trivial if 99% of data is one class. Never use as a primary metric for imbalanced problems.
- **AUROC on heavily imbalanced data**: Can look good while precision is terrible. Use AUPRC instead.
- **Averaging metrics across folds**: Report mean ± std, not just mean. High variance across folds signals instability.
- **Metric hacking**: Optimizing too aggressively for a single metric degrades unmeasured qualities. Use multiple metrics as guardrails.
