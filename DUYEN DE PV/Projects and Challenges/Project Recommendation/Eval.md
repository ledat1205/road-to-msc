Good framing — evaluation in MLOps is not a one-time step, it's a **gate system** that runs continuously across the model lifecycle. Let me rewrite it properly.

---

## Evaluation as a Gate System

The core MLOps principle: **no model reaches production without passing explicit gates**. Each gate is automated, logged in MLflow, and blocks promotion if thresholds aren't met.

```
Training run
     ↓
Gate 1: Offline evaluation      ← blocks promotion to Staging
     ↓
Gate 2: Shadow mode             ← runs in production without serving results
     ↓
Gate 3: Canary deployment       ← serves small % of traffic
     ↓
Gate 4: Full A/B test           ← measures business impact
     ↓
Production
     ↓
Gate 5: Continuous monitoring   ← ongoing, triggers rollback or retrain
```

---

## Gate 1 — Offline Evaluation (Staging Gate)

Runs automatically at the end of every training DAG in Airflow. Logged to MLflow. Blocks promotion if any threshold fails.

**What gets evaluated:**

For Two-Tower (Home):

- Retrieval quality — Recall@K, NDCG@K, MRR on time-based holdout
- Segmented by user tier (new / casual / power) and item tier (head / torso / long tail)
- Compared against current production model — must not regress

For LightFM + SBERT (PDP):

- Retrieval quality on held-out co-view pairs
- SBERT: proxy metric (automatic) + human evaluation sample (quarterly)
- Coverage — % of catalog with >= 10 similar products in KVDal
- Intra-list diversity — top-10 results shouldn't all be identical products

**Train/serve consistency check — always part of Gate 1:**

This is the MLOps-specific addition most teams skip. Before any model ships, you verify that what the model trained on matches what it will see at serving time:

- Compare session feature distributions between BigQuery reconstruction and Redis
- Compare batch feature distributions between Feast offline store and Feast online store
- Flag any feature with null rate spike or distribution drift above threshold

If skew is detected here — before the model ships — you can fix the pipeline rather than debugging a live production issue.

**Promotion logic:**

```
IF recall@50 >= production_model_recall@50 × 0.98   # allow 2% regression tolerance
AND coverage >= 0.90
AND no feature with KS p-value < 0.05
AND human eval score >= 0.6 (quarterly check)
THEN promote to Staging
ELSE fail DAG, alert, block deployment
```

All thresholds and results logged to MLflow model registry with the model version.

---

## Gate 2 — Shadow Mode

The new model runs **in parallel with production** for 3–5 days but its results are not served to users. You compare outputs silently.

**What you're checking:**

- Do the two models retrieve substantially different candidate sets? High divergence isn't necessarily bad but warrants investigation.
- Does the new model's latency profile match expectations under real traffic? ONNX inference time, FAISS search time — offline benchmarks don't always reflect production load.
- Does the new model produce any null or degenerate outputs (empty candidate sets, all identical items) on real requests that synthetic eval didn't surface?
- Are the feature values arriving at the new model consistent with what it trained on? This is the live version of the train/serve skew check.

**Why shadow mode before canary:**

Gate 1 uses synthetic eval traffic. Shadow mode uses real production traffic — real user feature distributions, real session patterns, real edge cases. It catches issues that clean eval datasets miss: users with unusual browsing histories, products with missing features, cold-start edge cases.

---

## Gate 3 — Canary Deployment

Route **5% of traffic** to the new model. Monitor for 24–48 hours before expanding.

**What you're watching:**

Hard guardrails — automatic rollback if breached:

- Null recommendation rate (% requests returning < 5 items) increases > 2x
- P95 serving latency exceeds budget (> 80ms Home, > 10ms PDP)
- Error rate on model inference spikes

Soft signals — human review required:

- CTR on recommendations drops > 5% vs control
- Add-to-cart rate drops

**Why canary before full A/B:**

Full A/B tests need 2+ weeks for statistical significance. Canary gives you fast feedback on catastrophic failures within hours. A model that crashes 5% of requests will show immediately in error rates — you don't need to wait for a two-week experiment to catch that.

---

## Gate 4 — Full A/B Test

Now you're measuring **business impact**, not just system health.

**Experiment design:**

Traffic split at user level (not request level) so each user consistently sees one variant. Deterministic assignment by hashing customer_id.

```
Control (10%):   current production model
Variant A (10%): new model
Holdout (80%):   not in experiment
```

**Metric hierarchy:**

Primary guardrail — conversion rate and GMV per session. This is what the business cares about. Everything else is diagnostic.

