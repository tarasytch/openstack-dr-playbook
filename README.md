# OpenStack Disaster Recovery Playbook

A production-grade DR framework for large-scale OpenStack cloud platforms.
Designed and operated at **VK Cloud** — a platform serving ~300 services across IaaS, PaaS and DBaaS layers.

---

## Overview

| Parameter | Value |
|-----------|-------|
| Scope | Full AZ failure simulation (control-plane + data-plane) |
| Method | Non-graceful shutdown via IPMI force poweroff |
| Cadence | Quarterly |
| Success criteria | No degradation in service SLO graphs |
| Teams involved | 5+ cross-functional engineering teams |
| Tracked in | Confluence + Jira |

---

## DR Test Methodology

Each DR test follows a structured sequence:

1. **Pre-checks** — verify replica status, replication lag, cluster health, HAProxy backends
2. **Shutdown** — IPMI force poweroff of target controller node (non-graceful)
3. **Failure verification** — confirm component enters expected degraded state within SLA window (5–10 min)
4. **Recovery validation** — verify automatic failover, check service logs, confirm HAProxy backend UP
5. **Post-test** — document findings, create Jira tickets for discovered issues, track to resolution

---

## Components Covered

### OpenStack Control Plane

| Component | Failure Scenario | Recovery Mechanism |
|-----------|-----------------|-------------------|
| Nova | VM creation/management unavailable | HA replica failover, MySQL master re-election |
| Cinder | Volume operations unavailable | HA replica failover |
| Neutron | Network management unavailable | HA replica failover |
| Glance | Image upload/download unavailable | HA replica failover |
| Octavia | Load balancer management unavailable | HA replica failover |
| Manila | File share management unavailable | Control + data plane failover |
| Sprut (SDN) | SDN control plane unavailable | Pod rescheduling, etcd lock re-acquisition |

### Data Layer

| Component | Failure Scenario | Recovery Mechanism |
|-----------|-----------------|-------------------|
| Galera (MariaDB) | DB connection loss | Multi-master, automatic failover |
| Stolon + PGBouncer | PostgreSQL unavailable | Stolon sentinel re-election |
| ClickHouse | Read/write degradation | Distributed table queue, replica recovery |
| Redis Sentinel | Cache/session unavailable | Sentinel master re-election |
| etcd | Key-value store unavailable | Quorum-based failover (3 replicas) |
| Kafka + ZooKeeper | Message queue degradation | Partition replication (3/3), lag recovery ~2-5 min |
| Memcached | Cache miss, Keystone latency | Stateless — no recovery needed |

### Infrastructure Layer

| Component | Failure Scenario | Recovery Mechanism |
|-----------|-----------------|-------------------|
| Consul | Service discovery unavailable | Quorum-based failover |
| HAProxy | Backend routing failure | Backend health check re-validation |
| VictoriaMetrics | Monitoring storage unavailable | Replication failover |
| Zabbix | Infrastructure monitoring gap | Service restart |
| IAM (Breeze/Dusk/Ocean) | Auth/token validation unavailable | Tarantool replica failover, overlord re-election |

---

## Incident Escalation Process

### Critical Incident Definition

An incident is considered **critical** when **both** conditions are met:

**Condition 1** — incident affects at least one of:
- VIP / enterprise clients
- 3+ clients simultaneously
- Critical services: VM operations, UI availability, billing

**Condition 2** — on-call engineer cannot determine resolution timeline within **20 minutes**

### On-Call Coverage

| Shift Type | Schedule |
|-----------|----------|
| Daily on-call | Weekdays, assigned per shift schedule |
| Weekly on-call | 24/7, covers nights and weekends |
| Contact in business hours | Team chat |
| Contact outside business hours | Phone call |

### Escalation Chain

```
Alert detected (Grafana + monitoring channel)
        |
        v
L1 — Client Support (on-call)
First response. Triage, initial assessment.
        |
        +-- Resolved
        |
        +-- Cannot resolve or assess in 20 min --> L2
        |
L2 — Technical Support (on-call, per shift schedule)
Component-level diagnosis and fix.
        |
        +-- Resolved --> Post-mortem, Jira ticket
        |
        +-- Escalated --> L2 Team Lead
        |
L3 — Engineering Management
Multi-component or prolonged incidents.
Involves platform operations director.
        |
        +-- Escalated --> L4
        |
L4 — Executive Level
Full platform degradation or AZ-level failure.
VK Tech leadership involved.
```

### Escalation Matrix by Service

| Business Scenario | Service | L1 | L2 |
|------------------|---------|----|----|
| VMs running / new VMs created | IaaS | Client Support on-call | Tech Support on-call |
| UI available | Frontend (Personal Account) | Client Support on-call | Tech Support on-call |
| UI available | Frontend (Docs, Status Page) | Client Support on-call | Tech Support on-call |
| Billing operational | Billing (Scrooge) | Client Support on-call | Tech Support on-call |
| Auth operational | IAM | Client Support on-call | Tech Support on-call |
| Backup operational | RunTime | Client Support on-call | Tech Support on-call |
| PaaS operational | EventHouse, VDI, XaaS | Client Support on-call | Tech Support on-call |
| Data platform operational | DBaaS, BigData, ML Platform | Client Support on-call | Tech Support on-call |

Each service has a defined 4-level escalation chain up to executive level.

---

## Key Findings from DR Tests

Real outcomes from production DR cycles:

- **Galera (Cinder/Glance)** — timeout on connection pool checkout; fixed via SQLAlchemy health check on pool re-connection
- **IAM (Breeze/Dusk)** — master failover with >7 min downtime; improved overlord policy for replica promotion
- **Octavia** — dnsmasq listener misconfiguration discovered; fixed post-test
- **Kafka** — replication lag spikes ~2-5 min during broker loss; acceptable, no SLO breach

---

## Monitoring Stack

- **Grafana** — primary observability dashboard per component
- **VictoriaMetrics** — metrics storage (cloud monitoring)
- **Zabbix** — infrastructure-level monitoring
- **VK WorkSpace** — alerting channel
- **SLI/SLO** — availability tracking baseline for DR pass/fail decision

---

## Confluence Structure

```
DR Process/
+-- DR Plan — [Platform] [Environment]
|   +-- Pre-checks runbook
|   +-- Component test matrix
|   +-- Results log
+-- Escalation Matrix
+-- Post-mortem templates
+-- Issue tracker (Jira integration)
```

---

## Related

- [Profile](https://github.com/tarasytch) — Andrey Bogatyrev, Program Manager
- Stack: OpenStack · Kubernetes · Galera · Kafka · Redis · etcd · Stolon · ClickHouse · Grafana · Consul · HAProxy
