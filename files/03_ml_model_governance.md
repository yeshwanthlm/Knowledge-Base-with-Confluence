# ML Model Governance & Deployment Standard

**Space:** AI / ML Platform > Standards  
**Owner:** ML Platform Team  
**Last Updated:** March 2025  
**Applies To:** All production ML models at NovaTech  
**Status:** Approved — Engineering All-Hands ratified Jan 2025

---

## Purpose

This document defines the standards, checkpoints, and approval workflows for deploying machine learning models into production at NovaTech. All models — regardless of team or use case — must follow this process before serving live traffic.

As of Q1 2025, we have **47 models in production** across Recommendations, Risk & Fraud, Search Ranking, and Personalisation. Poorly governed model releases have caused two P1 incidents in the past 12 months, both traced to inadequate pre-production testing. This standard exists to prevent recurrence.

---

## Model Classification

All models are assigned a risk tier that determines the required approval level:

| Tier | Criteria | Examples | Approvals Required |
|------|----------|----------|-------------------|
| **Tier 1 — Critical** | Directly affects revenue, payments, or safety | Fraud detection, credit scoring, payment routing | ML Platform + Legal/Compliance + Eng VP |
| **Tier 2 — High Impact** | Affects significant user experience or business metric | Homepage recommendations, search ranking, churn prediction | ML Platform + Product Lead |
| **Tier 3 — Standard** | Internal tooling or low-exposure features | Email send-time optimiser, internal content classifier | ML Platform review only |
| **Tier 4 — Experimental** | Shadow mode only, no user impact | A/B test variants, research prototypes | Team lead sign-off |

---

## Model Development Lifecycle

```
Research & Ideation
      │
      ▼
Model Design Doc (required for Tier 1 & 2)
      │
      ▼
Development & Experimentation
  └── MLflow experiment tracking (mandatory)
  └── Unit tests on model logic
      │
      ▼
Validation Gate (ML Platform Review)
  └── Offline metrics meet threshold
  └── Fairness evaluation passed
  └── Data lineage documented
      │
      ▼
Shadow Mode Deployment (minimum 7 days)
  └── Run in parallel, no user impact
  └── Compare predictions vs current model
      │
      ▼
Canary Release (1% → 5% → 25% → 100%)
  └── Online metrics monitored per stage
  └── Automatic rollback if guard rails breached
      │
      ▼
Full Production
  └── Model card published
  └── Monitoring dashboards live
  └── Drift detection active
      │
      ▼
Regular Review (Tier 1: monthly, Tier 2: quarterly)
```

---

## Required Artefacts

Every model must have the following before promotion to production:

### 1. Model Card

Maintained in the ML Registry (SageMaker Model Registry). Must include:

- **Model name and version**
- **Use case description** — What decision does this model inform?
- **Training data** — Source, date range, preprocessing steps, known biases
- **Features** — Full feature list with descriptions and data sources
- **Target variable** — Definition and labelling methodology
- **Performance metrics** — Offline metrics with confidence intervals
- **Fairness evaluation** — Performance breakdown across demographic segments (where applicable)
- **Known limitations** — What the model does NOT do well
- **Deployment constraints** — Latency requirements, throughput, fallback behaviour
- **Owner** — Team and individual accountable for model health

### 2. MLflow Experiment

All experiments must be logged to `mlflow.company.internal` under the correct project:

```python
import mlflow

mlflow.set_tracking_uri("https://mlflow.company.internal")
mlflow.set_experiment("/production/recommendations/homepage-ranker")

with mlflow.start_run(run_name="v2.4.1-lgbm-baseline"):
    mlflow.log_param("n_estimators", 500)
    mlflow.log_param("learning_rate", 0.05)
    mlflow.log_param("feature_count", 87)
    
    mlflow.log_metric("AUC-ROC", 0.847)
    mlflow.log_metric("precision_at_10", 0.312)
    mlflow.log_metric("ndcg_at_10", 0.681)
    
    mlflow.sklearn.log_model(model, "model")
```

### 3. Data Lineage Document

Describes the full lineage of training data:
- Where does each feature come from in the data platform?
- What transformations were applied?
- What date range was used for training vs validation vs test?
- Are there any data quality issues or known biases in the training set?

---

## Validation Gate — Required Metrics

### Tier 1 Models (Fraud, Risk)

| Metric | Minimum Threshold | Notes |
|--------|-------------------|-------|
| AUC-ROC | ≥ 0.92 | Evaluated on held-out test set |
| Precision at threshold | ≥ 0.85 | At operating point |
| False Positive Rate | ≤ 0.03 | Critical — impacts real customers |
| Latency P99 | ≤ 50ms | Load tested at 2× expected peak |
| Fairness gap (by gender) | ≤ 2% | Differential positive prediction rate |
| Fairness gap (by age band) | ≤ 3% | Same metric |

### Tier 2 Models (Recommendations, Search)

| Metric | Minimum Threshold |
|--------|-------------------|
| NDCG@10 | ≥ 5% improvement over current production model |
| Click-through rate (offline simulation) | ≥ current model |
| Latency P99 | ≤ 100ms |
| Coverage | ≥ 95% of items receivable at least 1 recommendation |

### Tier 3 Models

Metrics defined per project. ML Platform review required but thresholds are team-defined.

---

## Shadow Mode Requirements

