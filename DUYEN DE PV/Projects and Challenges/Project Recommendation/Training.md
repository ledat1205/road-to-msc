Let me walk through both training pipelines in full detail.

---

## Home: Two-Tower Training

### What you're trying to learn

The model learns to map users and items into the same 64-dim embedding space such that **users are close to items they'll interact with**. The training signal is implicit feedback — a PDP view is a positive signal that user U is interested in item I.

---

### Step 1 — Build the training dataset from BigQuery

The core table is `pdp_view_raw`. Each row is one (user, item, timestamp) interaction.

sql

```sql
WITH raw_interactions AS (
  SELECT
    customer_id,
    product_id,
    session_id,
    event_time,
    primary_cate_id,
    event_type
  FROM `tiki-dwh.recommendation.pdp_view_raw_*`
  WHERE _TABLE_SUFFIX BETWEEN
    FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY))
    AND FORMAT_DATE('%Y%m%d', CURRENT_DATE)
    AND customer_id IS NOT NULL  -- exclude anonymous
),

-- time-based train/eval split
split AS (
  SELECT *,
    CASE
      WHEN event_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP, INTERVAL 3 DAY)
      THEN 'eval'
      ELSE 'train'
    END AS split
  FROM raw_interactions
)
```

**Why time-based split, not random:** if you split randomly, a user's future interactions leak into training. The model memorizes rather than generalizes. Time split forces the model to predict future behavior from past behavior — which is exactly the serving scenario.

---

### Step 2 — Reconstruct session features

At serving time, session features come from Redis. During training, you reconstruct them from the raw event log using window functions:

sql

```sql
session_features AS (
  SELECT
    customer_id,
    product_id,
    session_id,
    event_time,
    primary_cate_id,
    split,

    -- how many products viewed before this event in the same session
    COUNT(*) OVER (
      PARTITION BY customer_id, session_id
      ORDER BY event_time
      ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
    ) AS session_view_count,

    -- ATCs before this event in the same session
    SUM(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) OVER (
      PARTITION BY customer_id, session_id
      ORDER BY event_time
      ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
    ) AS session_atc_count,

    -- most viewed category so far in this session
    APPROX_TOP_COUNT(primary_cate_id, 1) OVER (
      PARTITION BY customer_id, session_id
      ORDER BY event_time
      ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
    )[OFFSET(0)].value AS session_top_cate

  FROM split
),

-- previous session features
session_summaries AS (
  SELECT
    customer_id,
    session_id,
    COUNT(*) AS total_views,
    APPROX_TOP_COUNT(primary_cate_id, 1)[OFFSET(0)].value AS top_cate,
    MIN(event_time) AS session_start
  FROM split
  GROUP BY customer_id, session_id
),

with_prev_session AS (
  SELECT
    s.*,
    LAG(total_views) OVER (
      PARTITION BY customer_id ORDER BY session_start
    ) AS last_session_view_count,
    LAG(top_cate) OVER (
      PARTITION BY customer_id ORDER BY session_start
    ) AS last_session_top_cate
  FROM session_summaries
)
```

The reconstructed values won't be byte-identical to Redis — but they're statistically consistent. Same distributions, same meaning. The model learns to use session features correctly either way.

---

### Step 3 — Join batch features

Join each (user, item) interaction with the batch features that were valid _at that point in time_. This is the point-in-time correctness problem — you don't want to use today's feature values for an interaction that happened 20 days ago.

sql

```sql
training_data AS (
  SELECT
    sf.customer_id,
    sf.product_id,
    sf.event_time,
    sf.split,

    -- session features (reconstructed)
    sf.session_view_count,
    sf.session_atc_count,
    sf.session_top_cate,
    ps.last_session_view_count,
    ps.last_session_top_cate,
    IFNULL(ps.last_session_view_count >= 5, FALSE) AS last_session_was_deep,

    -- batch user features (from Feast offline store / BQ snapshot)
    uf.age_group,
    uf.gender,
    uf.region,
    uf.num_pdp_views_d30,
    uf.num_atc_d30,
    uf.num_purchases_d365,
    uf.primary_cate_id AS top_cate,
    -- ... remaining 18 user batch features

    -- batch item features
    pf.primary_cate_id,
    pf.sale_price,
    pf.avg_rating_all,
    pf.num_views_d30,
    pf.ctr_d30,
    -- ... remaining 15 item batch features

    -- request features (derived from event_time)
    EXTRACT(HOUR FROM sf.event_time) AS hour_of_day,
    EXTRACT(DAYOFWEEK FROM sf.event_time) AS day_of_week,
    EXTRACT(DAYOFWEEK FROM sf.event_time) IN (1, 7) AS is_weekend

  FROM session_features sf
  LEFT JOIN with_prev_session ps
    ON sf.customer_id = ps.customer_id
    AND sf.session_id = ps.session_id
  LEFT JOIN customer_features_snapshot uf  -- point-in-time snapshot
    ON sf.customer_id = uf.customer_id
  LEFT JOIN product_features pf
    ON sf.product_id = pf.product_id
)
```

