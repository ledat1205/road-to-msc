![[Data_Quality_In_Depth_Shopee_Vietnam_2026.docx]]
https://docs.google.com/document/d/1ydbV2ZqO7v0p1g2gshd5-X7pz5bYe1Bp/edit
---

**The fraud landscape first — what you're actually detecting**

Affiliate and ad fraud comes in a few main flavors:

- **Click fraud** — bots or click farms generating fake clicks to inflate affiliate commissions
- **Install fraud** — fake app installs triggered programmatically to earn CPI (cost-per-install) payouts
- **Conversion fraud** — fake orders placed and then refunded/cancelled after commission is paid
- **Cookie stuffing** — affiliate cookie dropped without a real click, stealing attribution
- **Ad impression fraud** — bots loading ads in hidden iframes, inflating CPM costs

Each has a distinct statistical fingerprint.

---

**1. Z-Score — catching volume spikes per affiliate/publisher**

The most immediate signal. A legitimate affiliate's daily click volume follows a relatively stable distribution. A fraudulent one suddenly sends 50,000 clicks on a Tuesday with no corresponding campaign change.

$$Z = \frac{x_{today} - \mu_{rolling}}{\sigma_{rolling}}$$

Apply this **per affiliate**, not globally. A Z-score on total platform clicks misses a single bad actor who is a small fraction of total volume. You need per-entity baselines.

What you track per affiliate daily:

- Click volume
- Click-to-install rate
- Install-to-order conversion rate
- Refund rate on attributed orders
- Average time-to-convert (how fast after click does conversion happen)

A fraudulent affiliate will show a Z-score spike on click volume accompanied by a Z-score _drop_ on conversion rate — more clicks, same or fewer real orders. That combination is the key signal.

---

**2. IQR — detecting impossible session metrics**

Fraud bots are fast. Real users browse, hesitate, read reviews. Bots click → convert in seconds.

Apply IQR fences to **time-to-convert** (seconds between affiliate click and order placement):

$$\text{Lower fence} = Q_1 - 3 \times \text{IQR}$$

Legitimate users converting in under 10 seconds is statistically rare. A cluster of conversions with time-to-convert of 2–5 seconds, all from the same affiliate, all falling below the lower fence — that's bot traffic.

Also apply IQR to:

- **Session duration** — bots have near-zero or suspiciously uniform session times
- **Pages visited per session** — bots often hit exactly 1 page (the landing page with the tracking pixel)
- **Order value** — fraud orders often cluster at minimum threshold to trigger commission (e.g., all orders exactly ₫200,000 — the minimum qualifying amount)

That last one is subtle and important. Fraudsters learn your commission threshold and script orders to just clear it. IQR on order value per affiliate will show a spike at the threshold with unusually low variance — real customers don't all order the same amount.

---

**3. PSI — detecting when an affiliate's traffic character changes**

A legitimate affiliate that suddenly switches to fraud traffic will show a distribution shift in their traffic features — device types, time-of-day patterns, geographic spread, OS versions.

$$\text{PSI} = \sum_{i=1}^{n}(A_i - E_i) \times \ln\left(\frac{A_i}{E_i}\right)$$

Compute PSI on a feature like **hour-of-day distribution** of clicks. Real human traffic has a circadian rhythm — peaks during waking hours, troughs at 3am. Bot traffic is either flat across 24 hours (naive bots) or suspiciously concentrated in off-hours.

If an affiliate's hour-of-day PSI jumps from 0.05 (stable) to 0.35 (significant drift) week-over-week, their traffic character has fundamentally changed. That warrants investigation even before any rule-based threshold is breached.

Apply PSI to per-affiliate distributions of:

- Hour of click
- Device type mix (mobile vs desktop)
- Province/city spread of clicks
- Browser/OS mix
- Screen resolution distribution (bots often use default/headless browser resolutions)

---

**4. Chi-Square — geographic and device anomalies**

Legitimate affiliate traffic for a Vietnam-targeted campaign should have a sensible geographic distribution — mostly HCMC, Hanoi, Da Nang, with some spread across provinces. If an affiliate sends 94% of clicks from one province with 3% of Vietnam's population, that's statistically improbable.

