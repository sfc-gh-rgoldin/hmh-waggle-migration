# Workload Sizing — Usage Patterns & Volume Projection

## Overview

This reference provides a structured methodology for collecting current-state workload patterns from the Redshift/Talend environment and projecting equivalent Snowflake resource sizing for the target architecture. The output is a right-sized architecture with defensible compute and storage projections.

## Why Collect Workload Metrics

Raw object counts (tables, rows, storage) are insufficient for proper sizing. What drives architecture decisions is **workload intensity** — how often queries run, how long they take, how much data they scan, and how much concurrency the system handles. These metrics must be collected from the current environment to produce a properly sized target architecture.

---

## Step 1: Collect Current-State Usage Metrics

### 1.1 Query Workload Profile (from Redshift)

If Redshift access is available, run these queries to profile the workload:

```sql
-- Daily query volume and compute hours (last 30 days)
SELECT DATE(starttime) AS query_date,
       COUNT(*) AS total_queries,
       SUM(elapsed)/1000000.0/3600 AS total_compute_hours,
       AVG(elapsed)/1000000.0 AS avg_seconds,
       PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY elapsed)/1000000.0 AS p95_seconds,
       MAX(elapsed)/1000000.0 AS max_seconds
FROM stl_query
WHERE starttime > GETDATE() - 30
  AND userid > 1  -- exclude system queries
GROUP BY 1
ORDER BY 1;

-- Query volume by user/service (identifies ETL vs. reporting vs. ad-hoc)
SELECT u.usename,
       COUNT(*) AS query_count,
       SUM(q.elapsed)/1000000.0/3600 AS total_hours,
       AVG(q.elapsed)/1000000.0 AS avg_seconds
FROM stl_query q
JOIN pg_user u ON q.userid = u.usesysid
WHERE q.starttime > GETDATE() - 30
  AND q.userid > 1
GROUP BY 1
ORDER BY total_hours DESC;

-- Concurrency profile (peak simultaneous queries per hour)
SELECT DATE(starttime) AS day,
       EXTRACT(HOUR FROM starttime) AS hour_of_day,
       MAX(concurrent) AS peak_concurrent
FROM (
  SELECT starttime,
         COUNT(*) OVER (
           ORDER BY starttime
           RANGE BETWEEN INTERVAL '1 second' PRECEDING AND CURRENT ROW
         ) AS concurrent
  FROM stl_query
  WHERE starttime > GETDATE() - 7 AND userid > 1
)
GROUP BY 1, 2
ORDER BY 1, 2;

-- Data scanned per query (proxy for warehouse sizing)
SELECT DATE(starttime) AS query_date,
       COUNT(*) AS queries,
       SUM(bytes_scanned)/1024/1024/1024.0 AS total_gb_scanned,
       AVG(bytes_scanned)/1024/1024/1024.0 AS avg_gb_scanned
FROM svl_query_summary s
JOIN stl_query q ON s.query = q.query
WHERE q.starttime > GETDATE() - 30
GROUP BY 1
ORDER BY 1;

-- WLM queue wait times (indicator of concurrency pressure)
SELECT service_class,
       COUNT(*) AS queries,
       AVG(queue_elapsed)/1000000.0 AS avg_queue_wait_sec,
       MAX(queue_elapsed)/1000000.0 AS max_queue_wait_sec,
       SUM(exec_elapsed)/1000000.0/3600 AS total_exec_hours
FROM stl_wlm_query
WHERE service_class_start_time > GETDATE() - 30
GROUP BY 1;

-- Nightly batch window duration
SELECT DATE(starttime) AS batch_date,
       MIN(starttime) AS batch_start,
       MAX(endtime) AS batch_end,
       DATEDIFF('minute', MIN(starttime), MAX(endtime)) AS batch_duration_min
FROM stl_query
WHERE starttime > GETDATE() - 30
  AND userid IN (SELECT usesysid FROM pg_user WHERE usename LIKE '%talend%' OR usename LIKE '%etl%')
GROUP BY 1
ORDER BY 1;

-- Storage and data change rate (CDC volume estimation)
SELECT "table" AS tablename,
       tbl_rows AS current_rows,
       size AS size_mb,
       pct_used
FROM svv_table_info
ORDER BY size DESC
LIMIT 50;
```

### 1.2 ETL/Ingestion Metrics (from Talend or AWS)

Collect these manually or from monitoring:

