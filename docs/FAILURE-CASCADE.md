# Failure Cascade Analysis: SwiftEats Monolith Under World Cup Load

## Executive Summary
The monolith can handle approximately 12,000 RPS. A World Cup Final promotional spike will generate 500,000 RPS. The system will fail catastrophically in cascading stages, losing approximately ₹189 crore in orders during a 45-minute outage.

---

## Section 1: Traffic Simulation Math

### The Arrival Calculation

```
Total push notification recipients:        180,000,000 users
Click-through rate on major promos:        ~8%
Active users clicking and opening app:     14,400,000 users

Conservative estimate (actual traffic):    10,000,000 active users in 60s spike
```

### API Calls Per User (First 60 Seconds)
When a user opens the app during a flash sale:
1. **GET /restaurants** - Fetch nearby restaurant list (10ms)
2. **GET /restaurant/:id** - View menu and details (15ms)
3. **POST /orders** - Place order with payment (500ms for synchronous payment call)

Total per user: ~3 API calls

### Peak RPS Calculation
```
Peak RPS = (Active Users × API Calls Per User) / Spike Window (seconds)
Peak RPS = (10,000,000 × 3) / 60
Peak RPS = 30,000,000 / 60
Peak RPS = 500,000 RPS
```

**Monolith capacity: ~12,000 RPS**
**Demand: 500,000 RPS**
**Ratio: 41.7× over capacity**

### Real-World Context
| System | Capacity (RPS) |
|--------|---|
| **SwiftEats Monolith** | ~12,000 |
| **This Event (Demand)** | ~500,000 |
| Twitter Peak (2013) | ~300,000 |
| Netflix Global Peak | ~800,000 |
| Google Search | ~8,500,000 |

---

## Section 2: Component Capacity Numbers

### Hard Limits of the Monolith

#### PostgreSQL Database
```
max_connections = 100 (configured limit)
Average query execution time = 20ms (menu loads)
Payment queries hold connection = 200–2,000ms (payment gateway wait)
Connection hold formula:
  Connections held = (Non-payment RPS × query_time) + (Payment RPS × payment_hold_time)
```

**Example calculation at 300 RPS with 30% payment mix:**
```
Non-payment connections: (300 × 0.7) × 0.020s = 4.2 connections
Payment connections: (300 × 0.3) × 1.0s = 90 connections
Total held: 94.2 / 100 max ≈ EXHAUSTED
```

At 500 RPS with 30% payment mix:
```
Non-payment: (500 × 0.7) × 0.020s = 7 connections
Payment: (500 × 0.3) × 1.0s = 150 connections
Total: 157 connections >> 100 max
POOL EXHAUSTED INSTANTLY
```

#### Node.js Event Loop
```
Single process, single CPU core
Typical capacity: 12,000–15,000 RPS before callback queue saturation
At 15,000 pending requests in queue: RSS memory hits 4GB heap limit → OOM crash
Average request processing: 50–100ms
Queue formation time: 500ms per RPS above 12K
```

#### Static Asset Serving (No CDN)
```
Restaurant images per request: ~20 images per menu view
Average image size: 200KB per image
Peak image bandwidth: 10M users × 20 images × 200KB = 40TB/minute
Server NIC capacity: ~1–2Gbps = 125–250GB/minute maximum
TIME TO NIC SATURATION: <2 minutes from spike start
```

#### Payment Gateway Hold Time
```
Razorpay/PayU synchronous call duration: 200–2,000ms (avg 800ms)
Network latency: 50–100ms
Payment provider processing: 100–500ms
Each payment call holds ONE database connection for the entire duration
```

---

## Section 3: The Cascade - Failure Sequence with RPS Triggers

### ⚠️ Failure 1: PostgreSQL Connection Pool Exhaustion
**Severity:** CRITICAL  
**Triggered at:** ~300–400 RPS (within 3 seconds of spike start)  
**Why:** 30% of requests make synchronous payment calls holding connections for 800ms–2000ms. At 300 RPS with 30% payment mix, pool exhaustion formula shows 94 connections held. At 400 RPS, the pool is completely consumed.

**What users see:**
- "Connection pool timeout" errors on the API
- Orders fail to place: `Error: connect timeout`
- Menu loads timeout after 10 seconds
- 99% of new requests get connection refused

**Cascade impact:**
- Database rejects all new connections
- Node.js event loop begins backing up as requests wait for a free connection
- Response times explode from 50ms → 5,000ms
- Memory consumption increases as requests queue

---

### ⚠️ Failure 2: Node.js Event Loop Saturation
**Severity:** CRITICAL  
**Triggered at:** ~12,000 RPS (within 5 seconds)  
**Why:** All requests waiting for a DB connection fill the event loop callback queue. Node.js can process callbacks at ~12K RPS max. Beyond this, queue grows unbounded.

