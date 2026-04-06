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






## What MLflow Actually Is

MLflow is an open-source platform that solves one specific problem: **you run many training experiments and quickly lose track of what you tried, what worked, and what's currently in production.**

Without MLflow your workflow looks like:

```
train_v1.py → model saved to /artifacts/model_v1.onnx
train_v2.py → model saved to /artifacts/model_v2.onnx   ← what changed?
train_v3.py → model saved to /artifacts/model_v3.onnx   ← which is production?
              ↑ three weeks later you have no idea what
                hyperparameters produced which result
```

MLflow gives you four things:

**Tracking** — every training run logs its hyperparameters, metrics, and artifacts automatically. You can look back at any run and know exactly what produced it.

**Model registry** — a catalog of model versions with explicit states. You know which version is in production, which is being evaluated, and what came before.

**Artifact storage** — ONNX files, FAISS indexes, similarity lists — stored and versioned alongside the run that produced them.

**Experiment comparison** — compare Recall@50 across 10 runs in a single table without manually reading log files.

---

## How MLflow Works Mechanically

MLflow has two components:

**Tracking server** — a process that receives logs from your training script and stores them. Can be local (just a folder) or remote (a server with a database backend).

**Tracking client** — the `mlflow` Python library you call inside your training script. It sends data to the tracking server.

```
Your training script          MLflow tracking server
──────────────────            ─────────────────────
mlflow.log_param(...)    →    stores in database
mlflow.log_metric(...)   →    stores in database
mlflow.log_artifact(...) →    stores file in artifact store (GCS)
```

For your GCP VM POC, the simplest setup is MLflow tracking server running on the same VM, with GCS as the artifact store.

---

## Integrating MLflow into Your GCP Training Pipeline

### Step 1 — Install and configure on the VM

```bash
pip install mlflow google-cloud-storage

# set artifact store to GCS bucket
export MLFLOW_TRACKING_URI="sqlite:///mlflow.db"  # local SQLite for POC
export MLFLOW_DEFAULT_ARTIFACT_ROOT="gs://tiki-mlops/mlflow-artifacts"

# start the MLflow UI (optional, for viewing runs)
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root gs://tiki-mlops/mlflow-artifacts \
  --host 0.0.0.0 \
  --port 5000
```

Access the UI at `http://{VM_EXTERNAL_IP}:5000` — open port 5000 in GCP firewall rules first.

---

### Step 2 — Two-Tower training with MLflow

```python
import mlflow
import mlflow.pytorch

mlflow.set_tracking_uri("sqlite:///mlflow.db")
mlflow.set_experiment("two_tower_home")

with mlflow.start_run(run_name="two_tower_v12"):

    # --- log hyperparameters upfront ---
    mlflow.log_params({
        'batch_size': 256,
        'temperature': 0.07,
        'learning_rate': 1e-3,
        'epochs': 10,
        'user_tower_dims': '28->128->64',
        'item_tower_dims': '15->128->64',
        'negative_strategy': 'in_batch',
        'training_window_days': 30,
        'train_eval_split_days': 3,
    })

    # --- training loop ---
    for epoch in range(num_epochs):
        train_loss = train_one_epoch(user_tower, item_tower, dataloader)
        recall_50, ndcg_50, mrr = evaluate(user_tower, item_tower, eval_set)

        # log metrics per epoch — MLflow tracks the full curve
        mlflow.log_metrics({
            'train_loss': train_loss,
            'recall_at_50': recall_50,
            'ndcg_at_50': ndcg_50,
            'mrr': mrr,
        }, step=epoch)

    # --- offline evaluation against production model ---
    prod_recall = get_production_model_recall()  # read from registry
    new_recall  = recall_50

    mlflow.log_metrics({
        'recall_vs_production': new_recall / prod_recall,
        'eval_coverage': compute_coverage(),
        'session_feature_ks_pvalue': compute_skew_check(),
    })

    # --- gate: only log model if metrics pass ---
    if new_recall >= prod_recall * 0.98 and compute_coverage() >= 0.90:

        # export and log ONNX user tower
        torch.onnx.export(user_tower, sample_input, "/tmp/user_tower.onnx")
        mlflow.log_artifact("/tmp/user_tower.onnx", artifact_path="model")

        # build and log FAISS index
        build_faiss_index(item_tower, "/tmp/item_index.faiss")
        mlflow.log_artifact("/tmp/item_index.faiss", artifact_path="model")

        # tag this run as candidate for production
        mlflow.set_tag("status", "candidate")
        mlflow.set_tag("gate_passed", "true")

    else:
        mlflow.set_tag("status", "failed_eval")
        mlflow.set_tag("gate_passed", "false")
        raise ValueError(f"Offline eval gate failed: recall={new_recall:.4f} vs production={prod_recall:.4f}")
```

---

### Step 3 — LightFM training with MLflow

```python
mlflow.set_experiment("pdp_lightfm")

with mlflow.start_run(run_name="lightfm_v8"):

    mlflow.log_params({
        'no_components': 64,
        'loss': 'warp',
        'learning_rate': 0.05,
        'epochs': 30,
        'training_window_days': 90,
        'min_coview_sessions': 10,
        'item_alpha': 1e-6,
    })

    model = LightFM(no_components=64, loss='warp', learning_rate=0.05)
    
    for epoch in range(30):
        model.fit_partial(interactions, epochs=1)
        recall, ndcg = evaluate_lightfm(model, eval_pairs)
        
        mlflow.log_metrics({
            'recall_at_50': recall,
            'ndcg_at_50': ndcg,
        }, step=epoch)

    # log final metrics
    mlflow.log_metrics({
        'kvdal_coverage': compute_kvdal_coverage(),
        'intra_list_diversity': compute_diversity(),
    })

    # save embeddings as artifact
    np.save("/tmp/lightfm_embeddings.npy", model.item_embeddings)
    mlflow.log_artifact("/tmp/lightfm_embeddings.npy", artifact_path="embeddings")
    mlflow.set_tag("status", "candidate")
```

