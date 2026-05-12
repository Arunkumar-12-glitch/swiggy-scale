# Incident Runbook: SwiftEats Backend Scale Event Response

**For:** Site Reliability Engineers responding to outages during high-traffic events  
**When:** Activated by high-severity alerts (5xx error rate >5%, latency P99 >2s, or explicit page)  
**Duration:** 30 seconds detection → 5 minutes triage → 10 minutes response  
**Owner:** On-call SRE (primary), DBA (backup), Payment team (if needed)

---

## STEP 1: DETECT — Alert Thresholds and Activation

You are reading this because one of these alerts fired. **Confirm which one, then go to Step 2.**

### Critical Alerts (IMMEDIATE response required)

| Alert Name | Metric | Threshold | Severity | Response |
|------------|--------|-----------|----------|----------|
| **APIError5xxSpike** | ALB: HTTP 500 errors | >5% of requests for 2 min | CRITICAL 🔴 | Page SRE on-call |
| **DatabaseConnectionPoolExhausted** | RDS: DatabaseConnections | ≥95/100 for 1 min | CRITICAL 🔴 | Page DBA |
| **NodeCPUSaturation** | EC2: CPU Utilization | ≥90% on ≥50% of fleet | CRITICAL 🔴 | Trigger auto-scaling |
| **RedisEvictionRate** | ElastiCache: EvictionRate | >100 evictions/sec for 2 min | CRITICAL 🔴 | Add cache nodes |
| **PaymentQueueBacklog** | SQS: ApproximateNumberOfMessages | ≥50,000 for 3 min | HIGH 🟠 | Check payment worker |
| **LatencyP99Spike** | ALB: TargetResponseTime | P99 >2 seconds for 2 min | HIGH 🟠 | Check root cause |
| **LoadBalancerUnhealthyInstances** | ALB: HealthyCount | <50% of fleet for 1 min | HIGH 🟠 | Check instance logs |

### Warning Alerts (Monitor, investigate next)

| Alert Name | Threshold | Action |
|------------|-----------|--------|
| Database CPU > 70% | Check query performance | Review slow-query logs |
| Redis Memory > 80% | Monitor eviction | Prepare for node addition |
| RDS Replication Lag > 500ms | Check network | Verify replica is healthy |
| SQS Visibility Timeout Exceed | Messages timing out | Scale worker processes |

### Dashboard to Open Immediately

```
AWS CloudWatch Dashboard: "SwiftEats-RealTime"
  
Open these 4 tiles in order:
  1. ALB: Request Count + Error Rate (top-level system health)
  2. RDS: DatabaseConnections + Query Performance Insights
  3. EC2: CPU Utilization across fleet (compute saturation)
  4. ElastiCache: Evictions + Hit Rate (cache effectiveness)

Open these optional tiles if alarm is not obvious:
  5. SQS: Approx Number of Messages (payment queue depth)
  6. CloudFront: Error Rate + Edge Latency (CDN health)
```

---

## STEP 2: TRIAGE — Identify Root Cause (30 seconds)

**Your job:** Determine which component failed FIRST. The first failure causes all others.

**Read the 4 dashboards in this exact order. The FIRST one that shows red = your root cause.**

### 2a. Check Database Connections

**Dashboard:** CloudWatch → RDS → DatabaseConnections

```
Looking for: Spike to ≥90 connections, or error messages

✅ HEALTHY:
  - DatabaseConnections = 10–50 (normal load)
  - Query latency P50 = 10–20ms
  - Query latency P99 = 50–100ms
  - Replication lag <100ms

🔴 UNHEALTHY (ROOT CAUSE: DB Pool):
  - DatabaseConnections = 100/100 (EXHAUSTED)
  - New connection requests: "too many connections" errors in app logs
  - Query latency P99 = 5000+ms (queries waiting for connection)
  - Replication lag >5 seconds (replica can't keep up)

If this is red → GO TO STEP 3a
If this is healthy → Go to 2b
```

### 2b. Check EC2 CPU Saturation

**Dashboard:** CloudWatch → EC2 → CPUUtilization (all instances)

```
Looking for: Sustained CPU >85% on majority of fleet

✅ HEALTHY:
  - CPU Utilization = 30–60% (normal headroom)
  - Avg response time = 50–100ms per request
  - Event loop healthy (no GC pauses >100ms visible in logs)

🔴 UNHEALTHY (ROOT CAUSE: Compute Saturation):
  - CPU >85% on ≥50% of instances
  - Response times: 100ms → 1000ms → 5000ms (degrading)
  - New instances not launching (or launched but not helping)
  - Memory RSS growing: 500MB → 2GB → approaching 4GB limit

If this is red → GO TO STEP 3b
If this is healthy → Go to 2c
```