| Metric | How to Get | Why It Matters |
|--------|-----------|----------------|
| **Daily CDC row volume** | Talend logs or binlog position delta | Drives Fivetran/Openflow credit consumption |
| **Nightly batch duration** | Talend run history | Maps to dbt transform warehouse active time |
| **Number of batch runs per day** | Talend scheduler | Determines warehouse wake-up frequency |
| **Peak extraction parallelism** | Talend server config | Affects ingestion warehouse sizing |
| **Data volume per sync (GB)** | S3 staging bucket metrics | Ingestion volume sizing |
| **Full refresh frequency** | Schedule (which tables do full reload vs. incremental) | Full refreshes are expensive on Snowflake |

### 1.3 Reporting & BI Metrics

| Metric | How to Get | Why It Matters |
|--------|-----------|----------------|
| **Report execution count/day** | JasperSoft logs or Redshift query log filtered by report user | Reporting warehouse sizing |
| **Peak concurrent report users** | JasperSoft session metrics or Redshift concurrency data | Multi-cluster warehouse config |
| **Report query avg runtime** | Redshift query log (filter by JasperSoft user) | Warehouse size needed |
| **Report scheduling (overnight vs. on-demand)** | JasperSoft config | Batch vs. interactive warehouse split |
| **Ad-hoc query volume** | Redshift query log (filter by analyst users) | Separate ad-hoc warehouse? |

### 1.4 API/App Layer Metrics

| Metric | How to Get | Why It Matters |
|--------|-----------|----------------|
| **API queries per second (peak)** | App layer metrics / Redshift query log | App warehouse sizing + multi-cluster |
| **API query avg latency** | App layer metrics | Determines if XS warehouse is sufficient |
| **API data freshness requirement** | Product team | Dynamic Table lag configuration |
| **Read replica query patterns** | Redshift read cluster query log | Understand app query shapes |

### 1.5 Storage Metrics

| Metric | How to Get | Why It Matters |
|--------|-----------|----------------|
| **Total compressed storage (TB)** | `svv_table_info` aggregate | Snowflake storage cost ($/TB/month) |
| **Daily net growth (GB/day)** | Storage trend over 30 days | Projected storage growth |
| **Time-Travel/history retention needed** | Business requirement (audit, recovery) | 1 vs. 90 day retention cost |
| **Fail-safe overhead** | Enterprise feature | +7 days of storage |

---

## Step 2: Map Metrics to Snowflake Credit Model

### 2.1 Snowflake Credit Rates (reference)

| Warehouse Size | Credits/Hour | Typical Use Case |
|---------------|-------------|-----------------|
| XS | 1 | Light API queries, metadata |
| S | 2 | Standard reporting, simple transforms |
| M | 4 | Complex transforms, dbt runs |
| L | 8 | Heavy analytics, ML training |
| XL | 16 | Large-scale batch processing |

| Service | Credit Model |
|---------|-------------|
| **Snowpipe** | ~0.06 credits per 1,000 files |
| **Snowpipe Streaming** | Per-second compute while active |
| **Dynamic Tables** | Credits for refresh compute |
| **Cortex AI Functions** | Per-token (model-dependent) |
| **Cortex Search** | Per-query + index build |
| **Cortex ML (Classification/Forecast)** | Training + inference credits |
| **Tasks** | Credits only when running |
| **Storage** | ~$23-40/TB/month (depending on region/edition) |

### 2.2 Workload-to-Warehouse Mapping

Use the collected metrics to estimate warehouse sizing:

```
INGESTION WAREHOUSE (WAGGLE_INGEST_WH):
  Size = XS (Fivetran/Openflow manages its own; if Snowpipe, XS is typical)
  Active hours/day = based on CDC frequency × sync duration
  Daily credits = size_credits × active_hours

TRANSFORM WAREHOUSE (WAGGLE_TRANSFORM_WH):
  Size = map from (current batch duration ÷ Snowflake speedup factor*)
  Active hours/day = batch runs × duration per run
  Daily credits = size_credits × active_hours
  *Snowflake typically 2-5x faster than Redshift for same workload

REPORTING WAREHOUSE (WAGGLE_REPORTING_WH):
  Size = map from avg report query runtime + data scanned
  Active hours/day = hours when reports are running
  Concurrency = peak concurrent ÷ queries_per_cluster → multi-cluster count
  Daily credits = size_credits × active_hours × avg_cluster_count

AI WAREHOUSE (WAGGLE_AI_WH):
  Size = M (Cortex functions need at least M for ML training)
  Active hours/day = estimated AI workload (start small, grow)
  Daily credits = size_credits × active_hours + cortex_token_costs

APP/API WAREHOUSE (WAGGLE_API_WH):
  Size = XS-S based on query complexity
  Active hours/day = app active hours (likely 12-16h for edu platform)
  Concurrency = peak QPS → multi-cluster config
  Daily credits = size_credits × active_hours × avg_cluster_count
```

