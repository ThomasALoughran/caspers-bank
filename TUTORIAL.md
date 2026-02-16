# Databricks Learning Tutorial: Casper's Kitchens for Banking Engineers

Welcome! This tutorial uses the **Casper's Kitchens** codebase as a hands-on learning platform for Databricks. While the demo models a ghost kitchen food-delivery business, the architecture patterns directly translate to banking use cases. Each exercise includes a "Banking Parallel" section showing how the same concept applies to financial services.

---

## Part 0: Codebase Orientation

### What is Casper's Kitchens?

A fully Databricks-native ghost kitchen and food-delivery platform that demonstrates the entire Databricks stack:

- **Data Ingestion** - Streaming events from a canonical dataset into Delta Lake
- **Data Pipelines** - Medallion architecture (Bronze/Silver/Gold) via Spark Declarative Pipelines
- **AI Agents** - LLM-powered refund and complaint triage agents
- **Reverse ETL** - Syncing analytical data to PostgreSQL (Lakebase)
- **Web Apps** - Operational UI for refund management

### Architecture at a Glance

```
                     databricks.yml (Infrastructure-as-Code)
                            |
                  "Casper's Initializer" Job
                     /      |        \
                Canonical   Lakeflow   Agent Stages...
                  Data      Pipeline
                    |          |
               events →  Bronze → Silver → Gold
              (Volume)    (raw)   (clean)  (aggregated)
```

### Key Directories

| Directory | Purpose | Banking Equivalent |
|-----------|---------|-------------------|
| `stages/` | Thin orchestration notebooks | Job step definitions |
| `data/canonical/` | Pre-generated event dataset + streaming replay | Transaction feed simulator |
| `pipelines/` | Spark Declarative Pipeline transformations | ETL/ELT pipeline logic |
| `jobs/` | Streaming job implementations | Real-time processing jobs |
| `apps/` | Web application (refund manager UI) | Internal operations dashboard |
| `utils/uc_state/` | Resource state tracking for cleanup | Infrastructure state management |

### Key Files to Read First

1. **`databricks.yml`** - The entire deployment configuration. Read this first.
2. **`claude.md`** - Architecture decisions and patterns.
3. **`data/canonical/README.md`** - How the streaming data source works.
4. **`pipelines/order_items/transformations/transformation.py`** - The medallion pipeline.

---

## Part 1: Understanding the Data (Exercises 1-3)

### Exercise 1: Explore the Event Schema

**Goal:** Understand the raw event data that flows through the system.

**Tasks:**
1. Open `data/canonical/README.md` and read the "Event Schema" and "Event Types" sections.
2. Open one of the parquet files conceptually by reading the schema definitions in `pipelines/order_items/transformations/transformation.py` (lines 30-46).
3. Answer these questions:
   - How many event types exist per order lifecycle?
   - What is the `sequence` field used for?
   - Why is `body` stored as a JSON string rather than a nested struct?
   - What does `location_id` represent?

**Banking Parallel:**
In banking, a similar event stream might look like:
| Kitchen Event | Banking Event |
|---------------|---------------|
| `order_created` | `transaction_initiated` |
| `gk_started` | `transaction_processing` |
| `gk_finished` | `compliance_check_passed` |
| `delivered` | `transaction_settled` |
| `driver_ping` | `status_update` / `audit_log_entry` |

Banking events track the lifecycle of transactions, loan applications, or KYC processes - same pattern, different domain.

---

### Exercise 2: Trace the Data Flow

**Goal:** Understand how data moves from raw events to analytics-ready tables.

**Tasks:**
1. Read `stages/canonical_data.ipynb` - this is the stage that bootstraps data.
2. Read `pipelines/order_items/transformations/transformation.py` - this is the medallion pipeline.
3. Draw a diagram (on paper or a tool) showing:
   - Where raw events land (Volume)
   - How Bronze ingests them (`all_events` function, line 14-25)
   - How Silver cleans and explodes them (`silver_order_items`, line 48-77)
   - How Gold aggregates them (4 gold tables, lines 82-157)