### 2c. Check Redis Cache Health

**Dashboard:** CloudWatch → ElastiCache → CachePerformance

```
Looking for: Hit rate drop OR eviction spike

✅ HEALTHY:
  - Hit Rate = 85–95% (most requests cached)
  - Evictions = 0–10/sec (rare)
  - Memory Usage = 30–70% of total capacity
  - CPU per node = <20%

🔴 UNHEALTHY (ROOT CAUSE: Cache Failure):
  - Hit Rate suddenly drops to <50%
  - Evictions spike to 1000+/sec
  - OR: Node is down/unreachable (node count drop)
  - OR: Memory full, evicting everything
  - Database queries suddenly spike (requests no longer finding cache)

If this is red → GO TO STEP 3c
If this is healthy → Go to 2d
```

### 2d. Check Payment Queue Depth

**Dashboard:** CloudWatch → SQS → ApproximateNumberOfMessages

```
Looking for: Queue depth >10,000 OR payment worker down

✅ HEALTHY:
  - Message count = 0–1,000 (messages processed immediately)
  - Message age max = <30 seconds
  - Worker processes: 5 workers, each consuming 200 msg/sec

🔴 UNHEALTHY (ROOT CAUSE: Payment Worker Down):
  - Message count >50,000 (backing up)
  - Message age max >5 minutes (stuck messages)
  - Worker dead/crashed (check ECS task logs)
  - DLQ messages increasing (payment calls failing)

If this is red → GO TO STEP 3d
If all 4 are healthy → Problem is not in these components
  → Possible: ALB unhealthy targets, deployment in progress, or network issue
  → GO TO STEP 3e: "Advanced Troubleshooting"
```

---

## STEP 3: RESPOND — Component-Specific Recovery Actions

### 3a: Database Connection Pool Exhausted

**Symptom:** DatabaseConnections = 100/100, new queries get "too many connections" error

**Root cause is one of:**
- (Scenario A) Synchronous payment calls holding connections for 800ms each
- (Scenario B) Slow query running, holding a connection for minutes
- (Scenario C) Connection leak in application code

**Recovery steps (in order):**

#### Action 3a.1: Verify Payment Worker is Running
```bash
# Check if payment worker service is alive
aws ecs describe-services \
  --cluster swiggy-prod \
  --services payment-worker \
  --query 'services[0].[runningCount,desiredCount]'

# Expected output: runningCount = desiredCount (e.g., 5 = 5)
# If not: aws ecs update-service --cluster swiggy-prod --service payment-worker --desired-count 5
```

**Signal of success:** Worker restarts, queue begins draining, database connections drop to <80 within 2 minutes.

---

#### Action 3a.2: Kill Long-Running Queries
```bash
# Connect to database
psql -h swiggy-prod-db.xxxxx.rds.amazonaws.com -U admin -d swiftdb

# Find long-running queries
SELECT pid, usename, state, query_start, query 
FROM pg_stat_activity 
WHERE state != 'idle' AND query_start < NOW() - INTERVAL '10 minutes'
ORDER BY query_start;

# Kill any query running >10 minutes
SELECT pg_terminate_backend(PID);
```

**Signal of success:** Terminated queries, connections freed, new requests go through.

---

#### Action 3a.3: Restart PgBouncer (if pool corruption)
```bash
# SSH into the PgBouncer instance
ssh ec2-user@pgbouncer-1.swiggy.internal

# Check bouncer status
sudo systemctl status pgbouncer

# Reload configuration (no connection loss)
sudo pgbouncer -R /etc/pgbouncer/pgbouncer.ini

# If still stuck, restart (may lose 10–20 in-flight connections)
sudo systemctl restart pgbouncer
```

**Signal of success:** Bouncer restarts, accepts new connections, pool recovers.

---

#### Action 3a.4: Check Application Code for Connection Leaks
```bash
# Get app logs - search for unclosed connections
kubectl logs -n prod deployment/swiftdb-app --tail=500 | grep "connection leak"

# Check if any middleware is holding connections
grep -r "connection.acquire" src/middleware --include="*.js"

# Look for .query() calls without proper callback cleanup
grep -r "\.query(" src/ | grep -v "\.then\|\.catch" | head -20
```

