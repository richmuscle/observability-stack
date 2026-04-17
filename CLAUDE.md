# CLAUDE.md — Observability Stack (Portfolio Project 5)

> Place at `~/dev/floorp/infra-tooling/observability-stack/CLAUDE.md`

## What this project is (SysAdmin lens)

The monitoring backbone for the other 5 projects. Prometheus + Grafana + Loki + Alertmanager, with defined SLOs, burn-rate alerting, and — the critical sysadmin differentiator — **a written runbook for every alert**. This is the project that proves richmuscle thinks about **on-call maturity and incident response**, not just "I installed Prometheus once."

**For a sysadmin, this is:**
- Capacity planning (disk, memory, CPU trends)
- Incident response (alerts → runbooks → root cause → remediation)
- Change verification (did that deploy break something?)
- Audit trail (logs in Loki, queryable)
- On-call sustainability (no noisy alerts, every page is actionable)

**What this project proves about richmuscle as a sysadmin:**
- Takes on-call seriously — every alert has a runbook
- Knows the difference between metrics and logs and when to use each
- Understands SLOs vs SLAs vs SLIs
- Designs alerts that a tired human can respond to at 2am
- Thinks about alert fatigue and signal/noise

## Scope (hard limits)

**In scope:**
- Prometheus with scrape configs for: node_exporter (from Project 2's hosts), cAdvisor, blackbox_exporter, ArgoCD (from Project 4), sample app (from Project 6)
- Alertmanager with routing, grouping, inhibition rules
- Grafana with 4-5 dashboards: host overview, service health, SLO burn rate, log search, cluster overview
- Loki + Promtail for log aggregation from all services
- **Recording rules** (SLI computation pre-computed)
- **Alerting rules** with multi-window multi-burn-rate pattern (Google SRE book)
- 3 real SLOs defined in SLOs.md:
  - API availability: 99.5% over 30d
  - Latency p95 < 500ms over 30d
  - Error rate < 1% over 30d
- Alert routing to Discord webhook OR email (pick one, document in ADR)
- **Runbook per alert** (non-negotiable — the whole project hinges on this)
- Synthetic test alert script (`trigger-test-alert.sh`)
- Grafana dashboards exported as JSON in repo
- **systemd artifacts:** if running on bare metal, Prometheus + Loki + Grafana as systemd units with hardening; if in k3s cluster, Helm-deployed with podMonitor/serviceMonitor CRs
- **journalctl + promtail pipeline:** Promtail scrapes journald, routes to Loki

**Explicitly out of scope:**
- Full distributed tracing (Tempo / Jaeger) — deferred; mention "logs + metrics sufficient at this scale"
- Multi-cluster federation — single cluster
- Long-term storage / Thanos / Mimir — local retention
- PagerDuty / Opsgenie integration — webhook is swap-in ready
- Custom exporters in Go / Python — use existing exporters only
- Log parsing at scale beyond Promtail defaults

## Required file structure

```
observability-stack/
├── README.md
├── ARCHITECTURE.md
├── SLOs.md                            # defined SLOs + rationale
├── RUNBOOK.md                         # top-level operational runbook
├── LICENSE
├── Makefile                           # up, down, reload, test-alert
├── .gitignore
├── docker-compose.yml                 # OR Helm values for Project 4 deployment
├── .github/
│   └── workflows/
│       ├── ci.yml                     # promtool + amtool validation
│       └── dashboard-validation.yml   # Grafana JSON schema check
├── DECISIONS/
│   ├── README.md
│   ├── ADR-001-prometheus-stack-choice.md
│   ├── ADR-002-loki-vs-alternatives.md
│   ├── ADR-003-slo-methodology.md
│   └── ADR-004-alert-routing.md
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/
│   │   ├── recording/
│   │   │   ├── slo.yml
│   │   │   └── sli.yml
│   │   └── alerting/
│   │       ├── availability.yml
│   │       ├── latency.yml
│   │       ├── infrastructure.yml
│   │       └── synthetic.yml
│   └── README.md
├── alertmanager/
│   ├── alertmanager.yml
│   └── templates/
│       └── default.tmpl
├── loki/
│   ├── loki-config.yml
│   └── promtail-config.yml            # includes journald scraping
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   └── dashboards/
│   └── dashboards/
│       ├── host-overview.json
│       ├── service-health.json
│       ├── slo-burn-rate.json
│       ├── log-search.json
│       └── cluster-overview.json
├── exporters/
│   ├── blackbox/
│   │   └── blackbox.yml
│   ├── node-exporter/
│   └── cadvisor/
├── runbooks/
│   ├── README.md                      # index
│   ├── high-error-rate.md
│   ├── high-latency.md
│   ├── disk-full.md
│   ├── certificate-expiry.md
│   ├── endpoint-down.md
│   ├── slo-budget-exhausted.md
│   └── host-down.md
├── scripts/
│   ├── trigger-test-alert.sh
│   ├── backup-grafana.sh
│   └── systemd/                       # if bare-metal deployment
│       ├── prometheus.service
│       ├── alertmanager.service
│       └── loki.service
└── screenshots/
    ├── host-overview.png
    ├── slo-dashboard.png
    └── alert-routing.png
```

## ADR topics (minimum set)

1. **ADR-001: Prometheus stack choice** — vs Datadog / New Relic / Grafana Cloud; why self-hosted for this project; when you'd switch to managed
2. **ADR-002: Loki vs alternatives** — vs ELK / OpenSearch; label model; scaling characteristics; why Loki fits small-shop ops
3. **ADR-003: SLO methodology** — multi-window multi-burn-rate (Google SRE book); 2-minute + 30-minute windows; why these targets for a portfolio system
4. **ADR-004: Alert routing** — webhook vs email vs PagerDuty; grouping strategy (by cluster, by service); inhibition rules (infra-level failure suppresses downstream app alerts)

## The SLO document (SLOs.md) — required quality

Structure, written like a real SRE would:

```markdown
# SLOs: Portfolio Infrastructure

## SLO 1: Sample App Availability

- **Service:** sample-web-app (Project 6 deployment)
- **SLI:** ratio of non-5xx responses to total responses
- **Target:** 99.5% over 30 days
- **Error budget:** 3.6 hours/month of allowed downtime
- **Why this target:** a portfolio demo; 99.9% would be aspirational for a single-node cluster. 99.5% is enough to exercise the alerting pattern meaningfully without demanding 24/7 babysitting.
- **Measured from:** Prometheus probe + in-app metric
- **Alert:** burns 2% budget in 1 hour (fast), or 10% budget in 6 hours (slow)

## SLO 2: Sample App Latency
(same structure)

## SLO 3: Platform Availability
(same structure)

## Error budget policy

When the error budget is >50% consumed in a window, deploys to that service require explicit sign-off. Purely demonstrative here, but the pattern is real.
```

## Runbook quality bar (critical)

Every alert has a runbook file, named after the alert. Each runbook:

```markdown
# Alert: HighErrorRate

## Symptom
What the on-call person sees when paged.

## Severity
P2 — business hours page; not a 3am wake-up

## Diagnosis
Specific commands, not "check the dashboard."

1. `curl -s 'http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{status=~"5.."}[5m]))/sum(rate(http_requests_total[5m]))'`
2. `kubectl logs -n sample-app deployment/sample-web-app --tail=200 | grep -i error`
3. Check the error-rate dashboard: [link]

## Remediation
Step-by-step. If "rollback the last deploy," put the command:
`argocd app rollback sample-web-app`

## Escalation
If diagnosis step 3 doesn't identify root cause within 15 minutes, escalate to...

## Prevention
What engineering work would prevent this alert from firing again.
```

## SysAdmin-specific README sections (required)

### `## What this monitors`
Concrete list. Every host, every service, every endpoint. No hand-waving.

### `## Capacity planning dashboards`
Which dashboards show disk growth, memory trends, CPU saturation. Forecast horizons documented.

### `## On-call walk-through`
"You just got paged. What do you do?" Step-by-step from page to resolution.

### `## Limitations and what I'd do differently`
- No distributed tracing — would add Tempo in a real multi-service system
- Single Prometheus — would federate or migrate to Mimir at real scale
- Retention limited by local disk
- No anomaly detection — alerts are all threshold-based
- No secondary comms channel for alerting (just webhook)

## Acceptance criteria

- [ ] `make up` starts the full stack
- [ ] `make test-alert` fires synthetic alert, routes successfully
- [ ] Grafana loads host-overview dashboard with live data
- [ ] `promtool check rules` passes for all rule files
- [ ] `amtool check-config` passes for alertmanager.yml
- [ ] Every alert rule has a corresponding runbook file
- [ ] Every runbook has specific commands, not placeholders
- [ ] SLOs.md defines 3 SLOs with justification
- [ ] 3 screenshots with real data
- [ ] CI green
- [ ] 25+ commits over 6+ calendar days

## Build sequence (Week 5)

**Day 30:** Prometheus + node_exporter + cAdvisor + blackbox_exporter. Scrape config. **Checkpoint.**

**Day 31:** Loki + Promtail (including journald scraping). Log ingestion verified. **Checkpoint.**

**Day 32:** Grafana with 4 dashboards. Provisioning configured. **Checkpoint.**

**Day 33:** Alertmanager. 3 alert rules (error rate, latency, disk). Webhook routing. ADR-004 draft. **Checkpoint.**

**Day 34:** **All runbooks written.** One per alert, with specific commands. ADR-003 (SLO methodology) draft. **Checkpoint.**

**Day 35 (weekend):** SLOs.md finalized. Dashboard screenshots. `trigger-test-alert.sh`. Full end-to-end synthetic alert. **Checkpoint.**

**Day 36 (weekend):** Integration — add Project 1 resources (CloudWatch metrics via exporter), Project 2's node_exporter targets, Project 4's ArgoCD metrics, Project 6's app metrics. Final README. **PR → merge.**

## What NOT to generate

- Don't skip runbooks (the whole project is the runbook discipline)
- Don't set SLOs you can't actually measure
- Don't use fake screenshots — real dashboards only
- Don't claim 99.99% on a single-node system
- Don't copy Grafana dashboards from the internet without attribution in an ADR

## SysAdmin voice (required tone)

> Monitoring without runbooks is just decoration. Most portfolio monitoring projects stop at "here are my dashboards." A real on-call rotation lives or dies by how fast you can go from "paged at 3am" to "root cause identified." I built this stack to monitor my other 5 portfolio projects, and I wrote a runbook for every alert as I implemented it — while the failure modes were still fresh. Every runbook has real kubectl / curl / journalctl commands, not "investigate the issue."

## Linux depth markers

- [ ] Promtail scrapes `journald` — documented in promtail-config.yml with a scrape_config
- [ ] RUNBOOK includes `journalctl -u prometheus --since '1 hour ago'` for Prometheus troubleshooting
- [ ] At least one systemd unit file for a component (if bare-metal deployment)
- [ ] Reference to `/var/log/journal/` persistence in ARCHITECTURE
- [ ] node_exporter's textfile collector mentioned in ADR (how sysadmins expose custom metrics)

## Cross-project connections

- **Scrapes Project 2:** node_exporter deployed by Project 2's Ansible role
- **Scrapes Project 4:** ArgoCD metrics, kube-state-metrics, cluster health
- **Scrapes Project 6:** sample app's `/metrics` endpoint
- **Ingests logs from Project 3:** WireGuard session logs forwarded to Loki
- **Deployed into Project 4's cluster:** this IS the monitoring component of that cluster
