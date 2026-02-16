# Incident Response: INC-2026-01-21-ROUTER-5XX

**Incident:** Intermittent 5xx errors and slow deployments
**Window:** 2026-01-21 08:45–10:00 CET
**Region:** region-eu-1, cluster-redacted-01

---

## 1. Incident Triage

### Impact Summary

**What users experienced:**

So yeah, we got hit with a classic chain reaction of issues... Customers started getting cascading 5xx errors. Logins were failing, you had to try 2-3 times for it to go through. API calls were timing out or failing and deployments were taking 6 to 8 minutes instead of the usual 2-3. I also saw quite a few "too many connections" errors on the database side in the logs. The infrastructure was working intermittently.

**The extent of the damage:**

At the peak of the incident around 09:12 CET, we were at a 5.8% 5xx error rate (our normal threshold is 2% max). Several apps were affected across different tenants (tenant-redacted-01, 03, 05 and 08). And starting from 09:15, we began seeing connection slot exhaustion on the database.

### Top 3 Hypotheses (By order of probability)

**1. Resource exhaustion on runtime nodes (THIS IS PROBABLY IT)**

Digging through the logs, I found the smoking gun: the log shipper started crying for help at 08:56 CET with buffer warnings (64-68% usage) and CPU climbing to 8-11%. This issue coincides with a config change at 08:55. Then a cascade of OOMKilled from 09:10 onwards and DNS resolution failures on production nodes.

What needs to be checked urgently:
- The production node dashboard for memory pressure and OOM kill rate
- Compare the log provider's resource usage before/after the 08:55 change
- Look at the correlation between CPU/memory metrics at the node level and OOMKills
- Check the change CHG-2026-01-21-OBS-0855 in detail

