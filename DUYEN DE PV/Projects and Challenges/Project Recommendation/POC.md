## What you'd set up

A single GCP Compute Engine VM with enough resources to handle all three training jobs sequentially.

**Machine spec for POC:**

```
Machine type:  n1-standard-8 (8 vCPU, 30GB RAM)
GPU:           1x NVIDIA T4 (for SBERT fine-tuning)
Disk:          200GB SSD (training data + artifacts)
OS:            Deep Learning VM image (comes with CUDA, Python pre-installed)
```

T4 is the cheapest GPU option on GCP and sufficient for fine-tuning a multilingual SBERT on your dataset size. No need for A100 for POC.

---

## What runs where on that VM

```
BigQuery (stays in BQ)          GCP VM
──────────────────────          ──────
Dataset preparation     →       Download parquet to VM disk
Feature joins           →       Two-Tower training (PyTorch + CPU/GPU)
Co-view aggregation     →       LightFM training (Python, CPU only)
Triplet construction    →       SBERT fine-tuning (PyTorch + T4 GPU)
                        →       FAISS index building
                        →       KVDal similarity list generation
                        →       Upload artifacts back to GCS
```

---

## Practical setup steps

**1. Create the VM:**

```bash
gcloud compute instances create tiki-ml-poc \
  --machine-type=n1-standard-8 \
  --accelerator=type=nvidia-tesla-t4,count=1 \
  --image-family=pytorch-latest-gpu \
  --image-project=deeplearning-platform-release \
  --boot-disk-size=200GB \
  --maintenance-policy=TERMINATE \
  --metadata="install-nvidia-driver=True"
```

`--maintenance-policy=TERMINATE` is important — GPU VMs can't live migrate, so you tell GCP to terminate instead of migrate during maintenance.

**2. Install dependencies:**

```bash
pip install torch sentence-transformers lightfm faiss-gpu \
            google-cloud-bigquery google-cloud-storage \
            pandas pyarrow mlflow
```

**3. Authenticate to GCP services:**

```bash
gcloud auth application-default login
# VM service account needs:
#   BigQuery Data Viewer
#   Storage Object Admin (read/write GCS)
```

**4. Pull training data from BigQuery to disk:**

```python
from google.cloud import bigquery
import pandas as pd

client = bigquery.Client(project='tiki-dwh')

# pull Two-Tower training dataset
df = client.query("""
    SELECT * FROM recommendation.two_tower_training_data
    WHERE split = 'train'
""").to_dataframe()

df.to_parquet('/data/two_tower_train.parquet')
```

**5. Train sequentially:**

```bash
# Two-Tower
python train_two_tower.py \
  --input /data/two_tower_train.parquet \
  --output /artifacts/user_tower.onnx \
  --faiss-output /artifacts/item_index.faiss

# LightFM
python train_lightfm.py \
  --input /data/coview_matrix.parquet \
  --output /artifacts/lightfm_embeddings.npy

# SBERT (uses GPU automatically)
python train_sbert.py \
  --input /data/triplets.parquet \
  --output /artifacts/sbert_model/

# Build KVDal similarity lists
python build_kvdal.py \
  --lightfm /artifacts/lightfm_embeddings.npy \
  --sbert /artifacts/sbert_model/ \
  --output /artifacts/kvdal_lists.parquet

# Upload everything to GCS
gsutil -m cp /artifacts/* gs://tiki-mlops/poc/
```

---

## Cost consideration for POC

```
n1-standard-8 + T4 GPU:   ~$0.50/hour
Training time estimate:
  Two-Tower:    2–3 hours
  LightFM:      1–2 hours
  SBERT:        3–4 hours (GPU)
  FAISS build:  30 min
  ──────────────────────
  Total:        ~7–10 hours → ~$5 total

Storage (200GB SSD):      ~$0.04/GB/month → negligible for POC
```

**Stop the VM when not training** — GPU VMs are expensive if left running idle. Use a startup script that shuts down automatically when training finishes:

```bash
# at end of your training script
gcloud compute instances stop tiki-ml-poc --zone=asia-southeast1-b
```

---

## What to skip for POC

- **Airflow orchestration** — just run scripts manually or as a simple bash pipeline
- **MLflow tracking server** — log locally to `/artifacts/mlflow/`, upload to GCS
- **Blue-green index deployment** — just overwrite artifacts in GCS directly
- **Monitoring** — manual spot checks after each training run

---

## The one gotcha

BigQuery export to VM disk can be slow for large tables. For POC, either:

- Export BigQuery → GCS first, then `gsutil cp` to VM (faster for large datasets)
- Or use `LIMIT` to work with a sample during initial POC runs:

```sql
SELECT * FROM recommendation.two_tower_training_data
WHERE split = 'train'
  AND event_date >= DATE_SUB(CURRENT_DATE, INTERVAL 7 DAY)  -- 7 days instead of 30
LIMIT 5000000
```

Get the pipeline working end-to-end on a data sample first, then scale to full data once you've verified everything runs correctly. That's the right POC approach — prove the pipeline before proving the scale.