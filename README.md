# Reliability Design: Investment Orders Microservice

> **Context:** Critical microservice handling all investment orders. Downtime at market open has direct financial cost. Being wrong costs more than being slow. This architecture is designed around real failure modes, not theoretical uptime numbers.

---

## Table of Contents

- [Overview](#overview)
- [Compute Architecture](#compute-architecture)
- [Database Architecture](#database-architecture)
- [Failover Strategy](#failover-strategy)
- [Backup Strategy](#backup-strategy)
- [SLO Definition](#slo-definition)
- [Error Budget Policy](#error-budget-policy)
- [Failure Mode Analysis](#failure-mode-analysis)

---

## Overview

### Design Principles

| Principle | Rationale |
|-----------|-----------|
| **Correctness over speed** | A wrong order is worse than a slow one. Every layer validates before committing. |
| **No single points of failure** | Every component has a standby. No single EC2 instance, no single AZ, no single region for extended outages. |
| **Failure isolation** | Reporting traffic never competes with write traffic. Health checks validate real dependencies, not just container liveness. |
| **Backups are only backups if tested** | Daily restore tests in staging. An untested backup is not a backup. |

---

## Compute Architecture

**Runtime:** AWS ECS Fargate. No EC2 instance management overhead, no patching risk, no capacity planning for individual hosts.

```
                        ┌─────────────────────────────────┐
                        │     Application Load Balancer    │
                        │  Health check: GET /health        │
                        │  (validates DB + cache, not       │
                        │   just container liveness)        │
                        └────────────┬────────────────────┘
                                     │
               ┌─────────────────────┴──────────────────────┐
               │                                            │
    ┌──────────▼──────────┐                    ┌──────────▼──────────┐
    │   AZ-1 (Primary)    │                    │   AZ-2 (Secondary)  │
    │  ECS Fargate Tasks  │                    │  ECS Fargate Tasks  │
    │  Min: 2 | Max: 10   │                    │  Min: 2 | Max: 10   │
    └─────────────────────┘                    └─────────────────────┘
```

### Key Decisions

- **Two AZs minimum.** Tolerates a full AZ outage with no manual intervention.
- **ALB health checks on `/health`.** The endpoint actively checks database connectivity and cache reachability. A container that cannot reach the DB is removed from rotation immediately.
- **Auto Scaling on CPU and request queue depth.** Order bursts at market open are request-driven, not compute-bound. CPU alone is not the right signal.
- **No spot instances.** The cost savings do not justify a mid-trading-day interruption.

---

## Database Architecture

**Engine:** Amazon RDS PostgreSQL with Multi-AZ.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Primary Region                           │
│                                                                 │
│  ┌──────────────┐   sync replication   ┌──────────────────┐    │
│  │  RDS Primary │ ──────────────────▶  │  RDS Standby     │    │
│  │    (AZ-1)    │   (automatic failover │    (AZ-2)        │    │
│  │              │    < 2 min)           │                  │    │
│  └──────┬───────┘                      └──────────────────┘    │
│         │                                                       │
│         │ async replication                                     │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │  Read Replica│   ← Reporting queries only                    │
│  │    (AZ-3)    │     Never competes with write traffic         │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

| Setting | Value | Reason |
|---------|-------|--------|
| Multi-AZ | Enabled | Synchronous standby, automatic failover |
| Failover time | < 2 minutes | RDS-managed, no manual DNS changes |
| Read replica | AZ-3 | Reporting isolation from transactional traffic |
| Point-in-time recovery | Enabled | Recover to any second within retention window |
| Automated backups | 7-day retention | Daily snapshots, continuous transaction logs |
| Weekly snapshots | 90-day retention | Compliance, audit, and rollback coverage |
| Restore testing | Daily in staging | Untested backups are not backups |

### Write Path Guarantee

All investment order writes go to the primary only. Read replicas are never used for order submission. Replication lag, even sub-second, is not acceptable here: a duplicate or missing order has regulatory and financial consequences.

---

## Failover Strategy

### In-Region Failover (AZ Failure)

Handled automatically by RDS Multi-AZ and ECS task redistribution. No manual action required. Expected recovery: **< 2 minutes.**

### Cross-Region Failover (Region Failure)

```
Primary Region (us-east-1)          DR Region (us-west-2)
─────────────────────────           ──────────────────────
RDS Primary ─── async ───────────▶  RDS Cross-Region Replica
                replication              (promotable)

Route 53 Health Checks
  ├── Primary endpoint healthy → route to primary
  └── Primary endpoint unhealthy → failover routing to DR
```

- **Active-passive in DR region.** Warm standby, not cold.
- **RDS cross-region read replica** that can be promoted to standalone primary.
- **Route 53 failover routing** for automatic DNS cutover on health check failure.
- **RTO:** < 30 minutes for full regional failover (replica promotion + ECS scale-up in DR region).
- **RPO:** < 5 minutes (cross-region replication lag under normal conditions).

### Runbook Requirement

A documented, tested runbook exists for cross-region promotion. It runs in a drill at least once per quarter. The failover process is not something you work out during an incident.

---

## Backup Strategy

| Backup Type | Frequency | Retention | Tested |
|-------------|-----------|-----------|--------|
| Automated RDS snapshots | Daily | 7 days | Yes — daily restore in staging |
| Manual weekly snapshots | Weekly | 90 days | Yes — monthly restore drill |
| Transaction logs (PITR) | Continuous | 7 days | Implicit in daily restore |
| Cross-region replica | Continuous async | N/A (live) | Yes — quarterly failover drill |

### Restore Testing

Automated daily job in staging:

1. Restore last automated snapshot to staging RDS instance
2. Run schema validation queries
3. Run data integrity checks (row counts, constraint checks, last N orders readable)
4. Alert on failure. A failed restore test is treated as an incident, not a warning.

---

## SLO Definition

### Primary SLO

> **99.95% availability on successful investment order submission**, measured over a rolling 30-day window.

That is roughly **22 minutes of allowable downtime per month.**

### Why This Metric, Not Just Uptime

Service uptime ("is the container running?") is the wrong signal. An order service that is up but failing to write to the database is 0% available from a business perspective. The SLO measures **successful order submission end-to-end:**

- The request reached the service
- Authentication and validation passed
- The order was written to the database
- A confirmation was returned to the caller

Any failure in that chain counts against the SLO.

### Time-Weighted SLO Windows

Not all minutes are equal. A uniform SLO treats 2am downtime the same as 9:30am market-open downtime. That is wrong.

```
Trading Hours (09:30–16:00 ET, weekdays)
  └── Downtime weight: 5x
  └── These minutes burn error budget faster

Extended Hours (07:00–09:30, 16:00–20:00 ET)
  └── Downtime weight: 2x

Off-Hours / Weekends
  └── Downtime weight: 1x
```

The 99.95% target is measured in weighted minutes. Deployments and maintenance windows are scheduled off-hours for this reason.

### Latency SLO (Secondary)

| Percentile | Target |
|------------|--------|
| p50 | < 100ms |
| p95 | < 500ms |
| p99 | < 1000ms |

Latency is a secondary SLO and does not burn the primary error budget. Sustained p99 > 1s triggers a review. Slow order submission has real business impact even when technically "successful."

---

## Error Budget Policy

**Error budget: 0.05% per rolling 30-day window (~22 minutes weighted)**

| Threshold | Action |
|-----------|--------|
| > 50% budget consumed in a single incident | Mandatory postmortem. Deployment freeze until root cause is confirmed. |
| > 75% budget consumed in the month | All non-critical deployments halted for remainder of window. Engineering leadership notified. |
| Budget exhausted (100%) | Full deployment freeze. Hotfixes require two-engineer sign-off. SLO breach notification to stakeholders. |

### Postmortem Requirements

Any incident consuming > 50% of the monthly budget requires a written postmortem within 48 hours:

- Timeline of the incident
- Root cause (not "human error" — the system condition that made human error possible)
- What monitoring missed or caught too late
- Specific remediation items with owners and deadlines
- What would have caught this earlier

---

## Failure Mode Analysis

| Failure | Detection | Response | RTO |
|---------|-----------|----------|-----|
| Single ECS task crash | ALB health check fails, task removed from rotation | ECS restarts and replaces task | < 60s |
| Full AZ outage | ALB + RDS detect AZ unreachable | RDS fails over to standby, ECS redistributes tasks | < 2 min |
| RDS primary failure | RDS Multi-AZ health monitoring | Automatic promotion of standby | < 2 min |
| Cache layer failure | `/health` endpoint detects cache unreachability | Service falls through to DB (cache-aside, degraded mode) | 0 (degraded) |
| Bad deployment (error spike) | Error rate alarm on 5xx > threshold | Automated rollback via ECS deployment circuit breaker | < 5 min |
| Full region outage | Route 53 health checks fail | Failover routing to DR region, manual replica promotion | < 30 min |
| Data corruption | PITR enabled, daily restore tests catch backup validity | Restore to last clean point-in-time | Depends on extent |

### Graceful Degradation Priority

When the system is under stress, it sheds load in this order. Correctness is not on the list.

1. **Shed reporting traffic first.** Read replicas saturated: reporting queries get 503.
2. **Rate limit non-critical consumers.** Internal analytics and dashboards wait.
3. **Queue non-time-sensitive operations.** Confirmations and notifications can be async.
4. **Order submission is last.** It does not degrade.

---

## Summary

The architecture rests on three explicit frameworks:

Multi-AZ is table stakes. Single-AZ for a service this critical is not a cost decision; it is a risk decision, and the risk is unacceptable.

The SLO measures business outcomes, not infrastructure metrics. 99.95% on successful order submission is harder to hit than 99.95% on container uptime. That is the point.

Untested plans are not plans. Backups are restored daily. Failover is drilled quarterly. The runbook is a living document.

---

