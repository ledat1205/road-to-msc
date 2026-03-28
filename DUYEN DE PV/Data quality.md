![[Data_Quality_In_Depth_Shopee_Vietnam_2026.docx]]
https://docs.google.com/document/d/1ydbV2ZqO7v0p1g2gshd5-X7pz5bYe1Bp/edit
---

Good correction. Analytics Engineer at Shopee is not a data infrastructure role — it sits closer to the transformation and modeling layer. The statistics that matter are the ones that show up in SQL, dbt models, data quality monitoring, and business metric design — not distributed systems internals.

Let me redo this properly.

---

**The right frame for Analytics Engineer**

Your job is to build the data layer that business teams trust and use. Statistics appears in three practical places:

- **Data quality monitoring** — detecting when your pipelines produce wrong numbers
- **Metric design** — building metrics that are statistically meaningful, not misleading
- **Business analysis support** — helping BI teams interpret results correctly, especially around A/B tests and campaign measurement

---

**1. Sampling — validating ETL transformations cheaply**

When you write a new dbt model or ETL transformation on a 200M row table, you don't validate it by running the full job. You sample first.

**The math — minimum sample size for a proportion:**

$$n = \frac{Z^2 \times p(1-p)}{E^2}$$

|Symbol|Meaning|
|---|---|
|$Z$|1.96 for 95% confidence|
|$p$|expected defect rate — use 0.5 if unknown (most conservative)|
|$E$|acceptable margin of error — 0.01 means ±1%|

**Walking through it:**

You want to check the null rate on a transformed column. You'll accept ±1% error at 95% confidence. You don't know the true null rate so use $p = 0.5$:

$$n = \frac{1.96^2 \times 0.5 \times 0.5}{0.01^2} = \frac{3.8416 \times 0.25}{0.0001} = 9{,}604 \text{ rows}$$

Out of 200M rows, you only need to check ~10,000. That's 0.005% of the data, and statistically you're guaranteed to catch any null rate above 1% with 95% confidence.

**Why $p = 0.5$ is conservative:** the numerator contains $p(1-p)$. This is maximized at $p = 0.5$ (gives 0.25). Any other value gives a smaller sample requirement. So $p = 0.5$ guarantees your sample is large enough regardless of the true rate.

**In SQL — Postgres TABLESAMPLE:**

```sql
-- Bernoulli sampling: each row has 0.1% chance of inclusion
-- Fast, statistically valid, no ORDER BY RANDOM() overhead
SELECT
  COUNT(*) AS sampled_rows,
  COUNT(*) FILTER (WHERE amount IS NULL) AS null_amount,
  ROUND(100.0 * COUNT(*) FILTER (WHERE amount IS NULL) / COUNT(*), 2) AS null_pct
FROM orders TABLESAMPLE BERNOULLI(0.1)  -- 0.1% of rows
WHERE created_at >= NOW() - INTERVAL '7 days';
```

**Where this applies for Analytics Engineer:**

- Validating a new dbt model transformation before full backfill
- Spot-checking data quality on large partitions without full table scan
- Estimating metrics on incoming raw data before pipeline completes
- Building test fixtures — how many records to include in a unit test for a SQL transform

---

**2. Statistical Significance in A/B Testing — supporting the BI team**

The BI team runs A/B tests on campaigns, UI changes, and recommendation tweaks. As the Analytics Engineer you own the data pipeline that feeds the test results. You need to understand the math well enough to build the right tables and flag when the BI team is drawing wrong conclusions.

**The two-proportion Z-test — the workhorse of A/B testing:**

$$Z = \frac{p_1 - p_2}{\sqrt{p(1-p)\left(\frac{1}{n_1} + \frac{1}{n_2}\right)}}$$

|Symbol|Meaning|
|---|---|
|$p_1$|conversion rate in group A (treatment)|
|$p_2$|conversion rate in group B (control)|
|$p$|pooled rate $= \frac{x_1 + x_2}{n_1 + n_2}$ where $x$ = conversions|
|$n_1, n_2$|number of users in each group|
|$Z$|test statistic — compare to 1.96 for 95% significance|

**Walking through the math with real numbers:**

