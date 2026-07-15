# Output Templates

Templates for each phase's .md output file.

---

## Phase 1 Output: `01-current-state-discovery.md`

```markdown
# HMH Waggle — Current State Discovery

**Generated**: [DATE]
**Participants**: [Names]
**Status**: [Complete / Partial — X sections need follow-up]

---

## Executive Summary

[2-3 sentences: what we learned, key findings, primary risks or opportunities]

---

## Current Architecture

### System Diagram

```mermaid
[Auto-generated Mermaid diagram based on confirmed architecture]
```

### Component Inventory

| Component | Technology | Version | Owner | Notes |
|-----------|-----------|---------|-------|-------|
| OLTP Database | Aurora MySQL | [version] | [team] | |
| ETL Tool | Talend [edition] | [version] | [team] | |
| Data Warehouse | Amazon Redshift | [node type] | [team] | |
| Reporting | JasperSoft | [version] | [team] | |
| ... | | | | |

---

## Data Model & Scale

### Redshift Clusters

| Cluster | Type | Tables | Rows | Storage | Purpose |
|---------|------|--------|------|---------|---------|
| Write | [node type] | 215 | ~3B | [TB] | staging, datamart |
| Read | [node type] | 65 | ~79M | [TB] | reporting, API |
| History | [node type] | [TBD] | [TBD] | [TBD] | archive |

### Key Tables (Top 10 by size)

| Table | Rows | Size | Zone | SLA | Notes |
|-------|------|------|------|-----|-------|

### Data Growth

- Daily ingest volume: [X GB/day]
- Growth rate: [X% monthly]
- Retention: [X years]

---

## Talend ETL Inventory

### Job Summary

| Category | Count | Avg Runtime | Complexity | Target Tool |
|----------|-------|-------------|------------|-------------|
| Extract (MySQL → S3) | | | | Fivetran |
| Load (S3 → Redshift) | | | | Fivetran/Snowpipe |
| Transform (staging → mart) | | | | dbt |
| Complex transform | | | | dbt + macros |
| Custom Java | | | | Snowpark |
| Outbound feeds | | | | Tasks + COPY INTO |
| Orchestration | | | | Airflow/Tasks |

### Critical Job Chains

[List the most important job dependency chains]

### Scheduling

- Scheduler: [Talend TAC / cron / other]
- Batch window: [start time] - [end time]
- Total nightly runtime: [X hours]

---

## Data Pipeline & Latency

| Consumer | Current Latency | Required Latency | Gap |
|----------|----------------|-----------------|-----|
| Teacher dashboards | | | |
| Admin reports | | | |
| Student-facing UI | | | |
| Datalake | | | |
| CSV exports | | | |

### CDC Status

- Currently implemented: [Yes/No]
- Method: [Talend / binlog / none]
- Streaming needs identified: [list]

---

## Downstream Consumers

### Reports

| System | Report Count | Active | Method | Repointing Complexity |
|--------|-------------|--------|--------|----------------------|
| JasperSoft | | | | |
| CSV exports | | | | |
| API consumers | | | | |
| Datalake feeds | | | | |

---

## Security & Compliance

| Requirement | Status | Notes |
|------------|--------|-------|
| FERPA compliance | [Yes/No/TBD] | |
| COPPA compliance | [Yes/No/TBD] | |
| Row-level security | [Yes/No/TBD] | |
| Column masking | [Yes/No/TBD] | |
| Encryption at rest | [Yes/No/TBD] | |
| Audit logging | [Yes/No/TBD] | |
| Data residency | [region] | |

---

## Operations & Team

### Team

| Person | Role | Availability | Snowflake Experience |
|--------|------|-------------|---------------------|
| Nagaraj | Dir AI Engineering | Executive sponsor | New |
| Mahesh Birajdar | Sr Data Architect | Technical lead | [level] |
| [others] | | | |

### Current Pain Points

1. [List identified pain points]

---

## AI Platform Requirements

| Capability | Priority | Use Cases | Target Users |
|-----------|----------|-----------|--------------|
| GenAI/LLM | [H/M/L] | | |
| Semantic Layer | [H/M/L] | | |
| Agents | [H/M/L] | | |
| Predictive ML | [H/M/L] | | |
| In-App AI | [H/M/L] | | |

---

## Migration Constraints

| Constraint | Details |
|-----------|---------|
| Timeline | |
| Hard deadlines | |
| Parallel run required | |
| Budget | |
| Team bandwidth | |
| School calendar constraints | |

---

## TBD Items (Needs Follow-Up)

| # | Question | Category | Who Can Answer | Priority |
|---|----------|----------|---------------|----------|
| 1 | | | | |
| 2 | | | | |

---

## Key Findings & Risks

### Top Findings
1. [Finding 1]
2. [Finding 2]
3. [Finding 3]

### Identified Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|

---

## Recommended Next Steps

1. [Action item 1] — Owner: [name]
2. [Action item 2] — Owner: [name]
3. [Action item 3] — Owner: [name]

---

*Share this document with your Snowflake team (Roy Ballard, Russ Goldin) to enable migration expert engagement.*
```

---

## Notes on Template Usage

- Replace all `[bracketed]` placeholders with actual values
- Mark items as `[TBD]` when the user doesn't have the answer yet
- Include Mermaid diagrams where architecture visualization helps
- Keep each output file under ~500 lines (split if needed)
- Use tables for structured data (easier to scan)
- End every file with "Share this document..." reminder
