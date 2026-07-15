---
name: hmh-waggle-migration
description: "HMH Waggle migration discovery and planning skill. Guides HMH's team through migrating their Waggle education platform from Redshift/Talend to Snowflake/Fivetran/dbt, plus AI platform architecture (Cortex, semantic views, agents, ML). Multi-phase: current state discovery, AIM assessment, architecture recommendations, migration plan. Triggers: waggle migration, HMH migration, waggle discovery, HMH redshift, waggle snowflake."
---

# HMH Waggle Migration & AI Platform Discovery

Guides HMH through a structured, multi-phase discovery and migration planning process for the Waggle education platform — from Redshift/Talend to Snowflake/Fivetran/dbt — plus designing the AI platform layer (GenAI, semantic layer, agents, predictive, in-app AI).

## Prerequisites

- **Snowflake connection**: Active Snowflake account with appropriate permissions
- **Redshift access** (optional for Phase 1, enables combined discovery+assessment): JDBC/ODBC connectivity to Waggle's Redshift read and write clusters
- **Migration plugin**: Will be installed automatically if not present when Redshift is connected

## Phase Selection

Ask the user which phase they want to run:

> Welcome to the HMH Waggle Migration Discovery skill. This guides you through evaluating and planning the migration from Redshift/Talend to Snowflake.
>
> **Available phases:**
> 1. **Discovery & Assessment** — Inventory your Redshift objects, run AIM assessment, analyze Talend jobs, collect business context. If you provide Redshift access, this combines automated inventory AND conversion assessment in one pass. Produces `01-current-state-discovery.md` and `02-aim-assessment.md`
> 2. **Architecture Recommendations** — Lift/shift vs. refactor analysis, target Snowflake architecture, AI platform design. Produces `03-target-architecture.md`
> 3. **Migration Plan** — Phased execution plan with wave prioritization, validation strategy, and cutover. Produces `04-migration-plan.md`
>
> Which phase would you like to start with? (If this is your first time, start with Phase 1.)

Read `references/hmh-context.md` for baked-in context about HMH/Waggle architecture.

---

## Phase 1: Current State Discovery

### Step 1.1 — Confirm Context

Read `references/hmh-context.md` and present the known architecture to the user:

> Based on our existing information about Waggle, here's what we know:
>
> - **Source**: Aurora MySQL (OLTP) → Talend ETL → S3 → Redshift (write cluster: 215 tables/~3B rows, read cluster: 65 tables/~79M rows)
> - **Downstream**: JasperSoft reports, CSV exports, Datalake, Waggle UI (via API)
> - **Proposed target**: Fivetran (CDC) → Snowflake (zone architecture) → dbt (transform) → Airflow (orchestrate)
>
> Is this still accurate? Has anything changed since our last conversation?

⚠️ **STOP**: Wait for confirmation or corrections before proceeding.

### Step 1.2 — Automated Collection (before asking questions)

Before asking manual questions, offer automated data collection methods that can pre-fill many answers:

> I can automatically collect much of the technical inventory. Which of these can you provide access to?
>
> 1. **Redshift connection** (JDBC/ODBC) — I'll query system catalogs to auto-collect: table/view/procedure counts, row counts, column types, sort/dist keys, storage sizes, schemas, dependencies, recent query patterns, WLM config, user/role inventory
>
> 2. **AWS CLI access** (credentials or assume-role) — I'll query: Redshift cluster configs (node types, count, storage), CloudWatch metrics (CPU, connections, query throughput), Cost Explorer (monthly Redshift/S3 spend), S3 bucket sizes
>
> 3. **Talend project export** — Export your Talend project as a ZIP or point me to the Git repo. I'll parse: job inventory, component types (tMap, tJava, etc.), job dependencies/DAG, context variables, scheduling metadata
>
> 4. **Documentation upload** — Drop architecture docs, runbooks, spreadsheets, or wiki exports here. I'll summarize and extract answers from them.
>
> 5. **MySQL/Aurora connection** — I'll inspect: source schema structure, table sizes, replication config, stored procedures that may contain transformation logic
>
> Select all that apply (or type "none" to answer questions manually).

⚠️ **STOP**: Wait for user response before proceeding.

#### If Redshift connection provided:

Use the AIM migration skill to connect and run these system catalog queries:

