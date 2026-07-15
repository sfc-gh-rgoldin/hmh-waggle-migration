# AI Capabilities Matrix — Snowflake Features for HMH

Maps each HMH AI requirement to specific Snowflake features with implementation guidance.

---

## 1. GenAI / LLM Capabilities

### Cortex Complete (LLM Function)

**What it does**: Call foundation models (Llama, Mistral, Gemma, Claude, GPT) directly in SQL or Python.

**HMH use cases**:
- Generate explanations for incorrect answers
- Summarize student performance narratives for parent reports
- Auto-generate quiz questions from curriculum content
- Translate content for multilingual students

**Implementation**:
```sql
SELECT SNOWFLAKE.CORTEX.COMPLETE(
    'llama3.1-70b',
    'Explain why the answer to "What is 3/4 + 1/2?" is 5/4, in terms a 4th grader would understand.'
) AS explanation;
```

**Latency**: 1-5 seconds depending on model and token count. Suitable for interactive (teacher tools) and batch (report generation).

**Cost**: AI credits per token. Optimize by choosing smallest effective model.

### Cortex Search (RAG)

**What it does**: Semantic search over documents/text with vector embeddings. Powers Retrieval-Augmented Generation.

**HMH use cases**:
- Intelligent tutoring: search curriculum materials for relevant explanations
- Teacher assistant: find relevant lesson plans, standards, exemplars
- Content discovery: "find exercises that teach fractions to struggling 3rd graders"

**Implementation**:
1. Create a Cortex Search Service over curriculum content table
2. Query returns relevant chunks with similarity scores
3. Feed top-k results as context to Cortex Complete for answer generation

**Latency**: Sub-second for search. 2-5s including LLM generation.

---

## 2. Semantic Layer / Natural Language Analytics

### Semantic Views

**What it does**: Define business metrics, dimensions, and relationships in YAML. Powers Cortex Analyst for natural language → SQL.

**HMH use cases**:
- District admin asks: "What's the average math proficiency in my elementary schools?"
- Principal asks: "Which teachers have the highest student growth this quarter?"
- Teacher asks: "Show me my students who are below grade level in reading"

**Implementation**:
```yaml
name: waggle_student_performance
tables:
  - name: mart_student_proficiency
    base_table: WAGGLE_DB.MARTS.STUDENT_PROFICIENCY
    dimensions:
      - name: district_name
      - name: school_name
      - name: grade_level
      - name: subject
      - name: teacher_name
    measures:
      - name: proficiency_score
        expr: AVG(score)
      - name: student_count
        expr: COUNT(DISTINCT student_id)
      - name: at_risk_count
        expr: COUNT_IF(risk_flag = 'HIGH')
    time_dimensions:
      - name: assessment_date
    filters:
      - name: current_year
        expr: school_year = '2025-2026'
```

### Cortex Analyst

**What it does**: Natural language interface over Semantic Views. Users ask questions in English, get SQL + results.

**HMH use cases**: All the "ask a question" scenarios above. Can be embedded in:
- Streamlit apps for admins
- API endpoint for in-app experiences
- Slack/Teams bot for quick queries

**Latency**: 2-5 seconds for query generation + execution.

---

## 3. Agentic Workflows

### Cortex Agents

**What it does**: Multi-step AI reasoning with tool use. Agent plans, executes tools (SQL, search, APIs), reasons over results, takes action.

**HMH use cases**:

| Agent | Trigger | Steps | Action |
|-------|---------|-------|--------|
| **Student Risk Monitor** | Weekly scheduled | Query risk scores → Identify newly at-risk students → Generate intervention summary | Email teacher with student list + recommendations |
| **Data Pipeline Monitor** | Fivetran sync failure | Detect failure → Query error logs → Diagnose root cause → Attempt auto-fix or escalate | Alert on-call + auto-retry if safe |
| **Report Generator** | Admin request | Query metrics → Generate narrative → Create chart → Format report | Deliver PDF/email |
| **Standards Alignment** | New content uploaded | Analyze content → Match to state standards → Suggest tagging | Tag in metadata system |

**Implementation**:
```sql
CREATE CORTEX AGENT waggle_risk_monitor
  TOOLS = (waggle_analyst_tool, email_notification_tool)
  INSTRUCTIONS = 'You are a student performance monitoring agent...';
```

---

## 4. Predictive Analytics & ML

### Cortex ML Functions

| Function | HMH Use Case | Input | Output |
|----------|-------------|-------|--------|
| **FORECAST** | Predict end-of-year scores | Historical assessment scores (time series) | Projected score trajectory |
| **ANOMALY_DETECTION** | Flag unusual student behavior | Activity patterns (logins, time-on-task) | Anomaly flag + score |
| **CLASSIFICATION** | Student risk categorization | Multi-feature input (scores, attendance, engagement) | Risk tier (High/Medium/Low) |
| **CONTRIBUTION_EXPLORER** | Understand score drivers | Performance data + features | Feature importance rankings |

### Custom ML (Snowpark ML)

For recommendation engines and adaptive learning:

```python
# Example: Collaborative filtering for content recommendations
from snowflake.ml.modeling.preprocessing import StandardScaler
from snowflake.ml.modeling.metrics import precision_recall_fscore_support

# Feature engineering in Snowpark
student_features = session.table("MARTS.STUDENT_ACTIVITY") \
    .group_by("student_id") \
    .agg(
        F.avg("time_on_task").alias("avg_time"),
        F.count("assignment_id").alias("assignments_completed"),
        F.avg("score").alias("avg_score")
    )
```

**Latency**:
- Batch scoring (nightly): Use Snowpark + Tasks
- Real-time scoring: Use Snowflake ML model registry + UDF for single-row inference (<100ms)

---

## 5. In-App AI (Student/Teacher Facing)

### Snowflake Postgres (for OLTP-speed queries)

**What it does**: PostgreSQL-compatible interface on Snowflake. Sub-10ms queries for app-serving patterns.

**HMH use cases**:
- Waggle UI reads student profiles, class rosters, assignment lists
- Low-latency lookups for real-time AI features (student context retrieval)
- Replace direct Aurora MySQL reads for analytics-enriched data

**When to use**: When app layer needs <50ms response time for point lookups. Not for analytical queries.

### Container Services (SPCS)

**What it does**: Run custom containers inside Snowflake (GPU-enabled). For custom models, fine-tuned LLMs, or complex serving.

**HMH use cases**:
- Host a fine-tuned education-specific LLM
- Run custom IRT (Item Response Theory) models for adaptive assessment
- Serve custom recommendation models with GPU inference

**When to use**: When Cortex built-in models aren't sufficient and you need custom model serving with data gravity.

### Streamlit in Snowflake

**What it does**: Build interactive data apps directly in Snowflake. Python + UI, no infrastructure.

**HMH use cases**:
- Admin dashboard with interactive drill-downs
- Teacher data exploration tool
- AI chatbot interface (backed by Cortex Agent)
- Migration monitoring dashboard (during this project)

---

## 6. Ontology & Knowledge Graph

### Recommended Ontology for Education Domain

Define entity relationships that power AI reasoning:

```
ENTITY: District
  HAS_MANY: Schools
  
ENTITY: School
  BELONGS_TO: District
  HAS_MANY: Classes

ENTITY: Class
  BELONGS_TO: School
  TAUGHT_BY: Teacher
  HAS_MANY: Sections

ENTITY: Section
  BELONGS_TO: Class
  HAS_MANY: Students (enrolled)

ENTITY: Student
  ENROLLED_IN: Section(s)
  COMPLETES: Assignment(s)
  TAKES: Assessment(s)

ENTITY: Assignment
  BELONGS_TO: Class
  ASSESSES: Standard(s)
  COMPLETED_BY: Student(s)

ENTITY: Assessment
  PRODUCES: Score
  MEASURES: Standard(s)

ENTITY: Standard
  BELONGS_TO: Subject + Grade Level
  ASSESSED_BY: Assignment(s), Assessment(s)
```

### Implementation in Snowflake

1. **Semantic Views**: Encode relationships as join paths and dimensional hierarchies
2. **Business Glossary** (Horizon Context, coming H2 2026): Formal ontology definitions
3. **Cortex Sense**: Auto-indexes ontology for agent routing
4. **Knowledge Graph tables**: `KG_NODE` + `KG_EDGE` pattern for complex traversal queries

### Why Ontology Matters for HMH

- **Disambiguation**: "my students" depends on who's asking (teacher → their sections, principal → their school)
- **Multi-hop reasoning**: "Which standards are my struggling students weakest in?" requires Student → Assessment → Score → Standard traversal
- **Agent routing**: Agent needs to know whether to query student table, class table, or assessment table based on the question
- **Metric consistency**: "proficiency" must be calculated the same way regardless of which tool/report uses it

---

## Feature Readiness Matrix

| Feature | Status | Edition Required | Notes |
|---------|--------|------------------|-------|
| Cortex Complete | GA | Standard+ | Multiple models available |
| Cortex Search | GA | Enterprise+ | RAG-ready |
| Cortex Analyst | GA | Enterprise+ | Requires Semantic Views |
| Semantic Views | GA | Standard+ | YAML definition |
| Cortex Agents | GA | Enterprise+ | Tool use, multi-step |
| Cortex ML (Forecast) | GA | Enterprise+ | Time series |
| Cortex ML (Anomaly) | GA | Enterprise+ | Unsupervised |
| Cortex ML (Classification) | GA | Enterprise+ | Supervised |
| Snowflake Postgres | PrPr | TBD | Apply via account team |
| Container Services | GA | Enterprise+ | GPU available |
| Streamlit in Snowflake | GA | Standard+ | No extra cost |
| Dynamic Tables | GA | Standard+ | Auto-refresh marts |
| Data Quality Monitoring | GA | Enterprise+ | Built-in DMFs |
| Cortex Code (CoCo) | GA | Standard+ | Already adopted by HMH |

**Recommendation**: Enterprise Edition minimum for AI platform. Business Critical if FERPA/COPPA compliance requires Tri-Secret Secure or HIPAA BAA.
