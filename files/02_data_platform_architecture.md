# Data Platform Architecture — AWS Modern Data Stack

**Space:** Data Engineering > Architecture  
**Owner:** Data Platform Team  
**Last Updated:** January 2025  
**Status:** Production  
**Version:** 3.2

---

## Overview

This document describes the current architecture of our data platform, which processes approximately **2.4 billion events per day** across all product surfaces. The platform supports real-time analytics (< 5 second latency), batch reporting (daily/weekly aggregations), and ML feature serving.

The stack is fully hosted on AWS and follows a **Medallion Architecture** (Bronze → Silver → Gold) for data quality and lineage management.

---

## Architecture Diagram

```
                        ┌──────────────────────────────────────────────┐
                        │              INGESTION LAYER                  │
   Product Events ─────▶│  Kinesis Data Streams (24 shards, 7d retain)  │
   Mobile SDK    ─────▶│  Kinesis Firehose (S3 delivery, 60s buffer)   │
   Server Events ─────▶│  MSK (Kafka 3.6, 6-broker cluster)            │
   3rd Party APIs ─────▶│  AWS Glue Connectors (Salesforce, Stripe)     │
                        └─────────────────┬────────────────────────────┘
                                          │
                        ┌─────────────────▼────────────────────────────┐
                        │             BRONZE LAYER (Raw)               │
                        │  S3: s3://company-datalake-prod/bronze/       │
                        │  Format: Parquet, partitioned by dt/hr        │
                        │  Retention: 7 years (compliance)             │
                        │  Compression: Snappy                          │
                        └─────────────────┬────────────────────────────┘
                                          │  AWS Glue ETL / Spark EMR
                        ┌─────────────────▼────────────────────────────┐
                        │            SILVER LAYER (Cleaned)            │
                        │  S3: s3://company-datalake-prod/silver/       │
                        │  Format: Delta Lake (ACID transactions)       │
                        │  PII masked, schema validated, deduped       │
                        │  Glue Data Catalog registered                │
                        └─────────────────┬────────────────────────────┘
                                          │  dbt (Airflow orchestrated)
                        ┌─────────────────▼────────────────────────────┐
                        │             GOLD LAYER (Business)            │
                        │  Redshift Serverless (analytics)             │
                        │  DynamoDB (low-latency feature serving)      │
                        │  Athena (ad-hoc SQL over S3)                 │
                        └─────────────────┬────────────────────────────┘
                                          │
                        ┌─────────────────▼────────────────────────────┐
                        │           CONSUMPTION LAYER                  │
                        │  Tableau / Looker (dashboards)               │
                        │  Jupyter / SageMaker (data science)          │
                        │  Internal APIs (feature store)               │
                        │  ML Training pipelines (SageMaker)           │
                        └──────────────────────────────────────────────┘
```

---

## Component Details

### Ingestion

#### Amazon Kinesis Data Streams
- **Stream:** `prod-event-stream`
- **Shards:** 24 (supporting ~120,000 events/sec peak throughput)
- **Retention:** 7 days
- **Consumer:** Lambda (real-time processing) + Firehose (S3 delivery)
- **Monitoring:** CloudWatch alarm on `GetRecords.IteratorAgeMilliseconds` > 60,000ms

#### Amazon MSK (Kafka)
- **Cluster:** `prod-kafka-cluster` (6 brokers, kafka.m5.2xlarge)
- **Topics:** 47 active topics, replication factor 3
- **Key topics:**
  - `user-events` — 16 partitions, 14-day retention
  - `payment-events` — 8 partitions, 90-day retention (compliance)
  - `ml-feature-updates` — 4 partitions, 3-day retention
- **Schema Registry:** AWS Glue Schema Registry (Avro schemas)
- **Monitoring:** MSK Connect, Prometheus + Grafana

#### AWS Glue Connectors (3rd Party)
| Source | Schedule | Records/Day | Connector |
|--------|----------|-------------|-----------|
| Salesforce CRM | Every 4 hours | ~500K | Glue Salesforce |
| Stripe Payments | Every 1 hour | ~200K | Custom Lambda |
| Google Ads | Daily 02:00 UTC | ~50K | Glue JDBC |
| Zendesk | Every 6 hours | ~30K | Custom Lambda |

---

### Storage — Medallion Layers

#### Bronze (Raw)
- **Location:** `s3://company-datalake-prod/bronze/`
- **Format:** Parquet with Snappy compression
- **Partitioning:** `source=<name>/year=YYYY/month=MM/day=DD/hour=HH/`
- **Retention:** 7 years (GDPR and SOX compliance requirement)
- **Write pattern:** Append-only, never modified after write
- **Size:** ~18 TB current, growing ~2.5 TB/month

#### Silver (Cleaned & Validated)
- **Location:** `s3://company-datalake-prod/silver/`
- **Format:** Delta Lake (enables time travel, ACID writes, schema evolution)
- **Key transformations applied:**
  - PII tokenisation (email, phone, IP address)
  - Schema validation against Glue Schema Registry
  - Deduplication using event UUID + 24-hour window
  - Null handling and type casting
  - Late-arriving data handling (2-hour watermark)
