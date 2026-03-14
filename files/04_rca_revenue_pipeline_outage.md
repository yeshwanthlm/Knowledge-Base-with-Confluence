# Root Cause Analysis — Revenue Pipeline Outage
## Incident ID: INC-2024-0847

**Space:** Engineering > SRE > Post-Mortems  
**Date of Incident:** November 14, 2024  
**Date of RCA:** November 19, 2024  
**Severity:** P1  
**Duration:** 3 hours 42 minutes (09:17 UTC – 12:59 UTC)  
**Status:** Resolved — All action items tracked in Jira  
**RCA Author:** Kiran Nair (Platform Engineering)  
**Reviewers:** Arjun Mehta, Priya Sharma, Ananya Krishnan

---

## Executive Summary

On November 14, 2024, the daily revenue reporting pipeline failed silently for 3 hours and 42 minutes, resulting in Finance not receiving the morning revenue report by the 08:00 UTC SLA. The root cause was a combination of:

1. A schema change deployed to the `payment-events` Kafka topic without following the Schema Registry change process
2. A gap in alerting — the pipeline failed but the dead-letter queue was not monitored
3. Downstream dependency on the revenue report caused two additional teams (Finance and Growth) to make incorrect business decisions based on previous day's data

**Customer/Business Impact:** No direct customer impact. Internal — Finance and Growth teams operated without accurate revenue data for approximately 4 hours during the APAC business day. One incorrect budget decision was made and subsequently reversed.

---

## Timeline

All times in UTC.

| Time | Event |
|------|-------|
| **08:00** | Revenue report SLA misses — Finance team does not receive expected Slack notification |
| **08:12** | Finance analyst manually checks Tableau dashboard, notes data is stale (showing previous day) |
| **08:17** | Finance analyst pings #data-platform: "Revenue dashboard showing yesterday's data, expected today's by 8am" |
| **08:31** | Data platform on-call (Kiran Nair) acknowledges. Begins investigation. |
| **08:45** | Kiran checks Airflow — `transform_revenue_daily` DAG shows green (succeeded). Checks Redshift — revenue table last updated at 00:12 UTC. Unexpected. |
| **09:02** | Kiran identifies that `stg_payment_events` model has 0 new rows since 23:47 UTC the previous night. |
| **09:17** | **Root cause identified:** `payment-events` Kafka topic schema was changed at 23:31 UTC the previous night. New field `payment_method_detail` added as non-nullable without default value. Glue streaming job began rejecting all messages to dead-letter queue. |
| **09:22** | Escalated to P1. Arjun Mehta (Platform Lead) joins. Priya Sharma (ML Platform) joins (ML feature store also affected). |
| **09:40** | Decision: Revert schema change in Schema Registry to restore ingestion. Schema Registry admin Ananya Krishnan called. |
| **10:05** | Schema reverted to v4.2.1. Glue streaming job restarted. Messages begin flowing from dead-letter queue replay. |
| **10:45** | Glue job caught up to real-time. Silver layer updated. |
| **11:20** | dbt run triggered manually: `dbt run --select tag:revenue` |
| **12:43** | Revenue data validated in Redshift. Finance notified. |
| **12:59** | Incident closed. Revenue dashboard updated and confirmed accurate. |
| **13:15** | Post-mortem meeting scheduled for Nov 19. |

---

## Root Cause Analysis

### Primary Root Cause: Uncontrolled Schema Change

At 23:31 UTC on November 13, a backend engineer (who was not on the data platform team) deployed a new version of the payments service that included a new Kafka message field: `payment_method_detail` (type: `STRING`, nullable: `false`).

This change was **not registered** through the Schema Registry change process defined in `Data Platform > Standards > Kafka Schema Change Process`. The engineer was unaware this process existed.

The AWS Glue streaming job consuming `payment-events` uses **FULL COMPATIBILITY** mode validation. When it encountered messages with the new non-nullable field not present in the registered schema v4.2.1, it treated each message as invalid and routed them to the dead-letter queue: `s3://company-datalake-prod/dlq/payment-events/`.

**Volume affected:** 847,293 payment event messages were written to the dead-letter queue between 23:31 UTC Nov 13 and 10:05 UTC Nov 14.

### Contributing Factor 1: No DLQ Monitoring Alert

The dead-letter queue for `payment-events` had no CloudWatch alarm configured. Messages silently accumulated with no automated notification. This is a process gap — the DLQ monitoring standard existed for Kinesis streams but had not been applied to Glue streaming jobs consuming MSK.

### Contributing Factor 2: dbt Model Did Not Fail

Because the Silver layer `stg_payment_events` table received 0 new rows (rather than failing to write), the dbt test `expect_table_row_count_to_be_between` (min: 50,000) was **not** configured for this table. The model ran successfully — it simply had nothing to process. This produced a false positive in Airflow.