**If connection leak found:** Patch and redeploy (see Step 4 - Rollback if needed).

---

#### Action 3a.5: Monitor Recovery
```bash
# Watch connections drain in real-time
watch -n 1 'aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=swiggy-prod-db \
  --start-time $(date -u -Iseconds -d "5 minutes ago") \
  --end-time $(date -u -Iseconds) \
  --period 10 \
  --statistics Average'

# Expected: Connections drop from 100 → 50 → 20 (over 2–5 minutes)
# If connections stay at 100: Escalate to on-call DBA
```

**Recovery complete when:** DatabaseConnections <50 and error rate drops below 5%.

---

### 3b: Node.js Compute Saturation (CPU >85%)

**Symptom:** EC2 CPU >85% on majority of fleet, response times degrading, error rate climbing

**Root cause:** Event loop is saturated or blocked

**Recovery steps:**

#### Action 3b.1: Verify Auto-Scaling is Working
```bash
# Check auto-scaling activity
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name swiftdb-app-asg \
  --max-records 10

# Expected: Recent scaling activities (launching new instances)
# If none: Scaling rule may be misconfigured or disabled
# To enable:
aws autoscaling enable-metrics-collection \
  --auto-scaling-group-name swiftdb-app-asg \
  --granularity "1Minute"
```

#### Action 3b.2: Check for Memory Leaks or GC Pauses
```bash
# SSH into high-CPU instance
ssh ec2-user@node-app-1.swiggy.internal

# Check Node process memory
ps aux | grep "node app.js"
# Look for RSS column - should be <1GB normally
# If >3GB: Memory leak likely, prepare for restart

# Check for GC pauses in logs
tail -100 /var/log/app/debug.log | grep "GC pause"

# If many GC pauses >500ms: Memory is fragmented or leaking
# Action: Restart the instance (load balancer will reroute)
```

#### Action 3b.3: Restart Instance with Graceful Shutdown
```bash
# Make the instance "drain" (stop accepting new connections)
curl -X POST http://localhost:3000/admin/drain

# Wait 30 seconds for existing connections to close
sleep 30

# Then restart
sudo systemctl restart swiftdb-app

# ALB will detect the restart and reroute traffic
# CPU should drop within 1 minute
```

#### Action 3b.4: Verify Scaling is Happening
```bash
# Check desired count vs running count
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names swiftdb-app-asg \
  --query 'AutoScalingGroups[0].[DesiredCapacity,Instances[*].InstanceId]'

# Expected: DesiredCapacity increasing (4 → 6 → 8 → 10...)
# If not: Go to Action 3b.2 (check scaling policy)
```

**Recovery complete when:** CPU drops to <70%, error rate <5%, response time returns to baseline.

---

### 3c: Redis Cache Miss Rate High (or Node Down)

**Symptom:** Cache hit rate drops from 90% to <50%, eviction rate spikes, or node is unreachable

**Recovery steps:**

#### Action 3c.1: Check if Node is Down
```bash
# Check ElastiCache cluster status
aws elasticache describe-cache-clusters \
  --cache-cluster-id swiftdb-cache-001 \
  --show-cache-node-info \
  --query 'CacheClusters[0].CacheNodes[*].[CacheNodeId,CacheNodeStatus]'

# Expected: All nodes show "available"
# If any show "creating" or "deleting": Node is being recovered
# Wait 5 minutes for node recovery

# If node shows "incompatible-parameters": Restart the node
aws elasticache reboot-cache-cluster \
  --cache-cluster-id swiftdb-cache-001 \
  --cache-node-ids-to-reboot <node-id>
```

#### Action 3c.2: Check Memory Pressure
```bash
# Get ElastiCache metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name BytesUsedForCache \
  --dimensions Name=CacheClusterId,Value=swiftdb-cache-001 \
  --start-time $(date -u -Iseconds -d "10 minutes ago") \
  --end-time $(date -u -Iseconds) \
  --period 60 \
  --statistics Average

# If BytesUsedForCache is >90% of MaxMemory:
#   → Cache is full and evicting
#   → Add temporary cache nodes (see Action 3c.3)
```

#### Action 3c.3: Add Temporary Cache Nodes
```bash
# Check current node count
aws elasticache describe-replication-groups \
  --replication-group-id swiftdb-cache \
  --query 'ReplicationGroups[0].MemberClusters'

# Add temporary nodes (this is instantaneous)
aws elasticache increase-replica-count \
  --replication-group-id swiftdb-cache \
  --new-replica-count 4

# Wait 5 minutes for replicas to sync
sleep 300

# Verify
aws elasticache describe-replication-groups \
  --replication-group-id swiftdb-cache \
  --query 'ReplicationGroups[0].[MemberClusters,CacheNodeType]'
```

