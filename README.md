# A/B Test Analysis — End-to-End Senior Data Analyst Framework

## Overview
This repository contains a complete end-to-end A/B test analysis of a landing page experiment. The analysis demonstrates the full senior data analyst workflow — from data validation and experiment design through statistical testing, effect size estimation, visualisation, and business conclusion. The methodology follows industry best practices used at data-mature companies and is designed to be reproducible, rigorous, and explainable to both technical and non-technical stakeholders.

The experiment tests whether a redesigned landing page (new_page) produces a higher user conversion rate than the existing page (old_page). The analysis uses a one-tailed Z-test for proportions as the primary statistical method, with Cohen's h as the effect size measure and a 95% confidence interval to bound the plausible range of the true effect.

## Data Dictionary
The dataset `ab_data.csv` contains one row per user session recorded during the experiment.

`user_id`: Unique identifier for each user
`timestamp`: The date and time at which the user was recorded in the experiment
`group`: The experimental group the user was assigned to
`landing_page`: The actual landing page the user was shown
`converted`: Whether the user completed the target conversion action

## Experiment Setup and Design
The following steps must be completed **before any data is collected**. Changing any of these parameters after seeing results constitutes p-hacking and invalidates the statistical guarantees of the test.

-----------------------------------------

### Step 1 — Define the Business Question
Start with the business context, not the statistics. The experiment should have a clear, falsifiable hypothesis grounded in a product change. In this case the product team redesigned the landing page with the explicit goal of increasing conversion rate. This directional intent is what justifies a one-tailed test.

Questions to answer at this stage:
- What change was made and why is it expected to improve the metric?
- Who is the target user population?
- What is the primary metric and why was it chosen over alternatives?
- Are there guardrail metrics that must not be harmed?
- What is the cost of shipping a change that does not work?
- What is the cost of missing a change that does work?

----------------------------------------

### Step 2 — Define Hypotheses Formally
For a one-tailed test testing whether the new page increases conversion:
```
H0: p_treatment ≤ p_control
H1: p_treatment > p_control
α  = 0.05
```

The null hypothesis in a one-tailed test must cover the entire region that is not the alternative. Writing H0 as equality only (p_treatment = p_control) is incorrect for a one-tailed test — it leaves the region where treatment is worse unaccounted for. The test statistic is computed at the boundary point (equality) regardless, so the ≤ is a statement of logical scope, not computation.

Choose one-tailed when:
- The business hypothesis is explicitly directional
- The team built the change specifically to improve a metric, not just to create a difference
- The cost of missing a positive effect outweighs the cost of missing harm

Choose two-tailed when:
- The direction of the effect is unknown
- You want to detect both improvement and degradation
- The analysis is exploratory rather than confirmatory
- Regulatory or scientific convention requires it

Never switch from one-tailed to two-tailed after seeing results. This is p-hacking. It doubles your effective α and inflates the false positive rate.

---------------------------------------

### Step 3 — Choose the Primary Metric
The primary metric must be chosen before the experiment runs. In this experiment the primary metric is conversion rate — the proportion of users who complete the target action.

Because the outcome is binary (0 or 1 per user), the correct test is a Z-test for proportions applied to raw user-level data. Do not aggregate to daily rates before testing. Aggregation reduces n from hundreds of thousands of users to the number of days in the experiment, losing statistical power and misrepresenting the unit of analysis.

The t-test is appropriate when the outcome is continuous (revenue per user, time on page, sessions per user). At large sample sizes the Z-test and t-test converge mathematically, but at small to medium sample sizes the distinction matters significantly.

----------------------------------------

### Step 4 — Define the Minimum Detectable Effect (MDE)

MDE is the minimum lift that justifies shipping the change, given its engineering cost, design cost, and business risk. It is a business decision, not a statistical one. It must be set before the experiment runs.

MDE is decided collaboratively:
- The Product Manager defines what lift is worth shipping the change for
- The Data Analyst translates that into a number and stress-tests it against historical benchmarks
- Engineering flags maintenance costs that raise the bar for what is worth shipping

Three approaches to setting MDE in industry:
1. The first approach is business viability thresholding. The PM asks what the minimum lift is that justifies the cost of the change. If the new page required two weeks of engineering time, a 0.1pp lift probably does not justify it. A 1-2pp lift might.