4. For each Gold table, write down what business question it answers:
   - `gold_order_header` - ?
   - `gold_item_sales_day` - ?
   - `gold_brand_sales_day` - ?
   - `gold_location_sales_hourly` - ?

**Banking Parallel:**
The medallion architecture is the standard for banking data platforms:

| Layer | Kitchen Use | Banking Use |
|-------|-------------|-------------|
| Bronze | Raw JSON events as-is | Raw transaction feeds, wire transfers, card swipes |
| Silver | Cleaned, exploded items with prices | Validated transactions with enriched metadata (merchant category, currency conversion) |
| Gold | Aggregated sales metrics | Risk aggregations, daily P&L, regulatory reports (e.g., LCR, CCAR) |

**Best Practice:** Banks must maintain Bronze data immutably for audit and regulatory purposes. Never transform in place at Bronze - always create new layers.

---

### Exercise 3: Understand Streaming Replay

**Goal:** Learn how the custom PySpark streaming data source works.

**Tasks:**
1. Read `data/canonical/caspers_data_source.py` (or the notebook version `caspers_streaming_notebook.py`).
2. Answer:
   - What does `SPEED_MULTIPLIER` control? What happens at `60.0` vs `3600.0`?
   - How does checkpoint-based state work? What happens if the job crashes and restarts?
   - Why is `trigger(availableNow=True)` used instead of continuous streaming?
3. Explain in your own words: Why is a pre-generated dataset + streaming replay better than a live generator for demos?

**Banking Parallel:**
Banks use similar replay patterns for:
- **Disaster recovery testing** - Replay a day's transactions to verify failover systems
- **Model backtesting** - Replay historical market data through updated risk models
- **Regulatory stress testing** - Replay crisis-period data (e.g., March 2020) to validate capital adequacy

**Best Practice:** Always design streaming pipelines with checkpoint-based recovery. In banking, losing or double-counting transactions is unacceptable. Exactly-once semantics matter.

---

## Part 2: Infrastructure & Deployment (Exercises 4-6)

### Exercise 4: Read and Understand databricks.yml

**Goal:** Understand Databricks Asset Bundles (DABs) configuration.

**Tasks:**
1. Open `databricks.yml` and identify:
   - How many targets are defined? What are they?
   - What parameters does the `default` target expose?
   - What is the task dependency graph for the `default` target? (Draw it)
   - Which stages are shared across multiple targets?
2. Compare the `free` target to the `default` target:
   - Which tasks are missing from `free`?
   - Why does `free` set `PIPELINE_SCHEDULE_MINUTES: "3"` while `default` uses `"0"`?
   - What does `"0"` mean for pipeline scheduling? (Hint: continuous mode)
3. Find the `sync` section. What files are excluded from deployment and why?

**Banking Parallel:**
Banks typically have multiple deployment targets too:

| Casper's Target | Banking Equivalent |
|----------------|-------------------|
| `free` | Development / sandbox environment |
| `default` | UAT (User Acceptance Testing) |
| `all` | Production (full feature set) |

**Best Practice:** Use Infrastructure-as-Code (like DABs) for all Databricks deployments in banking. Manual UI configurations create audit gaps and make disaster recovery impossible. Every resource should be traceable to a versioned configuration file.

---

### Exercise 5: Understand the Stage Pattern

**Goal:** Learn the thin orchestration layer pattern.

**Tasks:**
1. Read any two stage notebooks from `stages/` (e.g., `canonical_data.ipynb` and `lakeflow.ipynb`).
2. For each stage, identify:
   - What parameters does it accept? (Look for `dbutils.widgets.get()`)
   - What external resources does it create? (Look for API calls)
   - How does it register resources with `uc_state`? (Look for `state.add()`)
   - What actual implementation does it call? (Look for `%run` or `dbutils.notebook.run()`)
3. Explain in your own words: Why should stages be "thin"? Why not put business logic directly in the stage?