---

### Step 4 — Model architecture and loss

python

```python
class UserTower(nn.Module):
    def __init__(self):
        super().__init__()
        self.cate_embedding = nn.Embedding(num_cates, 8)
        self.platform_embedding = nn.Embedding(4, 4)
        self.fc1 = nn.Linear(28, 128)
        self.fc2 = nn.Linear(128, 64)

    def forward(self, batch_features, session_features, request_features):
        # embed categorical features
        cate_emb = self.cate_embedding(session_features['session_top_cate'])
        platform_emb = self.platform_embedding(request_features['platform'])

        x = torch.cat([
            batch_features,    # 18 dims
            session_features,  # 6 dims (including embedded cates)
            request_features,  # 4 dims
        ], dim=-1)             # → 28 dims total

        x = F.relu(self.fc1(x))
        x = self.fc2(x)
        return F.normalize(x, dim=-1)  # L2 normalize → unit sphere

class ItemTower(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(15, 128)
        self.fc2 = nn.Linear(128, 64)

    def forward(self, item_features):
        x = F.relu(self.fc1(item_features))
        x = self.fc2(x)
        return F.normalize(x, dim=-1)
```

**Loss — in-batch negatives:**

python

```python
def training_step(batch):
    user_emb = user_tower(
        batch['user_batch_features'],
        batch['session_features'],
        batch['request_features']
    )  # [B, 64]

    item_emb = item_tower(
        batch['item_features']
    )  # [B, 64]

    # similarity matrix: every user vs every item in batch
    logits = torch.matmul(user_emb, item_emb.T)  # [B, B]
    logits = logits / temperature  # temperature scaling

    # diagonal = true positives, off-diagonal = implicit negatives
    labels = torch.arange(B)
    loss = F.cross_entropy(logits, labels)
    return loss
```

With batch size B=256, you get 256 positives and 65,280 implicit negatives per step — no explicit negative sampling needed.

---

### Step 5 — Export

After training:

python

```python
# export user tower as ONNX (for serving)
torch.onnx.export(user_tower, sample_user_input, "user_tower.onnx")

# pre-compute all item embeddings
all_item_embeddings = []
for batch in item_dataloader:
    emb = item_tower(batch)
    all_item_embeddings.append(emb)

item_embeddings = torch.cat(all_item_embeddings)  # [N_items, 64]

# build FAISS index
index = faiss.IndexFlatIP(64)  # inner product = cosine on unit sphere
index.add(item_embeddings.numpy())
faiss.write_index(index, "item_index.faiss")
```

The item tower gets **discarded after this**. Only the ONNX user tower and FAISS index go to serving.

---

## PDP: LightFM + Sentence-BERT Training

Two separate models, combined at write time into KVDal.

---

### Part A — LightFM interaction similarity

**What you're learning:** which products tend to be viewed together in the same session — capturing behavioral co-occurrence.

**Step 1 — Build the session co-view matrix:**

sql

```sql
-- product pairs co-viewed in same session
co_view_pairs AS (
  SELECT
    a.product_id AS product_a,
    b.product_id AS product_b,
    COUNT(DISTINCT a.session_id) AS session_count
  FROM pdp_view_raw a
  JOIN pdp_view_raw b
    ON a.session_id = b.session_id
    AND a.product_id < b.product_id  -- avoid duplicates
    AND b.event_time BETWEEN a.event_time
        AND TIMESTAMP_ADD(a.event_time, INTERVAL 10 MINUTE)
  WHERE a.event_date >= DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY)
    AND a.primary_cate_id = b.primary_cate_id  -- same cate for similar
  GROUP BY 1, 2
  HAVING session_count >= 10  -- filter noise
)
```

For complementary products, remove the same-cate filter and join against `complementary_cate_pairs`.

**Step 2 — Build interaction matrix and train LightFM:**

python

```python
from lightfm import LightFM
from scipy.sparse import coo_matrix

# build sparse session × product matrix
# rows = sessions, cols = products, values = view count
interactions = coo_matrix(
    (data['count'], (data['session_idx'], data['product_idx'])),
    shape=(n_sessions, n_products)
)

model = LightFM(
    no_components=64,
    loss='warp',        # WARP loss: optimizes ranking directly
    learning_rate=0.05,
    item_alpha=1e-6,    # L2 regularization
)

model.fit(
    interactions,
    epochs=30,
    num_threads=8,
    verbose=True
)

# extract product embeddings
item_embeddings = model.item_embeddings  # [n_products, 64]

# compute pairwise similarity for top-N per product
# done in batches to avoid O(n²) memory
for product_id in all_products:
    query_emb = item_embeddings[product_idx]
    scores = item_embeddings @ query_emb  # cosine similarity
    top_n = np.argsort(scores)[::-1][:200]
    # write to staging table
```

**Why WARP loss over BPR:** WARP (Weighted Approximate-Rank Pairwise) directly optimizes for the rank of the positive item — it samples negatives until it finds one that violates the ranking, then updates. This is more aligned with retrieval quality than BPR which just maximizes the positive-negative margin regardless of rank.