Group A (new recommendation algorithm): 12,400 users, 744 purchases → $p_1 = 0.060$ Group B (control): 12,100 users, 605 purchases → $p_2 = 0.050$

Pooled rate: $$p = \frac{744 + 605}{12400 + 12100} = \frac{1349}{24500} = 0.0551$$

Standard error: $$SE = \sqrt{0.0551 \times 0.9449 \times \left(\frac{1}{12400} + \frac{1}{12100}\right)} = \sqrt{0.0521 \times 0.0001648} = \sqrt{0.00000859} = 0.00293$$

Z-score: $$Z = \frac{0.060 - 0.050}{0.00293} = \frac{0.010}{0.00293} = 3.41$$

Since $3.41 > 1.96$, the result is statistically significant at 95% confidence. The new algorithm genuinely improves conversion, not just by luck.

**Why the Analytics Engineer needs to know this:**

You build the table that feeds this test. If your pipeline double-counts users (uniqueness issue), $n_1$ and $n_2$ are inflated, the standard error shrinks, and the Z-score looks bigger than it should — false significance. If you misattribute users to groups (consistency issue), $p_1$ and $p_2$ are wrong from the start.

The statistical result is only as good as the data engineering behind it.

**In SQL — building an A/B test results table:**

```sql
-- A/B test results pipeline — feed this into BI dashboards
WITH experiment_users AS (
  SELECT
    user_id,
    variant,                    -- 'treatment' or 'control'
    assigned_at
  FROM ab_assignments
  WHERE experiment_id = 'rec_algo_v2'
    AND assigned_at BETWEEN '2024-11-01' AND '2024-11-14'
),
conversions AS (
  SELECT DISTINCT
    e.user_id,
    e.variant,
    1 AS converted
  FROM experiment_users e
  JOIN orders o
    ON e.user_id = o.user_id
    AND o.created_at >= e.assigned_at   -- only count post-assignment orders
    AND o.created_at <= e.assigned_at + INTERVAL '7 days'  -- within window
    AND o.status = 'completed'
),
results AS (
  SELECT
    e.variant,
    COUNT(DISTINCT e.user_id)              AS users,
    COUNT(DISTINCT c.user_id)              AS converters,
    ROUND(COUNT(DISTINCT c.user_id) * 1.0
      / COUNT(DISTINCT e.user_id), 4)      AS conversion_rate
  FROM experiment_users e
  LEFT JOIN conversions c USING (user_id)
  GROUP BY e.variant
)
SELECT
  r.*,
  -- Pooled standard error across both variants
  ROUND(SQRT(
    (SUM(converters) OVER () * 1.0 / SUM(users) OVER ())
    * (1 - SUM(converters) OVER () * 1.0 / SUM(users) OVER ())
    * (1.0/users + 1.0/SUM(users) OVER ())
  ), 6) AS standard_error
FROM results r;
```

---

**3. Confidence Intervals — metric reporting with uncertainty**

One of the most common mistakes in e-commerce BI reporting: a metric is reported as a single number with no uncertainty range. "Conversion rate is 4.2%" — but what's the margin of error? Is that 4.2% ± 0.1% or ± 2%? Very different business implications.

**The formula:**

$$\text{CI} = \hat{p} \pm Z \times \sqrt{\frac{\hat{p}(1-\hat{p})}{n}}$$

|Symbol|Meaning|
|---|---|
|$\hat{p}$|observed proportion (e.g., conversion rate)|
|$Z$|1.96 for 95% CI, 1.645 for 90% CI|
|$n$|sample size|
|$\sqrt{\frac{\hat{p}(1-\hat{p})}{n}}$|standard error of the proportion|

**Walking through it:**

You observe 4.2% conversion rate from 800 users in a small regional campaign:

$$SE = \sqrt{\frac{0.042 \times 0.958}{800}} = \sqrt{\frac{0.04024}{800}} = \sqrt{0.0000503} = 0.00709$$

$$\text{CI} = 0.042 \pm 1.96 \times 0.00709 = 0.042 \pm 0.0139$$

So the true conversion rate is somewhere between **2.81% and 5.59%** with 95% confidence. That's a huge range — the campaign might be doing nothing or doing great. You shouldn't make budget decisions from 800 users.

Now run the same campaign with 50,000 users:

