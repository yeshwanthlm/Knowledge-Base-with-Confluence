# AWS Cloud FinOps Guide — Cost Optimisation Playbook

**Space:** Engineering > Cloud Platform > FinOps  
**Owner:** Cloud Platform Team  
**Last Updated:** February 2025  
**Quarter:** Q1 2025 Planning Edition  
**Status:** Active — reviewed and updated quarterly

---

## Overview

NovaTech's total AWS spend for 2024 was **$4.2M** across all accounts. This guide documents the FinOps practices, cost allocation standards, and optimisation playbooks used by the Cloud Platform team to manage and reduce cloud spend while supporting engineering growth.

**2025 target:** Hold total spend below $4.5M despite a projected 40% increase in compute workloads.

Our AWS environment:
- **12 AWS accounts** (1 management + 11 workload accounts, via AWS Organizations)
- **Primary region:** us-east-1 (85% of spend)
- **DR region:** eu-west-1 (12% of spend)
- **Experimentation region:** us-west-2 (3%)

---

## Cost Allocation Framework

### Account Structure

| Account | Purpose | Monthly Budget | Owner |
|---------|---------|----------------|-------|
| `prod-core` | Production workloads | $180K | Platform Engineering |
| `prod-data` | Data platform (S3, Glue, Redshift, MSK) | $45K | Data Engineering |
| `prod-ml` | ML training and inference (SageMaker, GPU) | $65K | ML Platform |
| `staging` | Pre-prod environments | $25K | Platform Engineering |
| `dev-shared` | Developer sandbox (shared) | $15K | All Engineering |
| `security` | Security tooling (GuardDuty, Security Hub) | $8K | Security |
| `monitoring` | Observability (Datadog, CloudWatch) | $12K | SRE |

### Tagging Standard

**All resources must have these tags.** Untagged resources are flagged daily and owners notified.

```
Environment:    prod | staging | dev
Team:           data-engineering | ml-platform | backend | frontend | sre
Service:        <service-name>   (e.g., payments-api, recommendation-engine)
CostCentre:     ENG-001 | ENG-002 | ENG-003 | DATA-001 | ML-001
Project:        <jira-project-key>   (e.g., DATA, REC, PLAT)
Owner:          <email>
```

Tagging compliance report: **AWS Config rule `required-tags`** — runs hourly, results in CloudWatch dashboard `FinOps-Tag-Compliance`. Current compliance: **94.2%** (target: 98%).

---

## Top Spend Breakdown (2024 Actuals)

| Service | 2024 Spend | % of Total | Notes |
|---------|-----------|------------|-------|
| Amazon EC2 | $1,480K | 35.2% | Includes EKS node groups |
| Amazon S3 | $380K | 9.0% | Data lake + backups + model artefacts |
| Amazon RDS / Aurora | $420K | 10.0% | 6 production clusters |
| Amazon SageMaker | $580K | 13.8% | Training + inference endpoints |
| Amazon Redshift | $310K | 7.4% | Serverless + provisioned legacy |
| Amazon MSK | $175K | 4.2% | 2 production Kafka clusters |
| Amazon CloudFront | $145K | 3.5% | CDN for web and mobile |
| AWS Glue | $220K | 5.2% | ETL jobs |
| Other | $490K | 11.7% | 80+ other services |
| **Total** | **$4,200K** | 100% | — |

---

## Optimisation Playbooks

### Playbook 1: EC2 Right-Sizing

**Savings potential:** $80–120K/year  
**Effort:** Medium (2 sprints)  
**Owner:** Platform Engineering

AWS Compute Optimizer is enabled across all accounts. Monthly review of recommendations:

```bash
# Pull right-sizing recommendations for all EC2 in an account
aws compute-optimizer get-ec2-instance-recommendations \
  --region us-east-1 \
  --filters name=Finding,values=OVER_PROVISIONED \
  --query 'instanceRecommendations[*].{
    Instance: instanceArn,
    CurrentType: currentInstanceType,
    RecommendedType: recommendationOptions[0].instanceType,
    SavingsMonthly: recommendationOptions[0].estimatedMonthlySavings.value
  }' \
  --output table
```