```sql
-- Object inventory
SELECT schemaname, tablename, 'table' as obj_type FROM pg_tables WHERE schemaname NOT IN ('pg_catalog','information_schema')
UNION ALL
SELECT schemaname, viewname, 'view' FROM pg_views WHERE schemaname NOT IN ('pg_catalog','information_schema');

-- Row counts and sizes
SELECT "table" AS tablename, tbl_rows AS row_count, size AS size_mb
FROM svv_table_info ORDER BY size DESC;

-- Sort and distribution keys
SELECT tablename, "column", type, encoding, sortkey, distkey
FROM pg_table_def WHERE schemaname NOT IN ('pg_catalog','information_schema');

-- Stored procedures
SELECT schemaname, proname, prosrc FROM pg_proc_info
WHERE schemaname NOT IN ('pg_catalog','information_schema');

-- Recent query patterns (last 7 days)
SELECT userid, query, starttime, endtime, elapsed/1000000.0 as seconds
FROM stl_query WHERE starttime > GETDATE() - 7
ORDER BY elapsed DESC LIMIT 50;

-- WLM configuration
SELECT * FROM stl_wlm_query ORDER BY service_class DESC LIMIT 20;

-- User/group inventory
SELECT usename, usecreatedb, usesuper FROM pg_user;
```

Summarize findings and pre-fill relevant sections in the output file.

#### If AWS CLI provided:

```bash
# Cluster configuration
aws redshift describe-clusters --output json

# Recent costs (last 3 months)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '-90 days' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Redshift","Amazon Simple Storage Service"]}}' \
  --metrics BlendedCost

# S3 bucket sizes
aws s3 ls --summarize --recursive s3://<waggle-bucket>/
```

#### If Talend project export provided:

Parse the project structure:
- Count `.item` files by type (tMysqlInput, tRedshiftOutput, tMap, tJava, tFileOutputDelimited, etc.)
- Extract job names from `*.properties` files
- Map parent-child job relationships (tRunJob references)
- Identify context variables and their groupings
- Count components per job (complexity proxy)

Present summary: "Found X jobs: Y simple extracts, Z transformations, W custom Java, etc."

#### If documentation uploaded:

Read uploaded files and extract answers to relevant questions. Summarize what was found and what still needs manual answers.

#### If MySQL/Aurora connection provided:

```sql
-- Schema inventory
SELECT table_schema, table_name, table_rows, data_length, index_length
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
ORDER BY data_length DESC;

-- Stored procedures (transformation logic)
SELECT routine_schema, routine_name, routine_type
FROM information_schema.routines
WHERE routine_schema NOT IN ('mysql','information_schema','sys');

-- Replication status
SHOW SLAVE STATUS;
SHOW BINARY LOG STATUS;
```

### Step 1.3 — Interactive Discovery Questions (for remaining gaps)

After automated collection, identify which questions from `references/question-bank.md` are still unanswered. Present ONLY the remaining questions.

Read `references/question-bank.md` and walk through each category interactively. For each section:

