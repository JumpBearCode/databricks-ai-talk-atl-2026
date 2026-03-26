# Databricks AI Talk ATL 2026 - Lakehouse & Lakebase Deep Dive

> Study notes and extended discussion based on a Databricks talk

---

## Table of Contents

1. [What is Lakebase? How Does It Differ from Lakehouse?](#1-what-is-lakebase-how-does-it-differ-from-lakehouse)
2. [Why is the Lakehouse OLAP?](#2-why-is-the-lakehouse-olap)
3. [Open Table Formats](#3-open-table-formats)
4. [Delta Lake vs Apache Iceberg](#4-delta-lake-vs-apache-iceberg)
5. [Multi-Platform Consumption of Iceberg Tables](#5-multi-platform-consumption-of-iceberg-tables)
6. [The Catalog Metadata Management Problem](#6-the-catalog-metadata-management-problem)
7. [Unity Catalog vs OneLake vs Snowflake Catalog](#7-unity-catalog-vs-onelake-vs-snowflake-catalog)
8. [External Table OLAP Performance](#8-external-table-olap-performance)
9. [Key Takeaways from the Talk: Why Lakebase?](#9-key-takeaways-from-the-talk-why-lakebase)
10. [Bidirectional Sync Between Lakehouse and Lakebase](#10-bidirectional-sync-between-lakehouse-and-lakebase)
11. [CDC (Change Data Capture)](#11-cdc-change-data-capture)

---

## 1. What is Lakebase? How Does It Differ from Lakehouse?

### Lakebase Overview

Databricks Lakebase is a **fully managed, serverless PostgreSQL database** built directly into the Databricks platform. It is an **OLTP (Online Transaction Processing) database** designed for production-grade applications requiring high-frequency reads and writes — supporting 10K+ queries per second, sub-10ms latency, and full ACID transaction support.

**In simple terms: Lakehouse is for analytics, Lakebase is for transactions/operations.**

### Core Comparison

| Dimension | Lakehouse | Lakebase |
|-----------|-----------|----------|
| **Workload Type** | OLAP — BI reports, ML training, ad-hoc queries | OLTP — real-time single-row reads/writes, app state management |
| **Latency** | Seconds to minutes (optimized for large-scale scans) | Sub-10ms (optimized for real-time apps) |
| **Typical Use Cases** | Dashboards, data reporting, machine learning | Shopping carts, inventory, user account updates, AI Agent state |
| **Engine** | Databricks SQL Warehouse (Photon) | Fully managed serverless PostgreSQL |
| **Data Format** | Delta Lake (open columnar format) | PostgreSQL row store + automatic sync with Delta Lake |

### Why Build Lakebase When Lakehouse Already Exists?

1. **Lakehouse unified analytics and AI**, but day-to-day **transactional operations** (purchases, account updates, sensor data writes) still required external databases
2. Syncing data between external OLTP databases and Lakehouse required complex ETL pipelines — high latency, high maintenance cost
3. The AI Agent era demands **real-time operational data read/write** alongside access to analytical results and ML models

**Lakebase's core value**: Embed OLTP capabilities directly into the Lakehouse platform, eliminating the need for external databases.

```
Traditional:   External OLTP DB  ←ETL→  Data Lake  ←ETL→  Data Warehouse
Lakehouse:                        Unified Analytics + AI (but OLTP still external)
Lakebase:      OLTP + OLAP + AI all unified on one platform with bidirectional sync
```

---

## 2. Why is the Lakehouse OLAP?

### Columnar Storage Defines Its Role

The Lakehouse is built on **Delta Lake**, with data stored in **Parquet** columnar format. Columnar storage is inherently optimized for analytical queries:

```
Query: SELECT AVG(revenue) FROM sales WHERE region = 'US'

Columnar: reads only revenue and region columns → fast
Row-based: reads all fields for every row → slow
```

### Lakehouse Cannot Efficiently Handle Single-Row Transactions

| OLTP Requirement | Lakehouse Performance |
|-----------------|----------------------|
| Single-row INSERT/UPDATE at millisecond latency | Delta Lake writes are batch-oriented (file writes), latency in seconds to minutes |
| 10K+ small transactions per second | SQL Warehouse is not designed for high-concurrency point lookups |
| Row-level locking / fine-grained concurrency | Delta Lake uses file-level optimistic locking |
| Low-latency point lookups (by primary key) | Columnar + file scanning is far less efficient than B-Tree indexes |

### Intuitive Analogy

```
Lakehouse (OLAP) = A library's analytics reporting system
  → Good at: "What were the top 100 most borrowed books last year?"
  → Not good at: "Check out this book for User A and update inventory now"

Lakebase (OLTP) = A library's checkout counter system
  → Good at: "Check out this book for User A, deduct inventory, log transaction"
  → Not good at: "Analyze borrowing trends for all users over the past year"
```

---

## 3. Open Table Formats

### The Problem: Before Open Formats

Data lakes were just raw Parquet/ORC files on object storage (S3, ADLS), with major issues:

- No transactions → partial writes lead to data inconsistency
- No update/delete → append-only, modifying one row required rewriting entire files
- No schema management → upstream schema changes break downstream consumers
- No time travel → no way to roll back bad writes
- Slow queries → engines have no idea which files contain relevant data, must scan everything

### The Solution: An Intelligent Metadata Layer on Top of Raw Files

```
┌─────────────────────────────────┐
│     Query Engines (Spark,       │  ← Any engine can read
│     Flink, Trino, Presto...)    │
├─────────────────────────────────┤
│   Open Table Format (metadata)  │  ← Delta / Iceberg / Hudi
│   • ACID transactions            │
│   • Schema evolution             │
│   • Time travel (snapshots)      │
│   • Row-level update/delete      │
│   • Data skipping                │
├─────────────────────────────────┤
│   File Format: Parquet / ORC     │  ← Actual data files
├─────────────────────────────────┤
│   Object Storage: S3/ADLS/GCS   │  ← Cheap underlying storage
└─────────────────────────────────┘
```

**What "Open" means**:
- **Open source**: Code is public, no single-vendor lock-in
- **Engine-agnostic**: Multiple compute engines can read/write the same data
- **Storage-agnostic**: Runs on S3, ADLS, GCS, or any object storage

---

## 4. Delta Lake vs Apache Iceberg

| Dimension | Delta Lake | Apache Iceberg |
|-----------|-----------|----------------|
| **Origin** | Databricks (2017) | Netflix (2017), donated to Apache Foundation |
| **Governance** | Databricks-led (Linux Foundation) | Apache Foundation, broader community |
| **Ecosystem** | Deep integration with Databricks/Spark | Engine-agnostic: Spark, Flink, Trino, Snowflake, AWS Athena, etc. |
| **Metadata** | `_delta_log/` with JSON transaction logs + Parquet checkpoints | Tree-structured manifest files |
| **Large Table Scalability** | Linear transaction log, checkpoints grow with table size | Tree-based metadata naturally scales to massive tables |
| **Partition Evolution** | Supported, but requires data rewrite | Hidden Partitioning: transparent to users, no data rewrite needed |

### Metadata Architecture Differences

```
Delta Lake: Linear log
  _delta_log/
    000000.json → 000001.json → ... → checkpoint.parquet
  Reading current state requires replaying from last checkpoint

Iceberg: Tree structure
       metadata.json
           │
     manifest list
        /     \
   manifest  manifest
    /   \      /   \
  file  file  file  file
  Tree structure enables fast file location for huge tables
```

### Hidden Partitioning (Iceberg's Killer Feature)

```
Traditional partitioning (Delta / Hive):
  Users must know partition structure to write efficient queries
  SELECT * FROM events WHERE event_date = '2025-03-26'  → full scan!

Iceberg Hidden Partitioning:
  partition by month(event_date)
  SELECT * FROM events WHERE event_date = '2025-03-26'  → automatic partition pruning!
  Partition strategy can be changed at any time without rewriting historical data
```

### How to Choose?

- **Choose Delta Lake**: Already in the Databricks ecosystem, Spark-centric team
- **Choose Iceberg**: Multi-engine environment, avoid lock-in, massive tables, need Hidden Partitioning

### Industry Trend

Iceberg is becoming the industry standard — Snowflake, AWS, Google, and Apple all deeply support it. Databricks embraces Iceberg compatibility through **UniForm** (automatically generates Iceberg metadata when writing Delta).

---

## 5. Multi-Platform Consumption of Iceberg Tables

### Core Idea: Store Once, Read from Multiple Engines

```
On-Prem DB
    │
    ▼ (ETL: Fivetran, ADF, Spark, etc.)
    │
Azure Storage Account (ADLS Gen2)
    │
    └── Iceberg Format (Parquet files + metadata)
         │
         ├── Unity Catalog → External Table ✅
         ├── Snowflake → External Table (Iceberg) ✅
         └── OneLake → Shortcut ✅
```

### Registration Examples

**Snowflake**:
```sql
CREATE ICEBERG TABLE my_table
  EXTERNAL_VOLUME = 'my_adls_vol'
  CATALOG = 'ICEBERG'
  METADATA_FILE_PATH = 'my_table/metadata/v1.metadata.json';
```

**Databricks (Unity Catalog)**:
```sql
CREATE TABLE my_catalog.my_schema.my_table
  USING ICEBERG
  LOCATION 'abfss://container@account.dfs.core.windows.net/warehouse/my_table';
```

**OneLake (Fabric)**: Create a Shortcut via UI → point to ADLS path → auto-detects Iceberg format.

### Key Considerations

- **Designate a single writer, others read-only** (concurrent multi-engine writes risk data corruption)
- **Choose a catalog management strategy**
- **Be aware of metadata refresh latency across platforms**

---

## 6. The Catalog Metadata Management Problem

### Core Issue: How Do Platforms Know When Data Is Updated?

An Iceberg catalog is essentially a pointer — telling engines "where is the current latest metadata.json."

**For static data, there's nothing to manage** — register once and every platform sees the columns, types, and data.

**The problem arises when data is continuously updated**:

```
Day 1: ETL writes → v1.metadata.json
Day 2: ETL appends → v2.metadata.json
Day 3: ETL appends → v3.metadata.json
```

### Refresh Behavior by Platform

| Platform | Auto-detects updates? | Latency | Manual Action |
|----------|----------------------|---------|---------------|
| Snowflake | ❌ No | Infinite (stays stale forever without refresh) | `ALTER TABLE REFRESH` + specify new metadata path |
| Databricks | ⚠️ Only for its own writes | Own writes: 0 / External writes: infinite | `REFRESH TABLE` |
| OneLake | ✅ Some auto-refresh | Minutes-level lag | Can manually trigger refresh |

### When Does "External Write" Happen?

- **Multi-team, multi-platform**: Team A uses Databricks Spark, Team B uses Flink
- **Cross-cloud pipelines**: AWS EMR writes Iceberg to ADLS, Azure Databricks reads
- **Third-party data providers**: Dump Iceberg-format data directly into your storage
- **Migration transition periods**: Moving from Snowflake to Databricks, both platforms run in parallel

---

## 7. Unity Catalog vs OneLake vs Snowflake Catalog

| Dimension | Unity Catalog | OneLake (Fabric) | Snowflake (Horizon + Polaris) |
|-----------|--------------|-------------------|-------------------------------|
| **Open Source** | ✅ Open-sourced | ❌ Proprietary | ⚠️ Polaris open-sourced, Horizon proprietary |
| **Native Format** | Delta Lake + UniForm | Delta Lake | Proprietary format + Iceberg support |
| **Access Control** | ABAC + row/column-level security | Fabric Workspace + Entra ID | RBAC + dynamic data masking |
| **Data Sharing** | Delta Sharing (open protocol) | Shortcuts + Fabric sharing | Secure Data Sharing (zero-copy) |
| **Unstructured Data** | ✅ Volumes | ✅ OneLake Files | ❌ |
| **AI/ML Governance** | ✅ Strongest (models, features, vector indexes) | ⚠️ Via Azure ML | ⚠️ Has ML but governance not as deep |
| **Multi-Engine Access** | ✅ Iceberg REST Catalog API | ⚠️ Mainly Fabric-internal engines | ⚠️ Mainly Snowflake engine |

### How to Choose?

- **All-in Databricks + Spark** → Unity Catalog
- **Microsoft ecosystem (Azure + Power BI + Office 365)** → OneLake (Fabric)
- **Core focus is SQL analytics + data sharing** → Snowflake
- **Multi-platform, avoid lock-in** → Unity Catalog (open source + Iceberg REST API + UniForm)

---

## 8. External Table OLAP Performance

### Why Are External Tables Slower Than Native Tables?

```
Native Table: Engine has full control over data layout, indexing, and optimization → very fast
External Table: Every query goes over the network to read external files, limited optimization → slower
```

### Approximate Performance Gap by Platform

| Platform | Native Table | External Table | Gap |
|----------|-------------|---------------|-----|
| Snowflake | 1x (baseline) | ~2-5x slower | No micro-partition optimization |
| Databricks | 1x (baseline) | ~1.1-1.5x (Iceberg) | Photon optimizes external tables too |
| OneLake | 1x (baseline) | ~1.2-3x (depends on caching) | Accelerated Shortcuts available |

### Practical Recommendation: Tiered Approach

- **Hot data** (frequent queries, high performance required) → Internalize as Native Table
- **Warm data** (occasional queries, some latency acceptable) → External Table
- **Cold data** (rarely queried, compliance retention) → External Table or unregistered

---

## 9. Key Takeaways from the Talk: Why Lakebase?

### Core Argument: Analytics vs Applications

> **Analytics**: You're only **observing** data
> **Apps**: The business can **write back / change** data

Frontline workers (managers, accountants, operators, field engineers) fundamentally **make changes to data** — they need **applications**, not dashboards.

> "With analytics, the data haunts the business. But with apps, the business can finally talk back."

### Three Pain Points of Traditional Databases

1. **Development is cumbersome**: Multiple environments each need their own database instances
2. **Expensive to manage**: DBA time spent on index tuning, capacity planning, patching — none of this is core business value
3. **Vendor lock-in**: Cloud providers keep adding proprietary hooks; once you use them, you can't migrate

### AI Agents Make It Worse

- Agent year-over-year growth is 400-500%
- Agents need high-frequency operational data read/write
- Traditional databases were not designed for this pattern

### Lakebase's Response

| Traditional DB Problem | Lakebase's Answer |
|-----------------------|-------------------|
| Cumbersome multi-env management | Serverless, no infrastructure to manage |
| High DBA operational cost | Fully managed PostgreSQL, auto-scaling |
| Vendor lock-in | Built on open-source PostgreSQL |
| Disconnected from analytics layer | Native Lakehouse integration, bidirectional sync |
| Not suitable for AI Agents | Purpose-built for agent state management |

---

## 10. Bidirectional Sync Between Lakehouse and Lakebase

### Direction A: Lakebase → Lakehouse (Lakehouse Sync)

Operational data automatically syncs to the analytics layer via built-in CDC.

```
App/Agent writes to Lakebase (PostgreSQL)
        │
        │  wal2delta extension reads WAL logs
        │  Auto-converts to Delta Table (SCD Type 2)
        ▼
Unity Catalog: lb_<table_name>_history
        │
        ▼
Can JOIN with any Delta table, GROUP BY, aggregate — full OLAP performance
```

### Direction B: Lakehouse → Lakebase (Synced Tables / Reverse ETL)

Analytical results flow back to the operational layer for low-latency app consumption.

```
Lakehouse Delta tables (ML recommendations, risk scores, etc.)
        │
        │  Synced Tables (Lakeflow Spark Pipeline)
        │  Supports Triggered or Continuous mode
        ▼
Lakebase PostgreSQL table
        │
        ▼
App / Agent reads via sub-millisecond SELECT
```

### Complete Bidirectional Architecture

```
                    ┌──────────────────────────────┐
                    │         Lakehouse             │
                    │      (Delta Lake, OLAP)        │
                    │  • JOIN / GROUP BY / aggregates │
                    │  • ML model training            │
                    │  • BI reporting                 │
                    └──────▲──────────┬──────────────┘
                           │          │
             Lakehouse Sync│          │ Synced Tables
              (CDC/WAL)    │          │ (Reverse ETL)
                           │          │
                    ┌──────┴──────────▼──────────────┐
                    │         Lakebase               │
                    │     (PostgreSQL, OLTP)          │
                    │  • App reads/writes user data    │
                    │  • Agent stores state/memory     │
                    │  • Real-time order/inventory     │
                    └────────────────────────────────┘
```

### Concrete Example: E-Commerce

1. User places order → App writes to Lakebase `orders` table **[OLTP write]**
2. Lakehouse Sync → orders auto-synced as Delta table **[CDC sync]**
3. Lakehouse analytics: regional sales trends, user behavior analysis **[OLAP analysis]**
4. ML model generates recommendations, stored as Delta table **[AI/ML]**
5. Synced Tables → recommendations synced to Lakebase **[Reverse ETL]**
6. User opens app → sub-millisecond personalized recommendations **[OLTP read]**

### Understanding "Analytics vs Apps" — Extended

```
Pure Analytics Mode (Dashboard):
  Data → Reports → Business users view → Then switch to another system to take action
  Data tells you what happened, but you can't directly act on it

Application Mode (App + Lakebase):
  See "User A credit score 720" → Click "Approve Loan" → Writes back to DB → Downstream auto-triggers
  One interface: see analytics AND take action, data flows both ways

AI Agent Mode:
  Agent detects anomaly (from Lakehouse analysis)
    → Auto-flags order in Lakebase as "pending review"
    → Notifies customer service
    → Agent handles and resolves
    → Result flows back to Lakehouse → Updates analytical models
  Complete loop: Analyze → Decide → Act → Feedback → Re-analyze
```

> **Dashboard = You look at data, data ignores you**
> **App = You interact with data, data responds to your actions**

---

## 11. CDC (Change Data Capture)

### Core Concept

CDC monitors every insert, update, and delete in a database, captures those changes in real time, and syncs them elsewhere.

Databases write a log before executing any write operation (PostgreSQL → WAL, MySQL → Binlog). CDC doesn't query the table itself — it **reads the log stream**.

### CDC vs Traditional Full-Table Queries

| | Traditional (Scheduled Full Query) | CDC (Log Stream) |
|---|---|---|
| Latency | Up to one sync cycle | Seconds or sub-seconds |
| Completeness | May miss data (DELETEs undetectable) | Logs capture all operations |
| Source DB load | Heavy (scans table each time) | Near-zero (only reads log files) |
| Change history | None | Fully preserved |

### How CDC Works in Lakebase

```
App writes to Lakebase
    │ INSERT INTO orders VALUES (1001, 'iPhone', 500)
    │ UPDATE orders SET amount = 600 WHERE id = 1001
    ▼
PostgreSQL WAL records these operations
    │
    ▼
wal2delta extension (built-in, no external tools needed)
    │
    ▼
Delta table: lb_orders_history (SCD Type 2)

┌──────┬─────────┬────────┬─────────────────────┬───────────┐
│ id   │ product │ amount │ _change_timestamp    │ _change_  │
│      │         │        │                      │ type      │
├──────┼─────────┼────────┼─────────────────────┼───────────┤
│ 1001 │ iPhone  │ 500    │ 2026-03-26 10:00:01 │ INSERT    │
│ 1001 │ iPhone  │ 600    │ 2026-03-26 10:05:32 │ UPDATE    │
└──────┴─────────┴────────┴─────────────────────┴───────────┘
```

Every change is appended as a new row, preserving full history. The Lakehouse can run OLAP analytics directly on this table.

### Common CDC Tools in the Industry

| Tool | Description |
|------|-------------|
| **Debezium** | Open source, most popular, supports PostgreSQL/MySQL/MongoDB |
| **AWS DMS** | AWS managed CDC service |
| **Fivetran** | SaaS data sync with built-in CDC |
| **Striim** | Enterprise-grade real-time CDC |
| **Lakebase built-in** | wal2delta — no external tools required |

---

## References

- [A New Era of Databases: Lakebase | Databricks Blog](https://www.databricks.com/blog/what-is-a-lakebase)
- [Databricks Lakebase is now Generally Available](https://www.databricks.com/blog/databricks-lakebase-generally-available)
- [Lakebase Product Page](https://www.databricks.com/product/lakebase)
- [Lakehouse Sync | Databricks Docs](https://docs.databricks.com/aws/en/oltp/projects/lakehouse-sync)
- [Serve Lakehouse Data with Synced Tables](https://docs.databricks.com/aws/en/oltp/instances/sync-data/sync-table)
- [Reverse ETL with Lakebase](https://www.databricks.com/blog/reverse-etl-lakebase-activate-your-lakehouse-data-operational-analytics)
- [How to use Lakebase as a transactional data layer for Databricks Apps](https://www.databricks.com/blog/how-use-lakebase-transactional-data-layer-databricks-apps)
- [Understanding Open Table Formats | Delta Lake](https://delta.io/blog/open-table-formats/)
- [Unity Catalog vs Snowflake Governance](https://www.celestinfo.com/unity-catalog-vs-snowflake-governance.html)
- [Mirroring Azure Databricks Unity Catalog in Microsoft Fabric](https://blog.fabric.microsoft.com/en-us/blog/unified-by-design-mirroring-azure-databricks-unity-catalog-in-microsoft-fabric-now-generally-available)
- [Databricks Introduces Lakebase for AI Workloads | InfoQ](https://www.infoq.com/news/2026/02/databricks-lakebase-postgresql/)