$$SE = \sqrt{\frac{0.042 \times 0.958}{50000}} = \sqrt{0.000000805} = 0.000897$$

$$\text{CI} = 0.042 \pm 1.96 \times 0.000897 = 0.042 \pm 0.00176$$

Now you're at **4.02% to 4.38%** — a tight range. You can make decisions with this.

**In SQL — reporting metrics with confidence intervals:**

```sql
-- Conversion rate with 95% confidence interval per campaign
SELECT
  campaign_id,
  COUNT(DISTINCT user_id)                         AS users,
  COUNT(DISTINCT user_id) FILTER (
    WHERE converted = TRUE
  )                                               AS converters,

  -- Point estimate
  ROUND(AVG(converted::int), 4)                   AS conversion_rate,

  -- 95% confidence interval
  ROUND(AVG(converted::int)
    - 1.96 * SQRT(
        AVG(converted::int) * (1 - AVG(converted::int))
        / COUNT(*)
      ), 4)                                        AS ci_lower,

  ROUND(AVG(converted::int)
    + 1.96 * SQRT(
        AVG(converted::int) * (1 - AVG(converted::int))
        / COUNT(*)
      ), 4)                                        AS ci_upper,

  -- Margin of error — useful for deciding if sample is large enough
  ROUND(1.96 * SQRT(
    AVG(converted::int) * (1 - AVG(converted::int))
    / COUNT(*)
  ), 4)                                           AS margin_of_error

FROM campaign_users
GROUP BY campaign_id
ORDER BY users DESC;
```

**Why Analytics Engineers need to build CI into their tables:** if you give BI teams a single conversion rate number, they will compare it to last month's number and declare a trend. If both numbers have a margin of error of ±2% and the observed difference is 0.8%, they're looking at noise. Building CI columns into your metric tables prevents this class of wrong business decision.

---

**4. Regression to the Mean — the most dangerous misinterpretation in campaign analytics**

This is not a formula you apply — it's a statistical trap you need to recognize and actively protect your pipeline outputs from being misused for.

**The concept:**

If you target a campaign at users who performed poorly last month (low GMV, lapsed buyers), they will almost certainly perform better this month — and most of that improvement has nothing to do with your campaign. Users who perform unusually poorly in one period naturally move toward the average in the next period by chance alone.

**The math:**

$$x_{predicted} = \mu + r \times (x_{observed} - \mu)$$

|Symbol|Meaning|
|---|---|
|$\mu$|population mean (e.g. average monthly GMV per user)|
|$r$|correlation between consecutive months' performance (typically 0.4–0.7)|
|$x_{observed}$|user's performance in the selection period|
|$x_{predicted}$|expected performance next period _with zero intervention_|

**Walking through it:**

Average monthly GMV per user = ₫500,000. Correlation between month 1 and month 2 GMV = 0.6. You target users who spent ₫0 last month.

Expected GMV next month with no campaign: $$x_{predicted} = 500000 + 0.6 \times (0 - 500000) = 500000 - 300000 = ₫200{,}000$$

So a "lapsed" user who spent ₫0 last month is expected to naturally spend ₫200,000 this month — before your campaign does anything. If your campaign reports that these users spent ₫220,000 on average, the true campaign lift is only ₫20,000, not ₫220,000.

**Why the Analytics Engineer needs to know this:**

You design the attribution table. If you build `campaign_attributed_gmv = gmv_during_campaign_period` without a control group, you're baking regression-to-the-mean inflation into every campaign report in the company. The fix is always the same: require a randomly held-out control group of the same user cohort, and report **incremental** lift, not absolute performance.

```sql
-- Correct incremental lift calculation
-- Requires experiment design with control group
WITH cohort_stats AS (
  SELECT
    variant,                          -- 'treatment' or 'control'
    COUNT(DISTINCT user_id)           AS users,
    AVG(gmv_post_campaign)            AS avg_gmv,
    STDDEV(gmv_post_campaign)         AS sd_gmv
  FROM campaign_cohorts
  GROUP BY variant
)
SELECT
  MAX(avg_gmv) FILTER (WHERE variant = 'treatment') AS treatment_gmv,
  MAX(avg_gmv) FILTER (WHERE variant = 'control')   AS control_gmv,

  -- True incremental lift — strips out regression to mean
  MAX(avg_gmv) FILTER (WHERE variant = 'treatment')
  - MAX(avg_gmv) FILTER (WHERE variant = 'control') AS incremental_lift_per_user,

  -- Lift as % of control (how much better vs doing nothing)
  ROUND(
    (MAX(avg_gmv) FILTER (WHERE variant = 'treatment')
     - MAX(avg_gmv) FILTER (WHERE variant = 'control'))
    / NULLIF(MAX(avg_gmv) FILTER (WHERE variant = 'control'), 0) * 100
  , 2)                                              AS lift_pct
FROM cohort_stats;
```