2. The second approach is historical benchmarking. Teams that run experiments regularly build a library of past lifts. If the last ten landing page tests moved conversion by 0.5-2pp, design future experiments to detect in that range.

3. The third approach is revenue impact modelling. Work backwards from business impact. At a baseline of 12% conversion across 1M monthly users, a 1pp lift equals 10,000 additional conversions per month. If each conversion is worth $50, that is $500K per month — clearly worth detecting.

Never reverse-engineer MDE to match the sample size you already have. This is intellectually dishonest and produces experiments that are designed to confirm rather than test.

------------------------------------------------------------------

### Step 5 — Run Power Analysis and Determine Required Sample Size
Four inputs are required:
- Baseline conversion rate from historical data or a pilot
- MDE from the business decision in Step 4
- Significance level α (typically 0.05)
- Statistical power 1-β (typically 0.80, meaning 80% chance of detecting a real effect)

At large-traffic companies the question shifts from sample size to experiment duration:
```
Required days = Required n per group / Daily unique users per group
```

Regardless of when the required sample size is hit, always enforce a minimum runtime of one to two full weekly cycles to capture:
- Weekday versus weekend behavioural differences
- Novelty effect decay — users behave differently when first exposed to something new
- Weekly seasonality in the target behaviour (especially important for travel and booking platforms)

----------------------------------------------------------------

### Step 6 — Design the Randomisation and Assignment Mechanism
Users must be randomly assigned to groups with no systematic bias. Checklist before launching:

- Assignment is at the user level, not the session or cookie level where possible
- Each user is assigned to exactly one group for the duration of the experiment
- The assignment mechanism is deterministic — the same user_id always maps to the same group
- No user can opt in or out of a group
- The assignment is independent of any user characteristic correlated with the outcome

After launch, verify randomisation immediately by checking group balance and running a Sample Ratio Mismatch (SRM) check. Do not wait until the experiment ends.

```python
from scipy.stats import chisquare

observed     = [n_treatment, n_control]
total        = sum(observed)
expected     = [total * 0.5, total * 0.5]
stat, p_srm  = chisquare(observed, expected)
```

If p_srm < 0.05, there is a statistically significant imbalance in group sizes. This indicates a bug in the assignment mechanism and the experiment result cannot be trusted regardless of what the primary metric shows.

-----------------------------------------------------------------

## Running the Experiment

### Data Validation Checklist
1. Before running any statistical test, validate the data completely. Statistical results are only as trustworthy as the data they are computed on.
2. Check for null values in all columns. Nulls in `converted` or `group` make rows unusable and may indicate logging failures.
3. Check for duplicate rows. A perfectly duplicate row indicates a data pipeline issue and should be removed.
4. Check for duplicate user_ids. Users appearing more than once were exposed to multiple experiences and their conversion signal is compromised. Remove all rows for any user_id that appears more than once — not just the duplicate row. Keeping one row for a duplicate user implicitly attributes a conversion to one experience when the other may have influenced it.
5. Check group and page alignment. Every control user must have seen old_page. Every treatment user must have seen new_page. Mismatches indicate assignment or logging failures. Quantify the contamination rate. If it exceeds 5%, investigate the root cause before proceeding.
6. Check experiment duration. Confirm the experiment ran for the pre-specified duration and captured the required sample size.
7. Check for temporal anomalies. Plot daily user counts per group. Sudden spikes or drops on specific days indicate external events, marketing campaigns, or logging outages that could confound results.

-----------------------------------------------------------

## Statistical Analysis — Step by Step

### Step 1 — Assumption Checks
Before running the Z-test, verify three assumptions are satisfied.
1. Random assignment is verified through group balance and the SRM check described above. Near-equal group sizes and a non-significant SRM chi-square test confirm randomisation is intact.
2. Independence means each user's conversion does not influence any other user's conversion. For a landing page experiment with no social or network component, independence is satisfied by design. Removing duplicate user_ids addresses the most common independence violation in practice.
3. The normal approximation condition ensures the sampling distribution of the proportion is approximately normal, which is what the Z-test relies on. Check this for each group:

```
n × p ≥ 10  AND  n × (1 - p) ≥ 10 #To check for normalization. Alternatively can use Shaphiro test
```