**Signal of success:** Hit rate recovers to >85%, evictions drop to <10/sec.

---

#### Action 3c.4: Warm Up Cache After Recovery
```bash
# If cache was completely lost, warm it up by triggering cache fills
curl -X POST http://localhost:3000/admin/cache-warmup

# This queries the database once per menu, caches all results
# Should complete in <30 seconds

# Verify cache hit rate recovering
watch -n 2 'aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CacheHits \
  --metric-name CacheMisses \
  --dimensions Name=CacheClusterId,Value=swiftdb-cache-001 \
  --start-time $(date -u -Iseconds -d "5 minutes ago") \
  --end-time $(date -u -Iseconds) \
  --period 60 \
  --statistics Sum'
```

**Recovery complete when:** Hit rate >85%, database connections drop <30.

---

### 3d: Payment Queue Backed Up (Worker Down)

**Symptom:** SQS ApproximateNumberOfMessages >50,000, messages aging >5 minutes

**Recovery steps:**

#### Action 3d.1: Check if Worker Service is Running
```bash
# Check ECS task status
aws ecs describe-services \
  --cluster swiggy-prod \
  --services payment-worker \
  --query 'services[0].[runningCount,desiredCount,deployments]'

# If runningCount < desiredCount:
#   → Tasks are failing to start
#   → Check ECS task logs

aws logs tail /ecs/payment-worker --follow --since 5m
```

#### Action 3d.2: Restart Worker Service
```bash
# Scale down then up (forces new task deployment)
aws ecs update-service \
  --cluster swiggy-prod \
  --service payment-worker \
  --desired-count 0

sleep 10

aws ecs update-service \
  --cluster swiggy-prod \
  --service payment-worker \
  --desired-count 5

# Wait 30 seconds for tasks to start
sleep 30

# Verify tasks are running
aws ecs describe-tasks \
  --cluster swiggy-prod \
  --tasks $(aws ecs list-tasks --cluster swiggy-prod --service-name payment-worker --query 'taskArns[]' --output text) \
  --query 'tasks[*].[taskArn,lastStatus]'
```

#### Action 3d.3: Monitor Queue Draining
```bash
# Watch queue depth in real-time
watch -n 5 'aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/xxx/payment-queue \
  --attribute-names ApproximateNumberOfMessages,ApproximateAgeOfOldestMessage'

# Expected: Messages drop from 50K → 40K → 20K (over 5 minutes)
# Queue processing rate: ~1000 messages/sec with 5 workers
```

#### Action 3d.4: Check Dead Letter Queue for Failed Payments
```bash
# If messages keep appearing in DLQ, payments are failing at provider
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/xxx/payment-dlq \
  --max-number-of-messages 10

# Inspect a DLQ message
# Check: Is Razorpay returning 500s? Network timeout? Invalid request?

# If provider is down:
#   → Cannot process payments until provider recovers
#   → Set payment status to "PENDING_RETRY"
#   → Alert Finance team that payments are delayed
#   → Continue retrying with exponential backoff
```

**Recovery complete when:** Queue depth <1,000, DLQ empty or slowly clearing.

---

### 3e: Advanced Troubleshooting (if none of 3a–3d apply)

**If all 4 components (DB, CPU, Cache, Queue) look healthy but errors continue:**

#### Action 3e.1: Check ALB Health Checks
```bash
# Verify target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Description]'

# Expected: All targets show "healthy"
# If any show "unhealthy": Check their logs
#   → /var/log/app/error.log
#   → Check if /health endpoint is responding
```

#### Action 3e.2: Check for Recent Deployment
```bash
# If errors started right after a deployment:
#   → Likely: code bug, not infrastructure
#   → Go to STEP 4: Rollback (roll back the app code)

# Check deployment history
aws ecs describe-task-definition \
  --task-definition swiftdb-app \
  --query 'taskDefinition.[taskDefinitionArn,revision,createdAt]'

# Check when last deployment happened
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutTaskDefinition \
  --max-results 5 \
  --query 'Events[*].[EventTime,Username]'
```