---

### Step 4 — SBERT training with MLflow

```python
mlflow.set_experiment("pdp_sbert")

with mlflow.start_run(run_name="sbert_v5"):

    mlflow.log_params({
        'base_model': 'paraphrase-multilingual-mpnet-base-v2',
        'batch_size': 64,
        'epochs_phase1': 3,
        'epochs_phase2': 2,
        'loss': 'MultipleNegativesRankingLoss',
        'hard_negative_ratio': 0.3,
        'training_window_days': 90,
    })

    # phase 1 — easy pairs
    model.fit(train_objectives=[(easy_loader, loss)], epochs=3,
              callback=lambda score, epoch, steps:
                  mlflow.log_metric('eval_score_phase1', score, step=epoch))

    # phase 2 — hard negatives
    model.fit(train_objectives=[(hard_loader, loss)], epochs=2,
              callback=lambda score, epoch, steps:
                  mlflow.log_metric('eval_score_phase2', score, step=epoch))

    # save model
    model.save("/tmp/sbert_model")
    mlflow.log_artifacts("/tmp/sbert_model", artifact_path="sbert_model")
    mlflow.set_tag("status", "candidate")
```

---

### Step 5 — Promotion script (replaces formal model registry for POC)

After all three models train and pass their gates, a promotion script moves artifacts from candidate to production in GCS and records the promotion in MLflow:

```python
def promote_to_production(experiment_name, run_id):

    client = mlflow.MlflowClient()
    run = client.get_run(run_id)

    # verify gate passed
    if run.data.tags.get('gate_passed') != 'true':
        raise ValueError(f"Run {run_id} did not pass eval gate")

    # copy artifacts from candidate path to production path in GCS
    artifact_uri = run.info.artifact_uri  # gs://tiki-mlops/mlflow-artifacts/{run_id}/
    prod_path = f"gs://tiki-mlops/models/{experiment_name}/production/"

    os.system(f"gsutil -m cp -r {artifact_uri}/model/* {prod_path}")

    # archive previous production
    prev_run_id = get_current_production_run_id(experiment_name)
    if prev_run_id:
        archive_path = f"gs://tiki-mlops/models/{experiment_name}/archive/{prev_run_id}/"
        os.system(f"gsutil -m cp -r {prod_path}* {archive_path}")

    # tag run as production in MLflow
    client.set_tag(run_id, "status", "production")
    client.set_tag(run_id, "promoted_at", datetime.now().isoformat())

    print(f"Promoted run {run_id} to production for {experiment_name}")
```

---

### Step 6 — Airflow DAG wiring it together

```python
with DAG('weekly_training_pipeline', schedule_interval='@weekly') as dag:

    wait_for_features = ExternalTaskSensor(
        task_id='wait_for_dbt',
        external_dag_id='dbt_feature_pipeline',
        external_task_id='feast_materialization',
    )

    train_two_tower = PythonOperator(
        task_id='train_two_tower',
        python_callable=run_two_tower_training,
        # raises ValueError if gate fails → DAG stops here
    )

    train_lightfm = PythonOperator(
        task_id='train_lightfm',
        python_callable=run_lightfm_training,
    )

    train_sbert = PythonOperator(
        task_id='train_sbert',
        python_callable=run_sbert_training,
    )

    promote_models = PythonOperator(
        task_id='promote_to_production',
        python_callable=promote_all_candidates,
        # only runs if all training tasks succeeded
    )

    notify = PythonOperator(
        task_id='slack_notification',
        python_callable=send_slack_summary,
        # sends MLflow run URLs so you can inspect results
    )

    wait_for_features >> [train_two_tower, train_lightfm, train_sbert]
    [train_two_tower, train_lightfm, train_sbert] >> promote_models
    promote_models >> notify
```

---

## What You Get From This

After one week of training runs, MLflow gives you:

```
Experiment: two_tower_home
┌─────────┬──────────┬─────────────┬──────────┬────────┬──────────┐
│ Run     │ recall@50│ ndcg@50     │ coverage │ status │ date     │
├─────────┼──────────┼─────────────┼──────────┼────────┼──────────┤
│ v12     │ 0.341    │ 0.218       │ 0.94     │ prod   │ Apr 06   │
│ v11     │ 0.328    │ 0.209       │ 0.93     │ archive│ Mar 30   │
│ v10     │ 0.301    │ 0.195       │ 0.91     │ archive│ Mar 23   │
│ v9      │ 0.289    │ 0.188       │ 0.89     │ failed │ Mar 16   │
└─────────┴──────────┴─────────────┴──────────┴────────┴──────────┘
```

You know exactly which model is in production, what it achieved, what changed between versions, and you can pull any previous artifact from GCS by run_id if you need to rollback.

---

## One-line interview framing

> "MLflow is the audit trail for the training pipeline — every run is reproducible, every promotion is intentional, and if something breaks in production I can identify exactly which model version and training run caused it and roll back to the previous artifact in GCS within minutes."