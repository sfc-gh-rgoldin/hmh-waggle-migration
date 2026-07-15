# HMH Waggle Migration — Exhaustive Question Bank

Instructions: Walk through each category with the user. Ask 3-5 questions at a time. Record answers inline. Mark unknowns as `[TBD]`.

---

## 1. Data Model & Scale

### 1.1 Redshift Clusters

- How many Redshift clusters are in use for Waggle (beyond the read/write pair)?
- What node types are the clusters (RA3, DC2, DS2)? How many nodes per cluster?
- What is the total storage consumed across clusters (TB)?
- What Redshift features are you actively using? (Spectrum, Federated Query, Data Sharing, AQUA, Materialized Views, Concurrency Scaling)
- What is the peak concurrent query load on the read cluster? Write cluster?
- Are there Workload Management (WLM) queues configured? How many and for what purposes?
- What is the typical query runtime for your longest/most critical queries?

### 1.2 Schema & Object Inventory

- How many schemas exist across both clusters?
- Breakdown of object types: tables (regular, temp, external), views (standard, late-binding, materialized), stored procedures, UDFs?
- Are there sort keys defined? What types (compound, interleaved)? On which tables?
- Are there distribution keys/styles (KEY, EVEN, ALL)? Which critical tables use which style?
- Are there any external schemas (Redshift Spectrum pointing to S3)?
- What encoding/compression schemes are used on columns?
- Are there any cross-database queries or cross-cluster queries?

### 1.3 Data Characteristics

- What is the data retention policy? How far back does historical data go?
- What is the daily data growth rate (GB/day or rows/day)?
- What is the largest single table (by rows and by bytes)?
- Are there any JSON/semi-structured columns stored in VARCHAR?
- What is the typical row width (bytes) for your fact tables vs. dimension tables?
- Are there any geospatial data types in use?
- What character encoding is used (UTF-8, Latin-1, etc.)?

---

## 2. Talend ETL Inventory

### 2.1 Job Landscape

- How many Talend jobs exist in total for Waggle?
- How are they organized (projects, folders, job groups)?
- What Talend edition is in use (Open Studio, Data Integration, Data Fabric)?
- What version of Talend is running?
- How many jobs run daily? Weekly? Monthly? Ad-hoc?
- What is the longest-running Talend job and how long does it take?
- What is the total runtime for all nightly batch jobs combined?

### 2.2 Job Complexity

- How many jobs use custom Java code or custom components (tJavaRow, tJava, tJavaFlex)?
- How many jobs use stored procedure calls within Talend (tMysqlSP, tRedshiftSP)?
- Are there lookup joins done in Talend vs. in Redshift?
- How many jobs do SCD (Slowly Changing Dimension) processing?
- Are there jobs that perform data quality checks or cleansing?
- How many jobs use tMap (complex transformations) vs. simple source-to-target?
- Are there any Talend jobs that call external APIs or web services?

### 2.3 Orchestration & Dependencies

- How are Talend jobs scheduled (Talend TAC, cron, external scheduler)?
- What are the job dependencies/DAG structure? Are there critical chains?
- How is error handling implemented (tLogCatcher, tDie, alerts)?
- What monitoring/alerting exists for job failures?
- Are there retry mechanisms or manual intervention procedures?
- How are job parameters/context variables managed (hardcoded, environment-specific, dynamic)?
- How many environments exist (dev, test, staging, prod)? How are promotions handled?

### 2.4 Talend → Replacement Mapping

- Which Talend jobs are pure extract (MySQL → S3)? These map to **Fivetran**.
- Which Talend jobs are transformations within Redshift (staging → datamart)? These map to **dbt**.
- Which Talend jobs handle outbound feeds (Redshift → JasperSoft, CSVs, Datalake)? These need a **separate strategy** (Snowflake Tasks, reverse ETL, or lightweight orchestrator).
- Are there Talend jobs that do both extract AND transform in one job? These need decomposition.
- Which jobs use Talend-specific connectors that have no direct equivalent?

---

## 3. Data Pipeline & Latency

### 3.1 Ingestion Patterns

- What is the current batch frequency from MySQL to Redshift (hourly, daily, real-time)?
- Is Change Data Capture (CDC) implemented today? If so, how (Talend CDC, MySQL binlog, Debezium, other)?
- Are full table loads or incremental loads the norm? What's the split?
- What is the acceptable latency from a change in MySQL to it being queryable in the DWH? (seconds, minutes, hours?)
- Are there any real-time/streaming requirements today or anticipated?
- Does the S3 intermediate layer serve any purpose beyond staging (e.g., data lake consumers reading raw S3 files)?

