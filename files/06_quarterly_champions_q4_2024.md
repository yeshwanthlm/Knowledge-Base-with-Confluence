# 🏆 Quarterly Champions — Q4 2024

**Space:** People & Culture > Recognition  
**Published:** January 8, 2025  
**Quarter:** October – December 2024  
**Nominated by:** Engineering Leads, Product Managers, People Team  
**Announced at:** All-Hands — January 7, 2025

---

> *"Our Quarterly Champions are the engineers, analysts, and leaders who didn't just do their jobs — they changed how we work. Each award below represents a real story of impact, collaboration, and craft."*  
> — Sunita Rao, CTO

---

## 🥇 Engineer of the Quarter

### Priya Sharma — ML Platform Team

**Award:** Engineering Excellence  
**Nominated by:** Kiran Nair, Arjun Mehta, Rahul Verma

Priya single-handedly redesigned the ML deployment pipeline this quarter, reducing the time from model training completion to production deployment from **11 days** to **under 4 hours**. This wasn't a minor improvement — it fundamentally changed what was possible for the ML team.

Before Priya's work, deploying a new model required coordinating across four teams, filing four Jira tickets, and waiting for a deployment window. Now, a data scientist can run `make deploy MODEL=<n>` and the CI/CD pipeline handles the rest: safety checks, shadow mode enrollment, canary routing, and monitoring dashboards — all automated.

**The numbers:**
- 3 Tier 2 models shipped in Q4 that previously couldn't have made it to production in the same quarter
- Average model deployment time: 11.2 days → 3.8 hours
- Zero production incidents attributable to model deployment in Q4 (vs 2 in Q3)

Priya also mentored two junior ML engineers through their first production model deployments and presented the new pipeline at an external ML meetup in December, bringing positive attention to NovaTech's engineering brand.

**What colleagues said:**
> *"Priya took a problem that everyone had accepted as 'just how it is' and refused to accept it. She built something that makes all of us faster."* — Kiran Nair, Platform Engineering

> *"The new deploy pipeline is one of the best pieces of internal tooling I've used at any company."* — Aditya Rao, ML Engineer

---

## 🥇 Data Champion of the Quarter

### Kiran Nair — Platform Engineering / Data Platform

**Award:** Data Excellence  
**Nominated by:** Arjun Mehta, Finance Analytics Team, Growth Team

Kiran led the full data platform incident response for INC-2024-0847 (November revenue pipeline outage) and then went several steps further: instead of just fixing the immediate issue, he rebuilt the monitoring layer for the entire data platform within 3 weeks.

Before the incident, the data platform had no freshness monitoring — teams only knew data was stale when they noticed it themselves. After Kiran's rebuild:

- **Grafana freshness dashboard** covering 47 production datasets, with SLA breach alerting to the right team automatically
- **DLQ monitoring** on all MSK consumer dead-letter queues (a gap that directly caused the November incident)
- **dbt row count freshness tests** on every P1 dataset — silent pipeline "success" with stale data is now impossible
- **Schema change CI/CD gate** — in progress, 80% complete (ships Q1 2025)

He did this while maintaining on-call responsibilities and without dropping any of his Q4 OKRs.

**The numbers:**
- 0 undetected data freshness issues since new monitoring deployed (Nov 22 onwards)
- 47 datasets now have automated SLA monitoring (vs 0 previously)
- Mean time to detect (MTTD) for data incidents: estimated 2.3 hours → 8 minutes

**What colleagues said:**
> *"After November, I expected Kiran to fix the specific problem. I didn't expect him to rebuild the entire monitoring layer. That's the difference between good and great."* — Arjun Mehta, Platform Lead

---

## 🥇 Collaboration Award

### Ananya Krishnan — Data Governance

**Award:** Cross-Team Excellence  
**Nominated by:** 6 teams across Engineering, Finance, Legal, and Product

Ananya is the reason that NovaTech's data governance went from "a document that exists somewhere" to "a thing people actually follow" in Q4 2024.

After the November pipeline incident revealed that backend engineers didn't know the Kafka schema change process existed, Ananya ran a full audit of data governance documentation, identified 14 processes that existed on paper but weren't known outside the data team, and then built a plan to fix it:

1. **Governance lunch-and-learn series** — 4 sessions in Dec, 67% of engineering attended at least one
2. **Schema change process page** rewritten in plain English, added to backend engineering onboarding checklist
3. **Data contract template** standardised and adopted by 3 teams in Q4 (previously none used it)
4. **GDPR/CCPA data inventory** completed for 2024 compliance audit — 3 months ahead of deadline