**What users see:**
- Mobile app shows perpetual loading spinner
- Requests timeout after 30–60 seconds
- Some older requests (from T+0s) finally complete after 45+ seconds with stale data
- New app opens get instant failure - database fully unavailable

**Cascade impact:**
- Pending requests accumulate: 1,000 → 10,000 → 100,000 queued in memory
- Node.js RSS memory: 100MB → 500MB → 2GB → 4GB (heap limit)
- Server begins garbage collection pauses of 5–10 seconds
- Eventually: OOM crash - process terminates

---

### ⚠️ Failure 3: Synchronous Payment Call Amplification  
**Severity:** CRITICAL  
**Duration:** 200–2,000ms per payment call  
**Triggered at:** Same time as Failure 1, *causes* it to trigger at lower RPS  

**The amplifier effect:**
```
Without payment calls:
  - DB connection hold time = 20ms (query only)
  - Max sustainable RPS = 100 connections ÷ 0.020s = 5,000 RPS

With payment calls:
  - DB connection hold time = 800ms (payment gateway wait)
  - Max sustainable RPS = 100 connections ÷ 0.800s = 125 RPS
  
Payment calls reduce maximum RPS by 40×
```

**What happens:**
- Every payment request holds a precious connection for 800ms
- At 500 RPS total, 150 payment calls (30%) are in flight, holding 120 connections for 800ms
- The remaining 100 non-payment requests starve - cannot get a connection
- Pool exhausts, non-payment requests queue, system collapses

---

### ⚠️ Failure 4: Promo Code Race Condition
**Severity:** HIGH  
**Triggered at:** ~T+10 seconds (concurrent with earlier failures)  

**The code flow (vulnerable):**
```sql
-- Request 1                           -- Request 2
SELECT budget FROM promos              SELECT budget FROM promos
WHERE id = 'world-cup-50-off'         WHERE id = 'world-cup-50-off'
-- Returns 1000 uses remaining        -- Returns 1000 uses remaining

-- 10 milliseconds later...

UPDATE promos SET budget = 999         UPDATE promos SET budget = 999
WHERE id = 'world-cup-50-off'         WHERE id = 'world-cup-50-off'
```

**The race condition:**
- Both requests read budget = 1000
- Both decrement to 999 and write back
- Net result: budget should be 998, but it's 999
- With 10M users, this race occurs millions of times
- Budget depletes in seconds (should take minutes)
- Millions of users receive "discount applied" but the promo is bankrupt

**Financial impact:**
- 50% discount promised on orders that shouldn't have been discounted
- System says promo is valid, order is confirmed
- Finance system later discovers the overspend
- Unplanned loss of ~₹20–40 crore in discounts

---

### ⚠️ Failure 5: Static Asset NIC Saturation  
**Severity:** HIGH  
**Triggered at:** ~T+8-12 seconds  

**The calculation:**
```
10M users opening app simultaneously
Average menu view: 20 images per restaurant
Average image size: 200KB (PNG compressed)

Total bandwidth demand: 10,000,000 × 20 × 200KB = 40,000,000,000 KB = 40 TB/minute
Server NIC capacity: ~1–2 Gbps
  - 1 Gbps = ~125 GB/minute
  - 2 Gbps = ~250 GB/minute
  
TIME TO SATURATION: 40,000 GB ÷ 250 GB/min = 160 minutes (if no other traffic)
But - Node.js only has ONE process, ONE NIC queue. 
Images arrive before API responses → API completely blocked.
TIME TO API BLOCKING: ~30–60 seconds of peak image load
```

**What users see:**
- Restaurant images load but API calls timeout entirely
- Users see beautiful menu photos but cannot proceed to checkout
- Thousands of pending image requests jam the network stack
- API becomes completely unreachable

---

### ⚠️ Failure 6: Monitoring Blindness (The Invisible Killer)
**Severity:** CRITICAL (for MTTD)  
**Triggered at:** T+0s (ongoing throughout incident)  

**What the on-call engineer sees:**
```
CloudWatch dashboard shows:
  - Error rate: 5% → 50% → 90% (red)
  - Response time P99: 50ms → 500ms → 5000ms (red)
  - CPU: 95% (red)
  - ...but WHICH component is broken?
```

**Without distributed tracing or structured logs, the engineer cannot answer:**
- Is the database pool exhausted? (No DB metrics dashboard for app-level pool)
- Is the payment gateway timing out?
- Did the promo code break?
- Is it a memory leak?
- Is the NIC saturated?

