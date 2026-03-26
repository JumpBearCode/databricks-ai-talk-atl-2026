# Databricks AI Talk ATL 2026 - Lakebase vs Lakehouse 深度解析

> 基于 Databricks 讲座内容的学习笔记与延伸讨论

---

## 目录

1. [Lakebase 是什么？为什么需要它？](#1-lakebase-是什么为什么需要它)
2. [为什么 Lakehouse 是 OLAP？](#2-为什么-lakehouse-是-olap)
3. [Lakehouse 与 Lakebase 的双向同步（CDC）](#3-lakehouse-与-lakebase-的双向同步cdc)
4. [Lakebase Branching：数据库的 Git](#4-lakebase-branching数据库的-git)

---

## 1. Lakebase 是什么？为什么需要它？

### Lakebase 概述

Databricks Lakebase 是一个**全托管、Serverless 的 PostgreSQL 数据库**，直接内置在 Databricks 平台中。它是一个 **OLTP（联机事务处理）数据库**，专门为需要高频读写的生产级应用而设计——支持每秒 10K+ 查询、亚 10 毫秒延迟、完整 ACID 事务支持。

**简单来说：Lakehouse 是做分析的，Lakebase 是做交易/操作的。**

### 核心对比

| 维度 | Lakehouse | Lakebase |
|------|-----------|----------|
| **工作负载类型** | OLAP（分析型）— BI 报表、ML 训练、ad-hoc 查询 | OLTP（事务型）— 实时读写单行数据、应用状态管理 |
| **延迟** | 秒到分钟级（适合大规模扫描聚合） | 亚 10 毫秒（适合实时应用） |
| **典型场景** | 仪表盘、数据报告、机器学习 | 购物车、库存系统、用户账户更新、AI Agent 状态管理 |
| **引擎** | Databricks SQL Warehouse (Photon) | 全托管 Serverless PostgreSQL |
| **数据格式** | Delta Lake（开放列存格式） | PostgreSQL 行存 + 与 Delta Lake 自动同步 |

### 讲座核心观点：分析 vs 应用的本质区别

> **Analytics（分析）**：你只是在**观察**数据
> **Apps（应用）**：业务可以**回写/改变**数据

一线人员（经理、会计、运维、现场工程师）的工作本质是**对数据做变更**，他们需要的是**应用**，不是仪表盘。

> "With analytics, the data haunts the business. But with apps, the business can finally talk back."

```
纯分析模式（Dashboard）：
  数据 → 报表 → 业务人员看 → 然后切换到其他系统去操作
  数据告诉你发生了什么，但你不能直接对数据做什么

应用模式（App + Lakebase）：
  看到"用户 A 信用评分 720" → 直接点击"批准贷款" → 写回数据库 → 下游自动触发
  一个界面，既看到分析，又完成操作，数据双向流动

AI Agent 模式：
  Agent 分析发现异常 → 自动在 Lakebase 中标记 → 通知客服 → 客服处理 → 结果回流分析层
  完整闭环：分析 → 决策 → 行动 → 反馈 → 再分析
```

> **Dashboard = 你看数据，数据不理你**
> **App = 你和数据互动，数据会响应你的操作**

### 为什么有了 Lakehouse 还需要 Lakebase？

1. **Lakehouse 解决了"分析"与"AI"的统一**，但日常的**事务型操作**（购买记录、账户更新、传感器数据写入）仍然需要依赖外部数据库
2. 数据在外部 OLTP 数据库和 Lakehouse 之间需要复杂的 ETL 管道来同步，延迟高、维护成本大
3. AI Agent 时代需要**实时读写操作数据**，同时又需要访问分析结果和 ML 模型

```
传统架构：  外部 OLTP DB  ←ETL→  Data Lake  ←ETL→  Data Warehouse
Lakehouse：                       统一分析 + AI（但 OLTP 仍在外部）
Lakebase：  OLTP + OLAP + AI 全部统一在一个平台，双向自动同步
```

### 传统数据库的三大痛点与 Lakebase 的回应

1. **开发繁琐**：多环境需要各自搭建独立数据库
2. **运维昂贵**：DBA 时间花在索引调优、容量规划、补丁升级——这些不是公司的核心价值
3. **厂商锁定**：云厂商不断加入专有特性，一旦用了就无法迁移

而 AI Agent 加剧了这些问题——Agent 年增长率 400-500%，需要高频读写操作数据，传统数据库不是为此设计的。

| 传统数据库的问题 | Lakebase 的回应 |
|----------------|----------------|
| 开发繁琐 / 多环境管理 | Serverless，无需管理基础设施 |
| DBA 运维成本高 | 全托管 PostgreSQL，自动扩缩 |
| 厂商锁定 | 基于开源 PostgreSQL |
| 与分析层割裂 | 原生集成 Lakehouse，双向同步 |
| 不适合 AI Agent | 专为 Agent 状态管理设计 |

---

## 2. 为什么 Lakehouse 是 OLAP？

### 列式存储决定了它的定位

Lakehouse 的底层是 **Delta Lake**，数据以 **Parquet** 列式格式存储。列式存储天然适合分析型查询：

```
查询：SELECT AVG(revenue) FROM sales WHERE region = 'US'

列式存储：只读 revenue 和 region 两列 → 快
行式存储：要读每一行的所有字段 → 慢
```

### Lakehouse 不支持高效的单行事务操作

| OLTP 需求 | Lakehouse 的表现 |
|-----------|-----------------|
| 单行 INSERT/UPDATE，毫秒级响应 | Delta Lake 的写入是批量的（写文件），延迟在秒到分钟级 |
| 每秒 10K+ 的小事务 | SQL Warehouse 不是为这种高并发点查设计的 |
| 行级锁 / 细粒度并发控制 | Delta Lake 的并发控制是文件级别的乐观锁 |
| 低延迟点查（按主键查一行） | 列存 + 文件扫描的方式，单行点查效率远不如 B-Tree 索引 |

### 直观类比

```
Lakehouse (OLAP) = 图书馆的分析报告系统
  → 擅长："过去一年借阅量最高的 100 本书是什么？"
  → 不擅长："现在帮用户 A 借出这本书，并更新库存"

Lakebase (OLTP) = 图书馆的借还书柜台系统
  → 擅长："帮用户 A 借出这本书，扣减库存，记录交易"
  → 不擅长："分析过去一年所有用户的借阅趋势"
```

---

## 3. Lakehouse 与 Lakebase 的双向同步（CDC）

### 什么是 CDC（Change Data Capture）？

CDC 就是"监听数据库的每一次增删改，实时捕获这些变更，然后同步到别的地方"。

数据库在执行写操作之前都会先写日志（PostgreSQL → WAL，MySQL → Binlog）。CDC 不去查表本身，而是**读取这个日志流**。

| | 传统方式（定时全量查询） | CDC（读日志流） |
|---|---|---|
| 延迟 | 最多等一个同步周期 | 秒级甚至亚秒级 |
| 完整性 | 可能漏数据（DELETE 捕获不到） | 日志记录了所有操作 |
| 对源库压力 | 大（每次扫表） | 几乎零（只读日志文件） |
| 变更历史 | 无 | 完整保留 |

Lakebase 内置了 **wal2delta** 扩展，无需额外部署 Debezium 等工具，CDC 开箱即用。

### 方向 A：Lakebase → Lakehouse（Lakehouse Sync）

操作数据通过内置 CDC 自动同步到分析层。

```
App/Agent 写入 Lakebase (PostgreSQL)
        │
        │  wal2delta 扩展读取 WAL 日志
        │  自动转写为 Delta Table (SCD Type 2)
        ▼
Unity Catalog: lb_<table_name>_history
        │
        ▼
可以和任何 Delta 表 JOIN、GROUP BY，享受 OLAP 性能
```

CDC 同步的具体效果：

```
App 写入 Lakebase:
    INSERT INTO orders VALUES (1001, 'iPhone', 500)
    UPDATE orders SET amount = 600 WHERE id = 1001

自动同步到 Delta 表：lb_orders_history (SCD Type 2)

┌──────┬─────────┬────────┬─────────────────────┬───────────┐
│ id   │ product │ amount │ _change_timestamp    │ _change_  │
│      │         │        │                      │ type      │
├──────┼─────────┼────────┼─────────────────────┼───────────┤
│ 1001 │ iPhone  │ 500    │ 2026-03-26 10:00:01 │ INSERT    │
│ 1001 │ iPhone  │ 600    │ 2026-03-26 10:05:32 │ UPDATE    │
└──────┴─────────┴────────┴─────────────────────┴───────────┘

每次变更追加一行，完整历史保留，Lakehouse 直接对这张表做 OLAP 分析。
```

### 方向 B：Lakehouse → Lakebase（Synced Tables / Reverse ETL）

分析结果回流到操作层，供应用低延迟读取。

```
Lakehouse Delta 表（ML 推荐结果、风控评分等）
        │
        │  Synced Tables (Lakeflow Spark Pipeline)
        │  支持 Triggered 或 Continuous 模式
        ▼
Lakebase PostgreSQL 表
        │
        ▼
App / Agent 亚毫秒级 SELECT 读取
```

### 完整闭环架构

```
                    ┌──────────────────────────┐
                    │       Lakehouse          │
                    │    (Delta Lake, OLAP)     │
                    │  • JOIN / GROUP BY / 聚合  │
                    │  • ML 模型训练             │
                    │  • BI 报表                │
                    └─────▲──────────┬──────────┘
                          │          │
            Lakehouse Sync│          │ Synced Tables
             (CDC/WAL)    │          │ (Reverse ETL)
                          │          │
                    ┌─────┴──────────▼──────────┐
                    │       Lakebase            │
                    │   (PostgreSQL, OLTP)       │
                    │  • App 读写用户数据         │
                    │  • Agent 存储状态/记忆      │
                    │  • 实时订单/库存更新         │
                    └───────────────────────────┘
```

### 具体例子：电商场景

1. 用户下单 → App 写入 Lakebase `orders` 表 **[OLTP 写入]**
2. Lakehouse Sync → orders 自动同步为 Delta 表 **[CDC 同步]**
3. Lakehouse 分析：各区域销售趋势、用户行为分析 **[OLAP 分析]**
4. ML 模型生成推荐结果，存为 Delta 表 **[AI/ML]**
5. Synced Tables → 推荐结果同步到 Lakebase **[Reverse ETL]**
6. 用户打开 App → 亚毫秒读取个性化推荐 **[OLTP 读取]**

---

## 4. Lakebase Branching：数据库的 Git

### 概念

Lakebase 的 Branching 就像 **Git for Database**——你可以在任意时间点对整个数据库创建一个分支，拥有完整数据，但**不复制任何数据**。

```
Production DB (100GB)
    │
    │  CREATE BRANCH（瞬间完成，额外存储 ≈ 0）
    │
    ├── Branch: dev-testing
    │     修改了 1GB 数据 → 只额外存储 1GB
    │     其余 99GB 和 Production 共享同一份物理数据
    │
    └── Branch: schema-migration-test
          修改了 500MB → 只额外存储 500MB
```

### 底层原理：Copy-on-Write（写时复制）

Lakebase 的 Branching 技术来自 Databricks 收购的 **Neon**——一个重新架构了 PostgreSQL 存储引擎的项目。

#### 传统 PostgreSQL vs Neon/Lakebase 架构

```
传统 PostgreSQL：
  计算 + 存储绑定在一起
  复制数据库 → 必须拷贝全部数据
  100GB 数据库 → 复制 = 200GB

Neon/Lakebase 架构：
┌─────────────────────────┐
│  Compute（计算层）       │  ← 标准 PostgreSQL 进程
└──────────┬──────────────┘
           │  "给我 page #42 在 LSN @100 时的版本"
┌──────────▼──────────────┐
│  Pageserver（页服务器）   │  ← 核心创新
│  • 不覆写旧页，追加新版本  │
│  • 通过 WAL 重建任意版本   │
└──────────┬──────────────┘
┌──────────▼──────────────┐
│  Object Storage (S3)     │  ← 所有历史版本持久化存储
└─────────────────────────┘
```

#### Branch 创建过程

```
Step 1: 用户说"在时间点 T 创建 branch"
Step 2: Pageserver 只记录一条元数据：
        "branch-dev 的起点 = 时间点 T 的 LSN"
Step 3: 完成。没有任何数据拷贝。

读操作：
  → 请求 page #42
  → branch 没改过 → 从 parent 拿 page #42 在时间点 T 的版本

写操作（Copy-on-Write）：
  → 修改 page #42
  → 只为 branch 创建 page #42 的新版本
  → 不影响 production
```

图示：

```
                    Page #1   Page #2   Page #3   Page #4
                    ─────────────────────────────────────
Production (T=100):   [A]       [B]       [C]       [D]

Branch 修改了 Page #2:

Production:           [A]       [B]       [C]       [D]
                       ↑         ↑         ↑         ↑
Branch:                │       [B']        │         │
                       │    (只存这一页)    │         │
                      共享      独立       共享      共享

额外存储 = 只有 Page #2 的新版本
```

### 与 Delta Lake 的架构类比

Lakebase Branching 和 Delta Lake 的版本管理**本质上是同一种思想**：

```
共同模式：
  1. 数据写入后不覆写（Immutable）
  2. 用日志/元数据记录"当前版本由哪些数据块组成"
  3. 不同版本/分支 = 不同的指针组合，指向同一批底层数据块
```

| | Delta Lake | Lakebase (Neon) |
|--|-----------|-----------------|
| 不可变数据单元 | Parquet file（几十 MB ~ 1GB） | Page（8KB） |
| 版本日志 | `_delta_log/` JSON 事务日志 | WAL (Write-Ahead Log) |
| 版本标识 | 版本号 (v0, v1, v2...) | LSN (Log Sequence Number) |
| 某个版本 = | 一组文件指针 | 一组页指针 |
| 历史访问 | Time Travel（只读） | Branch（可读写） |
| 清理机制 | VACUUM 清理旧文件 | GC 清理超出保留窗口的旧页 |

**关键区别**：Delta Lake 的粒度是文件级（几十 MB ~ 1GB），Lakebase 是页级（8KB）。粒度更细，所以 Copy-on-Write 的成本极低，branch 上的可写操作几乎零开销。

### 对比 Snowflake Time Travel 和 Delta Lake Time Travel

|  | Lakebase Branch | Snowflake Zero-Copy Clone | Delta Lake Time Travel |
|--|----------------|--------------------------|----------------------|
| **本质** | 可读写的完整数据库分支 | 表/库的元数据克隆 | 查询历史版本（只读） |
| **可以写入？** | ✅ 自由增删改 | ✅ clone 独立可写 | ❌ 只读 |
| **任意时间点？** | ✅ 保留窗口内任意时间点（0-30天） | ❌ 只能 clone 当前状态 | ✅ 按版本号或时间戳 |
| **隔离性** | ✅ 存储层隔离 | ✅ 写入后独立 | N/A（只读） |
| **存储开销** | 只有被修改的页（8KB 粒度） | 只有被修改的 micro-partition | 保留旧版本文件 |
| **合并回 parent？** | ❌ 目前不支持 | ❌ 不支持 | N/A |

### 实际 Use Case

#### 1. 安全的 Schema Migration 测试

```
# 从当前时间点创建 branch
CREATE BRANCH schema_test FROM main;

# 在 branch 上测试 schema 变更
ALTER TABLE orders ADD COLUMN discount DECIMAL(5,2);
UPDATE orders SET discount = 0.1 WHERE category = 'VIP';

# 跑测试，验证应用兼容性...
# 通过 → 在 production 执行同样的 migration
# 失败 → DROP BRANCH schema_test; 零影响
```

#### 2. AI Agent 的沙盒环境

```
# 创建 branch 作为 Agent 沙盒
CREATE BRANCH agent_sandbox FROM main;

# Agent 在沙盒里自由操作
UPDATE products SET price = price * 0.8;  -- 全场 8 折

# 人工 review → 满意则在 production 执行，不满意则丢弃 branch
```

#### 3. 基于历史数据点的调试

```
# "昨天下午 3 点之后数据就不对了"
CREATE BRANCH debug_branch FROM main AT '2026-03-25 15:00:00';

# 在 branch 上排查，对比当前数据和历史数据
# 找到根因后修复 production
```

### 本质总结

```
Delta Lake Time Travel  = 只读快照（坐时光机回去看历史）
Snowflake Zero-Copy Clone = 当前状态的可写副本（克隆现在）
Lakebase Branch         = 任意时间点的可写分叉（在任意历史时刻开辟平行宇宙）
```

> **Time Travel = 坐时光机回去看历史**
> **Lakebase Branch = 在任意历史时间点开辟一个平行宇宙，随便折腾，不影响主时间线**

---

## 参考资料

- [A New Era of Databases: Lakebase | Databricks Blog](https://www.databricks.com/blog/what-is-a-lakebase)
- [Databricks Lakebase is now Generally Available](https://www.databricks.com/blog/databricks-lakebase-generally-available)
- [Lakebase Product Page](https://www.databricks.com/product/lakebase)
- [Lakehouse Sync | Databricks Docs](https://docs.databricks.com/aws/en/oltp/projects/lakehouse-sync)
- [Serve Lakehouse Data with Synced Tables](https://docs.databricks.com/aws/en/oltp/instances/sync-data/sync-table)
- [Reverse ETL with Lakebase](https://www.databricks.com/blog/reverse-etl-lakebase-activate-your-lakehouse-data-operational-analytics)
- [How to use Lakebase as a transactional data layer for Databricks Apps](https://www.databricks.com/blog/how-use-lakebase-transactional-data-layer-databricks-apps)
- [Databricks Introduces Lakebase for AI Workloads | InfoQ](https://www.infoq.com/news/2026/02/databricks-lakebase-postgresql/)
- [Branches | Databricks Docs](https://docs.databricks.com/aws/en/oltp/projects/branches)
- [Neon Architecture Overview](https://neon.com/docs/introduction/architecture-overview)
- [Deep Dive into Neon Storage Engine](https://neon.com/blog/get-page-at-lsn)