$$\chi^2 = \sum_{i=1}^{k} \frac{(O_i - E_i)^2}{E_i}$$

Where $O_i$ is the observed proportion of clicks from province $i$ for this affiliate, and $E_i$ is the expected proportion based on either population distribution or the platform's baseline traffic mix.

The same test applies to device distribution. If the platform average is 78% mobile, 22% desktop, and an affiliate sends 99% desktop traffic — the chi-square test quantifies how unlikely that is by chance.

This is powerful because a fraudster can fake individual data points but it's hard to fake an entire distribution that matches organic platform traffic.

---

**5. Benford's Law — commission and conversion amount fabrication**

When fraudsters generate fake conversion events programmatically, they tend to produce amounts that are either suspiciously uniform or don't follow Benford's distribution.

$$P(d) = \log_{10}\left(1 + \frac{1}{d}\right)$$

Apply to order amounts attributed to each affiliate. Legitimate e-commerce order amounts follow Benford's Law closely — organic customer behavior produces naturally distributed leading digits. Scripted fake orders almost never do, because the script either uses fixed amounts, random uniform amounts, or threshold-hugging amounts — all of which violate Benford's distribution.

If an affiliate's attributed order amounts show $\chi^2$ significantly above the Benford expectation, it's a fabrication signal worth flagging.

---

**6. SPC run rules — catching slow ramp-up fraud**

This is the most sophisticated and the one most people miss. Professional fraud operations don't start at full volume — they ramp up slowly to stay under single-day anomaly thresholds. Z-score on any given day looks fine. But the trend over 2–3 weeks is systematically increasing.

SPC run rules catch this:

- 6 consecutive days of increasing click volume from an affiliate → alert, even if no single day triggers a Z-score alert
- 8 consecutive days of conversion rate below the affiliate's own historical mean → alert

The math: the probability of 8 consecutive points on the same side of the mean by chance is $(0.5)^8 = 0.4%$ — statistically significant without needing a single extreme value.

This is particularly relevant for affiliate fraud because fraudsters deliberately stay below per-day thresholds while accumulating commission over weeks.

---

**7. Correlation-based detection — the ratio signal**

This isn't a single formula but a family of ratio metrics that should remain stable for legitimate affiliates:

$$\text{Click-to-install rate} = \frac{\text{installs}}{\text{clicks}}$$

$$\text{Install-to-order rate} = \frac{\text{orders}}{\text{installs}}$$

$$\text{Commission-to-GMV ratio} = \frac{\text{commission paid}}{\text{GMV attributed}}$$

For a legitimate affiliate, these ratios are relatively stable over time. Fraud causes specific ratio breakdowns:

- Click fraud: clicks spike, install rate collapses
- Install fraud: installs spike, order rate collapses
- Conversion fraud: orders spike, refund rate spikes after commission cutoff date

Track these ratios per affiliate daily and apply Z-score to the ratios themselves, not just raw counts. The ratio anomaly is often cleaner than the volume anomaly because it's self-normalizing.

---

**How these stack together in a real system**

In practice you'd build a scoring system rather than a binary flag. Each statistical signal contributes a score, and an affiliate exceeds a threshold only when multiple signals fire together:

|Signal|Score if triggered|
|---|---|
|Click volume Z-score > 3|+20|
|Conversion rate Z-score < -2|+25|
|Time-to-convert IQR below lower fence > 30% of sessions|+30|
|Hour-of-day PSI > 0.2|+20|
|Geographic chi-square p < 0.01|+15|
|SPC run rule on 6-day trend|+20|
|Benford's deviation significant|+15|

Affiliates scoring above 50 get flagged for manual review. Above 80 get auto-suspended pending investigation. This reduces false positives — a single signal firing could be a legitimate campaign change, but five signals firing simultaneously is a very different story.

---

The deeper point: fraudsters can game any single check if they know it exists. But gaming Z-score on volume while simultaneously maintaining a natural hour-of-day distribution, a believable geographic spread, Benford-compliant order amounts, and stable conversion ratios is extremely hard to do at scale. The power of layering statistical methods is that it raises the cost of fraud beyond what most operations find profitable.

Want me to build out the SQL and Python implementation for this scoring system as a document?