#### Action 3e.3: Check Network / CDN
```bash
# If images are failing to load (user-visible):
#   → Check CloudFront
aws cloudfront get-distribution-config \
  --id XXXXX \
  --query 'DistributionConfig.Origins[0].DomainName'

# Check CloudFront error rate
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name 4xxErrorRate \
  --dimensions Name=DistributionId,Value=XXXXX \
  --start-time $(date -u -Iseconds -d "10 minutes ago") \
  --end-time $(date -u -Iseconds) \
  --period 60 \
  --statistics Average
```

#### Action 3e.4: Escalate to On-Call Lead
```
If you reach here, the issue is beyond standard playbook.
Escalate to:
  - Lead SRE: @on-call-lead in Slack
  - CTO: @cto if critical path is broken
```

---

## STEP 4: ROLLBACK — When to Roll Back and How

### When to Roll Back (Automatic Rollback Criteria)

Roll back the application code if ALL of these are true for >5 minutes:
- 5xx error rate >20% (not improving)
- Last stable deployment was >2 hours ago
- Root cause is NOT clearly an external dependency (Razorpay down, AWS regional issue)

**Do NOT roll back the database schema.** Rolling back code only. Database must never roll back.

### How to Roll Back

```bash
# Get the previous stable task definition
aws ecs describe-task-definition \
  --task-definition swiftdb-app \
  --query 'taskDefinition.[taskDefinitionArn,containerDefinitions[0].image]'

# Identify the previous version (e.g., swiftdb-app:123)
# Get the previous stable version:
aws ecs list-task-definitions \
  --family-prefix swiftdb-app \
  --sort DESCENDING \
  --max-results 10 \
  --query 'taskDefinitionArns'

# Update the service to use a previous stable version
aws ecs update-service \
  --cluster swiggy-prod \
  --service api \
  --task-definition swiftdb-app:122  # <-- Previous version number

# Verify the rollback
aws ecs describe-services \
  --cluster swiggy-prod \
  --services api \
  --query 'services[0].deployments[*].[taskDefinition,desiredCount,runningCount]'

# Wait 2 minutes for new tasks to start and propagate
sleep 120

# Check if error rate improved
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/swiftdb-alb/xxx \
  --start-time $(date -u -Iseconds -d "5 minutes ago") \
  --end-time $(date -u -Iseconds) \
  --period 60 \
  --statistics Sum
```

### If Rollback Works
```
✅ Error rate dropped below 5%? → Rollback successful
   - Keep the previous version running
   - Investigate what broke in the new version
   - Patch the code
   - Re-deploy when ready (not during event)
   - Document in postmortem (see Step 5)
```

### If Rollback Does NOT Help
```
❌ Error rate still >20%? → Issue is not code
   - The problem is infrastructure (database, cache, network)
   - Go back to STEP 2 and re-triage
   - If you rolled back code, roll forward again (previous steps will fix it)
```

---

## STEP 5: POSTMORTEM — Document the Incident

**Timing:** Postmortem must begin within 24 hours of incident end.  
**Duration:** 30 minutes to 1 hour  
**Participants:** Whoever responded + Engineering Lead + Product Lead

**Use this template:**

### Postmortem Template: [Incident Name and Date]

---

**Incident Title:** [Brief description, e.g., "World Cup Promo Event - Database Connection Pool Exhaustion"]  
**Date:** [Date incident occurred]  
**Duration:** [Start time] to [End time] (total: X minutes)  
**Severity:** [CRITICAL / HIGH / MEDIUM]

---

#### TIMELINE (be specific with timestamps from logs)

Include: WHAT happened, WHEN it happened, WHERE it shows up (which component), and WHO detected it.

**Instructions for filling this in:**
- Go to AWS CloudWatch
- Export logs and metrics for the incident time window
- For each major event, include the exact timestamp (e.g., T+03:47 UTC)
- Reference the specific metric/log that shows the issue
- Format: `T+[time]: [what happened] (evidence: [metric name] = [value])`

Example:
```
T+20:05:30 UTC: Push notification sent to 180M users
T+20:05:45 UTC: 2M concurrent app opens detected (ALB request count spike)
T+20:06:00 UTC: Database connections reach 100/100 (RDS DatabaseConnections metric)
T+20:06:15 UTC: App errors spike to 50% (ALB HTTP 500 count)
T+20:06:45 UTC: On-call engineer paged (alerts firing)
T+20:07:10 UTC: Triage complete - DB pool identified as root cause
...
T+20:50:00 UTC: Error rate drops below 5% - incident resolved
```

---

#### ROOT CAUSE