**Result:**
- First 20–45 minutes spent investigating by blind trial-and-error
- Restart the app server (doesn't help - it's the database)
- Flush the database (makes it worse - connections drop, reconnection storm)
- Check recent deployments (nothing changed)
- Finally: scale database CPU (doesn't help - pool is the bottleneck)
- Finally: restart database (database loses all connections, forces reconnection storm)
- System takes 30 more minutes to stabilize

**Total MTTD (Mean Time To Detect + Mean Time To Resolve): 45–90 minutes**

---

## Section 4: Timeline of Catastrophic Failure

```
T+0s:     Push notification sent to 180M users
T+0s:     Swiggy app receives notification, starts showing "50% OFF - TAP NOW"
T+1s:     First 1M users tap notification, app startup begins
T+2s:     5M concurrent API requests in flight
T+3s:     PostgreSQL connection pool reaches 98/100 connections
          Payment calls are holding ~90 connections (30% of 300 RPS × 800ms)
          First "connection pool timeout" errors appear
T+4s:     Pool reaches 100/100 - NEW requests get immediate connection refused
          "Error: connect timeout" floods error logs
T+5s:     Node.js event loop callback queue backs up
          Pending requests: 5,000 (waiting for a free DB connection)
          Median response time: 50ms → 500ms
T+6s:     Queue grows to 15,000 pending requests
          Heap memory: 100MB → 1.2GB
          P99 response time: 5000ms (users waiting 5 seconds for a response)
T+7s:     20,000 pending requests in memory
          GC pause: 3-second freeze every 2 seconds
          All new requests time out immediately
T+8s:     Image requests begin saturating the NIC
          Restaurant menu images are competing with API requests for bandwidth
          API response time: 8000–10000ms (if connection even acquired)
T+9s:     Heap reaches 3.5GB
          GC pauses extend to 5 seconds
          Pending queue: 50,000 requests
T+10s:    Promo race condition hits critical mass
          Budget check reads valid, but budget is actually exhausted
          1M+ users placed orders with "discount applied" but promo was already bankrupt
          System shows "50% off applied" for orders that shouldn't have had it
T+12s:    Heap reaches 4GB (Node.js heap limit)
          OOM killer triggers - Node.js process terminates
          Load balancer receives connection refused
T+13s:    Load balancer detects healthy node down
          No backup servers to route to
          ALL traffic: "Connection Refused"
T+15s:    On-call Slack channel starts flooding with page alerts
          Alerts: "API errors 99%", "P99 latency 30s", "Database errors"
T+20s:    On-call engineer wakes up, reads Slack
          Dashboard shows all metrics red
          No clear root cause - database, app, payment gateway all look bad
T+25s:    Engineer restarts the application
          New app process starts, immediately tries to create 100 DB connections
          Connection storm creates a queue at the database
          Problem persists
T+30s:    Engineer checks if a recent deployment caused it
          No deployments in the last 2 hours
          Escalates to database team
T+45m:    After systematic investigation of all components,
          Engineer identifies the root cause: "Database connection pool exhausted
          due to synchronous payment calls holding connections for 800ms"
T+50m:    Decision made: Implement circuit breaker for payment calls
          Async the payment flow (send to queue, don't wait)
T+60m:    Code change deployed
          Payment calls no longer hold connections
          Database responds normally, queued requests start clearing
T+75m:    System is 95% recovered
          Most pending requests have completed (now in past 75 seconds of queue)
          Promo overspend detected: ₹44 crore in extra discounts issued
T+90m:    System fully recovered
          Analyzing the incident
T+120m:   Post-mortem begins
          Documenting: What failed, why, what to change
          
IMPACT SUMMARY:
  - Duration: 45 minutes from failure to recovery
  - Orders processed: 0 (system unreachable)
  - Orders lost: 10M (users who clicked the promo, tried to order, got errors)
  - Revenue lost: ₹189 crore (4.2 crore/minute × 45 minutes)
  - Unplanned discounts: ₹44 crore (promo race condition)
  - App rating: 4.3 → 1.8 (users leave 1-star reviews for outage)
  - Social media: #SwiftEatsDown trends for 8 hours
```

---

## Section 5: Why This Is Not a Mystery - It Is Physics

Every component has a throughput limit. When you exceed it, it fails in a specific, deterministic way:

1. **Database pool limit (100 connections)** is the first chokepoint
2. **Payment call duration (800ms)** amplifies the pressure on the pool
3. **Node event loop limit (12K RPS)** is exceeded as requests back up
4. **Memory limit (4GB heap)** is hit as queued requests accumulate
5. **Network bandwidth limit (1–2Gbps)** is exceeded by static assets

None of these fail by chance. All fail because load exceeds capacity. The engineer who understands these numbers is the engineer who can predict the failure before it happens and design a system that doesn't fail.

---

## Key Insights for Architecture Redesign

The cascade reveals three critical design failures:

1. **Single point of failure**: One Node.js process, one database. When one fails, everything fails.
2. **Synchronous payment calls hold database connections**: 800ms × 500 RPS = 400 concurrent payment holds. Only 100 connections exist. This is the primary amplifier.
3. **No caching layer**: Every request hits the database. 80% of requests are restaurant menu reads that could be cached for 5 minutes.
4. **No CDN**: 40TB/minute of image bandwidth saturates the origin server.

The redesigned architecture (in the next document) eliminates each of these through specific, justified components.