### 3.2 Ingestion Tool Evaluation (Fivetran vs. Snowflake-Native)

The proposed architecture uses Fivetran, but Snowflake has native ingestion options that may eliminate the need for a separate tool. Assess which fits best:

- How many distinct source systems need ingestion (just Aurora MySQL, or others)?
- Is the source MySQL connector the only one needed, or are there other databases/SaaS apps feeding Waggle?
- What is the required ingestion latency? (This determines which options are viable)
  - **Minutes (5-15)**: Fivetran CDC, Snowflake Openflow, Snowpipe
  - **Seconds (<10s)**: Snowpipe Streaming v2
  - **Sub-second**: Snowflake Postgres (direct OLTP reads)
- Is there a preference for managed (hands-off) vs. self-managed ingestion?
- Is there an existing Fivetran contract or relationship at HMH (across other products)?
- Would you prefer fewer vendors (consolidate into Snowflake-native) or best-of-breed (Fivetran for connectors)?
- Are there non-database sources (SaaS APIs, event streams, file drops) that also need ingestion?

**Snowflake-native ingestion options to evaluate:**

| Option | Best For | Latency | MySQL Support | Complexity |
|--------|----------|---------|---------------|------------|
| **Openflow** | Visual pipelines, CDC from databases | Minutes | Yes (GA connector) | Low (no-code UI) |
| **Snowpipe Streaming v2** | High-throughput event streams, real-time | Seconds | Via SDK/Kafka Connector | Medium |
| **Snowpipe** | File-based ingestion from S3 | Minutes | N/A (file-based) | Low |
| **Dynamic Tables** | Continuous transformation (replaces batch) | Minutes | N/A (Snowflake-to-Snowflake) | Low |
| **Snowflake Postgres** | OLTP-speed reads, app-serving | Real-time | N/A (serves from Snowflake) | Low |
| **Fivetran** | Broadest connector library, fully managed | Minutes | Yes (CDC) | Lowest (SaaS) |

**Key trade-off questions:**
- If ONLY Aurora MySQL needs ingestion → **Openflow** may fully replace Fivetran (fewer vendors, lower cost, native Snowflake)
- If multiple SaaS sources also need ingestion → **Fivetran** has broader connector coverage
- If sub-10s latency needed → **Snowpipe Streaming** (neither Fivetran nor Openflow achieve this today)
- If Waggle app needs to read from the DWH at OLTP speed → **Snowflake Postgres** for the serving layer

### 3.2 Data Freshness SLAs

- What SLAs exist for data freshness in reports? (e.g., "reports must reflect data as of 6am")
- Are there different SLAs for different consumer types (teachers vs. administrators vs. district)?
- What happens when SLAs are missed? Is there an escalation process?
- Are there any regulatory requirements around data timeliness (e.g., state reporting deadlines)?

### 3.3 Streaming & Event Processing

- Does Waggle generate real-time events (student interactions, quiz completions, etc.)?
- If yes, where do these events go today? (Kafka, Kinesis, direct DB writes, S3?)
- Are there use cases where sub-second latency is required?
- Would you benefit from near-real-time analytics (e.g., teacher sees student progress updating live)?

---

## 4. Downstream Consumers

### 4.1 JasperSoft Reporting

- How many JasperSoft reports exist total?
- How many are actively used (run at least once per month)?
- What is the report generation method (scheduled, on-demand, embedded in Waggle UI)?
- Do reports use direct SQL queries, stored procedures, or JasperSoft domains/views?
- What version of JasperSoft is running? Is there a plan to modernize reporting tools?
- Are there any reports that require sub-second response time (embedded/interactive)?
- Who owns/maintains the JasperSoft reports (Waggle team, BI team, individual users)?

### 4.2 CSV Exports & File Feeds

- How many CSV export jobs exist?
- Where do they land (Windows Server, S3, SFTP)?
- Who consumes them (internal teams, external partners, state agencies)?
- What is the format/schema of these exports? Are they documented?
- Are there SLA requirements on export delivery timing?
- Are any exports regulatory-mandated (state reporting, compliance)?

### 4.3 Datalake Integration

- What technology is the Datalake (S3 raw files, Delta Lake, Iceberg, Parquet)?
- What consumes from the Datalake (other HMH products, data science, external)?
- Is there a bidirectional flow (Datalake feeds back into Waggle ADB/TDB)?
- Does data from other HMH products need to join with Waggle data in the Datalake?

### 4.4 API Consumers

- Does the Waggle UI query Redshift directly or through an API layer?
- Are there other applications or services that query Redshift?
- What are the concurrency requirements (how many simultaneous API-driven queries)?
- What latency requirements exist for API-driven queries (p50, p99)?