**What was the single, deepest technical cause?** Not "database was slow" but "PgBouncer pool_size = 100, and 30% of requests are synchronous payment calls holding connections for 800ms each, causing pool exhaustion at 394 RPS instead of theoretical 5,000 RPS."

**Instructions for filling this in:**
- Open FAILURE-CASCADE.md and identify which failure occurred
- Explain the numbers that made it fail (e.g., "pool size X, payment hold time Y, RPS Z")
- Do NOT blame people ("engineer didn't..." - blame systems)
- Do NOT say "we didn't expect..." - we should have expected this

---

#### IMPACT

**What happened to users?**

- Total users affected: [How many users could not place orders?]
- Estimated orders lost: [Order attempts that failed]
- Revenue impact: [If known, or "TBD"]
- App Store rating impact: [Did users leave bad reviews?]
- Duration: [Total minutes system was unavailable]

**Instructions:**
- Quantify the impact in business terms, not just technical terms
- Use: "1M users unable to place orders for 45 minutes = 50K lost orders = ₹X crore"
- Include customer support ticket surge (if any)

---

#### WHAT WORKED (Positive Observations)

- [What did we do right?]
  - Example: "ALB health checks detected the unhealthy instances and rerouted traffic within 30 seconds"
  - Example: "Redis cache prevented database from complete loss — only 20% of requests had to wait"

**Instructions:**
- Even in a failed incident, something worked
- Don't skip this section — it's morale and learning
- Minimum: 2–3 things that worked

---

#### WHAT DIDN'T WORK (Negative Observations)

- [What made the incident longer / worse?]
  - Example: "Payment queue worker didn't have enough visibility — took 10 minutes to realize it was down"
  - Example: "Rollback decision was slow — operator wasn't sure if rollback or scaling was the right choice"

**Instructions:**
- Be honest about failures
- Don't blame people (blame systems/processes)
- Focus on what slowed down recovery

---

#### ACTION ITEMS

**Each action item must have:**
1. **What:** Specific task (not vague)
2. **Why:** Why this prevents the incident from recurring
3. **Who:** Which person/team owns it
4. **When:** Due date (e.g., "2 weeks", "before next event")
5. **Status:** (Initially "not started")

**Example action items:**

| # | Action | Why | Owner | Due Date | Status |
|---|--------|-----|-------|----------|--------|
| 1 | Increase PgBouncer pool_size from 100 to 300 | Pool exhaustion at 394 RPS is too low for 500K RPS events | DBA (Alice) | 1 week | Not started |
| 2 | Add distributed tracing (Jaeger) to all services | On-call engineer took 30 min to find root cause. With traces, should be <5 min | Platform (Bob) | 3 weeks | Not started |
| 3 | Create "Scale Event Checklist" and test it | Engineer didn't know when to scale up vs roll back. Checklist removes ambiguity | SRE Lead (Carol) | 2 weeks | Not started |
| 4 | Monitor circuit breaker metrics for payment calls | Circuit breaker should have triggered automatically when payment latency spiked. Currently not monitored | Backend Lead (Dan) | 1 week | Not started |

---

#### LESSONS LEARNED (Optional Reflection)

- What surprised us?
- What should we do differently next time?
- What assumptions were wrong?

---

**Postmortem Deadline:** 24 hours after incident  
**Escalation:** If any action item is not owned within 24 hours, escalate to Engineering Manager

---

## Quick Reference: When to Call Who

| Situation | Who to Call | Slack Channel | Escalation |
|-----------|------------|---------------|-----------|
| Database connection pool | DBA on-call (@dba-oncall) | #database | @db-manager |
| Node.js / App crashes | Backend Lead (@backend-lead) | #backend | @cto |
| Cache / Redis issues | Platform Engineer (@platform-oncall) | #infrastructure | @platform-manager |
| Payment queue / worker | Payment Team Lead (@payment-lead) | #payments | @payment-manager |
| Not sure | SRE Lead (@sre-lead) | #incidents | @engineering-lead |

---

## Runbook Test Checklist (Before Calling This Done)

- [ ] Can a first-year engineer follow this without asking questions?
- [ ] Every command has clear expected output ("Expected: X appears in logs")
- [ ] Every decision point has clear indicators ("If DatabaseConnections = 100")
- [ ] Recovery steps include how to verify success (before moving to next step)
- [ ] Postmortem template is fillable by someone not in the incident
- [ ] All AWS CLI commands are copy-paste ready (not pseudocode)
- [ ] Time estimates are realistic (30 sec detection, 5 min triage, 10 min response)