### 2.3 Serverless & AI Consumption Estimation

| Component | Estimation Method |
|-----------|------------------|
| **Fivetran (if used)** | Per-MAR (Monthly Active Row) pricing from Fivetran — not Snowflake credits. Estimate from CDC row volume. |
| **Openflow (if used)** | Credits consumed by ingestion runtime. Estimate: ~1 credit/hour per connector active. |
| **Dynamic Tables** | Credits = refresh frequency × data processed per refresh. Use current materialized view refresh patterns as proxy. |
| **Cortex Analyst** | ~0.01-0.05 credits per query (depends on model). Estimate from projected NL query volume. |
| **Cortex Agents** | ~0.03-0.10 credits per invocation (model + tool calls). Estimate from projected automation count. |
| **Cortex ML (Classification)** | Training: varies by dataset size. Inference: ~0.001 credits per prediction. |
| **Cortex Complete** | Per-token: varies by model. Estimate from projected GenAI call volume × avg tokens. |
| **Cortex Search** | Index build + per-query cost. Estimate from document corpus size + query volume. |
| **Tasks/Scheduling** | Serverless tasks: ~1.5x credit multiplier vs. warehouse. Estimate from job count × avg runtime. |

---

## Step 3: Build the Cost Projection

### 3.1 Monthly Projection Template

Present this table to the user, filling values from collected metrics:

| Workload | Warehouse Size | Active Hrs/Day | Credits/Day | Monthly Credits | Notes |
|----------|---------------|----------------|-------------|-----------------|-------|
| Ingestion (Fivetran-managed) | — | — | — | — | Fivetran cost is separate ($) |
| Ingestion (Openflow/Snowpipe) | XS (1 cr/hr) | ___ | ___ | ___ | Based on CDC frequency |
| Transform (dbt) | S-M (2-4 cr/hr) | ___ | ___ | ___ | Based on batch duration |
| Reporting (JasperSoft) | S (2 cr/hr) | ___ | ___ | ___ | Multi-cluster: ×___ |
| Ad-hoc Analytics | S (2 cr/hr) | ___ | ___ | ___ | Analyst workload |
| App/API Layer | XS-S (1-2 cr/hr) | ___ | ___ | ___ | Multi-cluster: ×___ |
| AI Platform (Cortex) | M (4 cr/hr) | ___ | ___ | ___ | Phase-in over time |
| Dynamic Tables | — | — | ___ | ___ | Refresh compute |
| Tasks/Orchestration | — | — | ___ | ___ | Serverless overhead |
| **Compute Subtotal** | | | | **___** | |
| Storage (___TB × $__/TB) | | | | | **$___/mo** |
| **TOTAL** | | | | **___ credits** | **+ $__ storage** |

### 3.2 Convert to Dollar Cost

```
Monthly cost = (total_credits × $/credit) + storage_cost + ingestion_tool_cost + dbt_cloud_cost
```

Pricing varies by edition and contract:
- Standard: ~$2.00/credit
- Enterprise: ~$3.00/credit  
- Business Critical: ~$4.00/credit
- On-Demand vs. Capacity pricing (pre-purchase discounts)

### 3.3 Current vs. Projected Comparison