---

**5. Percentile Metrics — building robust KPIs that don't lie**

Most BI dashboards report averages. Average order value, average session duration, average seller response time. Averages are easy to compute but statistically fragile — one massive B2B order can pull the average up and make the entire metric misleading for the 99% of typical orders.

**The problem with averages — a concrete example:**

10 orders with amounts: ₫100k, ₫120k, ₫90k, ₫110k, ₫95k, ₫105k, ₫115k, ₫100k, ₫108k, ₫5,000k

Average = ₫594,300 — completely unrepresentative of the 9 normal orders. Median (p50) = ₫107,500 — accurate picture of a typical order.

**The right percentile metrics to build for e-commerce KPIs:**

```sql
-- Robust seller performance metrics using percentiles
-- p50 = typical experience, p95 = tail experience, p99 = worst cases
SELECT
  DATE_TRUNC('week', created_at)       AS week,
  seller_id,

  -- Order value distribution
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY amount) AS p50_order_value,
  PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY amount) AS p90_order_value,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY amount) AS p95_order_value,

  -- Fulfillment time distribution (hours from order to ship)
  PERCENTILE_CONT(0.50) WITHIN GROUP (
    ORDER BY EXTRACT(EPOCH FROM (shipped_at - created_at))/3600
  ) AS p50_fulfillment_hours,

  PERCENTILE_CONT(0.95) WITHIN GROUP (
    ORDER BY EXTRACT(EPOCH FROM (shipped_at - created_at))/3600
  ) AS p95_fulfillment_hours,   -- SLA breach metric

  -- Average for comparison — show alongside percentiles
  ROUND(AVG(amount)) AS avg_order_value,

  -- Ratio of p90 to p50 — measures distribution skewness
  -- High ratio = lots of outlier orders pulling the average up
  ROUND(
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY amount)
    / NULLIF(PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY amount), 0)
  , 2) AS p90_p50_ratio

FROM orders
WHERE status = 'completed'
  AND shipped_at IS NOT NULL
GROUP BY 1, 2
ORDER BY 1 DESC, p95_fulfillment_hours DESC;
```

**The statistical insight behind p90/p50 ratio:**

If p90/p50 = 1.2, the distribution is compact — most orders are similar in value and the average is reliable. If p90/p50 = 8.0, the distribution is heavily skewed — the average is meaningless and you need to segment or cap before reporting. This ratio is a single-number summary of distribution skewness that doesn't require computing skewness directly.

**Where Analytics Engineers apply this:**

- SLA monitoring: "95% of orders ship within 24 hours" is a p95 metric — more meaningful than average shipping time
- Seller performance tiers: rank sellers by p50 fulfillment time, not average (robust to occasional delays)
- Pricing analytics: report p50 order value per category alongside average to flag when averages are being distorted
- Customer experience: "typical checkout time" should be p50, not average

---

**The meta-point for your interview**

If an interviewer asks "how do you make sure the metrics in our dashboards are reliable?" — the senior Analytics Engineer answer covers three layers:

First, the **pipeline layer** — your SQL and dbt models are correct, deduplicated, and cover the right time windows. This is the engineering foundation.

Second, the **statistical validity layer** — the metrics are designed to be robust. You use percentiles not just averages, you report confidence intervals not just point estimates, you require control groups for campaign measurement.

Third, the **interpretation layer** — you actively flag misinterpretations before they reach business decisions. You know when a result is regression to the mean, when a sample is too small to act on, and when two metrics that look different are actually within each other's confidence intervals.

Most candidates only cover the first layer. Covering all three is what gets you the offer.