# Phase 2: AIM Assessment — Detailed Instructions

## Overview

Phase 2 uses Snowflake's AIM (AI Migration) tooling to perform automated assessment of the Waggle Redshift codebase. This produces a detailed complexity/risk report and identifies what can be auto-converted vs. what needs manual remediation.

## Prerequisites

1. **Phase 1 complete** — `01-current-state-discovery.md` exists with answers to key questions
2. **Redshift connectivity** — JDBC/ODBC access to both read and write clusters
3. **Migration plugin installed** — The Snowflake migration plugin must be active

## Step 2.1 — Ensure Migration Plugin

Check if the migration plugin is installed:

```
cortex plugin list
```

If `snowflake-migration` is not listed, install it:

```bash
cortex plugin install https://github.com/Snowflake-Labs/cortex-code-migrations/tree/preview/plugin
```

Then reload: `/plugin reload`

## Step 2.2 — Connect to Redshift and Extract DDLs

Use the AIM migration skill to connect to Redshift and extract code:

> I'm going to connect to your Redshift clusters to extract all DDLs (table definitions, views, stored procedures, functions).
>
> I'll need:
> - **Host**: Your Redshift cluster endpoint(s)
> - **Port**: (default 5439)
> - **Database**: The database name(s)
> - **User**: A read-only user with access to system catalogs
> - **Password**: (will be handled securely)
>
> Which cluster should we start with — read or write?

⚠️ **STOP**: Wait for connection details. Do NOT proceed without valid credentials.

Once connected, invoke the AIM extraction:

1. Extract DDLs from `pg_catalog` and `information_schema`
2. Extract stored procedures and functions
3. Extract view definitions (including late-binding views)
4. Extract user-defined functions
5. Capture table statistics (row counts, sizes, last modified)

**AIM prompt to use:**
> Use the migration skill to extract code from my Redshift database. Connect using [credentials provided]. Extract all DDLs, views, stored procedures, and functions. Save output to a local directory.

## Step 2.3 — Run AIM Assessment

Once DDLs are extracted, run the assessment:

**AIM prompt to use:**
> Run an assessment on the extracted Redshift code. Generate a comprehensive report covering:
> - Object inventory (counts by type)
> - Complexity scoring per object
> - Conversion feasibility (auto-convertible vs. manual)
> - Unsupported Redshift features identified
> - Dependencies graph
> - Risk areas and recommendations

The assessment will produce an HTML report. Summarize key findings for the output .md file.

## Step 2.4 — SnowConvert AI Analysis

Run SnowConvert on the extracted code:

**AIM prompt to use:**
> Convert the extracted Redshift DDLs to Snowflake using SnowConvert AI. Report on:
> - Conversion success rate (% auto-converted)
> - EWIs (Errors, Warnings, Informational) breakdown
> - Objects requiring manual remediation
> - Redshift-specific features that need alternatives (sort keys → clustering, dist keys → elimination, etc.)

## Step 2.5 — Talend Job Analysis

For Talend jobs (which AIM doesn't auto-extract), perform manual analysis based on Phase 1 answers:

### Classification Framework

| Category | Replacement | Complexity |
|----------|-------------|------------|
| **Simple extract** (MySQL → S3) | Fivetran connector | Low |
| **Simple load** (S3 → Redshift COPY) | Fivetran or Snowpipe | Low |
| **SQL transform** (staging → datamart via SQL) | dbt model | Low-Medium |
| **Complex transform** (tMap, joins, SCD) | dbt model + macros | Medium |
| **Custom Java** (tJava, tJavaRow) | Snowpark Python/Java or stored proc | High |
| **API calls** (web service, REST) | Snowflake external functions or Tasks | Medium-High |
| **Outbound feeds** (→ files, JasperSoft) | Snowflake Tasks + COPY INTO or reverse ETL | Medium |
| **Orchestration logic** (conditional, loops) | Airflow DAGs or Snowflake Tasks | Medium |

Ask the user to categorize their Talend jobs:

> Based on your knowledge of the Talend jobs, approximately how many fall into each category?
> 1. Simple extracts (MySQL → S3)
> 2. Simple loads (S3 → Redshift)
> 3. SQL-based transformations
> 4. Complex transformations (tMap, lookups, SCD)
> 5. Custom Java code
> 6. API/web service calls
> 7. Outbound feeds (exports)
> 8. Pure orchestration (job triggers, dependencies)

## Step 2.6 — Risk Assessment

Based on all analysis, categorize risks:

### High Risk Items
- Objects with conversion errors (EWI Errors)
- Custom Java in Talend jobs
- Redshift-specific features without direct equivalents
- Outbound feed dependencies (JasperSoft, file exports)
- Stored procedures with complex logic

### Medium Risk Items
- Late-binding views (need careful handling)
- Sort/distribution key-dependent query patterns
- Concurrency-dependent workloads
- SCD processing logic

### Low Risk Items
- Standard DDLs (tables, views)
- Simple SQL queries
- Basic extract/load jobs

## Step 2.7 — Generate Output

Create `02-aim-assessment.md` with these sections:

```markdown
# HMH Waggle — AIM Assessment Report

## Executive Summary
[2-3 paragraph summary of findings, conversion feasibility, key risks]

## Object Inventory
| Object Type | Count | Auto-Convertible | Manual Required | Notes |
|-------------|-------|-------------------|-----------------|-------|

## Conversion Analysis
### Success Rate: [X]%
### EWI Breakdown
| Severity | Count | Top Categories |
|----------|-------|----------------|

### Key Conversion Issues
[List specific issues and recommended remediations]

## Talend Job Analysis
| Category | Count | Target Tool | Effort Estimate |
|----------|-------|-------------|-----------------|

## Risk Matrix
### High Risk
### Medium Risk
### Low Risk

## Recommendations
1. [Prioritized list of recommendations]

## Next Steps
- Items to address before Phase 3
- Additional information needed
- Recommended POC scope
```