---

### Part B — Sentence-BERT title similarity

**What you're learning:** semantic similarity from product names — captures cases where co-view data is sparse (new products, niche items).

**Step 1 — Fetch training data:**

sql

```sql
-- anchor-positive pairs from co-views
SELECT
  p_anchor.product_name AS anchor_text,
  p_positive.product_name AS positive_text,
  p_anchor.primary_cate_id
FROM co_view_pairs cvp
JOIN dim_product_full p_anchor ON cvp.product_a = p_anchor.product_id
JOIN dim_product_full p_positive ON cvp.product_b = p_positive.product_id
WHERE p_anchor.product_name IS NOT NULL
  AND p_positive.product_name IS NOT NULL

UNION ALL

-- augment with purchase pairs (stronger signal)
SELECT
  p_a.product_name,
  p_b.product_name,
  p_a.primary_cate_id
FROM purchase_pairs pp
JOIN dim_product_full p_a ON pp.product_a = p_a.product_id
JOIN dim_product_full p_b ON pp.product_b = p_b.product_id
```

**Hard negatives — same category, never co-viewed:**

sql

```sql
hard_negatives AS (
  SELECT
    p_anchor.product_name AS anchor_text,
    p_neg.product_name AS negative_text
  FROM dim_product_full p_anchor
  JOIN dim_product_full p_neg
    ON p_anchor.primary_cate_id = p_neg.primary_cate_id
    AND p_anchor.product_id != p_neg.product_id
  WHERE NOT EXISTS (
    SELECT 1 FROM co_view_pairs cvp
    WHERE (cvp.product_a = p_anchor.product_id AND cvp.product_b = p_neg.product_id)
    OR    (cvp.product_b = p_anchor.product_id AND cvp.product_a = p_neg.product_id)
  )
  LIMIT 5000000
)
```

**Step 2 — Fine-tune:**

python

```python
from sentence_transformers import SentenceTransformer, InputExample
from sentence_transformers.losses import MultipleNegativesRankingLoss
from torch.utils.data import DataLoader

model = SentenceTransformer('paraphrase-multilingual-mpnet-base-v2')

# curriculum: start easy (in-batch negatives only), add hard negatives later
# Phase 1: easy pairs
easy_examples = [
    InputExample(texts=[anchor, positive])
    for anchor, positive in easy_pairs
]

# Phase 2: hard triplets
hard_examples = [
    InputExample(texts=[anchor, positive, hard_negative])
    for anchor, positive, hard_negative in hard_triplets
]

loss = MultipleNegativesRankingLoss(model)

# phase 1
loader = DataLoader(easy_examples, batch_size=64, shuffle=True)
model.fit(train_objectives=[(loader, loss)], epochs=3)

# phase 2: fine-tune further with hard negatives
hard_loader = DataLoader(hard_examples, batch_size=64, shuffle=True)
model.fit(train_objectives=[(hard_loader, loss)], epochs=2)
```

---

### Part C — Combine and write to KVDal

After both models are trained, combine their scores per product pair and write the final lists:

python

```python
for product_id in all_products:

    # LightFM scores (behavioral)
    lightfm_scores = get_lightfm_scores(product_id)  # {product_id: score}

    # SBERT scores (semantic)
    sbert_scores = get_sbert_scores(product_id)       # {product_id: score}

    # combine — union of candidates, weighted sum where both exist
    all_candidates = set(lightfm_scores) | set(sbert_scores)

    combined = {}
    for candidate in all_candidates:
        score = (
            w_interaction * lightfm_scores.get(candidate, 0) +
            w_title       * sbert_scores.get(candidate, 0) +
            w_cate        * cate_cooccurrence_score(product_id, candidate)
        )
        combined[candidate] = score

    # top-200 similar, top-200 complementary — written separately
    top_similar = sorted(combined.items(), key=lambda x: -x[1])[:200]

    kvdal.write(
        key=product_id,
        value={
            'similar': top_similar,
            'complementary': top_complementary
        }
    )
```

---

## Side by side summary

```
                HOME (Two-Tower)          PDP (LightFM + SBERT)
                ────────────────          ─────────────────────
Data source     pdp_view_raw (30d)        pdp_view_raw (90d)
Training signal implicit views            session co-view pairs
Negatives       in-batch implicit         in-batch + hard negatives
Loss            CrossEntropy on           MNR loss (SBERT)
                similarity matrix         WARP loss (LightFM)
Output          ONNX user tower           product-product score lists
                + FAISS item index        written to KVDal
Retraining      weekly                    weekly (LightFM)
cadence                                   monthly (SBERT)
New items       re-encode item tower      re-encode SBERT embeddings
                → update FAISS            → recompute similarity lists
```

The key architectural difference: Home training produces **two towers** where the item side becomes a static index. PDP training produces **pairwise scores** that get written directly as lookup lists — no embedding index needed at serving time.