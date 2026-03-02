# Fairness Checklist for ML Systems

## When This Applies

Every ML system encodes biases from its training data. This checklist applies to all projects, not just those labeled "sensitive." High-stakes applications (credit, hiring, healthcare, criminal justice, insurance) require more rigorous treatment, but even "low-stakes" systems (recommendations, content ranking) shape user experiences in ways that can be systematically unfair.

## Phase 1: Problem Formulation Audit

Before any data or modeling work, assess fairness risks at the problem level.

### Questions to Answer

1. **Who is affected by this model's predictions?** List the stakeholder groups. Include people who don't directly interact with the system but are affected by its outputs.

2. **What are the failure modes for different groups?** False positives and false negatives have different impacts on different populations. Map these out.
   - Example: A hiring model that rejects qualified candidates. False negatives disproportionately affect underrepresented groups = discriminatory impact.
   - Example: A medical screening tool that flags healthy patients. False positives waste resources but false negatives miss disease = different cost by prevalence.

3. **Does the historical data reflect the world as it should be?** Historical data encodes past decisions, including biased ones. If past loan decisions were discriminatory, a model trained on them will reproduce that discrimination.

4. **Is the target variable itself biased?** Labels derived from human decisions may embed bias. "Successful employee" based on past performance reviews may reflect reviewer bias, not actual performance.

5. **Are there proxy variables for protected attributes?** ZIP code proxies for race. Name proxies for gender and ethnicity. University proxies for socioeconomic status. These can encode discrimination even when protected attributes are excluded.

## Phase 2: Data Audit

### Representation Analysis

- **Group sizes**: Compute sample sizes for each demographic group. Underrepresented groups will have noisier predictions. If a group has <5% of training data, flag it.
- **Label distribution by group**: Does the positive class rate differ across groups? If so, is that difference real-world base rate or selection bias?
- **Feature availability by group**: Are features missing more often for some groups? Missing data patterns can introduce bias.
- **Collection bias**: Was the data collected in a way that systematically excludes or underrepresents certain populations?

### Historical Bias Check

- If labels come from past human decisions, audit those decisions for known biases.
- If possible, compare automated labels against a bias-audited subset of human-reviewed labels.
- Consider whether the training data time period includes known discriminatory practices.

## Phase 3: Fairness Metrics

No single metric captures "fairness." Choose metrics based on the specific fairness definition relevant to your use case.

### Metric Definitions

**Demographic Parity (Statistical Parity)**:
- Predictions positive at equal rates across groups: P(ŷ=1|A=a) = P(ŷ=1|A=b)
- When to use: When equal access or equal allocation is the goal (e.g., ad exposure)
- Limitation: Ignores base rate differences. A calibrated model can violate demographic parity.

**Equal Opportunity**:
- True positive rate equal across groups: P(ŷ=1|y=1,A=a) = P(ŷ=1|y=1,A=b)
- When to use: When the cost of false negatives differs by group and you want qualified members of all groups to be equally likely to be selected
- Most commonly used fairness metric in practice

**Equalized Odds**:
- Both true positive rate AND false positive rate equal across groups
- When to use: When both types of errors must be balanced across groups
- Stricter than equal opportunity

**Predictive Parity**:
- Precision equal across groups: P(y=1|ŷ=1,A=a) = P(y=1|ŷ=1,A=b)
- When to use: When positive predictions trigger costly actions (investigations, interventions)

**Calibration**:
- Predicted probabilities match observed rates within each group: P(y=1|ŷ=p,A=a) = p for all groups
- When to use: When predicted probabilities drive decisions (risk scores)

**Important**: Some of these metrics are mathematically incompatible — you cannot satisfy all of them simultaneously except in trivial cases. Choose the metric(s) that best align with your fairness requirements and document why.

### Measuring Disparities

For your chosen metric(s), compute:

1. **Absolute disparity**: |metric(group_a) - metric(group_b)|
2. **Ratio**: metric(group_a) / metric(group_b) — the "four-fifths rule" in employment contexts requires this ratio to be ≥0.8
3. **Worst-group performance**: The minimum metric value across all groups

Set acceptable thresholds BEFORE evaluating the model. Post-hoc rationalization of disparities is not acceptable.

## Phase 4: Bias Mitigation

If fairness metrics reveal unacceptable disparities, apply mitigations. Organized by pipeline stage:

### Pre-processing (Data Level)

- **Resampling**: Oversample underrepresented groups or undersample overrepresented groups. Simple but can reduce model quality.
- **Reweighting**: Assign higher sample weights to underrepresented groups. Preserves all data.
- **Label correction**: If labels are known to be biased, apply correction factors based on audit results.
- **Feature selection**: Remove or transform features that are strong proxies for protected attributes. Test that removal actually reduces bias (it doesn't always).

### In-processing (Model Level)

- **Constrained optimization**: Add fairness constraints to the training objective. Libraries: Fairlearn, AI Fairness 360.
- **Adversarial debiasing**: Train an adversary that tries to predict the protected attribute from model outputs; penalize the main model for being predictable.
- **Calibration**: Post-training calibration per group to ensure predicted probabilities are accurate within each group.

### Post-processing (Decision Level)

- **Threshold adjustment**: Use different classification thresholds per group to equalize the chosen fairness metric. Simple and effective but requires knowing group membership at inference time.
- **Reject option classification**: In uncertain cases, defer to human review rather than making a potentially biased automated decision.

### Mitigation Trade-offs

Every mitigation involves trade-offs:
- Fairness interventions typically reduce overall accuracy slightly
- Different fairness metrics conflict — improving one may worsen another
- Strong interventions can reduce model utility to the point where it's no longer helpful
- Document the trade-offs explicitly and get stakeholder agreement

## Phase 5: Ongoing Monitoring

Fairness isn't a one-time check. Monitor continuously.

- Track fairness metrics over time with the same rigor as performance metrics
- Alert on significant disparity increases
- Re-audit when the model is retrained or when the population changes
- Monitor for feedback loops that could amplify initial biases over time
- Periodic human audits on samples from each group

## Documentation

For every ML system, document:

1. **Fairness definition chosen and why**: Which metric(s), what threshold, what justification
2. **Groups evaluated**: Which demographic groups, how they were defined, sample sizes
3. **Results**: Metric values per group, disparities found
4. **Mitigations applied**: What was done, what trade-offs were accepted
5. **Ongoing monitoring plan**: What's tracked, how often, what triggers re-evaluation
6. **Limitations acknowledged**: What biases may exist that couldn't be measured or mitigated

This documentation is not optional. It's necessary for accountability, auditing, and stakeholder trust.

## Red Lines

Refuse to build or deploy ML systems that:
- Use protected attributes directly to make adverse decisions (unless legally required, e.g., insurance in some jurisdictions)
- Have known significant fairness violations with no mitigation plan
- Operate in high-stakes domains without any fairness evaluation
- Lack documentation of fairness considerations
- Are specifically designed to circumvent anti-discrimination laws
