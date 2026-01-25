### Topic 1: Data Pipeline Orchestration with Apache Airflow

This topic assesses his maintenance of 50+ DAGs with Python/PySpark/SQL, ensuring 95%+ SLA for ML and reporting. Questions focus on DAG design, operators, scheduling, and troubleshooting for depth.

1. **Question**: Walk through the structure of one of your complex Airflow DAGs. How did you incorporate PySpark tasks? 
	**Expected Answer**: DAG as Python script with tasks (e.g., BashOperator for scripts, PythonOperator for custom logic, SparkSubmitOperator for PySpark jobs). Dependencies via >> or set_downstream. Example: Ingestion task >> Transformation (PySpark SQL) >> Load to ClickHouse. Ensured idempotency with task retries.
2. **Question**: How did you achieve 95%+ SLA compliance in your Airflow setups? What monitoring did you implement? 
	**Expected Answer**: SLAs via Airflow's SLA feature (sla_miss_callback), with retries (default_retries=3, retry_delay=5min). Monitoring: StatsD exporter to Prometheus, alerts on task duration > threshold. Custom sensors for external dependencies.
3. **Question**: Explain how you handled dynamic DAGs or parameterized workflows in Airflow for varying data volumes. 
	**Expected Answer**: Used macros ({{ ds }}) and XCom for params. For dynamics: SubDAGs or TaskGroups, or external config (e.g., JSON in GCS). Example: BranchPythonOperator to skip tasks based on data size.
4. **Question**: What challenges arose with task dependencies in 50+ DAGs, and how did you optimize them? 
	**Expected Answer**: Cycles avoided via DAG linting; fan-out/fan-in with DummyOperators. Optimization: Pooling for resource limits, priority_weight for critical paths. Reduced cross-DAG waits with ExternalTaskSensor.
5. **Question**: Describe integrating Airflow with Kafka for triggering streaming-related tasks. **Expected Answer**: KafkaSensor to poll topics for messages, then trigger DAG. Or webhook via Airflow API. Ensured at-least-once with offset tracking in XCom.
6. **Question**: How did you manage secrets and configurations in your Airflow DAGs securely? **Expected Answer**: Airflow Connections/Variables (encrypted backend like AWS SSM). In code: get_connection() for creds. No hardcoding; used Fernet for encryption.
7. **Question**: Discuss scaling Airflow for high-throughput (e.g., for 2 TB/day integrations). What executor did you use? 
	**Expected Answer**: CeleryExecutor with Redis/RabbitMQ for distributed tasks. Scaled workers via KubernetesExecutor. Tuned scheduler_heartbeat_sec for faster polling.
8. **Question**: How did you implement error notification and recovery in your DAGs? 
	**Expected Answer**: on_failure_callback to Slack/Email. Recovery: Catchup=False for non-backfillable, or manual reruns via CLI. Custom operators for rollback.
9. **Question**: What testing strategies did you use for Airflow DAGs before production? **Expected Answer**: Unit tests with pytest-airflow, mocking tasks. Integration: LocalExecutor in dev, DAG validation (airflow dags test). CI with GitLab.
10. **Question**: How would you version-control and deploy updates to your 50+ DAGs without downtime? 
	**Expected Answer**: Git for DAG files, deployed via sync to Airflow server. Used image tags in Kubernetes for rolling updates. Pause/resume DAGs during deploy.