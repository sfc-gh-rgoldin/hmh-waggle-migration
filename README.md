# Skills-Powered Discovery: "Reverse RFP" for Migration Projects

A Cortex Code skill that automates customer discovery and migration scoping. Instead of weeks of meetings and async Q&A, the SE encodes their methodology into an autonomous multi-phase workflow the customer runs against their own environment.

## The Problem

Traditional discovery for migration projects:
- 3-4 weeks elapsed time
- 15-20 hours of SE time
- 4-5 meetings
- High risk of miscommunication and missed requirements
- Output quality varies by SE

## The Solution

Build a personalized skill. Share it with the customer. They run it in ~1 hour. You analyze 4 comprehensive documents.

**Total time: same-day turnaround. SE effort: a few hours (build + analyze). Meetings: 1 (focused on decisions).**

## Case Study: HMH Waggle

**Customer goal**: Migrate the Waggle adaptive learning platform from Redshift/Talend to Snowflake  
**Environment**: Aurora MySQL → Talend ETL → Redshift (us-east-1)  
**Who ran it**: Mahesh Birajdar (Sr Data Architect), active Cortex Code user

### Results

| Metric | Value |
|--------|-------|
| Questions answered automatically | 185 of 201 (92%) |
| Items needing human judgment | 6 |
| Time from initial meeting to skill delivery | <1 day |
| Time from skill delivery to customer response | ~3 business days |
| Time to scoping meeting with PS | 7 business days after initial meeting |

### What the Skill Produced

| Document | Contents |
|----------|----------|
| `01-current-state-discovery.md` | Full inventory: 429 tables, 555 ETL jobs, 563 stored procs, team, consumers, constraints |
| `02-aim-assessment.md` | Conversion feasibility (85-90% auto-convertible), EWI report, risk matrix, effort estimation |
| `03-target-architecture.md` | Zone design, warehouse sizing, RBAC, FERPA compliance model, cost projection (20-50% savings) |
| `04-migration-plan.md` | 6-wave execution plan with validation criteria, rollback procedures, resource plan |

### 201 Questions Across 10 Categories

| Section | Questions |
|---------|-----------|
| Data Model & Scale | 21 |
| Talend ETL Inventory | 26 |
| Data Pipeline & Latency | 21 |
| Downstream Consumers | 21 |
| Security & Compliance | 20 |
| Operations & Monitoring | 16 |
| AI Platform Requirements | 26 |
| Migration Logistics | 21 |
| Cost & Business Case | 15 |
| Snowflake-Specific Decisions | 14 |

## 4-Phase Gated Workflow

Each phase is gated — output is reviewed before proceeding. The customer can run all 4 phases in a single Cortex Code session.

1. **Current State Discovery** — Inventories all sources, tables, ETL jobs, consumers, team, constraints
2. **AIM Assessment** — Analyzes code for conversion feasibility, identifies blockers, scores risk
3. **Target Architecture Design** — Generates zone architecture, warehouse design, security model, cost projection
4. **Migration Execution Plan** — Produces wave-by-wave plan with validation criteria, rollback procedures, staffing

## How It Was Built

After the HMH customer meeting, one prompt in Cortex Code:

> "I want to create a skill for the customer to run internally. It should discover current state (Aurora MySQL, Talend, Redshift), assess conversion feasibility, design the target Snowflake architecture, and produce an execution plan. Make it a multi-phase gated workflow they can run autonomously. Fully analyze per the meeting transcript and customize to HMH and this project."

## How to Use

### Prerequisites
- Someone at the customer who can sign into the technical systems (for automated discovery)
- Someone who can answer direction/scope questions (may be the same person)

### Run the Skill
```
# In Cortex Code (CLI or Desktop)
/hmh-waggle-migration
```

The skill will guide the user through phase selection and interactive discovery.

## Adapting for Other Projects

This pattern is repeatable for any structured SE engagement:

1. Have a customer meeting. Understand the project scope.
2. Open Cortex Code. Describe your methodology.
3. CoCo builds the skill. Review and iterate.
4. Share the skill with the customer. They run it.
5. Analyze the output. Hold ONE meeting on items that require human judgment.

## Presentation

[Skills-Powered Discovery: A New SE Engagement Model — HMH Waggle Case Study](https://docs.google.com/presentation/d/12pu8tfsw-bFThMh5iYb02P58pCA7quXjwEPtfEfV7tw)

## Author

Russ Goldin | Senior Solution Engineer | russ.goldin@snowflake.com
