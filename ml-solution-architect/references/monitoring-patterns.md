# Monitoring Patterns for ML Systems

## Three Layers of ML Monitoring

ML monitoring extends beyond traditional application monitoring. You need visibility into system health, data quality, and model performance — each with different signals and response times.

### Layer 1: System Health (Minutes to Detect)

Standard infrastructure monitoring. Alert immediately on failures.

**Metrics**:
- Inference latency: p50, p95, p99. Set SLOs based on user experience requirements.
- Throughput: requests per second. Alert on unusual drops (upstream failure) or spikes (abuse/traffic surge).
- Error rate: HTTP 5xx, model serving errors, timeout rate.
- Resource utilization: CPU, memory, GPU utilization, disk usage.
- Queue depth: for async inference, monitor pending request queues.

**Tooling**: Standard observability stack — Prometheus/Grafana, Datadog, CloudWatch, or equivalent. No ML-specific tooling needed.

**Response**: Standard incident response. If the model server is down, it's an infrastructure problem, not an ML problem.

### Layer 2: Data Quality (Hours to Days to Detect)

Monitor input data for signals that the model's operating conditions have changed.

**Schema Validation** (catch immediately):
- Missing or null values exceeding historical thresholds
- New categorical values not seen during training
- Type mismatches (string where float expected)
- Out-of-range values (negative ages, prices above maximum)
- Record counts deviating significantly from expected volume

**Distribution Monitoring** (detect within hours/days):
- **Population Stability Index (PSI)**: Compare current feature distributions to a reference (training data or a recent baseline). PSI > 0.1 is a yellow alert; PSI > 0.2 is a red alert.
- **Kolmogorov-Smirnov (KS) Test**: For continuous features, test whether current distribution differs from reference. Set significance level based on alert fatigue tolerance.
- **Chi-squared test**: For categorical features, test whether category frequencies have shifted.
- **Per-feature monitoring**: Monitor the top 10-20 most important features (from feature importance analysis). You don't need to monitor every feature.

**Recommended approach**: Compute distribution statistics on sliding windows (e.g., hourly or daily). Compare against the training data distribution and against the recent past (e.g., same window last week). Alert when both comparisons show drift — this reduces false alarms from normal variation.

### Layer 3: Model Performance (Days to Weeks to Detect)

Ground truth arrives with delay. Monitor what you can, when you can.

**When ground truth is available quickly (hours/days)**:
Example: click-through rate, conversion within session, fraud confirmed within days.
- Track primary metric on rolling windows
- Compare against historical baseline and retrained-model benchmark
- Alert when metric degrades beyond threshold (define threshold as % drop from baseline)

**When ground truth is delayed (weeks/months)**:
Example: loan defaults (months), cancer diagnosis confirmation (weeks), long-term engagement.
- **Proxy metrics**: Monitor short-term signals correlated with long-term outcomes
- **Prediction distribution stability**: If the model's output distribution shifts (e.g., suddenly predicting higher risk for everyone), something changed
- **Confidence/uncertainty monitoring**: Track model confidence scores. A drop in average confidence or increase in uncertain predictions signals difficulty
- **Business KPI correlation**: Monitor downstream business metrics that the model influences

**When ground truth is never available**:
Example: some recommendation systems, content moderation at scale.
- Rely entirely on proxy metrics, prediction distribution monitoring, and business KPIs
- Periodic human evaluation on samples to spot-check quality

## Drift Detection

### Concept Drift vs. Data Drift

**Data drift** (covariate shift): Input feature distributions change, but the relationship between features and target stays the same. Example: customer demographics shift over time.

**Concept drift**: The relationship between features and target changes. Example: what constitutes "spam" evolves with new spam techniques.

**Detection approach**:
- Monitor feature distributions for data drift (Layer 2 monitoring)
- Monitor model performance metrics for concept drift (Layer 3 monitoring)
- If features drift but performance stays stable, the model is robust — no action needed
- If performance drops without feature drift, concept drift is likely — retrain on recent data