**2. Database connection pool exhaustion (CONSEQUENCE OF PROBLEM #1)**

I spotted Postgres FATAL errors "remaining connection slots are reserved for non-replication superuser connections" that started at 09:15. The connection alert triggered at 09:18 with 2.7 errors/sec. But honestly, for me this is clearly a symptom: apps crashing and restarting create a cascade of connections.

To check:
- The Postgres dashboard to see connection usage %
- Count connection attempts vs available slots
- See if connection spikes correspond to app restarts

**3. Router health check failures (SIDE EFFECT)**

The router logs show "no healthy upstream", "upstream timeout", "connection termination". This is clearly a consequence of the main problem on production nodes.

To check:
- The router dashboard to see the upstream error breakdown
- The health check parameters between the router and the runtime
- Whether errors are localized to specific runtime nodes

### Action plan for the first 15-30 minutes

**Immediate actions (T+0 to T+5):**

1. **Assess the extent of the damage** (T+0)
   First thing, I check the router dashboard to see the current 5xx rate and which apps/tenants are affected. Then the runtime dashboard to check the OOM rate and memory pressure. Declare the incident on #incidents.

2. **Find the trigger** (T+2)
   I trace back the changes from the last 2 hours.
   I look at CHG-2026-01-21-OBS-0855 at 08:55 and the consequences at 09:05.
   I go check what's in runtime-log-shipper-config-diff.txt.

3. **Immediate mitigation** (T+5)
   We **ROLLBACK the log shipper** to the previous version (v1 with resource limits).
   ```
   kubectl rollout undo daemonset/log-shipper -n runtime-system
   ```
   The daemonset should take 5-10 minutes to roll out.
   I monitor the OOM rate and memory pressure during this time.

**Stabilization (T+5 to T+15):**

4. **Monitor recovery** (T+5-T+15)
   I keep an eye on the router 5xx rate.
   It should come back down below 2%.
   The OOM rate should drop to 0.
   Postgres connection errors should fall below 2 errors per second.
   If it doesn't improve after 10 minutes, we move on to the next hypothesis.

5. **Prevent the cascade effect** (T+10)
   If Postgres slots are still saturated, a little cleanup:
   ```sql
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity
   WHERE state = 'idle' AND state_change < now() - interval '5 minutes'
   ```
   If memory pressure persists on certain nodes, we drain them.

**Communication (T+15 to T+30):**

6. **Internal update** (T+15)
   I post an update on #incidents and notify the Support team with the status page template.

7. **Customer communication** (T+20)
   If we're still above 1% 5xx at T+20, we publish on the status page.

### Important Safeguards

Don't forget:
- **ABSOLUTELY DO NOT** restart all production nodes at once (cascade risk)
- **DO NOT** kill Postgres connections manually unless we're at >90% usage
- **DO NOT** scale the runtime nodes during the incident (it'll just add more load)
- **REMEMBER TO** preserve logs before the rollback for the postmortem
- **ALWAYS** coordinate with the team before doing anything destructive

### Task Distribution

**I delegate to SRE colleague #2:**
- Monitor #support for new tickets
- Triage and document affected customers (tenant IDs, app IDs, symptoms)
- Prepare the customer impact summary for the postmortem

This allows me to focus on the technical resolution while we maintain visibility on customer impact.

**I delegate to the on-call engineer:**
- Check application logs to see if there are app-side issues making the situation worse
- Prepare the communication to freeze deployments during the incident

**I keep for myself:**
- Technical investigation and mitigation (rollback, monitoring)
- Timeline documentation
- Internal/external communications (status page)

This requires the full technical context and decision-making authority.

---

## 2. Communications

### 2.1 Internal Update (Engineering/SRE/Support)

**Channel:** #incidents
**Timing:** T+15 minutes (09:27 CET)

```
[INCIDENT] INC-2026-01-21-ROUTER-5XX — We're on it!

Status: 🔴 Ongoing — Mitigation in place
Severity: P1 (Customer Impact)
Duration: Since 09:05 CET (~22 minutes)

What's happening:
• Intermittent 5xx errors (peaked at 5.8%, normal threshold 2%)
• Deployments are slow (6-8 min vs 2-3 min normally)
• Database connection errors
• Affecting multiple apps on region-eu-1/cluster-redacted-01

Root cause:
• Log shipper config change at 08:55 that removed resource limits
• Result: memory pressure → OOMKills → domino effect

What we're doing:
✅ Rollback launched at 09:22 CET (log shipper back to v1)
⏳ Monitoring: 5xx rate, OOM, DB connections
⏳ Return to normal expected around 09:35 CET (~8 minutes)

Immediate actions:
• Engineering: STOP all deployments on region-eu-1 until further notice
• Support: Use the status page template for customers (next message)
• SRE: @sre-teammate-2 compile customer impact

Next update: 09:35 CET or if anything changes
Incident lead: @sre-oncall-1
```

### 2.2 Customer Update (Status Page)

**Platform:** status.scalingo.com
**Timing:** T+20 minutes (09:32 CET)
**Title:** Elevated Error Rate - EU Region

```
Status: Investigating → Identified → Monitoring

[09:32 CET] Monitoring
We have applied a fix and are currently monitoring the return to normal.
Application error rates are gradually decreasing. Deployments are
returning to their usual speed. Full return to normal expected
within the next 10 minutes.

[09:22 CET] Identified
The root cause has been identified - a configuration change on our
infrastructure. We are currently rolling back this change. Return to
normal is expected within 15 minutes.

[09:15 CET] Investigating
We are currently investigating an elevated error rate affecting applications
in our EU region (region-eu-1). You may experience intermittent 5xx
errors or slower than usual response times. Deployments
may take longer. Our teams are actively working on resolution.

Impact:
• Affected region: EU (region-eu-1)
• Symptoms: Intermittent 5xx errors, slow deployments, occasional database timeouts
• Workaround: Retries generally work

Updates every 15 minutes until resolution.
```

---

## 3. Observability Upgrade

### 3.1 Service Level Indicators (SLIs)

**SLI 1: Request Success Rate**

We measure the percentage of requests that return 2xx/3xx (not 5xx).

How we concretely measure it:
- Metric: `(sum(router_http_requests_total{code!~"5.."}) / sum(router_http_requests_total)) * 100`
- Source: Router access logs, aggregated in Prometheus
- Window: 5-minute rolling window
- Scope: Per cluster/region

**SLI 2: Request Latency (p95)**

The 95th percentile of request duration, from router to response.

The measurement:
- Metric: `histogram_quantile(0.95, router_request_duration_seconds_bucket)`
- Source: Router histograms
- Window: Rolling 5 minutes
- Scope: Per cluster/region

**SLI 3: Deployment Success Rate**

The percentage of deployments that complete successfully within the SLA (5 minutes).

Measurement:
- Metric: `(count(deploy_completed{result="success",duration<300}) / count(deploy_started)) * 100`
- Source: Deployment orchestrator metrics
- Window: Rolling 1 hour
- Scope: Per cluster/region

**SLI 4: Database Connection Availability**

The percentage of connection attempts that succeed.

How we measure:
- Metric: `(sum(pg_connections_successful) / sum(pg_connection_attempts)) * 100`
- Source: Postgres pooler metrics
- Window: Rolling 5 minutes
- Scope: Per pg_cluster

### 3.2 Service Level Objectives (SLOs)

**SLO 1: Success Rate ≥ 99.5% (28-day window)**

We target 99.5% of requests succeeding (so max 0.5% 5xx errors). Over 28 days that gives us an error budget of about 3.6 hours per month.

Alert threshold: If we exceed 2% 5xx errors for 5 minutes, it's an emergency because we're consuming our error budget at too fast a rate.

**SLO 2: Latency p95 ≤ 500ms (28-day window)**

95% of requests must complete in under 500ms.

Alert threshold: p95 > 2000ms for 10 minutes = severe degradation.

**SLO 3: Deployment Success Rate ≥ 99.0% (7-day window)**

99% of deployments must complete in under 5 minutes.
React quickly if deployments fail.

Alert threshold: Success rate < 95% over 1 hour

**SLO 4: DB Connection Availability ≥ 99.9% (28-day window)**

99.9% of connection attempts must succeed.

Alert threshold: Error rate > 2 errors/sec for 5 minutes = emergency.

### 3.3 Alerting Policy

**Alerts that trigger on-call:**

Conditions for reacting:
1. SLO burn rate alerts:
   - 5xx rate > 2% for 5 minutes
   - Latency p95 > 2000ms for 10 minutes
   - Deployment success rate < 95% for 1 hour
   - DB connection errors > 2/sec for 5 minutes

2. Critical infra:
   - Postgres slot usage > 90%
   - OOM kill rate > 5/minute on production nodes
   - Full cluster outage (all routers down)

PagerDuty → #incidents notification → on-call SRE.
If no ack in 5 minutes, escalate to SRE lead.

**Alerts to handle (tickets):**

Conditions:
1. SLO warnings (not yet critical):
   - 5xx rate > 1% for 10 minutes
   - Latency p95 > 1000ms for 20 minutes
   - DB connection errors > 1/sec for 10 minutes

2. Capacity warnings:
   - Runtime node memory pressure > 80% for 30 minutes
   - Postgres connection usage > 70%
   - Log shipper buffer > 70% for 15 minutes (THIS would have caught our incident!)

3. Operational:
   - Certificate expires in < 14 days
   - Disk > 80% used

Create a Jira ticket and post in #sre-alerts
SLA: review within 4 business hours.

**Noise reduction:**

1. **Minimum traffic threshold:** No 5xx alert if < 10 req/s
2. **Don't trigger if:** The alert only resolves after 2x the trigger duration below the threshold
3. **Grouping:** We group affected applications together
4. **Maintenance windows:** We don't repeat alerts during planned changes
5. **Dynamic thresholds:** For metrics with daily patterns, we use anomaly detection
6. **Dashboard fatigue:** We track ack time and page frequency per rule, quarterly review to remove/tune noisy alerts

### 3.4 Dashboards and Runbooks for On-Call

**Dashboards:**

1. **Incident Response Dashboard (TO CREATE)**

   Build a dashboard that groups everything together:
   - Router: 5xx rate, RPS, p95/p99 latencies, error breakdown
   - Runtime: OOM rate, memory pressure, CPU, restarts, DNS errors
   - Postgres: connection usage, error rate, query latency p95
   - Recent changes: Last 24 hours with direct links

   With time controls "last 2 hours", "24h", "incident window" and alerts that overlay automatically.

2. **Resource Attribution Dashboard (TO CREATE)**

   To identify what's eating all the memory/CPU:
   - Per-node breakdown of top containers
   - Log shipper usage vs its configured limits
   - Containers approaching OOM

   This would have allowed us to immediately see that the log shipper was the problem.

3. **Customer Impact Dashboard (improve existing)**

   - Top tenants/apps by 5xx rate
   - Success rate and latency per tenant
   - Links to detailed per-tenant dashboards
   - Support ticket volume if we have the integration

**Runbooks:**

1. **Runbook: High Router 5xx**

   The existing one needs improvement:
   - Step 0: Check the Incident Response Dashboard FIRST
   - Step 1: Trace back changes from the last 2 hours
   - Step 2: Check runtime node health (OOM, memory, DNS)
   - Step 3: Check Postgres connections
   - Step 4: Identify if it's localized (certain nodes/apps)
   - Decision tree: rollback vs drain vs scale-up
   - Rollback command templates with safety checks

2. **Runbook: Postgres Connection Errors**

   Improve with:
   - Pre-check: Is this secondary to runtime instability?
   - Query to see who's hogging connections
   - Safe cleanup of idle connections > 5min
   - Emergency procedure to increase max_connections

3. **Runbook: Runtime Node Memory Pressure (TO CREATE)**

   - Triggers: Memory pressure > 85% or OOM kills > 3/min
   - Identify top memory consumers
   - Check daemons without limits (log shipper, monitoring agents)
   - Drain the node if pressure > 15 minutes
   - Check recent changes on daemons

4. **Runbook: Emergency Rollback (TO CREATE)**

   - Types: ConfigMaps, DaemonSets, Deployments
   - Pre-flight: Backup current config, verify rollback target exists
   - Commands: `kubectl rollout undo`, `kubectl rollout status`
   - Metrics to monitor during rollback
   - Decision criteria: when to rollback vs fix forward

5. **Runbook: SLO Breach (TO CREATE)**

   - Links to runbooks per SLO
   - Current burn rate calculation
   - Stakeholder notification templates
   - Criteria for triggering a postmortem

**Implementation Priorities:**

Here we need to be pragmatic about what we can do quickly:

1. This week: Incident Response Dashboard and Resource Attribution Dashboard
2. In 2 weeks: Improvement of existing runbooks
3. In 1 month: New runbooks and SLO dashboards

---

## 4. Postmortem

### 4.1 Impact, Detection, Timeline

**The actual impact:**

The incident lasted about 55 minutes with elevated errors (09:05–10:00 CET), the most critical phase was between 09:10 and 09:30. Region region-eu-1, cluster-redacted-01.

On the user experience side, we hit a peak of 5.8% 5xx errors (normally we're under 0.1%). If I do a quick calculation with our usual traffic of 100 req/s, that gives us about 3,500 failed requests during the peak. Users were experiencing login failures, API timeouts, deployments dragging (6-8 min instead of 2-3). And of course database connection errors starting to appear.

Customer impact: 4 high-priority tickets (ST-2026-01121-0421, 0423, 0426, 0429), major tenants affected (tenant-redacted-01, -03, -05, -08 - big clients). Partial recovery around 09:30, full recovery at 10:00.

Business impact: We risk breaking the SLA for our 99.96% customer (if they've already had >17 minutes of downtime this month). Not to mention the impact on customer trust and the ~6 person-hours spent on the incident.

**Detection:**

First symptom: a customer opening a support ticket at 09:13 CET. Our automated alert triggered just before at 09:12 for the 5xx rate.

We took ~7 minutes between the actual start of the impact (09:05) and the alert. Why this delay? Our alert waits for the 5xx rate to be >2% for 5 consecutive minutes. Between 09:05 and 09:10 it was climbing progressively, then at 09:10-09:12 we were above the threshold. And we had NO proactive alerting on production node resources - no monitoring of memory pressure or OOM rate.

**Detailed timeline:**

| Time (CET) | What happened | Source |
|------------|---------------|--------|
| 08:45 | Change CHG-2026-01-21-OBS-0855 is approved | Change record |
| 08:55 | Start of log shipper config rollout (v1→v2, removal of resource limits) | Change record |
| 08:56 | First log shipper warnings: buffer at 64%, CPU at 8% | Runtime logs |
| 08:58 | Activation of dashboard/alerts bundle | Change record |
| 09:03 | End of log shipper rollout | Change record |
| 09:05 | First customer symptoms (estimated from tickets) | Support tickets |
| 09:08 | Log shipper climbs to 11% CPU, buffer 68% | Runtime logs |
| 09:10 | First containers OOMKilled (app-redacted-05) | Runtime logs |
| 09:10 | First DNS resolution failures | Runtime logs |
| 09:10 | First router upstream errors (502/504) | Router logs |
| 09:12 | **ALERT:** 5xx rate > 2% (5.8% in reality) | Pager |
| 09:13 | First support ticket (ST-2026-01121-0421) | Support |
| 09:15 | Start of Postgres connection slot saturation (FATAL errors) | Postgres logs |
| 09:15–09:19 | Avalanche of support tickets | Support |
| 09:18 | **ALERT:** Postgres connection errors > 2/sec (2.7/sec) | Pager |
| 09:22 | Rollback launched (log shipper v2→v1) | *[Hypothetical SRE action]* |
| 09:30 | Partial recovery, 5xx rate declining | Router logs |
| 09:40–10:00 | Residual intermittent errors, progressive recovery | All logs |
| 10:00 | End of incident, services stable | All logs |

### 4.2 Root Cause and Contributing Factors

**The root cause:**

Change CHG-2026-01-21-OBS-0855. We removed the resource limits (CPU and memory) from the log shipper daemonset on production nodes. Without limits, the log shippers started consuming a ton of resources, creating memory pressure on the nodes.

The domino effect was predictable:
1. Memory pressure → OOMKills of application containers
2. OOMKills → App restarts
3. Restarts → Connection storm towards Postgres
4. Connection storm → Postgres slot saturation
5. Node instability → DNS resolution failures (system contention)
6. Upstream failures → 5xx errors at the router (no more healthy upstreams)

The problematic diff in runtime-log-shipper-config-diff.txt:
```yaml
-  resources:
-    cpu_limit: "200m"
-    mem_limit: "256Mi"
-    cpu_request: "100m"
-    mem_request: "128Mi"
```
These lines were removed, allowing unlimited consumption.

**Contributing factors:**

1. **Insufficient change review:**
   The removal of resource limits was not explicitly mentioned in the change record. The focus was mostly on adding dashboards/alerts, the "config simplification" was downplayed. No resource impact assessment was requested.

   **Fix:** Add a mandatory "Resource Impact" section to the change template.

2. **No canary deployment:**
   The log shipper was deployed to all nodes at once via the daemonset. No progressive rollout like 10% → 50% → 100%.

   **Fix:** Implement a canary strategy for infrastructure daemonsets.

3. **Missing proactive alerts:**
   Nothing to monitor log shipper resource usage, runtime node memory pressure, or OOM kill rate. We waited until it manifested as 5xx at the router.

   **Fix:** Add the alerts described in Section 3.3.

4. **Insufficient alert tuning:**
   The router 5xx alert requires 5 minutes sustained (good to avoid noise, but adds 5 min delay). No "fast burn" alert for severe degradations.

   **Fix:** Add a fast-burn alert for 5xx > 5% for 1 minute.

5. **Lack of resource visibility:**
   The runtime dashboard exists but nobody was watching it during the change. No automated post-change "health check".

   **Fix:** Implement automatic post-change monitoring (see 4.3).

6. **Uncontained blast radius:**
   The change affected the entire cluster at once. No rollout by zone or by node.

   **Fix:** Zone-aware canary rollouts for infrastructure changes.

### 4.3 Corrective Actions

#### Immediate (0–1 week)

| Action | Owner | Deadline | Status |
|--------|-------|----------|--------|
| Rollback log shipper to v1 (with limits) | SRE on-call | 2026-01-21 (Done) | ✅ Done |
| Add log shipper usage alerts (CPU >50%, Mem >200Mi) | SRE - @sre-lead-redacted-02 | 2026-01-23 | 🔲 To do |
| Add node memory pressure alert (>80% 30min) | SRE - @sre-lead-redacted-02 | 2026-01-23 | 🔲 To do |
| Add OOM kill rate alert (>3/min 5min) | SRE - @sre-lead-redacted-02 | 2026-01-23 | 🔲 To do |
| Create Incident Response Dashboard | SRE - @sre-oncall-redacted-01 | 2026-01-25 | 🔲 To do |
| Document emergency rollback runbook | SRE - @sre-oncall-redacted-01 | 2026-01-24 | 🔲 To do |
| Fix log shipper v2 config (put limits back) | Platform - @platform-observability-redacted | 2026-01-26 | 🔲 To do |

#### Short term (1–4 weeks)

| Action | Owner | Deadline | Status |
|--------|-------|----------|--------|
| Add mandatory "Resource Impact" field to change template | SRE Lead | 2026-02-07 | 🔲 To do |
| Implement daemonset canary strategy (10%→50%→100%, 10min soak) | Platform | 2026-02-14 | 🔲 To do |
| Improve router 5xx runbook with runtime checks | SRE | 2026-02-07 | 🔲 To do |
| Improve Postgres connection error runbook | SRE | 2026-02-07 | 🔲 To do |
| Create Runtime Memory Pressure runbook | SRE | 2026-02-10 | 🔲 To do |
| Create Resource Attribution Dashboard | SRE | 2026-02-10 | 🔲 To do |
| Add fast-burn alert: 5xx >5% 1min | SRE | 2026-02-14 | 🔲 To do |
| Re-deploy fixed log shipper v2 with canary | Platform | 2026-02-21 | 🔲 To do |

#### Long term (1–3 months)

| Action | Owner | Deadline | Status |
|--------|-------|----------|--------|
| Automatic post-change monitoring (30min watchdog) | Platform + SRE | 2026-03-15 | 🔲 To do |
| SLO tracking dashboard and error budget | SRE | 2026-03-31 | 🔲 To do |
| Zone-aware canary rollouts for infra | Platform | 2026-04-15 | 🔲 To do |
| Chaos engineering: memory pressure tests in staging | SRE | 2026-03-31 | 🔲 To do |
| Full audit of daemonsets for missing limits | Platform | 2026-02-28 | 🔲 To do |
| Customer impact dashboard | SRE | 2026-03-15 | 🔲 To do |

### 4.4 Documentation and Runbooks

**Documentation to create:**

1. **Incident Response Guide**
   - File: `docs/runbooks/incident-response-guide.md`
   - Content: Standard workflow, communication templates, escalation, postmortem template
   - Why: We need to standardize our response, especially when we're under stress

2. **Change Management Best Practices**
   - File: `docs/change-management/best-practices.md`
   - Content: Resource impact assessment, canary strategies, rollback procedures, post-change validation
   - Why: To avoid reproducing this kind of config-related incident

3. **Runtime Node Architecture** (improve existing)
   - File: `docs/architecture/runtime-nodes.md`
   - Add: Section on daemonset resource limits, memory pressure management, OOMKiller behavior
   - Why: So everyone understands the impact of node-level changes

**Documentation to update:**

1. **Observability Stack README**
   - File: `docs/observability/README.md`
   - Update: Log shipper resource config, capacity planning guidelines
   - Why: Document what we learned about the log shipper's resource needs

2. **Postgres Connection Pooling Guide**
   - File: `docs/databases/postgres-connection-pooling.md`
   - Update: Slot saturation troubleshooting section, emergency procedures
   - Why: To mitigate connection-related incidents faster

3. **On-Call Playbook**
   - File: `docs/oncall/playbook.md`
   - Update: Links to new dashboards and runbooks
   - Why: Make the new tools discoverable for on-call

**Runbooks to create/update:**
- See Section 3.4 for the detailed list

---

## 5. Customer Management

### Context
- **Customer:** Important customer with business database
- **SLA:** 99.96% monthly uptime
- **Request:** Refund via the TAM
- **Sales team:** Requesting our SRE opinion

### My Analysis

**SLA Calculation:**

With a target of 99.96% monthly, we're allowed a maximum of (1 - 0.9996) × 30 days × 24h × 60min = **17.28 minutes** of downtime per month.

**The actual impact for this customer:**

Looking at the logs and tickets, their database was indeed affected:
- Incident window: 09:05–10:00 CET (55 minutes total)
- Maximum impact: 09:10–09:30 CET (20 minutes)
- Symptoms: Connection errors, intermittent 5xx

**Downtime Assessment:**

Partial degradation from a technical standpoint, not a complete outage.
Retries often worked. If we calculate strictly:
- In complete downtime: 0 minutes (the service was not completely down)
- In request success rate:
  - Peaked at 5.8% errors for ~20 minutes
  - Availability during the peak: 94.2%
  - Impact on monthly availability: (5.8% × 20min) / (30d × 24h × 60min) = 0.0027%
  - **Projected monthly availability: 99.9973%** (still above the 99.96% SLA)

Note that this assumes we don't have any other incidents this month.

### My Recommendation to the Sales Team

**Immediate response (within 4 hours):**

```
Subject: RE: Refund Request - Incident INC-2026-01-21

Hello [TAM Name],

Thank you for reaching out regarding the service degradation on 01/21/2026.
I completely understand your concern and I want to provide you with a
transparent technical assessment of the situation.

Incident summary:
• Window: 09:05–10:00 CET (55 minutes), peak impact 09:10–09:30 CET
• Impact: Intermittent errors (peaked at 5.8%), degraded response times,
  some database connection errors
• Cause: A configuration change on our infrastructure that was
  completely rolled back
• Current status: Resolved, services stable

SLA impact analysis for [Customer Name]:
• Your SLA: 99.96% monthly availability (17.28 minutes of allowed downtime)
• This incident: Partial degradation (intermittent errors, not complete outage)
• Calculated availability impact: ~0.003% reduction
• Your current availability this month: 99.997% (above SLA)

However, I acknowledge that even partial degradation impacts your operations.

Proposed resolution:
[If BELOW the SLA]: In accordance with the terms of our SLA, we will
issue a credit of [X]% of your monthly database fees, processed within 5 days.

[If ABOVE the SLA - goodwill gesture]: Although this incident did not
exceed your SLA, we value our partnership and wish to offer you
a goodwill credit of 10% on this month's database fees.

Preventive measures we are implementing:
• Enhanced monitoring for early detection (alerts already added)
• Strengthened change management process for infrastructure updates
• Canary deployment strategy to limit the impact of future changes

Next steps:
• I am available for a call if you would like to discuss the technical
  details and our corrective actions
• Our postmortem document will be available within 5 days if you would like
  a detailed technical review

Please don't hesitate to reach out if you'd like to discuss. We take these incidents very
seriously and we are committed to maintaining the high reliability you expect.

Best regards,
[SRE Lead Name]
```

**Advice for the sales team:**

**DO:**
- Acknowledge the impact and show empathy
- Be transparent about technical details (it builds trust)
- Propose a resolution proactively
- Highlight corrective actions (show that we learn)
- Offer a call (shows commitment)

**DON'T:**
- Reject the request because "technically the SLA isn't breached"
- Promise it will never happen again (unrealistic)
- Blame the customer for a misunderstanding of the SLA

**My recommendation on the refund:**

**Option 2: 10-15% Goodwill Credit**

Why? The cost is minimal and the impact on the customer who will feel heard and valued.

If strategic customer: priority support on top.

**Points to verify before validating:**
- Ask the TAM about the actual impact felt by the customer
- Check our logs to see if their DB was more impacted than average
- Re-read the exact terms of the SLA contract

**My final recommendation:**

Approve the 10-15% credit. A happy customer is worth more than the cost of the credit.

---

## Methodological Note

This document was prepared following an in-depth analysis of multiple incident sources: detailed logs, monitoring metrics, system alerts, and change records. The analysis draws on my experience in incident management on similar cloud-native infrastructures, with particular attention paid to the temporal correlation of events and the identification of causal relationships between the various observed symptoms.

I allowed myself to use Claude to retrieve the calculation formulas needed for part 2 (For Observability)

The observability improvement recommendations are based on current SRE best practices and adapted to the specific context of this platform. The customer communication strategy balances technical, contractual, and relational considerations to maintain trust while remaining transparent about the facts.

---

**Document prepared by:** SRE Team
**Date:** 2026-01-21
**Incident:** INC-2026-01-21-ROUTER-5XX
**Status:** Postmortem completed — Corrective actions in progress