**Banking Parallel:**
This separation of orchestration from implementation is critical in banking:
- **Orchestration layer** = Job definitions that compliance can audit
- **Implementation layer** = Business logic that engineers can test independently
- **State management** = Resource tracking for regulatory reporting ("What systems processed this data?")

**Best Practice:** In banking, orchestration layers should be auditable and unchanging between environments. The same stage definition should work in dev, UAT, and prod - only parameters change.

---

### Exercise 6: Explore Unity Catalog State Management

**Goal:** Understand resource lifecycle tracking.

**Tasks:**
1. Read `utils/uc_state/README.md` and `utils/uc_state/state_manager.py` (first 100 lines).
2. Answer:
   - What problem does `uc_state` solve that `databricks bundle destroy` cannot?
   - What types of resources can it track?
   - Where is state stored? (Hint: it's a Delta table)
   - What is the cleanup workflow? (Two commands)
3. Think about: What happens if a stage creates a resource but crashes before registering it with `uc_state`?

**Banking Parallel:**
Resource lifecycle management is critical in banking for:
- **Cost governance** - Orphaned compute resources cost money
- **Security compliance** - Untracked endpoints are security risks
- **Regulatory audits** - "Show me every system that has access to customer data"

**Best Practice:** Always track every resource programmatically. Manual resource management leads to "shadow IT" - untracked infrastructure that violates compliance policies. Consider implementing a similar state management pattern for your bank's Databricks resources.

---

## Part 3: Data Pipelines Deep Dive (Exercises 7-9)

### Exercise 7: Analyze the Medallion Pipeline

**Goal:** Deeply understand the Bronze-Silver-Gold transformation.

**Tasks:**
1. Open `pipelines/order_items/transformations/transformation.py`.
2. For the **Bronze** layer (`all_events`, lines 14-25):
   - What format are files read in? (`cloudFiles` with JSON)
   - What does `cloudFiles` provide that regular file reading doesn't?
   - Where does the path `/Volumes/{CATALOG}/{SCHEMA}/{VOLUME}` point to?
3. For the **Silver** layer (`silver_order_items`, lines 48-77):
   - What filtering happens? (Only `order_created` events)
   - What does `F.explode("body_obj.items")` do? Why is this important?
   - What is `extended_price` and how is it calculated?
   - Why is the table partitioned by `order_day`?
4. For the **Gold** layer (lines 82-157):
   - What does `F.approx_count_distinct` do and why use it instead of `F.countDistinct`?
   - What does `withWatermark("order_ts", "3 hours")` mean for streaming?
   - Why do some gold tables use watermarks but `gold_order_header` doesn't?

**Banking Parallel:**
A banking medallion pipeline might look like:

```
Bronze: Raw card transactions (JSON from payment processor)
    → Silver: Validated transactions (parsed, deduplicated, enriched with merchant info)
        → Gold: Daily merchant category spend per customer
        → Gold: Hourly fraud risk scores by region
        → Gold: Monthly regulatory reporting aggregates
```

**Best Practice:** In banking, Silver-layer transformations should include:
- **Deduplication** - Payment processors can send duplicate messages
- **Currency normalization** - Convert all amounts to base currency
- **PII tokenization** - Replace account numbers with tokens at Silver, never expose raw PII in Gold

---

### Exercise 8: Add a New Gold Table (Hands-On)

**Goal:** Practice adding a new analytical aggregation.

**Task:** Add a new Gold table called `gold_location_brand_day` that answers: "How much revenue did each brand generate at each location per day?"

**Steps:**
1. Open `pipelines/order_items/transformations/transformation.py`.
2. Study the existing Gold tables for patterns.
3. Add a new function after the existing Gold tables:

```python
@dlt.table(
    name           = "gold_location_brand_day",
    partition_cols = ["day"],
    comment        = "Gold - brand revenue and order count per location per day."
)
def gold_location_brand_day():
    return (
        dlt.read_stream("silver_order_items")
           .withWatermark("order_ts", "3 hours")
           .groupBy(
               "location_id", "brand_id",
               F.col("order_day").alias("day")
           )
           .agg(
               F.approx_count_distinct("order_id").alias("orders"),
               F.sum("qty").alias("items_sold"),
               F.sum("extended_price").alias("revenue")
           )
    )
```

4. Before saving, answer: Why do we use `withWatermark` and `approx_count_distinct` here?

**Banking Parallel:**
This is equivalent to adding a new regulatory reporting aggregate:
- "What is the total loan exposure per product type per branch per day?"
- "What is the daily transaction volume per merchant category per region?"

**Best Practice:** When adding Gold tables in banking:
- Always include a watermark for streaming aggregations (prevents unbounded state)
- Partition by date for efficient regulatory queries ("Show me data for Q4 2024")
- Use approximate functions where exact counts aren't required (saves compute cost)

---

### Exercise 9: Understand Data Quality

**Goal:** Learn how to add data quality checks to pipelines.

**Tasks:**
1. Review the Silver table definition. Notice it trusts the incoming data format.
2. Research Databricks Expectations (DLT data quality constraints). Look at how `@dlt.expect` and `@dlt.expect_or_drop` work.
3. Design (on paper) quality checks for the Silver table:
   - `valid_order_id`: `order_id IS NOT NULL`
   - `valid_price`: `price > 0`
   - `valid_quantity`: `qty > 0 AND qty < 100`
   - `valid_location`: `location_id IN (1, 2, 3, 4)`
4. Which checks should use `expect` (warn only) vs `expect_or_drop` (discard bad rows) vs `expect_or_fail` (halt pipeline)?

**Banking Parallel:**
Data quality is non-negotiable in banking:

| Quality Rule | Kitchen Version | Banking Version |
|-------------|----------------|-----------------|
| Null check | `order_id IS NOT NULL` | `account_number IS NOT NULL` |
| Range check | `price > 0` | `transaction_amount > 0 AND transaction_amount < daily_limit` |
| Referential | `location_id IN (1,2,3,4)` | `branch_id EXISTS IN branches_table` |
| Timeliness | `ts < current_timestamp()` | `settlement_date <= T+2` |

**Best Practice:** In banking, use `expect_or_fail` for critical compliance checks (e.g., missing account numbers) and `expect_or_drop` for non-critical data issues (e.g., malformed optional fields). Always log quality metrics - regulators will ask for them.

---

## Part 4: AI Agents & ML (Exercises 10-11)

### Exercise 10: Understand the Refund Agent

**Goal:** Learn how LLM agents integrate with Databricks.

**Tasks:**
1. Read `stages/refunder_agent.ipynb` to understand how the agent is built.
2. Identify:
   - What LLM model is used? (Look for the `LLM_MODEL` parameter)
   - What tools (UC functions) does the agent have access to?
   - How is the agent deployed? (MLflow + Model Serving)
   - What structured output does it produce? (refund_usd, refund_class, reason)
3. Read `stages/refunder_stream.ipynb` to see how agent decisions are applied at scale via streaming.

**Banking Parallel:**
Banks are deploying similar AI agents for:
- **Fraud detection triage** - LLM analyzes transaction patterns and explains suspicious activity
- **Credit decisioning** - Agent reviews application data and provides recommendation with reasoning
- **Compliance screening** - Agent checks transactions against sanctions lists and explains matches

**Best Practice:** Banking AI agents must:
- **Log all decisions** with full reasoning (MLflow tracking) for regulatory explainability
- **Use structured outputs** (not free text) for downstream automation
- **Have human oversight** - agents recommend, humans approve (especially for amounts above thresholds)
- **Be version-controlled** - every model version must be reproducible for audit

---

### Exercise 11: Design a Banking Agent (Thought Exercise)

**Goal:** Apply the agent pattern to a banking use case.

**Task:** Design (on paper) a **Suspicious Transaction Alert Agent** using the same architecture as the refund agent.

1. Define the agent's **tools** (UC functions it can call):
   - `get_transaction_details(txn_id)` - Returns transaction amount, merchant, timestamp
   - `get_customer_profile(customer_id)` - Returns typical spending patterns
   - `get_merchant_risk_score(merchant_id)` - Returns merchant risk rating
   - What other tools would be useful?

2. Define the agent's **decision output**:
   ```json
   {
     "alert_level": "low | medium | high | critical",
     "recommended_action": "approve | hold | block | escalate",
     "confidence": 0.85,
     "reason": "Transaction of $5,000 at overseas merchant is 3x customer's typical spend..."
   }
   ```

3. Define the **decision logic** (similar to the refund agent's percentile-based rules):
   - If transaction < P50 of customer history: `approve`
   - If P50 < transaction < P95: `hold` for review
   - If transaction > P99: `block` and `escalate`

4. How would you stream these decisions at scale? (Hint: look at `refunder_stream.ipynb`)

---

## Part 5: Governance & Security (Exercises 12-13)

### Exercise 12: Unity Catalog for Data Governance

**Goal:** Understand how Unity Catalog provides governance.

**Tasks:**
1. In `databricks.yml`, find every reference to `CATALOG`. Trace how the catalog name flows through the system.
2. In the pipeline transformation, find where `CATALOG` is used to construct the Volume path.
3. Answer:
   - How does Unity Catalog enable multi-tenant isolation? (Each deployment gets its own catalog)
   - What happens if two teams deploy with the same `CATALOG` name?
   - How would you implement environment separation (dev/staging/prod)?

**Banking Parallel:**
Unity Catalog is essential for banking governance:
- **Data lineage** - "Where did this number in the regulatory report come from?" Trace from Gold back to Bronze.
- **Access control** - Fine-grained permissions on catalogs, schemas, and tables.
- **Audit logging** - Every data access is logged for compliance.
- **Data classification** - Tag columns as PII, PHI, or confidential.

**Best Practice:** In banking:
- Use separate catalogs per environment: `bank_dev`, `bank_uat`, `bank_prod`
- Use separate schemas per business domain: `retail_banking`, `wealth_management`, `compliance`
- Apply column-level access control on PII fields
- Enable audit logging on all production catalogs

---

### Exercise 13: Security Review Checklist

**Goal:** Develop a security mindset for Databricks deployments.

**Task:** Review the codebase with a security lens. For each item, check if the codebase handles it well and note what a bank would need to add:

1. **Secrets Management**
   - Are any credentials hardcoded? (Check all notebooks and config files)
   - Where should secrets be stored? (Databricks Secret Scopes)

2. **Network Security**
   - Does the app expose any public endpoints? (Check `apps/refund-manager/`)
   - How would you secure API endpoints in banking?

3. **Data Encryption**
   - Is data encrypted at rest? (Delta Lake on cloud storage)
   - Is data encrypted in transit? (HTTPS for API calls)

4. **Access Control**
   - Who can run the initializer job?
   - Who can read the Gold tables?
   - How would you implement least-privilege access?

5. **Audit Trail**
   - Can you trace every data transformation?
   - Can you prove what data existed at a specific point in time? (Delta Lake time travel)

**Banking Best Practice:** A banking Databricks deployment must additionally:
- Enable **Private Link** (no public internet access to workspace)
- Use **Customer-Managed Keys (CMK)** for encryption
- Implement **IP access lists** for workspace access
- Enable **enhanced security monitoring** for all clusters
- Configure **data exfiltration protection** to prevent unauthorized data download

---

## Part 6: Putting It All Together (Exercises 14-15)

### Exercise 14: End-to-End Feature Implementation

**Goal:** Practice the full development workflow.

**Task:** Add a new feature: **"Peak Hours Analysis"** - a Gold table that identifies the busiest hours for each location.

**Steps:**
1. **Understand the requirement:**
   - New Gold table: `gold_location_peak_hours`
   - Columns: `location_id`, `hour_of_day` (0-23), `day_of_week` (1-7), `avg_orders`, `avg_revenue`
   - Purpose: Help kitchen managers staff appropriately

2. **Plan (before coding):**
   - Which pipeline file needs modification?
   - What Silver table does this read from?
   - What aggregation functions will you use?
   - Should this use a watermark?

3. **Implement:**
   - Add the Gold table definition to `pipelines/order_items/transformations/transformation.py`
   - Follow the existing patterns exactly

4. **Validate your thinking:**
   - Would this table work with streaming data?
   - Is the partitioning strategy correct?
   - Would this work across all 4 deployment targets?

**Banking Parallel:**
This is equivalent to building a "Peak Transaction Volume Analysis" for capacity planning - when do ATMs need more cash? When should fraud monitoring be at highest alert?

---

### Exercise 15: Design a Banking Data Platform

**Goal:** Synthesize everything you've learned into a banking architecture.

**Task:** Using the patterns from Casper's Kitchens, design a **Retail Banking Transaction Platform** on paper:

1. **Data Sources** (like the canonical dataset):
   - Card transactions (real-time stream)
   - Wire transfers (batch, daily)
   - ATM withdrawals (real-time)
   - Customer profile updates (CDC - change data capture)

2. **Medallion Pipeline** (like `transformation.py`):
   - **Bronze:** Raw transaction events
   - **Silver:** Validated, enriched transactions (add merchant info, currency conversion, deduplication)
   - **Gold Tables:**
     - `gold_daily_transaction_summary` (per customer per day)
     - `gold_merchant_category_spend` (per category per month)
     - `gold_branch_performance` (per branch per day)
     - `gold_suspicious_activity` (flagged transactions)

3. **AI Agents** (like the refund agent):
   - Fraud detection agent
   - Credit limit adjustment agent

4. **Operational Layer** (like Lakebase + Apps):
   - Transaction monitoring dashboard
   - Fraud alert management UI

5. **Governance** (like Unity Catalog + uc_state):
   - Catalog structure for dev/uat/prod
   - Access control matrix (who sees what)
   - Data retention policies (7-year regulatory requirement)

---

## Quick Reference: Databricks Concepts You'll Learn

| Concept | Where It's Used | What It Does |
|---------|----------------|--------------|
| **Databricks Asset Bundles (DABs)** | `databricks.yml` | Infrastructure-as-code deployment |
| **Unity Catalog** | Throughout (CATALOG parameter) | Data governance, access control, lineage |
| **Spark Declarative Pipelines** | `pipelines/` | Streaming ETL with quality constraints |
| **Medallion Architecture** | `transformation.py` | Bronze/Silver/Gold data layering |
| **Structured Streaming** | `data/canonical/` | Real-time data processing |
| **Delta Lake** | All tables | ACID transactions, time travel, versioning |
| **MLflow** | Agent stages | Model lifecycle management |
| **Model Serving** | Agent endpoints | Real-time ML inference |
| **Volumes** | Data ingestion | Managed file storage in Unity Catalog |
| **Lakebase** | `stages/lakebase.ipynb` | PostgreSQL for operational workloads |
| **Databricks Apps** | `apps/` | Custom web applications |
| **Jobs & Workflows** | `databricks.yml` tasks | Orchestration and scheduling |

---

## Suggested Learning Path

| Week | Focus | Exercises |
|------|-------|-----------|
| 1 | Data & Schema Understanding | 1, 2, 3 |
| 2 | Infrastructure & Deployment | 4, 5, 6 |
| 3 | Pipeline Deep Dive | 7, 8, 9 |
| 4 | AI & ML Integration | 10, 11 |
| 5 | Governance & Security | 12, 13 |
| 6 | Synthesis & Design | 14, 15 |

---

## Additional Resources

- [Databricks Documentation](https://docs.databricks.com/)
- [Delta Lake Documentation](https://docs.delta.io/)
- [Spark Structured Streaming Guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [Unity Catalog Best Practices](https://docs.databricks.com/en/data-governance/unity-catalog/best-practices.html)
- [Databricks Asset Bundles](https://docs.databricks.com/en/dev-tools/bundles/index.html)
- [Medallion Architecture](https://docs.databricks.com/en/lakehouse/medallion.html)