- **Processing:** AWS Glue 4.0 (Spark 3.3), `G.2X` workers, auto-scaling 5–50 workers
- **SLA:** Silver tables updated within 30 minutes of raw landing

#### Gold (Business-Ready)
Curated datasets consumed by analysts and ML systems.

**Redshift Serverless namespace:** `prod-analytics`  
**Base RPU:** 64, max 512  
**Key schemas:**

| Schema | Owner | Tables | Refresh |
|--------|-------|--------|---------|
| `finance` | Finance Analytics | 12 | Daily 06:00 UTC |
| `product` | Product Analytics | 28 | Hourly |
| `marketing` | Growth Team | 15 | Daily 07:00 UTC |
| `ml_features` | ML Platform | 34 | Every 15 min |

---

### Orchestration — Apache Airflow

**Deployment:** Amazon MWAA (Managed Workflows)  
**Version:** Airflow 2.8  
**DAG count:** 143 active DAGs  
**Environment:** `prod-mwaa-env` (mw1.medium, 2–10 workers)

Key DAG categories:
- `ingestion_*` — 23 DAGs managing source pulls
- `transform_*` — 58 DAGs running dbt models
- `ml_*` — 31 DAGs for feature engineering and model training
- `reporting_*` — 18 DAGs for dashboard data refresh
- `qa_*` — 13 DAGs running data quality checks

**DAG SLAs:**
- P1 DAGs (revenue/finance): alert if not complete by defined SLA
- P2 DAGs (product metrics): 2-hour SLA breach triggers PagerDuty

---

### Transformation — dbt

**dbt version:** 1.8  
**Adapter:** dbt-redshift  
**Models:** 312 models across 8 packages  
**Materialisation strategy:**

| Layer | Materialisation | Refresh |
|-------|----------------|---------|
| Staging (`stg_*`) | View | On demand |
| Intermediate (`int_*`) | Ephemeral | N/A |
| Mart (`mart_*`) | Incremental | Scheduled |
| Reporting (`rpt_*`) | Table | Daily full refresh |

**Testing:** dbt tests run on every production run:
- `not_null` and `unique` on all primary keys
- `accepted_values` on enum columns
- Custom SQL tests for business logic
- **Current test pass rate:** 99.3% (12 known failing tests under investigation)

---

### Data Quality — Great Expectations

Integrated at the Silver → Gold promotion step.

**Expectation suites:** 24 active suites  
**Checks run daily:** ~4,800  
**Current pass rate:** 98.7%

Sample expectations:
```python
# Revenue dataset
expect_column_values_to_not_be_null("transaction_id")
expect_column_values_to_be_between("amount_usd", min_value=0, max_value=1_000_000)
expect_column_values_to_match_regex("currency_code", r"^[A-Z]{3}$")
expect_table_row_count_to_be_between(min_value=100_000, max_value=5_000_000)
```

---

## SLAs and On-Call

| Dataset | SLA | P1 Alert | Owner |
|---------|-----|----------|-------|
| Daily Revenue Report | Available by 08:00 UTC | Yes | Finance Analytics |
| Real-time Event Stream | < 5s end-to-end | Yes | Data Platform |
| ML Feature Store | < 15 min lag | Yes | ML Platform |
| Marketing Attribution | Available by 09:00 UTC | No | Growth |

On-call rotation: `#data-platform-oncall` — weekly rotation, see PagerDuty schedule `data-platform-prod`.

---

## Disaster Recovery

| Component | RPO | RTO | Strategy |
|-----------|-----|-----|----------|
| S3 Data Lake | 0 (S3 versioning) | 1 hour | Cross-region replication to us-west-2 |
| Redshift | 8 hours | 4 hours | Automated snapshots, restore to new cluster |
| MSK | 24 hours | 2 hours | Multi-AZ, cross-region backup |
| Airflow | 1 hour | 2 hours | MWAA in secondary region |

---

## Costs (Q4 2024 Actuals)

| Service | Monthly Cost | Notes |
|---------|-------------|-------|
| S3 Storage + Requests | $12,400 | ~340TB total |
| Kinesis | $3,200 | 24 shards |
| MSK | $4,800 | 6 brokers |
| Glue ETL | $6,100 | ~1,400 DPU-hours/month |
| Redshift Serverless | $9,800 | avg 180 RPU |
| MWAA | $2,200 | mw1.medium |
| **Total** | **$38,500** | — |

**Budget for 2025:** $45,000/month (growth buffer for 2× data volume increase).

---

## Runbooks & Related Docs

- [Pipeline Failure Runbook](../runbooks/pipeline-failure)
- [Redshift Performance Tuning Guide](../guides/redshift-performance)
- [dbt Style Guide](../standards/dbt-style-guide)
- [Data Contract Template](../templates/data-contract)
- [PII Handling Policy](../../compliance/pii-policy)