If either condition fails, switch to Fisher's Exact Test or a bootstrapped proportion test. Both are valid for small samples and make no normality assumptions. At large sample sizes (>10,000 per group) the condition is essentially always satisfied.

### Step 2 — Run the Z-Test for Proportions
Pooled SE is used because the Z-test assumes H0 is true — both groups are assumed to have the same proportion under the null, so you estimate one shared rate from all data combined.
For a one-tailed test with alternative='larger', the p-value is the probability mass in the right tail only:
For a two-tailed test the p-value is the combined probability in both tails:

A p-value above 0.5 in a one-tailed test always means the observed effect went in the opposite direction to the hypothesis.

### Step 3 — Compute the Confidence Interval
The 95% CI on the difference uses unpooled SE because you are no longer assuming H0 is true — you are estimating the true difference with uncertainty bounds:

The CI is always reported as two-sided for stakeholder communication, even when the test is one-tailed. The two-sided CI is more conservative and more intuitive — it directly answers the question of whether any upside is plausible.

The upper bound of the CI is the most business-relevant number. It represents the most optimistic scenario the data supports. If the CI upper bound is below your MDE, the new experience cannot deliver meaningful lift under any statistically plausible scenario. This is the strongest possible business argument for not shipping.

### Step 4 — Compute Effect Size (Cohen's h)
Cohen's h is the standard effect size measure for comparing two proportions. It applies an arcsine transformation to stabilise variance across the proportion scale before taking the difference:

```python
cohen_h = 2 * np.arcsin(np.sqrt(p_treatment)) - 2 * np.arcsin(np.sqrt(p_control))
```

Interpretation benchmarks established by Jacob Cohen:
- Absolute value below 0.2 is small to negligible
- Absolute value between 0.2 and 0.5 is small to medium
- Absolute value between 0.5 and 0.8 is medium to large
- Absolute value above 0.8 is large

At large sample sizes, statistical significance and practical significance diverge. A 0.01pp difference with 10 million users will produce p < 0.05 but Cohen's h near zero. Always report effect size alongside p-value to prevent decisions based on statistically significant but practically meaningless results.

------------------------------------------------------------------------

## Readout and Business Conclusion

A complete readout contains five components. Every component is necessary. Presenting only the p-value is insufficient for a senior analyst.

**Component 1 — Statistical verdict**
State clearly whether you reject or fail to reject H0, and report the exact p-value and Z-statistic. Never say "the null is proven" or "there is no effect" — the correct statement is "we fail to reject H0" or "there is no statistically significant evidence of an effect."

**Component 2 — Practical significance verdict**
Compare the observed effect and CI upper bound to the pre-committed MDE. A result can be statistically significant but practically insignificant (common at large sample sizes) or practically suggestive but statistically inconclusive (common at small sample sizes). Report both verdicts separately.

**Component 3 — Effect size**
Report Cohen's h and its magnitude interpretation. This gives the audience a sense of how big the difference is in standardised terms, independent of sample size.

**Component 4 — Business risk quantification**
Translate the statistical result into business terms. Compute the worst-case conversion loss from the CI lower bound and express it as an estimated number of users or revenue impact per period. Make the cost of shipping anyway concrete.

**Component 5 — Recommendation and next steps**
State the recommendation clearly and without ambiguity. Do not soften the conclusion to spare stakeholder feelings. Then immediately provide a constructive forward path — what to investigate, what to redesign, and what the next experiment should test.

-------------------------------------------------------------------

## Post-Readout — What to Do After the Experiment Closes

### If the result is a definitive null

1. A definitive null result is one where the test was adequately powered, the required sample size was reached, and the CI upper bound is below the MDE. This is not a failed experiment — it is a successful one that answered the question correctly.
   
2. Document the result formally in your experimentation log including the exact hypothesis, test parameters, result, and recommendation. This becomes part of your historical benchmark library for future MDE setting.

3. Conduct a segment analysis to check whether the null result at the population level is masking a positive effect in a specific subgroup. Run this analysis only across pre-specified segments — device type, new versus returning users, geography, traffic source. If a segment shows directional improvement above MDE, design a targeted follow-up experiment for that segment specifically. Never ship globally based on a segment finding from a globally null experiment.

