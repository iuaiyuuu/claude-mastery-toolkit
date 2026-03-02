# Deployment Patterns for ML Systems

## Serving Architectures

### Batch Inference

**When to use**: Predictions needed on a schedule (daily, hourly), not per-request. Examples: daily recommendation refresh, weekly churn scoring, nightly report generation.

**Architecture**:
```
Scheduler (cron/Airflow/Prefect)
  → Data extraction (SQL/API)
  → Feature computation
  → Model prediction (batch)
  → Write results to database/cache
  → Downstream consumers read from store
```

**Advantages**: Simple, debuggable, cost-efficient (spot instances), no latency requirements.
**Disadvantages**: Stale predictions, wasted compute on unchanged inputs, doesn't work for truly real-time needs.

**Implementation considerations**:
- Size your batch job for your data volume. Don't load 100M rows into memory — chunk it.
- Write predictions to a fast-read store (Redis, DynamoDB, PostgreSQL with proper indexing) for serving.
- Always include a timestamp with predictions so consumers know the freshness.
- Build idempotent jobs — rerunning should produce the same output without side effects.

### Real-time Inference

**When to use**: Predictions needed per request with latency constraints. Examples: fraud detection, search ranking, dynamic pricing, real-time personalization.

**Architecture**:
```
Client request
  → API gateway / load balancer
  → Feature store (precomputed features) + real-time feature computation
  → Model server (TF Serving, Triton, custom Flask/FastAPI)
  → Post-processing / business logic
  → Response
```

**Advantages**: Fresh predictions, handles new inputs immediately.
**Disadvantages**: Latency pressure, scaling complexity, feature computation must be fast.

**Implementation considerations**:
- Separate feature computation from model inference. Precompute expensive features in batch; compute only real-time features on-request.
- Use a feature store (Feast, Tecton, or a simple Redis cache) to serve precomputed features with low latency.
- Model serving: TensorFlow Serving, Triton Inference Server, or a simple REST API. Choose based on model framework and scale.
- Plan for auto-scaling: define scaling metrics (request rate, latency percentile) and min/max instances.
- Add request timeouts and circuit breakers. A slow model server shouldn't cascade-fail the application.

### Hybrid (Batch + Real-time)

**When to use**: Base predictions precomputed in batch, personalized or time-sensitive aspects computed in real-time. Examples: recommendation systems (batch candidate generation + real-time re-ranking), search (batch index + real-time personalization).

**Architecture**:
```
Batch pipeline → candidate/feature store
Client request → retrieve batch results → real-time model for re-ranking/personalization → response
```

### Edge / On-Device Inference

**When to use**: Offline capability needed, extreme latency sensitivity, privacy constraints (data can't leave device). Examples: mobile ML, IoT, embedded systems.

**Considerations**:
- Model size constraints: typically <50MB, often <10MB for mobile.
- Optimization: quantization (INT8, FP16), pruning, knowledge distillation, architecture-specific (MobileNet, EfficientNet-Lite).
- Runtimes: TensorFlow Lite, ONNX Runtime, Core ML, TensorRT.
- Update mechanism: model updates via app updates or dynamic model download (with versioning and rollback).

## Pipeline Automation

### Training Pipeline

Every training run should be reproducible from its configuration alone:

```
Data snapshot (versioned)
  → Data validation (schema, distributions, leakage checks)
  → Feature engineering (deterministic transforms)
  → Train/validation/test split (deterministic)
  → Model training (pinned seeds, logged hyperparameters)
  → Evaluation (metrics on held-out test)
  → Model registration (if performance threshold met)
  → Artifact storage (model, metrics, config, data version)
```

**Orchestration options** (by complexity):
- **Simple**: Makefile or shell scripts with clear step ordering
- **Medium**: Prefect, Airflow, or Dagster for DAG-based pipelines
- **Full MLOps**: Kubeflow Pipelines, Vertex AI Pipelines, SageMaker Pipelines

Choose the simplest orchestration that meets your needs. A Makefile that works is better than a Kubeflow pipeline you can't debug.

### Inference Pipeline

Feature computation in inference must exactly match training. This is non-negotiable.

**Strategies for consistency**:
- **Shared code**: Same feature transformation functions used in both training and inference. Package them as a library.
- **Feature store**: Precompute and serve features from a shared store. Training reads historical features; inference reads current features.
- **Model includes preprocessing**: Bundle preprocessing into the model artifact (sklearn Pipeline, TF preprocessing layers). Simplest approach for simpler models.

### CI/CD for ML

**Testing**:
- Unit tests for feature engineering functions (deterministic transforms)
- Integration tests for the full pipeline on a small data sample
- Model quality gate: new model must meet minimum metrics on the test set before deployment
- Shadow deployment: run new model alongside current model, compare outputs before switching

**Deployment strategies**:
- **Shadow mode**: New model runs in parallel, outputs logged but not served. Validate before promoting.
- **Canary**: Route small percentage of traffic to new model, monitor for regressions, gradually increase.
- **Blue-green**: Two environments, instant switch. Good for batch predictions.
- **A/B test**: Route traffic based on user segments. Use for measuring business impact.

**Rollback**: Always have a one-command rollback to the previous model version. This must be tested before it's needed.

## Model Versioning and Registry

Track every model with:
- **Unique version identifier**: Auto-incremented or content-hash based.
- **Training data version**: Git hash or snapshot ID of the data used.
- **Code version**: Git commit hash of the training code.
- **Hyperparameters**: Full configuration used for training.
- **Metrics**: All evaluation metrics on the test set.
- **Lineage**: Which data, code, and config produced this model.

**Registry options**: MLflow Model Registry, Weights & Biases, simple database table with S3 artifact paths. The tool matters less than the discipline of using it.

## Cost Optimization

### Training Costs
- Use spot/preemptible instances for training (with checkpointing for interruption tolerance)
- Right-size GPU selection: don't use A100 for a LightGBM model
- Cache expensive feature computations — don't recompute on every experiment
- Set compute budgets for hyperparameter search

### Inference Costs
- Batch inference is typically 5-10x cheaper than real-time for the same volume
- CPU inference is sufficient for most tree-based and simple models — don't default to GPU
- Model optimization (quantization, pruning, distillation) can reduce inference cost 2-10x
- Auto-scaling with scale-to-zero for low-traffic models
- Request batching for GPU inference servers (higher throughput per dollar)

### Cost Estimation Template
Before deploying, estimate:
- Training cost: instance type × hours × retrain frequency
- Inference cost: instance type × count × utilization × hours/month
- Storage cost: model artifacts + features + logs
- Data pipeline cost: compute + storage + data transfer

Compare against the business value of the model's predictions to justify the investment.

## Scaling Considerations

**Scale dimensions to plan for**:
- Request volume: peak vs. average, growth trajectory
- Data volume: training data growth rate affects pipeline design
- Model complexity: larger models need more resources per prediction
- Feature freshness: more real-time features = more infrastructure

**Start simple, scale as needed**: A single FastAPI server with a LightGBM model handles surprisingly high throughput. Don't architect for Google-scale unless you have Google-scale traffic.