**Process:**
1. Run report weekly (automated in Lambda, results to #cloud-cost-reports)
2. For instances with > $50/month savings opportunity, open Jira ticket for owning team
3. Owning team has 2 weeks to right-size or document why current size is required
4. Re-check after 30 days

**2024 savings achieved:** $94K via right-sizing 47 instances (avg: 2 size classes down).

---

### Playbook 2: Reserved Instances & Savings Plans

**Current coverage:** 62% of eligible compute under Savings Plans  
**2025 target:** 75% coverage  
**Annual savings vs on-demand:** $320K

**Savings Plans active:**

| Plan Type | Commitment | Term | Region | Monthly Commitment | Expiry |
|-----------|-----------|------|--------|-------------------|--------|
| Compute Savings Plan | Flexible | 3yr | All | $48,000 | Dec 2026 |
| EC2 Instance SP | m5/r5 family | 1yr | us-east-1 | $22,000 | Mar 2025 |
| SageMaker SP | ML training | 1yr | us-east-1 | $12,000 | Jun 2025 |

**Renewal schedule:**
- EC2 SP expires March 2025 — renewal analysis due February 2025 ← **ACTION REQUIRED**
- SageMaker SP expires June 2025 — review in April 2025

**How to analyse coverage:**
```bash
# Check Savings Plans utilisation (should be > 90%)
aws ce get-savings-plans-utilization \
  --time-period Start=$(date -d "30 days ago" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --query 'Total.{Utilization: Utilization, UnusedCommitment: UnusedCommitment}'
```

If utilisation drops below 80%, review whether workloads have scaled down and adjust commitment.

---

### Playbook 3: S3 Storage Optimisation

**Savings potential:** $25–40K/year  
**Current S3 bill:** $380K/year

**Intelligent Tiering enabled on:**
- `company-datalake-prod` — Bronze/Silver layers (archive old partitions)
- `company-ml-artefacts` — model artefacts > 90 days old

**Lifecycle policies in place:**

| Bucket | Rule | Action |
|--------|------|--------|
| `datalake-prod/bronze/` | Objects > 365 days | Move to S3 Glacier Instant Retrieval |
| `datalake-prod/bronze/` | Objects > 2555 days (7yr) | Expire (delete) |
| `ml-artefacts/` | Objects > 90 days + no GetObject in 30 days | S3 Intelligent-Tiering |
| `app-logs/` | All objects > 30 days | S3 Glacier Deep Archive |
| `app-logs/` | All objects > 365 days | Expire |

**Quick wins to implement (Q1 2025):**
- [ ] Enable S3 Storage Lens for all buckets — identify largest objects and access patterns
- [ ] Review `ml-experiments/` bucket — suspected 8TB of orphaned experiment artefacts
- [ ] Apply lifecycle policy to `dev-snapshots/` (currently no policy, 4TB unmanaged)

---

### Playbook 4: RDS / Aurora Optimisation

**Current RDS spend:** $420K/year  
**Savings target:** $60K/year via migration and right-sizing

| Cluster | Engine | Instance | Monthly Cost | Recommendation |
|---------|--------|----------|-------------|----------------|
| `prod-payments-db` | Aurora PostgreSQL 15 | r6g.2xlarge (2 nodes) | $4,200 | No change — P1, sized for peak |
| `prod-user-db` | Aurora PostgreSQL 15 | r6g.xlarge (2 nodes) | $2,800 | Evaluate Aurora Serverless v2 |
| `prod-reporting-db` | RDS PostgreSQL 14 | db.r5.2xlarge | $3,100 | Migrate to Redshift Serverless |
| `staging-*` (3 clusters) | Various | db.t3.medium | $800 total | Implement auto-stop for off-hours |
| `dev-*` (4 clusters) | Various | db.t3.small | $600 total | Migrate to Aurora Serverless v2 |

**Immediate action:** Stage clusters should auto-stop from 20:00–08:00 UTC (weekdays) and all weekend. Estimated saving: $8K/year per cluster.

---

### Playbook 5: SageMaker Cost Control

**Current SageMaker spend:** $580K/year (training: $220K, inference: $360K)

**GPU training cost controls:**
```python
# Always set max_run in estimator to prevent runaway jobs
estimator = Estimator(
    instance_type="ml.g4dn.2xlarge",
    instance_count=2,
    max_run=60 * 60 * 8,   # 8 hour hard stop — REQUIRED
    use_spot_instances=True,              # Use Spot for training
    max_wait=60 * 60 * 12,  # 12 hour max wait for Spot
    checkpoint_s3_uri=f"s3://checkpoints/{job_name}/",  # Enable checkpointing
)
```

**Spot instance usage:**  
Training jobs use Spot instances where possible. Current Spot savings: **$78K/year** (avg 70% Spot discount).

**Inference endpoint rightsizing:**  
Scale-to-zero for non-production endpoints:
```python
# Dev/staging endpoints — stop outside business hours
# Implemented via EventBridge + Lambda
schedule = {
    "scale_down": "cron(0 20 ? * MON-FRI *)",   # 8pm UTC weekdays
    "scale_up":   "cron(0 7 ? * MON-FRI *)"     # 7am UTC weekdays
}
```
Endpoints that scale to zero when idle: 8 dev endpoints (saving $12K/year).

---

## FinOps Governance

### Weekly Cost Review

Every Monday, the Cloud Platform team reviews:
1. Previous week's spend vs budget (AWS Cost Explorer)
2. Anomaly alerts from AWS Cost Anomaly Detection
3. Top 5 fastest-growing services
4. Untagged resource report

**Slack:** Automated weekly spend digest posted to `#cloud-costs` every Monday 09:00 UTC.

### Budget Alerts

| Account | Monthly Budget | Alert at 80% | Alert at 100% |
|---------|---------------|-------------|--------------|
| `prod-core` | $180K | $144K → #cloud-costs | $180K → on-call page |
| `prod-ml` | $65K | $52K → #cloud-costs | $65K → #cloud-costs + ML lead |
| `dev-shared` | $15K | $12K → #cloud-costs | $15K → freeze new resources |

```bash
# Create budget alert (example for prod-core)
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "prod-core-monthly",
    "BudgetLimit": {"Amount": "180000", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {"NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 80},
      "Subscribers": [{"SubscriptionType": "SNS", "Address": "arn:aws:sns:us-east-1:..."}]
    }
  ]'
```

### Quarterly Business Review (QBR)

Every quarter, the Cloud Platform team presents to Engineering leadership:
- Actual spend vs budget
- Cost per product feature (unit economics)
- Optimisation initiatives completed + savings achieved
- Next quarter forecast and budget request
- Benchmark vs industry (% revenue spent on infrastructure)

**Target unit economics (2025):**
- Cost per 1M API requests: ≤ $2.50
- Cost per daily active user: ≤ $0.04
- ML inference cost per prediction: ≤ $0.0003

---

## 2025 Optimisation Roadmap

| Initiative | Owner | Target Saving | Planned Quarter |
|------------|-------|--------------|-----------------|
| EC2 Savings Plan renewal (3yr) | Platform Eng | $110K/yr | Q1 2025 |
| Aurora Serverless v2 migration (3 clusters) | Backend | $35K/yr | Q1 2025 |
| S3 orphaned artefact cleanup | Data Eng + ML | $20K/yr | Q1 2025 |
| Graviton migration (EKS node group) | Platform Eng | $45K/yr | Q2 2025 |
| Redshift Serverless consolidation | Data Eng | $28K/yr | Q2 2025 |
| SageMaker Savings Plan (training) | ML Platform | $40K/yr | Q2 2025 |
| Cross-AZ data transfer reduction | Platform Eng | $18K/yr | Q3 2025 |
| **Total target** | — | **$296K/yr** | — |

---

## Contacts

| Role | Name | Slack |
|------|------|-------|
| FinOps Lead | Vikram Joglekar | @vikram.joglekar |
| Cloud Platform | Arjun Mehta | @arjun.mehta |
| Finance liaison | Deepa Menon | @deepa.menon |
| Cost reporting | — | #cloud-costs |
| Anomaly alerts | — | #cloud-cost-alerts |