4. Commission qualitative research to understand why the page underperformed. User testing sessions, session recordings, and click heatmaps can identify friction points that quantitative data cannot. Use these findings to form a stronger hypothesis for the next experiment.

5. Redesign the hypothesis with the segment findings and qualitative insights before running the next experiment. A stronger hypothesis produces a more targeted change, which is more likely to produce a detectable effect.

### If the result is positive (reject H0)

Before shipping, verify the following:

1. Confirm the experiment ran for the full pre-specified duration and captured the required sample size. A significant result after only three days may reflect a novelty effect rather than a genuine improvement. Let the experiment run to completion before acting.

2. Check for Sample Ratio Mismatch. Even with a positive result, an SRM indicates the assignment mechanism was compromised and the result may not be valid.

3. Audit guardrail metrics. A positive primary metric does not mean the change is net positive. Check bounce rate, session duration, return visit rate, revenue per user, and any other pre-specified guardrail metrics. If any guardrail is significantly harmed, do not ship without explicit business sign-off on the trade-off.

4. Check segment-level results. A positive overall effect can hide harm in specific segments. Verify that no key segment is significantly worse than control before shipping globally.

5. Run a phased rollout rather than a full immediate ship. Expose 5-10% of production traffic first and monitor for 24-48 hours. Check for infrastructure issues, unexpected interactions with other running experiments, and edge cases not captured in the test environment. If guardrails hold, ramp gradually to 100%.

----------------------------------------------------------------

## Pitfalls and Mistakes to Avoid

### In experiment design

1. Never set MDE after seeing the data. This is reverse-engineering significance and guarantees you will always find what you are looking for. MDE must be locked before data collection begins based purely on business value, not available sample size.

2. Never skip the power analysis. Running an experiment without computing the required sample size means you do not know whether a null result is a true null or simply insufficient data. These require completely different responses.

3. Never run an experiment for less than one full weekly cycle. Behaviour on weekdays differs from weekends in almost every consumer context. A three-day experiment capturing only weekdays is not representative of your full user population.

4. Never assign users at the session or cookie level when user-level randomisation is possible. Session-level assignment allows the same user to be in both groups across sessions, contaminating the comparison.

### In data validation

1. Never silently drop contaminated rows without quantifying and documenting the contamination. Report the contamination rate as part of your methodology. If it exceeds 5%, investigate the root cause.

2. Never keep one row for duplicate user_ids. Both rows must be removed because the user's conversion signal is compromised regardless of which row you keep.

3. Never proceed if SRM is detected. An imbalanced group ratio indicates a bug in the assignment mechanism. The experiment result is not interpretable until the root cause is identified and fixed.

### In statistical analysis

1. Never use a t-test on aggregated daily conversion rates. Aggregation reduces your effective sample size from the number of users to the number of days, inflating variance and producing unreliable p-values. Apply the Z-test to raw user-level binary data.

2. Never report only the p-value. A p-value without a confidence interval and effect size is an incomplete result. With large sample sizes, even negligible effects become statistically significant. Effect size is what tells you whether the result matters to the business.

3. Never interpret p > 0.05 as proof that there is no effect. The correct statement is "we fail to reject H0" — meaning the data does not provide sufficient evidence of an effect. This is not the same as proving the null is true. The distinction matters when communicating to stakeholders who may use a null result to justify inaction without proper investigation.

4. Never change the test type, metric, α, or hypothesis after seeing results. This is p-hacking regardless of how it is framed. All analytical decisions must be locked in a pre-registration document before data collection begins.

5. Never peek at results and stop early without a pre-specified sequential testing framework. Optional stopping — running until you see p < 0.05 then stopping — inflates the false positive rate dramatically. At ten peeks with α = 0.05, the effective false positive rate approaches 40%.

6. Never run multiple metrics and report only the ones that reached significance. This is the multiple comparisons problem. If you test ten metrics and report the two that hit p < 0.05, the probability that at least one is a false positive is above 40%. Apply Bonferroni correction or Benjamini-Hochberg FDR control when testing multiple metrics.

### In the readout

1. Never soften a null result to spare stakeholder feelings. A clear "do not ship" recommendation delivered with evidence and a constructive forward path is more valuable than an ambiguous conclusion that wastes engineering resources on a change that does not work.