### Drift Monitoring Implementation

```
Reference distribution (training data or baseline window)
  ↓
Current window (e.g., last 24 hours of production data)
  ↓
Statistical tests (PSI, KS, chi-squared per feature)
  ↓
Aggregation (how many features drifted? which ones?)
  ↓
Alert routing (severity based on feature importance × drift magnitude)
```

**Window sizing**: Too short = noisy, too many false alarms. Too long = slow to detect real drift. Start with daily windows and adjust based on your data velocity.

**Baseline refresh**: Update the reference distribution periodically (e.g., after each retrain) so you're comparing against what the current model was trained on.

## Alerting Strategy

### Alert Design Principles

- **Actionable alerts only**: Every alert should have a clear response procedure. If you can't act on it, don't alert.
- **Severity levels**: Critical (model serving down, data pipeline broken), Warning (drift detected, performance declining), Info (model retrained, new data available).
- **Alert fatigue prevention**: Start with high thresholds and tighten as you understand normal variation. A noisy alert system gets ignored.
- **Composite alerts**: Combine signals rather than alerting on each independently. "Feature X drifted AND performance dropped 3%" is more actionable than either alone.

### Recommended Alert Configuration

| Signal | Threshold | Severity | Response |
|---|---|---|---|
| Model serving error rate | >1% for 5 minutes | Critical | Investigate serving infrastructure |
| Inference latency p99 | >2x SLO for 10 minutes | Critical | Scale up or investigate bottleneck |
| Input null rate | >2x historical rate | Warning | Check upstream data pipeline |
| PSI on top features | >0.2 | Warning | Investigate drift, consider retrain |
| Primary metric drop | >5% relative vs. baseline | Warning | Error analysis, consider retrain |
| Prediction distribution shift | KS test p<0.001 | Info | Log for investigation, compare with performance |
| Training pipeline failure | Any failure | Critical | Debug and rerun |

## Retraining Triggers

### Scheduled Retraining
- Simplest approach. Retrain on a fixed cadence (daily, weekly, monthly).
- Set cadence based on how fast your data distribution changes.
- Always validate the new model against the current production model before deploying.

### Performance-triggered Retraining
- Retrain when monitored performance drops below a threshold.
- Requires ground truth with reasonable delay.
- Include a cooldown period — don't retrain every time metrics fluctuate.

### Drift-triggered Retraining
- Retrain when data drift exceeds a threshold, even before performance degrades.
- More proactive than performance-triggered, catches issues earlier.
- Risk of unnecessary retraining if the drift doesn't affect performance.

### Recommended Approach
Combine scheduled (as a floor) with performance/drift triggers (for responsiveness):
- Scheduled retrain every [cadence based on domain], AND
- Trigger retrain if primary metric drops >X% OR PSI on critical features >0.2
- Always require new model to beat current model on recent validation data before swap
- Log all retraining events, their triggers, and the before/after metrics

## Feedback Loops

ML systems can create feedback loops where the model's predictions influence future training data:

- **Positive loops**: Model recommends item A → users click A → model learns A is popular → recommends A more. Can reduce diversity and create filter bubbles.
- **Negative loops**: Model flags transaction as fraud → transaction blocked → never confirmed as fraud → model has fewer positive examples → detection worsens.

**Mitigations**:
- Exploration: Randomly show non-recommended items to a small fraction of traffic to maintain data diversity.
- Counterfactual logging: Record what the model would have predicted for unchosen actions.
- Offline evaluation on held-out data from before the model was deployed.
- Monitor diversity metrics alongside accuracy metrics.

## Dashboards

Build a single dashboard with three sections:

1. **Operational**: Latency, throughput, error rates, resource utilization. Real-time, last 24 hours.
2. **Data Quality**: Feature distribution stability, null rates, volume. Last 7-30 days.
3. **Model Performance**: Primary metric trend, prediction distribution, slice-level metrics. Last 30-90 days.

Keep it simple. One dashboard that gets checked daily is better than ten dashboards that get ignored.