1. Present the category name and explain why these questions matter
2. Present ALL questions in that category at once (most categories are ≤10 questions — don't split them up)
3. Skip questions already answered by automated collection (note: "Auto-collected from [source]")
4. Record answers
5. Move to next category

**Category order:**
1. Data Model & Scale
2. Talend ETL Inventory
3. Data Pipeline & Latency
4. Downstream Consumers
5. Security & Compliance (FERPA/COPPA)
6. Operations & Monitoring
7. AI Platform Requirements
8. Migration Logistics & Constraints
9. Cost & Business Case

For each answer received, note it in the output. If the user says "I don't know" or "need to check", mark it as `[TBD — needs engineering team input]`.

### Step 1.4 — Workload Patterns & Volume Sizing

Read `references/workload-sizing.md` Steps 1 and 5.

Collect workload metrics required to properly size the target Snowflake architecture. These go **beyond** object inventory and capture the intensity and patterns of the current environment.

**If Redshift access was provided in Step 1.2**, run the workload profiling queries from `references/workload-sizing.md` Section 1.1 automatically:
- Daily query volume and compute hours (30 days)
- Query volume by user/service (ETL vs. reporting vs. ad-hoc breakdown)
- Concurrency profile (peak simultaneous queries per hour)
- Data scanned per query (GB)
- WLM queue wait times
- Nightly batch window duration

**For all deployments**, ask supplemental workload questions (Section 5 of `references/workload-sizing.md`):

> To properly size the target architecture and understand your workload patterns, I need a few additional data points:
>
> **Activity patterns:**
> 1. How many hours/day is Redshift actively processing queries (vs. idle)?
> 2. What is your peak concurrent query count? When does it peak?
> 3. How long does your nightly ETL batch run (start to finish)?
> 4. Are there periods of zero activity (nights, weekends, school breaks)?
>
> **Data freshness needs:**
> 5. Which tables need near-real-time freshness (<5 min)? Which are fine with daily?
> 6. For reporting marts — is 1-minute, 5-minute, or 1-hour freshness acceptable?
>
> **AI platform volume (Year 1 estimate):**
> 7. How many natural-language analytics queries/day do you expect from teachers/admins?
> 8. How large is the content corpus for search/RAG (documents/pages)?
> 9. How many students would ML models score, and how often?
>
> **Budget context:**
> 10. What is your target monthly budget for the data platform (or current total platform cost to stay under)?
> 11. Is the ROI story about cost savings or value creation (AI enables new revenue)?

⚠️ **STOP**: Wait for user answers. Record in the output file under a "Workload Patterns & Sizing Inputs" section.

### Step 1.5 — Snowflake-Specific Decisions

Present questions from Section 10 of the question bank (Snowflake-Specific Decisions). These require architectural judgment and cannot be auto-collected.

### Step 1.6 — Generate Output File

Create `01-current-state-discovery.md` using the template from `references/output-templates.md`. Include:
- All answered questions with responses
- Architecture diagrams (current state in Mermaid)
- **Workload patterns summary** (query volumes, compute hours, concurrency, batch durations, data freshness requirements)
- TBD items clearly marked
- Summary of key findings and risks
- Recommended next steps

Write the file to the user's current working directory.

> ✅ Phase 1 complete. Output written to `01-current-state-discovery.md`.
>
> **Key findings summary:**
> [Summarize top 3-5 findings]
>
> **Items needing follow-up:**
> [List TBD items]

### Step 1.7 — AIM Assessment (combined — runs automatically if Redshift was connected)

**If Redshift credentials were provided in Step 1.2**, proceed directly to AIM assessment without requiring the user to start a new phase. The DDLs are already extracted — use them now.

If Redshift was NOT connected in Step 1.2, inform the user:

> To produce the conversion assessment (`02-aim-assessment.md`), I need Redshift access. Would you like to provide credentials now, or run Phase 1 again later with Redshift access?

**When Redshift DDLs are available**, read `references/phase2-assessment.md` and execute:

1. Run AIM assessment on extracted code (complexity scoring, dependency graph)
2. Run SnowConvert AI conversion analysis (auto-convert rate, EWI breakdown)
3. Classify Talend jobs using the replacement mapping framework (from Phase 1 Talend data or manual categorization)
4. Generate risk matrix (high/medium/low)
5. Output: `02-aim-assessment.md`

This produces BOTH output files in a single session — no duplicate Redshift connection needed.

> ✅ Assessment complete. Output written to `02-aim-assessment.md`.
>
> **Conversion feasibility**: [X]% auto-convertible
> **Key risks**: [Top 3 risks]
>
> When ready, run this skill again and select Phase 2 (Architecture Recommendations).

---

## Phase 2: Architecture Recommendations

Read `references/phase3-architecture.md` for detailed instructions.

**Summary:**
1. Review Phase 1 outputs (`01-current-state-discovery.md` and `02-aim-assessment.md`)
2. Present lift/shift vs. refactor decision framework
3. Design target Snowflake architecture (data zones, compute, security)
4. Design data ingestion layer (evaluate Fivetran vs. Openflow vs. Snowpipe Streaming)
5. Design transformation layer (dbt, Dynamic Tables, stored procs)
6. Design AI platform layer (read `references/ai-capabilities-matrix.md`)
7. **Build workload sizing & cost projection** — read `references/workload-sizing.md` Steps 2-4. Map collected workload patterns to Snowflake warehouse sizes, estimate resource consumption per workload, build monthly projection, produce scenario comparison (conservative/expected/optimistic), and 12-month ramp model.
8. Output: `03-target-architecture.md` with Mermaid diagrams and **full Cost Model section**

---

## Phase 3: Migration Execution Plan

Read `references/phase4-migration-plan.md` for detailed instructions.

**Summary:**
1. Review all prior phase outputs
2. Define migration waves (prioritized by dependency + risk)
3. Design parallel run strategy
4. Define validation criteria per wave
5. Plan cutover sequence
6. Define rollback procedures
7. Estimate resource requirements
8. Output: `04-migration-plan.md`

---

## Sharing Results

All output `.md` files are designed to be:
- Shared internally within HMH for review and input
- Shared back to the Snowflake team (Roy Ballard, Russ Goldin) for migration expert engagement
- Used as input to the AIM migration agent for automated execution

Remind the user at the end of each phase that they can share the output with their Snowflake team.