| Component | Current Annual Cost | Projected Snowflake Annual | Delta |
|-----------|--------------------|-----------------------------|-------|
| Compute (Redshift RI + on-demand) | $___ | $___ (credits) | |
| ETL Licensing (Talend) | $___ | $0 (replaced by dbt/native) | |
| Storage (Redshift + S3) | $___ | $___ (Snowflake storage) | |
| Ingestion Tool | $0 (Talend included) | $___ (Fivetran or $0 if Openflow) | |
| Orchestration | $___ (existing) | $___ (Tasks/Airflow) | |
| AI Platform | $0 (doesn't exist) | $___ (Cortex credits) | |
| Operational overhead (people hours) | $___ | $___ (reduced?) | |
| **TOTAL** | **$___/year** | **$___/year** | **___% change** |

---

## Step 4: Sensitivity Analysis & Growth Modeling

### 4.1 Key Assumptions to Flag

- **Snowflake speedup factor**: Assume 2-5x vs. Redshift (conservative: 2x, optimistic: 5x)
- **Auto-suspend savings**: Snowflake only charges when active. If current Redshift is always-on but workload is bursty, savings are significant.
- **Concurrency scaling**: If current WLM queues show wait times, multi-cluster warehouses may cost more but deliver better UX.
- **AI platform ramp**: Cortex costs will start near $0 and grow as AI features are adopted. Model as Phase 1 (minimal), Phase 2 (moderate), Phase 3 (full AI).

### 4.2 Growth Scenarios

Present three scenarios:

| Scenario | Assumption | Monthly Credits | Annual Cost |
|----------|-----------|----------------|-------------|
| **Conservative** | Warehouse size = current compute equivalent, minimal AI, no auto-suspend savings | ___ | $___ |
| **Expected** | 3x speedup, moderate auto-suspend savings, Phase 1 AI | ___ | $___ |
| **Optimistic** | 5x speedup, full auto-suspend, aggressive right-sizing | ___ | $___ |

### 4.3 12-Month Ramp Model

Credits won't hit steady-state on day one. Model the ramp:

| Month | Compute Credits | AI Credits | Storage ($) | Fivetran ($) | Total |
|-------|----------------|-----------|-------------|-------------|-------|
| 1-2 (Foundation) | ~100/mo | 0 | Minimal | Setup | Low |
| 3-4 (Core migration) | ~500/mo | 0 | Growing | Active | Medium |
| 5-6 (Full migration) | ~800-1200/mo | ~50 | Steady | Active | Target |
| 7-12 (Steady state + AI) | ~800-1200/mo | ~100-300 | Steady | Active | Full |

---

## Step 5: Questions to Ask for Workload Sizing

These questions supplement the general discovery (Section 9 of the question bank) and are **specifically required** for proper architecture sizing. Ask them during Phase 1 or as a follow-up:

### 5.1 Activity Patterns

1. How many hours per day is your Redshift cluster actively processing queries (vs. idle)?
2. What is your peak query concurrency (max simultaneous queries)? When does this peak occur?
3. How long does your nightly batch window run (start to finish)?
4. How many dbt/transformation runs would you expect per day? (currently Talend runs = ___)
5. Are there periods of zero activity (nights, weekends, school breaks)?

### 5.2 Data Freshness Needs

6. Which tables need near-real-time freshness (<5 min)? Which are fine with daily refresh?
7. For reporting marts — what target lag is acceptable? (1 min, 5 min, 1 hour, daily?)
8. Would you prefer lower resource consumption with less frequent refreshes, or pay more for lower latency?

### 5.3 AI Platform Volume

9. In Year 1, how many AI queries/day do you expect? (NL analytics for teachers, agent automations)
10. How many documents/pages would be indexed for search/RAG? (curriculum, content library)
11. How many students would ML models score? How frequently? (daily, weekly)
12. Is there a hard budget ceiling for AI experimentation, or is it "prove value first, optimize later"?

### 5.4 Budget Context

13. What is your target monthly budget for the data platform? (Or: what's the current total platform cost you'd like to stay under?)
14. Would you prefer capacity pricing (commit upfront for lower rate) or on-demand (pay as you go, higher rate)?
15. Is the ROI story primarily about cost savings or value creation (AI enables new revenue)?

---

## Output Template

The cost model section in `03-target-architecture.md` should include:

```markdown
## Cost Model & Resource Projection

### Current State Annual Cost
| Component | Annual Cost |
|-----------|-------------|
| Redshift (compute) | $ |
| Redshift (storage) | $ |
| Talend licensing | $ |
| S3 (staging/archive) | $ |
| Operational labor | $ |
| **Total** | **$** |

### Projected Annual Cost (Steady State)
| Workload | Warehouse Size | Consumption/Month | Annual Consumption | Annual Cost |
|----------|---------------|-------------------|--------------------| ------------|
| Ingestion | XS | | | |
| Transformation | S-M | | | |
| Reporting | S (multi-cluster) | | | |
| App/API | XS-S | | | |
| AI Platform | M | | | |
| Serverless (Tasks, DTs) | — | | | |
| **Subtotal Compute** | | | | **$** |
| Storage (___TB) | | | | **$** |
| Ingestion tool (if applicable) | | | | **$** |
| dbt Cloud (if applicable) | | | | **$** |
| **TOTAL** | | | | **$** |

### Scenario Comparison
| Scenario | Annual Projected | vs. Current | Savings/(Cost) |
|----------|-----------------|-------------|----------------|
| Conservative | $ | $ | |
| Expected | $ | $ | |
| Optimistic | $ | $ | |

### Key Assumptions
- [ ] Performance factor: ___x vs. current platform
- [ ] Auto-suspend savings: ___% of compute hours recaptured
- [ ] AI ramp: estimated consumption by Month 6
- [ ] Edition: Enterprise
- [ ] Storage region: us-west-2

### 12-Month Ramp
[Table or chart showing monthly resource consumption during migration and ramp to steady state]
```