---

## 5. Security & Compliance

### 5.1 Data Classification

- What categories of PII exist in Waggle data? (student names, emails, grades, addresses, SSNs?)
- Is there student data subject to **FERPA** (Family Educational Rights and Privacy Act)?
- Is there data subject to **COPPA** (Children's Online Privacy Protection Act) for students under 13?
- Are there state-specific student data privacy laws that apply (e.g., California SOPIPA)?
- Is data classified into sensitivity tiers? What are they?

### 5.2 Access Control

- How is Redshift access controlled today (IAM roles, database users, groups)?
- How many distinct access patterns/roles exist?
- Is there row-level security (e.g., district A can only see district A data)?
- Is there column-level masking (e.g., mask student SSN for non-admin users)?
- How are service accounts managed (Talend, JasperSoft, API layer)?
- Is there an integration with an identity provider (Okta, Azure AD, etc.)?

### 5.3 Encryption & Network

- Is data encrypted at rest in Redshift (KMS keys)?
- Is data encrypted in transit (SSL/TLS for all connections)?
- Is Redshift in a private VPC? Are there VPN/PrivateLink requirements?
- Are there IP allowlisting or firewall rules that would affect Snowflake connectivity?
- Are there data residency requirements (must stay in a specific AWS region)?

### 5.4 Audit & Compliance

- Are there audit logging requirements (who accessed what data when)?
- How long must audit logs be retained?
- Are there regular compliance audits (SOC 2, ISO 27001, state audits)?
- Is there a Data Protection Officer or privacy team involved in architecture decisions?

---

## 6. Operations & Monitoring

### 6.1 Current Operations

- What is the on-call/support model for the Waggle data pipeline?
- What monitoring tools are in use (CloudWatch, Datadog, Grafana, custom)?
- What alerting is in place for pipeline failures or data quality issues?
- How frequently do pipeline failures occur? What is the typical MTTR?
- Is there a runbook or playbook for common failure scenarios?
- What is the maintenance window (if any) for Redshift?

### 6.2 Data Quality

- Are there data quality checks in the current pipeline? Where do they run?
- What types of checks (null counts, freshness, row counts, duplicates, referential integrity)?
- How are data quality issues surfaced and resolved?
- Is there a data quality SLA or metric being tracked?

### 6.3 Team & Capacity

- How many people are on the Waggle data/analytics team?
- What is their skill set (SQL, Python, dbt, Snowflake, Talend, AWS)?
- Is there Snowflake experience on the team already?
- Who would be the primary engineers executing the migration?
- Is there budget for external help (SI, Snowflake PS, FDE)?
- What is the team's bandwidth — is this a full-time dedicated effort or alongside BAU?

---

## 7. AI Platform Requirements

### 7.1 GenAI / LLM

- What GenAI use cases are you envisioning? (content generation, summarization, Q&A, tutoring?)
- Are these internal (teacher/admin tools) or external (student-facing)?
- What data would the LLMs need access to (student records, curriculum content, assessment results)?
- Are there existing LLM/AI models in use today? What providers (OpenAI, AWS Bedrock, etc.)?
- What are the latency requirements for GenAI responses (real-time chat vs. batch processing)?
- Are there content safety/guardrails requirements for student-facing AI?

### 7.2 Semantic Layer / Natural Language Analytics

- Who would use natural language querying? (district admins, school principals, teachers?)
- What types of questions would they ask? (e.g., "How are my 3rd graders performing in math?")
- What is the domain ontology? (districts → schools → classes → students → assignments → grades)
- Are there business rules that define metrics? (e.g., "proficiency" = score >= 80%, "at-risk" = declining 3 consecutive assessments)
- Would this replace or supplement JasperSoft reporting?

### 7.3 Agentic Workflows

- What multi-step processes would benefit from AI agents? (e.g., "identify struggling students, generate intervention plan, notify teacher")
- Are there operational workflows that could be automated (data pipeline monitoring, report generation, anomaly detection)?
- Who would configure/maintain agents? (central data team, product team, individual teachers?)
- What tools/actions should agents be able to take? (query data, send notifications, update records, generate content?)

### 7.4 Predictive Analytics & ML

- What prediction targets are relevant? (student performance, dropout risk, engagement, learning path optimization?)
- What features/signals are available for prediction (attendance, assignment completion, time-on-task, assessment scores)?
- What is the prediction horizon (next week, next quarter, next year)?
- Are there existing ML models that would need to migrate?
- What is the expected inference latency (batch scoring vs. real-time)?

### 7.5 In-App AI (Student/Teacher Facing)

- What in-app AI experiences are envisioned? (intelligent tutoring, auto-grading, content recommendation, chatbot?)
- What latency requirements exist for in-app AI? (must respond in <200ms? <2s?)
- What data does the app layer need access to for AI (student profile, learning history, curriculum)?
- Is the app layer currently reading from Redshift or from the API/cache layer?
- Would Snowflake Postgres (OLTP-speed queries) be relevant for serving the app layer?
- How would AI features be A/B tested and measured?

---

## 8. Migration Logistics & Constraints

### 8.1 Timeline & Priority

- What is the ideal completion date for the migration?
- Are there hard deadlines (contract renewals, budget cycles, school year boundaries)?
- What is driving urgency? (Talend license renewal date, Redshift RI expiration, executive mandate?)
- Is there a specific school calendar constraint (e.g., can't migrate during testing season)?
- Can you do a phased migration (some tables/jobs first) or must it be all-at-once?

### 8.2 Parallel Run & Cutover

- Is a parallel run period required (both Redshift and Snowflake active)?
- If yes, how long should the parallel run last?
- What validation criteria must be met before cutover? (100% data match? Report parity? Performance benchmarks?)
- Who has authority to approve the cutover?
- What is the rollback plan if issues are found post-cutover?
- Can Waggle UI/app be updated incrementally (some APIs point to Snowflake while others still on Redshift)?

### 8.3 Dependencies & Blockers

- Are there other systems that depend on Redshift being available (shared data, cross-team queries)?
- Are there upstream systems that would need to change (anything feeding into Aurora MySQL)?
- Are there contractual or procurement dependencies (Fivetran contract, dbt Cloud license, Airflow hosting)?
- Are there infrastructure dependencies (VPC peering, IAM roles, network routes)?
- Is there a change management / CAB process for production changes?

### 8.4 Testing Strategy

- What testing environments exist today?
- Can you stand up a Snowflake test environment that mirrors production?
- Who performs UAT (User Acceptance Testing) for reports and data?
- Are there automated test suites for data validation?
- How will you validate that Snowflake produces identical results to Redshift?

---

## 9. Cost & Business Case

### 9.1 Current Spend

- What is the annual AWS Redshift cost (including RI, storage, data transfer)?
- What is the annual Talend licensing cost?
- What is the S3 storage and transfer cost attributable to Waggle?
- What is the total annual cost of the current Waggle data platform (infra + licensing + people)?
- Are there upcoming cost events (RI expiration, Talend renewal, cluster resize needed)?

### 9.2 ROI Expectations

- What is the expected ROI model (cost savings, speed-to-market, AI capabilities, reduced maintenance)?
- Is there a budget allocated for this migration?
- Is there a cost comparison requirement (current state vs. Snowflake projected)?
- What value would AI capabilities add? Is there a revenue model for AI features?
- Is there executive sponsorship for the spend? (Nagaraj has confirmed yes — but who approves budget?)

### 9.3 Vendor Considerations

- Is there an existing Fivetran relationship or contract?
- Is there an existing dbt Cloud relationship or would dbt Core (open source) be preferred?
- Is there an Airflow instance already (MWAA, self-hosted, Astronomer)?
- Are there preferred cloud partners or marketplace vendors?
- Has procurement been involved yet?

---

## 10. Snowflake-Specific Decisions

### 10.1 Architecture Choices

- Single Snowflake account or multi-account (dev/test/prod)?
- Should Waggle data live in the existing HMH Snowflake org or a new account?
- What Snowflake edition is appropriate (Standard, Enterprise, Business Critical)?
- Multi-cluster warehouse for concurrency or separate warehouses per workload?
- Do you want to leverage existing NWEA Snowflake infrastructure or start fresh for Waggle?

### 10.2 Feature Preferences

- Dynamic Tables vs. dbt for transformation (or hybrid)?
- Snowpipe Streaming vs. Fivetran for ingestion (trade-offs: latency vs. simplicity)?
- Snowflake Tasks vs. Airflow for orchestration?
- Snowflake Notebooks for data exploration and development?
- Snowflake Postgres for OLTP workloads (replacing direct Aurora MySQL reads)?

### 10.3 Data Sharing & Collaboration

- Would Waggle data need to be shared with other HMH products via Snowflake Data Sharing?
- Are there external data providers (Marketplace datasets) that would enrich Waggle data?
- Would district/school customers need direct Snowflake access (Reader accounts, Data Clean Rooms)?
- Is there cross-product analytics needed (e.g., join Waggle data with NWEA MAP assessment data)?