2. Never present a positive result as a recommendation to ship immediately. Always verify experiment duration, SRM, guardrail metrics, and segment-level effects before recommending a full ship. Recommend phased rollout as the default.

3. Never let a statistically significant result override guardrail metric violations. A conversion rate improvement that comes with a meaningful increase in bounce rate or drop in revenue per user is not a net positive. The business must explicitly sign off on any trade-off before shipping.

4. Never use uncertainty as a backdoor to justify a predetermined conclusion. If the CI upper bound slightly exceeds MDE but p > 0.05, do not interpret this as a signal to ship. The correct response is to resolve the uncertainty through a follow-up experiment, not to use the most optimistic statistical scenario to bypass the pre-committed decision rule.

### In post-experiment analysis

1. Never run a segment analysis after a null result and report a positive segment finding as if it were the primary result. Post-hoc segment analysis is exploratory by definition. Any finding must be validated in a new pre-specified experiment targeting that segment specifically.

2. Never run the experiment longer after seeing p = 0.07 hoping it crosses 0.05. Running longer is only valid if the experiment has not yet reached the pre-specified required sample size. Running longer because the p-value disappointed you is optional stopping and inflates the false positive rate.

3. Never reuse the same dataset for a follow-up experiment. Users who were already exposed to the treatment have a different baseline response than naive users. Always collect fresh data for each experiment.

-------------------------------------------------------------

## Key Statistical Concepts Reference

**P-value** is the probability of observing a test statistic as extreme as or more extreme than the one computed, assuming the null hypothesis is true. It is not the probability that the null is true, the probability that the result occurred by chance, or the probability that the alternative is true.

**Confidence interval** is the range of values within which the true population parameter is estimated to lie with a given probability. A 95% CI does not mean there is a 95% probability the true value lies in this specific interval — it means that if you repeated the experiment many times, 95% of the constructed intervals would contain the true value.

**Effect size** is a standardised measure of the magnitude of a difference, independent of sample size. Cohen's h is the appropriate measure for comparing two proportions. It applies an arcsine transformation to equalise variance across the proportion scale before computing the difference.

**MDE** is the minimum detectable effect — the smallest true effect the experiment is designed to detect with the pre-specified power. It is a business commitment, not a statistical parameter. Setting MDE is equivalent to pre-committing your definition of practical significance.

**Power** is the probability of correctly rejecting a false null hypothesis. At power = 0.80, you accept a 20% chance of a false negative — missing a real effect. Higher power requires larger sample sizes.

**SRM** (Sample Ratio Mismatch) is a statistically significant imbalance in group sizes relative to the intended split. It indicates a bug in the assignment mechanism and invalidates the experiment result regardless of the primary metric outcome.

**P-hacking** is the inflation of false positive rates through undisclosed analytical flexibility — changing metrics, stopping rules, test types, or segmentations after seeing data. It makes random noise look like signal and at scale leads to shipping features that do not work.

**Novelty effect** is the tendency for users to behave differently simply because something looks new, rather than because it is genuinely better. It inflates treatment performance in the early days of an experiment and decays over time. Enforcing minimum experiment duration and checking the daily trend plot are the primary mitigations.

---

## Results Summary — This Experiment

The experiment ran for 21 days with approximately 143,000 users per group after cleaning. The new landing page produced a conversion rate of 11.87% compared to 12.02% for the old page — a difference of -0.1447 percentage points in the wrong direction.

The one-tailed Z-test produced a Z-statistic of -1.1945 and a p-value of 0.8839, providing no evidence to reject the null hypothesis. The 95% confidence interval on the difference was [-0.3821pp, +0.0927pp]. The upper bound of 0.0927pp represents only 4.6% of the pre-committed MDE of 2pp — meaning even under the most optimistic statistical scenario, the new page cannot deliver meaningful lift. Cohen's h was -0.0045, indicating a negligible effect size.

With 143,000 users per group — approximately 40 times the required sample size to detect a 2pp lift — this is a definitive null result, not an underpowered one. The absence of signal with this much data is itself a strong signal.

Recommendation: Do not ship the new landing page. Conduct pre-specified segment analysis across device type, user type, geography, and traffic source. Commission qualitative research on the new page to identify friction points. Use findings to redesign the hypothesis before the next experiment.