### Contributing Factor 3: Alert Routing Gap

The `transform_revenue_daily` Airflow DAG has a SLA miss alert configured — but the alert goes to `#airflow-alerts`, a low-signal channel with 200+ daily messages. The Finance team's SLA notification (via Slack bot) does depend on the DAG, but only fires when the DAG itself fails — not when it succeeds with stale data.

---

## Impact Assessment

| Affected System | Impact | Duration |
|----------------|--------|----------|
| Daily Revenue Report | Stale (showing Nov 13 data) | 4h 43m |
| Finance Tableau Dashboard | Stale | 4h 43m |
| ML Feature Store (`payment_features`) | Stale features served to 3 models | 3h 20m |
| Growth Attribution Report | Downstream dependency — also stale | 4h 43m |

**Business decisions affected:**
1. Finance team approved a minor budget reallocation at 09:00 UTC based on incorrect revenue figures. Reversed at 14:00 UTC after corrected data was confirmed. No financial loss resulted.
2. Growth team's daily spend review was delayed by 5 hours.

---

## What Went Well

- The Finance team spotted the issue within 12 minutes of SLA breach — good data consumer awareness
- Dead-letter queue replay was clean — all 847K messages were successfully re-processed with no data loss
- Rollback of schema change was executed within 50 minutes of root cause identification
- Communication to stakeholders was clear and timely throughout the incident

---

## What Did Not Go Well

- A schema change affecting a production data pipeline was deployed with no data platform team involvement or awareness
- The DLQ for a P1 pipeline had no monitoring
- Airflow DAG success status masked the underlying data quality issue
- The revenue SLA alert only covers DAG failure, not stale data

---

## Action Items

| ID | Action | Owner | Priority | Due Date | Status |
|----|--------|-------|----------|----------|--------|
| AI-847-1 | Add CloudWatch alarm on all MSK consumer DLQs (message count > 100 in 5min) | Kiran Nair | P1 | Nov 22, 2024 | ✅ Completed |
| AI-847-2 | Add row count freshness test to all P1 dbt models (fail if row count = 0 or last updated > SLA window) | Sonal Kapoor | P1 | Nov 29, 2024 | ✅ Completed |
| AI-847-3 | Add "Schema Change Process" to backend engineering onboarding checklist | Ananya Krishnan | P2 | Dec 6, 2024 | ✅ Completed |
| AI-847-4 | Run lunch-and-learn with backend engineers on Kafka Schema Registry process | Ananya Krishnan | P2 | Dec 13, 2024 | ✅ Completed |
| AI-847-5 | Move P1 DAG SLA alerts to dedicated #data-sla-alerts channel with reduced noise | Arjun Mehta | P2 | Dec 6, 2024 | ✅ Completed |
| AI-847-6 | Build data freshness monitoring dashboard (Grafana) for all P1 datasets | Kiran Nair | P2 | Jan 10, 2025 | ✅ Completed |
| AI-847-7 | Implement schema change approval workflow in CI/CD pipeline (block merge if Schema Registry not updated) | Platform Eng | P1 | Jan 31, 2025 | 🔄 In Progress |
| AI-847-8 | Review and assign DLQ monitoring to all 47 Glue streaming jobs | Sonal Kapoor | P2 | Dec 20, 2024 | ✅ Completed |

---

## Recurrence Prevention

The combination of AI-847-1 (DLQ alerting), AI-847-2 (freshness tests), and AI-847-7 (schema CI/CD gate) closes all three root cause vectors. Similar incidents have been estimated to have a **92% prevention rate** once AI-847-7 is complete.

---

## Lessons Learned

1. **Silent success is dangerous.** A pipeline that says it succeeded but produced no output is often worse than a pipeline that fails loudly. Freshness SLAs need to be first-class monitoring concerns, not afterthoughts.

2. **Schema changes are data changes.** Any change to an event schema is a data platform change, regardless of which team made it. The process needs to be enforced at the CI/CD level, not just communicated in documentation.

3. **DLQ without monitoring is a false safety net.** Dead-letter queues are only valuable if someone knows they have messages in them.

---

## Appendix: Schema Registry Change Process (Summary)

For reference, the correct process for changing a Kafka topic schema:

1. File a Jira ticket in the `DATA` project with label `schema-change`
2. Tag `#data-platform` and get acknowledgement from a data platform engineer
3. Run compatibility check: `./scripts/schema-check.sh --topic <topic> --new-schema <schema.avsc>`
4. Deploy schema change to Schema Registry **before** deploying the producing service
5. Monitor consumer lag and DLQ for 30 minutes post-deployment
6. Confirm with data platform engineer that ingestion is healthy

Full process: `Data Engineering > Standards > Kafka Schema Change Process`