Beyond the data governance work, Ananya was the bridge between the ML team and Legal during the ethics review for the new credit risk model — a review that could have taken months but was completed in 3 weeks because of how effectively she facilitated it.

**What colleagues said:**
> *"Ananya has a rare ability to translate between technical teams and compliance/legal in a way that doesn't frustrate anyone. That is genuinely hard and she makes it look easy."* — Priya Sharma, ML Platform

---

## 🏅 Honourable Mentions

### Sonal Kapoor — Data Engineering
Shipped the Q4 data quality initiative that brought the Great Expectations test pass rate from 96.1% to 98.7% — identifying and resolving 12 long-standing failing tests, including two that had been marked "known issue" for over a year. Sonal documented every fix and turned the investigation into a reusable playbook for the team.

---

### Deepa Menon — Finance Analytics
Built the first self-service revenue attribution model that eliminated 4 weekly hours of manual Finance reporting. The model now runs automatically each morning and has been adopted by the Growth team as well. Deepa learned dbt independently over a long weekend and shipped the first version the following Monday — nobody asked her to.

---

### Aditya Rao — ML Engineering
Led the migration of 3 production ML endpoints from `ml.c5.4xlarge` to `ml.c5.2xlarge` instances after his own benchmarking showed latency was within SLA at half the cost — saving $18K/year. He then wrote up his methodology and shared it with the team as a guide for future right-sizing exercises.

---

### Rahul Verma — ML Platform / Governance
Completed the ethics review framework in Q4, including the fairness evaluation tooling integrated into the ML pipeline. Every new model now automatically generates a fairness report before it can proceed to production. This was a gap that had been on the roadmap for 18 months — Rahul shipped it in 6 weeks.

---

## 📊 Team Performance Highlights — Q4 2024

### Engineering Metrics

| Metric | Q3 2024 | Q4 2024 | Change |
|--------|---------|---------|--------|
| Deploy frequency (deploys/day) | 8.2 | 12.4 | ▲ 51% |
| Mean time to recovery (MTTR) | 47 min | 28 min | ▲ 40% |
| P1 incidents | 3 | 1 | ▲ 66% |
| On-call escalations | 14 | 9 | ▲ 36% |
| Test coverage (production services) | 71% | 78% | ▲ 7pp |

### Data Platform Metrics

| Metric | Q3 2024 | Q4 2024 | Change |
|--------|---------|---------|--------|
| Pipeline SLA adherence | 94.2% | 98.8% | ▲ 4.6pp |
| Data quality test pass rate | 96.1% | 98.7% | ▲ 2.6pp |
| MTTD for data incidents | ~2.3 hrs | 8 min | ▲ 94% |
| Unmonitored production datasets | 47 | 0 | ✅ Complete |

### ML Platform Metrics

| Metric | Q3 2024 | Q4 2024 | Change |
|--------|---------|---------|--------|
| Models shipped to production | 2 | 5 | ▲ 150% |
| Avg model deployment time | 11.2 days | 3.8 hours | ▲ 98% |
| Production model incidents | 2 | 0 | ✅ Clean quarter |
| Models with fairness reports | 0% | 100% | ✅ New standard |

---

## 🎯 Q1 2025 Focus Areas

Based on Q4 retrospectives, the following themes will drive recognition in Q1 2025:

1. **Developer Velocity** — Initiatives that help engineers ship faster and with more confidence
2. **Cost Efficiency** — Cloud FinOps wins and unit economics improvements
3. **Knowledge Sharing** — Documentation, runbooks, lunch-and-learns that make the whole team better
4. **Reliability** — Improvements to MTTR, monitoring coverage, and incident reduction

> **Nominate a colleague:** Submit nominations anytime in the `#recognition` Slack channel. Q1 2025 nominations close March 28, 2025.

---

## 🏆 Past Champions Archive

| Quarter | Engineer of Quarter | Data Champion | Collaboration |
|---------|-------------------|---------------|---------------|
| Q4 2024 | Priya Sharma | Kiran Nair | Ananya Krishnan |
| Q3 2024 | Vikram Joglekar | Sonal Kapoor | Rahul Verma |
| Q2 2024 | Arjun Mehta | Deepa Menon | Aditya Rao |
| Q1 2024 | Ananya Krishnan | Priya Sharma | Kiran Nair |
| Q4 2023 | Rahul Verma | Arjun Mehta | Sonal Kapoor |

---

*Recognition programme managed by the People Team. Questions? Ping @people-team in Slack or email people@novatech.io*