Before any canary traffic, models must run in shadow mode for a minimum of:
- **Tier 1:** 14 days
- **Tier 2:** 7 days
- **Tier 3:** 3 days

Shadow mode comparison metrics must be documented. Significant divergence from the current model (> 15% difference in prediction distribution) requires investigation before proceeding.

---

## Production Deployment — SageMaker

All models are deployed via **Amazon SageMaker Endpoints**.

### Deployment Config Standards

```python
# Standard deployment configuration
endpoint_config = {
    "EndpointConfigName": f"{model_name}-v{version}-config",
    "ProductionVariants": [
        {
            "VariantName": "AllTraffic",
            "ModelName": model_name,
            "InstanceType": "ml.c5.2xlarge",  # default; GPU for deep learning
            "InitialInstanceCount": 2,          # minimum 2 for HA
            "InitialVariantWeight": 1.0,
        }
    ],
    "DataCaptureConfig": {
        "EnableCapture": True,
        "SamplingPercentage": 10,   # 10% of requests logged for monitoring
        "DestinationS3Uri": f"s3://company-ml-data-capture/{model_name}/",
        "CaptureOptions": [
            {"CaptureMode": "Input"},
            {"CaptureMode": "Output"}
        ]
    }
}
```

### Auto-scaling Policy

All Tier 1 and Tier 2 endpoints use target-tracking auto-scaling:

```python
scaling_policy = {
    "TargetValue": 70.0,   # 70% invocations per instance
    "PredefinedMetricSpecification": {
        "PredefinedMetricType": "SageMakerVariantInvocationsPerInstance"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
}

# Min/max instances by tier
TIER_1_MIN_MAX = (3, 20)
TIER_2_MIN_MAX = (2, 10)
TIER_3_MIN_MAX = (1, 5)
```

### Canary Rollout Stages

```yaml
# canary-config.yaml — managed by ML Deployment Service
stages:
  - name: canary-1pct
    traffic_weight: 1
    duration_hours: 24
    guard_rails:
      error_rate_threshold: 0.005
      latency_p99_threshold_ms: 150
      
  - name: canary-5pct
    traffic_weight: 5
    duration_hours: 48
    guard_rails:
      error_rate_threshold: 0.003
      business_metric_degradation: 0.02   # 2% degradation triggers rollback
      
  - name: canary-25pct
    traffic_weight: 25
    duration_hours: 72
    guard_rails:
      error_rate_threshold: 0.002
      business_metric_degradation: 0.01
      
  - name: full
    traffic_weight: 100
```

---

## Production Monitoring

### Required Monitors (All Tiers)

1. **Data Drift Detection** — Statistical test (KS test) on input features daily. Alert if drift score > 0.15 for any key feature.
2. **Prediction Distribution Monitoring** — Alert if prediction score distribution shifts significantly (PSI > 0.2).
3. **Latency Monitoring** — P50, P90, P99 latency tracked. Alert if P99 > agreed threshold.
4. **Error Rate** — CloudWatch alarm on `ModelInvocationErrors` > 0.5%.
5. **Business Metric Correlation** — Weekly report correlating model output with downstream business metric (click rate, conversion, fraud catch rate).

### Model Retraining Triggers

Models must be retrained when any of the following occur:
- Feature drift score exceeds 0.25 (automatic alert via SageMaker Model Monitor)
- Business metric correlation drops > 10% vs baseline (monthly review)
- Underlying data schema changes
- Scheduled retraining cycle reached (Tier 1: monthly, Tier 2: quarterly)

---

## Rollback Procedure

```bash
# Immediate rollback to previous model version
# Step 1: Identify previous good endpoint config
aws sagemaker list-endpoint-configs \
  --name-contains "${MODEL_NAME}" \
  --sort-by CreationTime \
  --sort-order Descending

# Step 2: Update endpoint to previous config
aws sagemaker update-endpoint \
  --endpoint-name "${ENDPOINT_NAME}" \
  --endpoint-config-name "${PREVIOUS_CONFIG_NAME}"

# Step 3: Monitor rollback completion (~5 min)
aws sagemaker describe-endpoint \
  --endpoint-name "${ENDPOINT_NAME}" \
  --query 'EndpointStatus'

# Step 4: Notify in #ml-incidents channel
# Step 5: Create incident ticket in Jira (ML project)
```

**Target rollback time:** < 10 minutes for Tier 1 models.

---

## Compliance & Ethics Checklist

Before Tier 1 or Tier 2 model launch, complete the Model Ethics Checklist:

- [ ] Has the model been tested for bias across protected characteristics (gender, age, race where applicable)?
- [ ] Is the model's decision explainable to affected users if required?
- [ ] Has Legal reviewed the use case for GDPR / CCPA compliance?
- [ ] Are there human override mechanisms for high-stakes decisions?
- [ ] Has the training data handling been reviewed by the Data Governance team?
- [ ] Is there a documented process for handling appeals/complaints?

---

## Contacts & Ownership

| Role | Name | Slack |
|------|------|-------|
| ML Platform Lead | Priya Sharma | @priya.sharma |
| ML Governance | Rahul Verma | @rahul.verma |
| Data Governance | Ananya Krishnan | @ananya.krishnan |
| SageMaker Support | — | #ml-platform |
| Incidents | — | #ml-incidents |

---

*Related docs: [Feature Store Guide](../feature-store/guide) | [A/B Testing Framework](../ab-testing) | [Data Contract Standard](../../data-platform/data-contracts)*