Secondary — add-to-cart rate. Strong purchase intent signal, faster to accumulate than conversion.

Tertiary — CTR. Easy to move, easy to game. Never the primary decision metric.

Quality guardrail — return rate and cancellation rate. These lag by 2–3 weeks. Run the experiment long enough to capture them. A model that improves CTR by misleading users about products will show up here.

**Segmentation that matters for your architecture:**

For Home — break results by session depth. Phase 2 (streaming session features) should outperform Phase 1 specifically in mid and late session (pageviews 5+). If there's no difference at session depth > 5, the streaming infrastructure isn't justified by the results.

For PDP — break results by catalog tier. SBERT's win over LightFM-only should be concentrated in long-tail products. If the improvement is uniform across head/torso/tail, something unexpected is happening.

**Minimum experiment duration:** 2 weeks. Non-negotiable. Weekly seasonality (weekday vs weekend) in e-commerce is significant enough to invalidate shorter experiments. Don't call early even if early results look strong — novelty effect inflates early CTR for any new recommendation surface.

**Decision criteria:**

```
Ship if:
  primary metric (GMV/session) improvement is statistically significant
  AND quality guardrail (return rate) does not degrade
  AND latency guardrails hold

Do not ship if:
  primary metric is flat even if CTR improved
    → model surfaces clickable but non-converting products
  return rate increases
    → model misleads users about product quality
  long-tail metrics don't improve for SBERT variant
    → SBERT not adding value beyond LightFM alone
```

---

## Gate 5 — Continuous Monitoring (Post-Production)

Evaluation doesn't end at deployment. This runs perpetually and triggers rollback or retraining automatically.

**Three layers of monitoring:**

**Data layer — pipeline health:**

Runs daily as part of the dbt/Feast pipeline:

- Null rate per feature in Feast online store
- Feature distribution drift vs 30-day baseline (PSI — Population Stability Index)
- Feast materialization lag — are batch features stale?
- Redis session hit rate — are session features being written correctly?
- KVDal coverage rate — what % of products have similarity lists?
- Trending channel health — are Redis sorted sets being updated?

PSI is preferred over KS test here because it gives a continuous score rather than a p-value, making it easier to set actionable thresholds:

```
PSI < 0.1   → no action needed
PSI 0.1–0.2 → monitor closely
PSI > 0.2   → alert, investigate, likely retrain needed
```

**Model layer — serving health:**

Near-real-time, running on streaming metrics:

- FAISS average similarity score — sudden drop suggests embedding quality regression
- Recommendation null rate — % requests returning fewer than minimum items
- P50/P95/P99 latency per API (Home and PDP separately)
- ONNX inference time
- Trending channel contribution rate — % of final recommendations coming from Redis vs FAISS

**Business layer — outcome metrics:**

Computed hourly from click and purchase event streams:

- Recommendation CTR — 1-hour rolling window vs 7-day baseline
- Add-to-cart rate from recommendations
- Null recommendation exposure rate — % of users seeing incomplete widgets

**Automated responses:**

```
Latency P95 > budget          → circuit breaker → fallback to popularity baseline
Null recommendation rate > 2x → alert on-call
Feature PSI > 0.2             → trigger unscheduled retrain
Business metric drop > 10%    → alert, pause experiment if running
Model inference error spike   → rollback to previous model version
```

**Retraining triggers:**

Not just time-based. Retraining is triggered by signals:

```
Weekly schedule               → standard retrain (planned)
Feature drift PSI > 0.2       → unscheduled retrain
Online metric sustained drop  → retrain after investigation confirms model cause
Catalog structure change       → retrain (new categories break existing embeddings)
New products > 10% of catalog → re-encode item tower + rebuild FAISS
                                 (no full retrain needed)
```

The distinction between full retrain and re-encode is important operationally — re-encoding new products into FAISS nightly is cheap and automated. Full retraining is expensive and should only trigger when the model itself is stale, not just the index.

---

## How to present this in the interview

Frame it as a pipeline, not a checklist:

> "Evaluation isn't a step that happens once before deployment. It's a gate system — offline evaluation gates promotion to staging, shadow mode catches real-traffic edge cases before users see them, canary catches catastrophic failures fast, A/B measures actual business impact over two weeks, and continuous monitoring detects when the production model starts to degrade so we retrain before users notice. The goal is that no human needs to manually decide whether a model is safe to ship — the gates make that decision automatically based on thresholds we've agreed on upfront."

That framing shows MLOps maturity — you understand that the hard problem isn't training a good model, it's building a system that reliably ships good models repeatedly.