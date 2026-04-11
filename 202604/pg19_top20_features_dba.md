# PostgreSQL 19 最值得关注的 20 个特性

> 作者视角：资深 DBA / 应用架构师
> 版本基线：PostgreSQL 19 (开发中，基于 commit log 分析)
> 整理时间：2025 年

---

## 前言

PostgreSQL 19 的 commit log 横跨数万条记录，涵盖新功能、性能优化、运维改进、可观测性增强等多个维度。本文从 DBA 及应用开发者的实际痛点出发，精选 20 个最具工程价值的特性，力求说清楚每个特性"为什么选它""没有它之前的问题""有了它之后的价值"以及"什么场景下用"。

---

## 一、运维与维护

### 1. REPACK 命令：VACUUM FULL 的现代替代品

**选取原因：** 这是 PG 19 最具里程碑意义的运维改进之一，直接影响大表维护的停机窗口。

**之前的问题：** `VACUUM FULL` 和 `CLUSTER` 会对表加 AccessExclusiveLock，期间所有 DML（读写）均被阻塞。生产环境中对大表执行这两个操作往往意味着数小时的业务中断，运维团队常常需要在凌晨开维护窗口。`CLUSTER` 命令本身名字含义模糊，容易与集群概念混淆。

**PG 19 的改变：** 引入全新 `REPACK` 命令，将 `VACUUM FULL`（清理膨胀）和 `CLUSTER`（按索引顺序重写）整合为一个语义清晰的命令，并在此基础上提供 `CONCURRENTLY` 选项。

```sql
-- 传统方式（会长时间锁表）
VACUUM FULL orders;
CLUSTER orders USING orders_created_at_idx;

-- PG 19 新方式：锁表时间极短
REPACK orders;
REPACK orders USING INDEX orders_created_at_idx;

-- 并发模式：几乎不影响业务
REPACK orders CONCURRENTLY;
```

**并发原理：** `REPACK CONCURRENTLY` 在 ShareUpdateExclusiveLock（与 VACUUM 同级）下创建新表副本，同时通过复制槽（Replication Slot）进行逻辑解码，捕获并发的数据变更，最后在极短的 AccessExclusiveLock 窗口内完成文件名交换（swap relfilenodes）。

**适用场景：** 表膨胀严重（dead tuple 比例高）、需要按索引重组数据以提升顺序扫描性能、7×24 高可用系统无法接受长时间锁表的生产环境。

---

### 2. 在线启用/禁用数据校验（Online Data Checksums）

**选取原因：** 数据完整性是所有存储系统的底线。长期以来，开启数据校验需要停机，这让许多运维团队望而却步。

**之前的问题：** 数据校验（`data_checksums`）只能在 `initdb` 时开启，或者通过 `pg_checksums` 工具在集群完全停止的情况下开启，对于运行中的生产数据库，这几乎是不可能的操作。

**PG 19 的改变：** 支持在运行中的集群动态开启或关闭数据校验，无需停机。后台启动一个独立的 checksum worker 进程，逐库逐表将所有 buffer 标记为脏页，触发写入时计算校验和。整个过程通过 procsignalbarrier 机制确保所有 backend 在正确时机切换到验证模式，避免误报。

```sql
-- 在线开启（后台异步执行）
ALTER SYSTEM SET data_checksums = on;
SELECT pg_reload_conf();

-- 查看进度
SELECT * FROM pg_stat_checksum_progress;
```

**适用场景：** 对历史遗留的无校验集群补充防护、合规要求强制开启数据完整性校验、在不影响业务的前提下完成安全加固。

---

### 3. autovacuum 支持并行 Vacuum Worker

**选取原因：** 对于拥有大量索引的宽表，autovacuum 长期以来是单线程瓶颈，严重影响回收速度。

**之前的问题：** 手动 `VACUUM` 支持 `PARALLEL` 选项（并行清理多个索引），但 autovacuum 始终禁用了并行能力，无论表上有多少索引。这导致对于有数十个索引的大表，autovacuum 往往跑不过业务产生的垃圾，表持续膨胀。

**PG 19 的改变：** 引入两个新配置项：
- `autovacuum_max_parallel_workers`：全局上限，默认 0（保持原有行为，需显式开启）
- `autovacuum_parallel_workers`：per-table 存储参数，精细控制单表并行度

```sql
-- 全局开启 autovacuum 并行
ALTER SYSTEM SET autovacuum_max_parallel_workers = 4;
SELECT pg_reload_conf();

-- 对特定宽表配置并行度
ALTER TABLE big_wide_table SET (autovacuum_parallel_workers = 3);
```

cost 参数的动态传播通过 DSM 段完成，leader worker 更新 cost 参数后，并行 worker 在 `vacuum_delay_point()` 时自动感知并应用。

**适用场景：** 索引数量多（如 10 个以上）的大表、写入压力大导致 autovacuum 追不上的场景、需要精细管理 autovacuum 资源消耗的生产环境。

---

### 4. `max_locks_per_transaction` 默认值提升至 128

**选取原因：** 这是一个"悄然影响大量系统"的默认值变更，升级后开箱即用的锁容量翻倍。

**之前的问题：** 默认值 64 在早期版本设计时已偏小。PG 19 对锁哈希表的内存管理进行了重构，去除了原有的"安全边距"（10% safety margin）并将哈希表大小固定化，使得锁空间分配更精确，但也意味着需要更大的基础容量。使用 pg_partman 分区管理、大量临时表或在单事务中操作多张表的场景，极易触及原有限制。

**PG 19 的改变：** 默认值从 64 提升至 128，同时哈希表改为固定大小，内存分配更可预测。

**升级注意事项：** 如果 `postgresql.conf` 中曾手动调整过 `max_locks_per_transactions`，升级后需要评估是否同步翻倍，以维持原有的锁空间。

**适用场景：** 分区表系统（一个查询可能锁定数十个分区子表）、多租户 SaaS 系统（每个 schema 对应一个租户）、批量数据处理脚本。

---

### 5. 事务 ID 老化警告阈值提升至 1 亿

**选取原因：** XID Wraparound 问题是 PostgreSQL DBA 夜间惊醒的噩梦之一。

**之前的问题：** 旧的警告阈值为 4000 万，对于高频 OLTP 系统，从出现警告到强制停库（autovacuum anti-wraparound）的时间窗口可能只有数小时，运维人员来不及响应。

**PG 19 的改变：** 将 XID 及 MultiXactId 的警告阈值从 4000 万提升至 **1 亿**，同时在警告消息中展示当前剩余 ID 数量的百分比，让运维人员更直观地了解紧迫程度。

```
WARNING:  database "mydb" must be vacuumed within 57,000,000 transactions
DETAIL:  To avoid a database shutdown, execute a database-wide VACUUM in "mydb".
         76% of transaction IDs are still available.  -- 新增百分比信息
```

**适用场景：** 所有生产系统均受益；尤其是高并发 OLTP（每秒数千 TPS）、批量导入场景，提前预警更从容。

---

## 二、可观测性与诊断

### 6. EXPLAIN (IO)：I/O 行为深度诊断

**选取原因：** SQL 慢查询分析长期缺少 I/O 层面的细节，BUFFERS 选项只给出命中数，无法反映实际的磁盘 I/O 行为和预读效果。

**之前的问题：** `EXPLAIN (ANALYZE, BUFFERS)` 可以看到 buffer 命中/未命中数量，但无法得知：这些 I/O 是同步还是异步发出的？预读距离是多少？实际完成了多少次真正的磁盘请求？

**PG 19 的改变：** 引入新的 `IO` 选项，对使用 ReadStream 的扫描节点（SeqScan、BitmapHeapScan、TidRangeScan）提供详细的 I/O 统计：

```sql
EXPLAIN (ANALYZE, IO)
SELECT * FROM orders WHERE status = 'pending';

-- 输出示例（新增 IO 字段）
-- Seq Scan on orders
--   I/O Read Requests: 1024
--   I/O Prefetch Distance: 256 blocks
--   I/O Sync Reads: 128
--   I/O Async Reads: 896
```

同时，`auto_explain.log_io = on` GUC 可以将 IO 诊断信息自动写入慢查询日志。

**适用场景：** 诊断全表扫描性能瓶颈、评估 `effective_io_concurrency` 和 `maintenance_io_concurrency` 设置效果、分析 io_uring 异步 I/O 的实际收益。

---

### 7. pg_stat_autovacuum_scores：autovacuum 决策透明化

**选取原因：** 长期以来，DBA 无法直接查询"autovacuum 为什么还不来清理这张表？"这类问题。

**之前的问题：** autovacuum 是否对某张表执行清理，取决于内部的 score 计算（基于 dead tuple 数量、上次 vacuum 时间等），但这个计算过程完全不透明，DBA 只能通过观察 `pg_stat_user_tables` 的变化来猜测。

**PG 19 的改变：** 新增系统视图 `pg_stat_autovacuum_scores`，每行对应当前数据库的一张表，实时展示：
- autovacuum score（当前分数）
- 是否会触发 vacuum / analyze
- dead tuple 统计与阈值

```sql
SELECT relname,
       av_score,
       needs_vacuum,
       needs_analyze,
       n_dead_tup,
       av_threshold
FROM pg_stat_autovacuum_scores
ORDER BY av_score DESC
LIMIT 20;
```

**适用场景：** 排查表膨胀问题、验证 autovacuum 参数调优效果、在 autovacuum 未按预期运行时进行诊断。

---

### 8. pg_stat_recovery：备库恢复状态统一视图

**选取原因：** 备库监控一直是个拼图游戏，需要调用多个函数才能获取完整状态。

**之前的问题：** 备库的恢复状态散落在 `pg_get_wal_replay_pause_state()`、`pg_last_xact_replay_timestamp()`、`pg_stat_wal_receiver` 等多个函数/视图中，且各函数各自加锁，难以保证时间点一致性。

**PG 19 的改变：** 新增 `pg_stat_recovery` 系统视图，通过单次 spinlock 原子地读取所有恢复状态，字段包括：
- 最后成功回放的 WAL LSN 及 timeline
- 当前正在回放的 WAL 记录 LSN
- WAL chunk 起始时间
- Promotion trigger 状态
- 最新 commit/abort 时间戳
- 恢复暂停状态

```sql
-- 在备库上执行
SELECT * FROM pg_stat_recovery;
```

需要 `pg_read_all_stats` 权限；主库上返回 0 行。

**适用场景：** 主备切换监控脚本、灾备演练状态确认、HA 中间件（Patroni/Repmgr）的健康检查增强。

---

### 9. pg_stat_lock：锁等待统计

**选取原因：** 锁竞争是 OLTP 系统性能问题的高频根因，但一直缺少历史维度的统计。

**之前的问题：** `pg_locks` 只能看到当前快照，锁等待发生和结束后，历史信息完全丢失。DBA 只能在问题发生时碰巧守在终端，或者依赖 `log_lock_waits` 的日志，缺乏结构化的统计数据。

**PG 19 的改变：** 新增统计类型 `PGSTAT_KIND_LOCK` 以及系统视图 `pg_stat_lock`，累计记录：
- `waits`：需要等待的锁获取次数
- `wait_time`：累计等待时间
- `fastpath_exceeded`：因 `max_locks_per_transaction` 限制导致无法走 fast-path 的次数

这些指标按锁标签类型（`LockTagType`）聚合，性能影响极低（仅在 `deadlock_timeout` 超时后才记录，fast-path 锁不影响）。

```sql
SELECT * FROM pg_stat_lock
ORDER BY wait_time DESC;
```

**适用场景：** 识别热点锁竞争的对象类型、评估 `deadlock_timeout` 设置合理性、监控分区表的锁竞争模式。

---

### 10. log_min_messages 支持按进程类型分级

**选取原因：** 统一的日志级别往往难以平衡"信息完整"与"日志噪音"的矛盾。

**之前的问题：** `log_min_messages` 是全局设置，对 backend、autovacuum、wal sender 等所有进程类型一视同仁。想详细记录 autovacuum 的行为，就不得不接受所有 backend 同等级别的日志洪流。

**PG 19 的改变：** `log_min_messages` 支持逗号分隔的 `type:level` 列表，可以对不同进程类型设置不同的日志级别，同时保持向后兼容（仍支持单个级别值）。

```ini
# postgresql.conf
# autovacuum 和 checkpoint 进程记录 DEBUG1，其他进程保持 WARNING
log_min_messages = 'WARNING,autovacuum:DEBUG1,checkpointer:LOG'
```

支持的进程类型包括 `backend`、`autovacuum`、`checkpointer`、`walwriter`、`walsender` 等。

**适用场景：** 调试 autovacuum 行为但不想产生大量 backend 日志、WAL sender 问题诊断、细粒度的日志管理策略。

---

## 三、SQL 功能扩展

### 11. UPDATE/DELETE FOR PORTION OF：时态数据原生支持

**选取原因：** 时态数据（随时间变化的历史记录）是金融、医疗、审计等行业的核心需求，PG 19 将其提升为原生语法。

**之前的问题：** 对于按时间范围存储历史的表（如 `valid_at daterange`），修改某个时间段的数据需要手动：先查出受影响的行，再计算时间范围的截取，再分别 INSERT "剩余"的历史片段，再 UPDATE 原行。逻辑复杂且容易出错。

**PG 19 的改变：** 直接在 `UPDATE`/`DELETE` 语法中声明时间范围，引擎自动处理时间段的截取和剩余片段的插入：

```sql
-- 修改 2001 年这一年内的有效数据
UPDATE employee_history
FOR PORTION OF valid_at FROM '2001-01-01' TO '2002-01-01'
SET salary = salary * 1.1
WHERE emp_id = 42;

-- 删除某段时间的历史记录
DELETE FROM employee_history
FOR PORTION OF valid_at FROM '2003-01-01' TO '2004-01-01'
WHERE emp_id = 42;
```

引擎自动插入时间范围外的"temporal leftover"行，保证历史完整性。

**适用场景：** SCD（缓慢变化维度）数据管理、人员历史档案系统、金融产品有效期管理、任何需要"按时间段修改历史"的业务场景。

---

### 12. INSERT ... ON CONFLICT DO SELECT：Upsert 的完美补全

**选取原因：** 这是一个被社区讨论多年、开发者呼声极高的功能，填补了 Upsert 语义的重要空缺。

**之前的问题：** `INSERT ... ON CONFLICT DO UPDATE` 可以处理冲突时更新，`DO NOTHING` 可以忽略冲突，但两者都无法方便地获取冲突的**已有行**并返回给调用方。常见的需求是：如果插入成功返回新行，如果冲突则返回已有行——这在之前需要多次往返（先 INSERT、再 SELECT）或复杂的 CTE 来实现。

**PG 19 的改变：**

```sql
-- 插入成功返回新行，冲突时返回已有行
INSERT INTO products (sku, name, price)
VALUES ('ABC-001', 'Widget', 9.99)
ON CONFLICT (sku) DO SELECT
RETURNING *;

-- 冲突时锁定已有行再返回
INSERT INTO orders (order_id, status)
VALUES (12345, 'pending')
ON CONFLICT (order_id) DO SELECT FOR UPDATE
RETURNING *;
```

可选的 `FOR UPDATE/SHARE` 子句允许在返回前对冲突行加锁，`WHERE` 子句可以过滤部分冲突行（仍会加锁）。

**适用场景：** 幂等 API 接口（"创建或获取"语义）、分布式系统的去重插入、电商库存扣减的并发安全实现。

---

### 13. SQL/PGQ：图数据库查询原生化

**选取原因：** 这是 PG 19 最具战略价值的新功能，让 PostgreSQL 成为首批实现 ISO/IEC SQL/PGQ 标准的主流关系型数据库之一。

**之前的问题：** 在 PostgreSQL 中实现图查询（如找到社交网络中的朋友的朋友）需要复杂的递归 CTE，不仅语法繁琐，而且性能难以预测，也无法表达复杂的图模式匹配。

**PG 19 的改变：** 引入完整的属性图（Property Graph）支持：

```sql
-- 定义属性图
CREATE PROPERTY GRAPH social_graph
  VERTEX TABLES (users)
  EDGE TABLES (
    friendships SOURCE KEY (from_user_id) REFERENCES users(id)
                DESTINATION KEY (to_user_id) REFERENCES users(id)
  );

-- 图模式匹配查询（GRAPH_TABLE）
SELECT p.name, f.name AS friend_name
FROM GRAPH_TABLE(social_graph
  MATCH (p:users) -[:friendships]-> (f:users)
  WHERE p.id = 42
  COLUMNS (p.name, f.name)
) AS result;

-- psql 查看图定义
\dG
```

新增 DDL：`CREATE/ALTER/DROP PROPERTY GRAPH`；新增系统目录；`pg_dump` 完整支持。

**适用场景：** 社交关系分析、供应链依赖图、知识图谱、欺诈检测中的关联关系挖掘、网络拓扑分析。

---

### 14. NOT IN 子查询自动转换为 Anti-Join

**选取原因：** 这是一个"悄悄让 SQL 快 10 倍甚至 100 倍"的查询优化，且完全透明、无需改写 SQL。

**之前的问题：** 历史上，`x NOT IN (SELECT y FROM ...)` 因 NULL 值的语义问题（`NULL = x` 返回 NULL 而非 FALSE，导致 NOT IN 为 NULL，进而过滤掉行）无法转换为 Anti-Join。优化器只能将其作为不透明的 SubPlan 执行（逐行扫描外层，每行都跑一次子查询），对大数据集效率极低。

**PG 19 的改变：** 当优化器能够证明以下条件时，自动将 NOT IN 子查询重写为 Anti-Join：
1. 比较两侧均不可能为 NULL（约束或表达式可推导）
2. 使用的比较操作符属于 B-tree 或 Hash 操作符族（保证不会返回 NULL）

```sql
-- 以下查询（假设 a.id 和 b.id 均有 NOT NULL 约束）
-- PG 18 及以前：SubPlan（慢）
-- PG 19：自动转换为 Anti-Join（快）
SELECT * FROM orders a
WHERE a.customer_id NOT IN (SELECT id FROM blacklist b);
```

**适用场景：** 数据清洗脚本中的排除查询、ETL 中的差集计算、报表中的"未完成"类查询。建议对历史 SQL 用 `EXPLAIN` 验证是否已触发此优化。

---

### 15. CREATE SCHEMA 支持更多对象类型

**选取原因：** DDL 原子性是数据库对象管理的重要原则，此前 CREATE SCHEMA 的限制让多对象场景下的部署脚本变得脆弱。

**之前的问题：** `CREATE SCHEMA` 仅支持在同一语句中创建表、视图、序列、索引等少数对象类型。函数、类型、域、排序规则等需要独立的 `CREATE` 语句，无法与 schema 创建原子绑定，而且外键等约束的执行顺序也可能触发"对象不存在"的依赖错误。

**PG 19 的改变：** 扩展支持：函数、存储过程、聚合函数、操作符、类型（含域）、排序规则、全文检索对象；外键约束统一延迟到所有子命令执行完毕后再检查，消除顺序依赖。

```sql
CREATE SCHEMA analytics
  CREATE TYPE status_enum AS ENUM ('active', 'inactive', 'pending')
  CREATE DOMAIN email AS TEXT CHECK (VALUE ~ '^[^@]+@[^@]+$')
  CREATE FUNCTION normalize_email(text) RETURNS email AS $$
    SELECT lower(trim($1))::email
  $$ LANGUAGE sql IMMUTABLE
  CREATE TABLE users (
    id serial PRIMARY KEY,
    email email NOT NULL,
    status status_enum DEFAULT 'active'
  );
```

**适用场景：** 多租户系统按 schema 隔离、应用模块化部署脚本、数据库版本管理工具（如 Flyway/Liquibase）。

---

## 四、性能优化

### 16. 外键检查批量索引探测（Fast-Path FK Batching）

**选取原因：** 外键约束是 OLTP 系统数据完整性的基石，但高并发批量插入时的 FK 检查一直是显著的性能瓶颈。

**之前的问题：** 每条带外键的 INSERT/UPDATE 触发一次 AFTER 触发器，每次触发都独立做一次 PK 索引探测（含 CommandCounterIncrement、snapshot、权限检查等开销），N 行插入就有 N 次独立的索引 B-tree 遍历。

**PG 19 的改变：** 引入两层优化：
1. **Fast-Path**（上个版本引入，约 1.8x 加速）：复用 FmgrInfo，减少函数调用开销
2. **Batched Index Probes**（PG 19 新增，叠加约 1.6x 加速）：将 64 行的外键值缓冲为一个批次，构造 `SK_SEARCHARRAY` 扫描键，让索引 AM 内部排序去重后一次性完成匹配，仅需单次 B-tree 遍历

综合加速效果：较未优化代码约 **2.9x** 提升（单列整型 FK，百万行，PK 表和索引均在内存中）。

```sql
-- 批量插入场景直接受益，无需任何 SQL 修改
INSERT INTO order_items (order_id, product_id, quantity)
SELECT * FROM staging_items;  -- 数百万行，FK 检查大幅加速
```

**适用场景：** 批量数据导入（含外键约束）、高并发订单/交易系统、数据迁移场景。

---

### 17. COPY FROM 的 SIMD 加速

**选取原因：** 数据导入是 DBA 日常工作中最频繁的操作之一，SIMD 加速带来的收益立竿见影。

**之前的问题：** `COPY FROM (FORMAT text/csv)` 逐字节扫描输入缓冲区寻找特殊字符（换行符、分隔符、引号等），在大文件导入时，这个逐字节扫描本身成为性能瓶颈。

**PG 19 的改变：** 利用 SIMD 指令（SSE2/AVX2 等）一次处理 16 或 32 字节，跳过无特殊字符的数据块，无需任何配置修改。

**适用场景：** ETL 管道的 CSV 批量导入、数据库初始化数据加载、日志导入分析系统。导入大型 CSV 文件时可以明显感受到速度提升。

---

### 18. 元组解包（Tuple Deformation）优化

**选取原因：** 元组解包是几乎所有 SQL 操作的底层热路径，其优化效果全局可见。

**之前的问题：** 从 heap 中读取一行数据时，需要逐字段解包（deform）。原有实现对每个字段逐位检查 NULL bitmap，对 attcacheoff（字段偏移缓存）的更新也发生在解包过程中，两者都是性能热点。

**PG 19 的改变：** 多项优化叠加：
1. `TupleDescFinalize()` 预计算所有字段的 `attcacheoff`，解包循环不再需要更新
2. NULL bitmap 改为按字节（8位）批量扫描，快速定位第一个 NULL 字段
3. 记录最大保证存在的属性编号（NOT NULL + 无 missing 值），对该范围内的字段跳过 NULL 检查
4. 专用的"无 NULL 快速路径"循环处理所有 offset 已缓存的字段

**适用场景：** 所有 SQL 查询均受益，宽表（列数多）、NOT NULL 列占多数的表效果尤为明显。

---

## 五、复制与高可用

### 19. wal_sender_shutdown_timeout：可控的关机等待

**选取原因：** 逻辑复制环境下，关机挂起是一个频繁被生产团队抱怨的问题。

**之前的问题：** PostgreSQL 关机（`pg_ctl stop`）时，WAL sender 会等待所有 pending 数据复制到接收端才退出。若逻辑复制的 apply worker 因锁等待或其他原因阻塞，关机可能挂起数分钟甚至更长，严重影响滚动升级和故障切换的时间窗口。

**PG 19 的改变：** 新增 `wal_sender_shutdown_timeout` GUC（默认 -1，即无限等待，保持原有行为），可以设置最大等待时间：

```ini
# postgresql.conf
# 最多等待 30 秒，超时后强制关机
wal_sender_shutdown_timeout = 30s
```

**注意事项：** 物理复制主备切换时，若超时导致数据未完全同步，可能影响切换安全性，建议物理复制场景谨慎评估。逻辑复制通常可以接受此限制，因为订阅方有独立的状态追踪。

**适用场景：** 逻辑复制为主的系统、DevOps CI/CD 流水线中的快速滚动重启、容器化环境中的优雅关机（grace period 有限制）。

---

### 20. CREATE SUBSCRIPTION ... SERVER：订阅管理解耦

**选取原因：** 逻辑复制的连接管理一直是运维痛点，硬编码连接字符串让多订阅场景的维护变得脆弱。

**之前的问题：** `CREATE SUBSCRIPTION` 要求直接写入连接字符串（含主机、端口、密码等），这意味着：主库迁移时需要逐一修改所有订阅；连接凭证嵌入在系统目录中，安全审计困难；多个订阅指向同一主库时，参数重复管理。

**PG 19 的改变：** 允许通过 FDW Server 名称创建订阅，利用现有的外部服务器和用户映射（User Mapping）基础设施：

```sql
-- 先定义外部服务器（复用 FDW 基础设施）
CREATE SERVER primary_db
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'primary.internal', port '5432', dbname 'myapp');

CREATE USER MAPPING FOR replicator
SERVER primary_db
OPTIONS (user 'replication_user', password 'secret');

-- 订阅通过服务器名引用，无需硬编码连接串
CREATE SUBSCRIPTION myapp_sub
SERVER primary_db
PUBLICATION myapp_pub;

-- 主库迁移时只需修改一处
ALTER SERVER primary_db OPTIONS (SET host 'new-primary.internal');
```

**适用场景：** 多订阅指向同一源库、主库 IP/密码轮换、统一的连接管理策略、多租户逻辑复制架构。

---

---

## 六、分区表管理

### 21. ALTER TABLE ... SPLIT PARTITION / MERGE PARTITIONS

**选取原因：** 分区表是现代大数据系统的标配，但分区边界的调整一直没有原生 DDL 支持，运维成本极高。

**之前的问题：** 如果需要将一个过大的分区拆成两个，或将多个冷数据分区合并为一个，必须手动进行繁琐操作：创建新分区、用 `INSERT ... SELECT` 搬移数据、删除旧分区，整个过程长时间持有 AccessExclusiveLock，对线上业务影响极大，且脚本容易出错。

**PG 19 的改变：** 引入两条原生 DDL：

```sql
-- 将一个大分区拆为两个（范围/哈希/列表分区均支持）
ALTER TABLE orders SPLIT PARTITION orders_2024
    INTO (
      PARTITION orders_2024_h1 FOR VALUES FROM ('2024-01-01') TO ('2024-07-01'),
      PARTITION orders_2024_h2 FOR VALUES FROM ('2024-07-01') TO ('2025-01-01')
    );

-- 将多个冷数据分区合并为一个归档分区
ALTER TABLE orders MERGE PARTITIONS (orders_2020, orders_2021, orders_2022)
    INTO orders_archive;
```

**当前限制：** 初版实现在单进程中执行数据迁移，全程持有 AccessExclusiveLock，不建议在大流量大表上直接使用，建议在维护窗口中操作。未来版本有望引入并发/低锁实现。

**适用场景：** 分区策略调整（月分区改为季分区）、冷热数据分区重组、历史数据归档整合。

---

### 22. autovacuum 表优先级评分调度

**选取原因：** 传统 autovacuum 按 pg_class 扫描顺序处理表，缺乏优先级感知，关键表可能排在不重要的表之后清理。

**之前的问题：** 当同时有多张表需要 vacuum 时，autovacuum worker 按扫描顺序处理，可能先清理一张不紧急的普通表，而接近 XID wraparound 的关键表却在队列末端等待。

**PG 19 的改变：** 引入基于比率的评分系统。每张表的分数为各指标（dead tuple 数、insert tuple 数、表年龄 relfrozenxid/relminmxid）与其对应阈值之比的最大值，分数越高代表越紧迫，worker 优先处理高分表。

```sql
-- 通过 pg_stat_autovacuum_scores 查看当前评分（即本文第 7 条特性）
SELECT relname, av_score, needs_vacuum, needs_analyze
FROM pg_stat_autovacuum_scores
ORDER BY av_score DESC NULLS LAST
LIMIT 10;
```

**适用场景：** 混合负载系统（部分表高频写入、部分表冷数据）、防止 XID wraparound 紧急情况的主动预防。

---

## 七、工具与运维增强

### 23. pg_get_database_ddl / pg_get_tablespace_ddl / pg_get_role_ddl

**选取原因：** PostgreSQL 长期有 `pg_get_viewdef()`、`pg_get_indexdef()` 等函数，但对数据库、表空间、角色这类全局对象缺乏 DDL 反查能力，给迁移、文档化、审计带来不便。

**之前的问题：** 想获取某个角色的完整创建语句，只能翻 `pg_authid` 系统目录手动拼接；数据库迁移时需要手写 `CREATE DATABASE` 语句，容易遗漏 collation、encoding、tablespace 等关键选项。

**PG 19 的改变：** 三个新函数，返回可直接执行的 DDL：

```sql
-- 反查数据库 DDL（需要 CONNECT 权限）
SELECT pg_get_database_ddl('myapp'::regdatabase);
-- 输出: CREATE DATABASE myapp ENCODING 'UTF8' LC_COLLATE 'zh_CN.UTF-8' ...
--       ALTER DATABASE myapp SET work_mem = '64MB';

-- 反查表空间 DDL
SELECT pg_get_tablespace_ddl('fast_ssd');

-- 反查角色 DDL（密码不包含在内）
SELECT pg_get_role_ddl('app_user'::regrole, 'memberships', 'true');
-- 输出: CREATE ROLE app_user LOGIN ...
--       GRANT analytics TO app_user;
```

**适用场景：** 数据库迁移文档生成、灾备演练的环境重建、配置审计、IaC（基础设施即代码）工具集成。

---

### 24. COPY TO 支持 JSON 格式输出

**选取原因：** JSON 已成为现代系统数据交换的事实标准，原生 COPY JSON 格式避免了应用层转换的开销和麻烦。

**之前的问题：** `COPY TO` 仅支持 text 和 csv 格式。要导出 JSON，必须借助 `row_to_json()` 函数查询再导出，或在应用层做格式转换，增加中间步骤。

**PG 19 的改变：**

```sql
-- 导出为 NDJSON（每行一个 JSON 对象，适合流处理）
COPY orders TO '/tmp/orders.ndjson' (FORMAT json);

-- 导出为完整的 JSON 数组（适合 REST API 直接读取）
COPY orders TO '/tmp/orders.json' (FORMAT json, force_array true);

-- 指定列导出
COPY (SELECT id, status, created_at FROM orders WHERE status = 'pending')
TO STDOUT (FORMAT json);
```

注意：JSON 格式目前仅支持 `COPY TO`，不支持 `COPY FROM`。

**适用场景：** 数据导出到 NoSQL 系统、API 数据快照导出、与数据管道（Kafka/Flink）的集成。

---

### 25. vacuumdb 新增 --dry-run 选项

**选取原因：** 在生产环境执行 `vacuumdb` 前，DBA 往往需要先确认将要清理哪些表，--dry-run 让这一过程变得安全可控。

**之前的问题：** `vacuumdb --all` 会直接执行 VACUUM/ANALYZE 命令，没有"先看后做"的预览模式，在不恰当的时机触发大规模 vacuum 可能造成 I/O 风暴。

**PG 19 的改变：**

```bash
# 仅打印将要执行的命令，不实际执行
vacuumdb --all --dry-run --verbose

# 输出示例：
# VACUUM mydb.public.orders;
# VACUUM mydb.public.customers;
# ANALYZE mydb.public.order_items;
```

**适用场景：** 生产数据库的维护窗口前验证、自动化运维脚本的安全测试、新 DBA 学习和培训。

---

### 26. pg_stat_progress_vacuum/analyze 新增来源和模式列

**选取原因：** VACUUM 运行时的来源和模式对于诊断问题至关重要，此前需要结合多个视图才能推断。

**之前的问题：** `pg_stat_progress_vacuum` 可以看到 vacuum 进度，但无法直接判断：这是手动触发的还是 autovacuum？是普通 vacuum 还是 aggressive（防 wraparound）模式？需要分别查 `pg_stat_activity` 才能拼凑答案。

**PG 19 的改变：** 新增字段：
- `pg_stat_progress_vacuum.mode`：`normal`、`aggressive`（接近 wraparound）、`failsafe`（紧急模式）
- `pg_stat_progress_vacuum.started_by`：`manual`、`autovacuum`、`autovacuum_wraparound`
- `pg_stat_progress_analyze.started_by`：`manual`、`autovacuum`

```sql
-- 一眼识别 vacuum 紧急程度和触发来源
SELECT pid, relid::regclass, phase, mode, started_by,
       heap_blks_scanned, heap_blks_total
FROM pg_stat_progress_vacuum;
```

**适用场景：** 监控告警（aggressive vacuum 触发时发告警）、autovacuum 行为分析、故障排查。

---

## 八、复制与逻辑解码增强

### 27. LISTEN/NOTIFY 性能大幅优化

**选取原因：** LISTEN/NOTIFY 是 PostgreSQL 原生的异步消息机制，广泛用于实时通知场景，但在大量监听者存在时性能一直较差。

**之前的问题：** 发送 NOTIFY 时，所有监听者（无论监听的 channel 是否匹配）都会被唤醒，然后各自检查队列。在多租户 pub/sub 场景（数百个监听连接但只有少数订阅特定 channel），产生大量无效唤醒。

**PG 19 的改变：** 引入共享哈希表精确跟踪"哪个进程监听哪个 channel"，NOTIFY 时只唤醒真正感兴趣的监听者；对于不感兴趣的监听者，直接在共享内存中推进其队列指针，完全避免唤醒。

**效果：** 在监听者多但感兴趣比例低的场景，NOTIFY 吞吐量有**整数倍**提升，延迟显著降低。对于每个监听者都接收所有消息的场景，无性能损失。

**适用场景：** 多租户 SaaS 的实时通知系统、微服务间的事件驱动架构、基于 LISTEN/NOTIFY 的任务队列。

---

### 28. wal_level='replica' 时动态启用逻辑解码

**选取原因：** 逻辑复制的 WAL 开销问题一直是"要么全开要么全关"的二选一，PG 19 实现了按需动态调节。

**之前的问题：** 使用逻辑复制必须将 `wal_level` 固定为 `logical`，即便没有活跃的逻辑槽，也持续产生逻辑级别的 WAL 记录（体积更大），带来不必要的存储和 CPU 开销。

**PG 19 的改变：** 当 `wal_level = replica` 时，系统自动感知逻辑复制槽的存在：
- 创建第一个逻辑槽时：自动提升有效 WAL 级别到 `logical`（同步执行）
- 最后一个逻辑槽被删除/失效后：自动降回 `replica`（通过 checkpointer 异步执行）

```sql
SHOW effective_wal_level;  -- 有逻辑槽时显示 logical，无时显示 replica
```

**适用场景：** 按需逻辑复制（如临时数据迁移后停用）、降低无逻辑复制需求时的 WAL 开销。

---

### 29. 逻辑复制初始同步改用 COPY table TO（分区表）

**选取原因：** 逻辑复制初始同步的性能直接影响新订阅的上线时间，对大型分区表尤为明显。

**之前的问题：** 逻辑复制的表同步对分区表使用 `COPY (SELECT ...) TO` 形式，优化器无法利用分区裁剪，会扫描所有分区的数据。

**PG 19 的改变：** 改用 `COPY table TO` 形式（PG 18 已支持分区表的此语法），让分区裁剪等优化得以生效，显著加速大型分区表的初始同步。

**适用场景：** 订阅大型分区表（如按时间分区的日志表、订单表）时，新建订阅的初始数据同步阶段。

---

### 30. 订阅级别 wal_receiver_timeout 设置

**选取原因：** 多订阅场景下，不同 publisher 的网络质量差异很大，统一的超时设置无法兼顾。

**之前的问题：** `wal_receiver_timeout` 是全局 GUC，所有订阅共享同一个超时值，无法为本地 publisher 和跨洋 publisher 分别配置。

**PG 19 的改变：**

```sql
-- 近距离 publisher，使用较短超时快速检测故障
CREATE SUBSCRIPTION local_sub
    CONNECTION 'host=local-primary ...' PUBLICATION my_pub
    WITH (wal_receiver_timeout = '10s');

-- 跨洋 publisher，使用较长超时容忍网络抖动
CREATE SUBSCRIPTION remote_sub
    CONNECTION 'host=us-primary ...' PUBLICATION my_pub
    WITH (wal_receiver_timeout = '120s');
```

**适用场景：** 多数据中心逻辑复制、混合云环境（本地 + 公有云）、需要精细化故障检测策略的高可用架构。

---

## 九、SQL 与查询优化补充

### 31. jsonpath 新增字符串操作方法

**选取原因：** jsonpath 是处理 JSON 数据的强力工具，但原有字符串处理能力有限，实际使用中常常需要绕回 SQL 层处理。

**之前的问题：** jsonpath 表达式中无法直接进行字符串的裁剪、大小写转换、替换等操作，要对 JSON 字段做字符串清洗，必须先提取再用 SQL 函数处理，语法繁琐。

**PG 19 的改变：** 新增 jsonpath 方法：`ltrim()`、`rtrim()`、`btrim()`、`lower()`、`upper()`、`initcap()`、`replace()`、`split_part()`

```sql
-- 直接在 jsonpath 中进行字符串操作
SELECT jsonb_path_query(data, '$.name.lower()') FROM users;

-- 链式调用：去空格 + 小写
SELECT jsonb_path_query_array(
    '{"emails": [" Admin@Example.COM ", "user@test.com"]}',
    '$.emails[*].btrim().lower()'
);
-- 结果: ["admin@example.com", "user@test.com"]
```

**适用场景：** JSON 数据清洗 ETL、对存储在 JSONB 中的用户输入进行规范化、复杂 JSON 查询的内联处理。

---

### 32. CHECK 约束支持 ENFORCED/NOT ENFORCED 切换

**选取原因：** NOT ENFORCED 约束在数据迁移和历史数据处理中极为实用，PG 19 将此能力从外键约束扩展到 CHECK 约束。

**之前的问题：** PG 17 引入了外键约束的 NOT ENFORCED 模式，但 CHECK 约束始终强制执行。在迁移历史脏数据时，若旧数据不满足新的 CHECK 约束，就无法添加约束。

**PG 19 的改变：**

```sql
-- 添加不强制执行的 CHECK 约束（不验证已有数据，也不拦截新写入）
ALTER TABLE products
    ADD CONSTRAINT chk_price_positive CHECK (price > 0) NOT ENFORCED;

-- 将已有约束切换为强制执行（会全表扫描验证现有数据）
ALTER TABLE products
    ALTER CONSTRAINT chk_price_positive ENFORCED;
```

**适用场景：** 历史脏数据迁移（先导入再逐步清洗）、数据质量监控（标记但不拦截违规数据）、ETL 暂存表的约束管理。

---

## 十、运维工具与监控补充

### 33. 密码即将过期提前告警

**选取原因：** 密码过期导致的业务中断是生产事故的高频原因之一，提前告警让运维有足够时间响应。

**之前的问题：** 用户密码过期时，连接直接失败，没有任何提前警示。运维团队往往在业务中断后才发现，需要在压力下紧急处理。

**PG 19 的改变：** 引入新 GUC `password_expiration_warning_threshold`（默认 7 天）。在密码到期前 N 天内，成功认证后发出 WARNING：

```
WARNING:  password will expire in 3 days
```

```ini
# postgresql.conf
password_expiration_warning_threshold = 14d  # 提前 14 天告警
```

同时引入了通用"连接告警"基础设施，未来可用于其他类型的连接时警告（如 MD5 弃用提醒）。

**适用场景：** 企业合规要求定期轮换密码的系统、数据库账号管理自动化（监控 WARNING 日志触发告警）。

---

### 34. 并行 TID Range Scan

**选取原因：** TID Range Scan 是按物理块范围扫描的高效方式，并行化后进一步提升大表范围扫描的吞吐。

**之前的问题：** `WHERE ctid BETWEEN '(0,1)' AND '(1000,0)'` 形式的 TID Range Scan 不支持并行，规划器有时会选择并行 Seq Scan（扫描不必要的块）来获得并行收益，而放弃更精准的 TID Range Scan。

**PG 19 的改变：** TID Range Scan 支持并行化，块范围按 chunk 分配给各个 worker，接近扫描末尾时 chunk 大小逐步缩小以确保 worker 同时结束。消除了规划器在 TID Range Scan 和并行 Seq Scan 之间的两难权衡。

**适用场景：** 基于 ctid 的分批处理脚本（大表数据迁移或清洗）、`pg_repack` 等工具的底层扫描、需要按块范围并行读取的自定义工具。

---

### 35. pg_available_extensions 新增 location 列

**选取原因：** 扩展来源不透明是运维安全审计的一个盲点，location 列让扩展来源一目了然。

**之前的问题：** 配置了多个 `extension_control_path` 时，`pg_available_extensions` 无法显示扩展文件实际存放在哪个目录，难以判断加载的到底是哪个路径下的版本。

**PG 19 的改变：** 新增 `location` 列：
- 系统内置扩展：显示 `$system`
- 用户自定义目录：超级用户可见实际路径，普通用户显示 `<insufficient privilege>`

```sql
SELECT name, default_version, location
FROM pg_available_extensions
ORDER BY location, name;
```

**适用场景：** 扩展安全审计、多版本扩展并存时的来源确认、自动化配置管理工具。

---

### 36. pg_buffercache 新增缓冲区强制置脏函数

**选取原因：** 这是一组面向开发者和 DBA 的测试/调试利器，用于精确控制 buffer 状态以复现问题或测试性能。

**之前的问题：** 没有办法通过 SQL 手动将特定 buffer 标记为脏（dirty），测试 checkpoint 行为、WAL 生成量时只能通过实际 DML 间接触发，难以精确控制实验条件。

**PG 19 的改变：** 新增三个超级用户专用函数（pg_buffercache 扩展）：

```sql
-- 将某张表的所有 buffer 标记为脏
SELECT pg_buffercache_mark_dirty_relation('mytable'::regclass);

-- 将整个 shared_buffers 中所有 buffer 标记为脏（慎用！）
SELECT pg_buffercache_mark_dirty_all();

-- 单个 buffer
SELECT pg_buffercache_mark_dirty(bufferid)
FROM pg_buffercache LIMIT 1;
```

**适用场景：** 测试 checkpoint 性能和 WAL 生成量、压测工具开发、复现脏页相关问题场景。

---

### 37. 进程终止信号来源记录 PID 和 UID

**选取原因：** 在多用户或多进程管理环境中，"是谁杀死了这个查询"是一个高频疑问，此前无法从日志中直接获取答案。

**之前的问题：** backend 被 `pg_terminate_backend()` 或外部 `kill` 终止时，日志中只有"termination signal received"，无法知道信号的发送者，给审计和排查带来困难。

**PG 19 的改变：** 在支持 `SA_SIGINFO` 的平台（Linux、FreeBSD 等主流系统）上，终止信号日志新增发送方信息：

```
ERROR:  terminating connection due to administrator command
DETAIL:  Signal sent by PID 12345, UID 1001.
```

**适用场景：** 多 DBA 团队环境下的操作审计、连接池（pgBouncer）行为追踪、自动化运维脚本的操作追溯。

---

### 38. checkpoint 完成日志包含请求标志

**选取原因：** checkpoint 日志是 DBA 判断系统负载的重要依据，完整信息让诊断更准确。

**之前的问题：** checkpoint 完成日志不含触发原因标志（CKPT_REQ_XLOG/CKPT_REQ_TIME 等），要了解完整原因，必须将完成消息与较早的启动消息关联对比，在高频 checkpoint 场景下非常困难。

**PG 19 的改变：** checkpoint 完成消息中直接包含请求标志，消息自包含：

```
LOG:  checkpoint complete: wrote 1024 buffers (12%); ... flags=CKPT_REQ_XLOG
```

**适用场景：** checkpoint 频率异常分析、`checkpoint_completion_target` 调优、I/O 压力诊断。

---

### 39. pg_restore 新增 --no-globals 选项

**选取原因：** 从 pg_dumpall 归档中选择性恢复数据库是常见需求，--no-globals 让部分恢复更加精细。

**之前的问题：** 从 pg_dumpall 生成的归档恢复时，无法跳过全局对象（角色、表空间），而目标服务器可能已有这些全局对象，造成冲突或报错。

**PG 19 的改变：**

```bash
# 从 pg_dumpall 归档中只恢复特定数据库，跳过全局对象
pg_restore --no-globals -d myapp pg_dumpall_archive.dump
```

**适用场景：** 从全量备份中选择性恢复单个数据库、跨版本迁移中分步骤处理全局对象和数据库、多租户环境下的精细化恢复操作。

---

## 十一、安全

### 40. standard_conforming_strings 永久锁定为 ON

**选取原因：** 这是消除历史安全隐患的关键清理，彻底消除了因字符串转义模式不一致导致的潜在 SQL 注入风险。

**之前的问题：** `standard_conforming_strings` 在 PG 9.1 后默认为 ON，但仍允许设置为 OFF。应用可能错误假设 OFF 模式导致转义处理不一致；某些条件下可能被利用来绕过字符串转义。

**PG 19 的改变：** 彻底禁止将 `standard_conforming_strings` 设置为 OFF，该 GUC 保留但任何尝试设为 OFF 的操作都会报错。同期移除了相关的 `escape_string_warning` GUC。

**升级注意事项：** 如果系统中有 `SET standard_conforming_strings = off` 或依赖旧式转义语法（`\'` 而非 `''`）的代码，需要在升级前清查和修复。

---

### 41. MD5 密码认证弃用警告

**选取原因：** MD5 作为密码哈希算法已不安全，PG 19 正式开始推动迁移到 SCRAM-SHA-256。

**之前的问题：** 大量历史系统仍使用 MD5 密码认证，数据库层面没有任何提示或告警，DBA 可能并不知道自己的系统存在安全隐患。

**PG 19 的改变：** 使用 MD5 密码成功认证后，服务器发出 WARNING，引导用户迁移到 SCRAM-SHA-256：

```
WARNING:  MD5 password authentication is deprecated
HINT:  Change the password with ALTER ROLE ... PASSWORD '...'.
```

**迁移方式：**

```sql
-- 重新设置密码（确保 password_encryption = 'scram-sha-256'）
ALTER ROLE myuser PASSWORD 'new_secure_password';
```

同时修改 `pg_hba.conf` 中 `md5` 为 `scram-sha-256`。

**适用场景：** 所有还在使用 MD5 认证的系统，这是计划弃用路线图上的第一步警告。

---

### 42. 禁止数据库、角色、表空间名称包含 CR/LF

**选取原因：** 名称中包含换行符会导致日志注入、配置文件解析异常等安全隐患，属于需要主动堵上的安全漏洞。

**之前的问题：** PostgreSQL 允许在数据库名、角色名、表空间名中包含回车（`\r`）和换行（`\n`）字符，恶意用户可以利用这一点伪造日志条目或注入 pg_hba.conf 配置片段。

**PG 19 的改变：** 相关 DDL 直接拒绝包含 CR/LF 字符的名称，并给出清晰的错误提示。

**适用场景：** 所有对外提供数据库创建接口的系统（如 DBaaS 平台），需要审查现有对象名是否包含此类字符（可查询系统目录排查）。

---

### 43. 统计视图补充 stats_reset 列

**选取原因：** 统计数据的时效性对于监控告警至关重要，知道"这组数据是什么时候开始统计的"才能正确解读数值。

**之前的问题：** `pg_statio_all_sequences` 和 `pg_stat_database_conflicts` 等视图缺少 `stats_reset` 列，在做环比分析或判断数据新鲜度时无从依据。

**PG 19 的改变：** 新增 `stats_reset` 列到：
- `pg_statio_all_sequences`（及相关视图）
- `pg_stat_database_conflicts`

```sql
SELECT schemaname, relname, blks_read, blks_hit, stats_reset
FROM pg_statio_all_sequences
ORDER BY blks_read DESC;
```

**适用场景：** 监控系统的数据一致性检查、滚动统计分析、重置统计后的基线建立确认。

---

## 特性总览

| # | 特性 | 类别 | 受益者 |
|---|------|------|--------|
| 1 | REPACK 命令 + CONCURRENTLY | 运维 | DBA |
| 2 | 在线启用数据校验 | 运维 | DBA |
| 3 | autovacuum 并行 Worker | 运维 | DBA |
| 4 | max_locks_per_transaction 默认 128 | 运维 | DBA / 开发者 |
| 5 | XID 警告阈值提升至 1 亿 | 运维 | DBA |
| 6 | EXPLAIN (IO) | 可观测性 | DBA / 开发者 |
| 7 | pg_stat_autovacuum_scores | 可观测性 | DBA |
| 8 | pg_stat_recovery | 可观测性 | DBA |
| 9 | pg_stat_lock | 可观测性 | DBA |
| 10 | log_min_messages 按进程类型分级 | 可观测性 | DBA |
| 11 | UPDATE/DELETE FOR PORTION OF | SQL 功能 | 开发者 |
| 12 | INSERT ... ON CONFLICT DO SELECT | SQL 功能 | 开发者 |
| 13 | SQL/PGQ 图查询 | SQL 功能 | 开发者 |
| 14 | NOT IN 自动转 Anti-Join | 查询优化 | 开发者 |
| 15 | CREATE SCHEMA 支持更多对象 | SQL 功能 | 开发者 / DBA |
| 16 | FK 批量索引探测（~2.9x 加速）| 性能 | 开发者 |
| 17 | COPY FROM SIMD 加速 | 性能 | DBA / 开发者 |
| 18 | 元组解包优化 | 性能 | 全局受益 |
| 19 | wal_sender_shutdown_timeout | 高可用 | DBA |
| 20 | CREATE SUBSCRIPTION ... SERVER | 复制 | DBA |
| 21 | SPLIT / MERGE PARTITION DDL | 分区管理 | DBA |
| 22 | autovacuum 表优先级评分调度 | 运维 | DBA |
| 23 | pg_get_database/tablespace/role_ddl() | 工具 | DBA |
| 24 | COPY TO 支持 JSON 格式 | SQL 功能 | 开发者 |
| 25 | vacuumdb --dry-run | 工具 | DBA |
| 26 | pg_stat_progress_vacuum/analyze 新增列 | 可观测性 | DBA |
| 27 | LISTEN/NOTIFY 性能优化 | 性能 | 开发者 |
| 28 | wal_level=replica 时动态启用逻辑解码 | 复制 | DBA |
| 29 | 逻辑复制初始同步优化（分区表）| 复制 | DBA |
| 30 | 订阅级别 wal_receiver_timeout | 复制 | DBA |
| 31 | jsonpath 新增字符串方法 | SQL 功能 | 开发者 |
| 32 | CHECK 约束 ENFORCED/NOT ENFORCED | SQL 功能 | DBA / 开发者 |
| 33 | 密码即将过期提前告警 | 安全 | DBA |
| 34 | 并行 TID Range Scan | 性能 | DBA / 开发者 |
| 35 | pg_available_extensions 新增 location 列 | 可观测性 | DBA |
| 36 | pg_buffercache 强制置脏函数 | 工具 | DBA |
| 37 | 进程终止信号记录发送方 PID/UID | 可观测性 | DBA |
| 38 | checkpoint 完成日志包含请求标志 | 可观测性 | DBA |
| 39 | pg_restore --no-globals | 工具 | DBA |
| 40 | standard_conforming_strings 永久 ON | 安全 | 全局 |
| 41 | MD5 密码认证弃用警告 | 安全 | DBA |
| 42 | 禁止对象名包含 CR/LF | 安全 | DBA |
| 43 | 统计视图补充 stats_reset 列 | 可观测性 | DBA |

---

## 结语

PostgreSQL 19 在延续其一贯工程严谨性的同时，呈现出几个明显的趋势：

**可观测性全面跃升**：EXPLAIN (IO)、pg_stat_autovacuum_scores、pg_stat_recovery、pg_stat_lock、pg_stat_progress_vacuum 的 mode/started_by 列、checkpoint 完整日志等，形成了一套从查询到系统级别的完整诊断体系，让 DBA 从"凭经验猜"走向"有数据说话"。

**运维操作在线化**：在线开启数据校验、REPACK CONCURRENTLY、动态 wal_level 调节，体现了 PostgreSQL 对零停机运维的持续投入，与云原生时代的运维需求高度契合。

**SQL 标准深度跟进**：SQL/PGQ 图查询、FOR PORTION OF 时态数据、CHECK 约束 NOT ENFORCED，让 PostgreSQL 在 SQL 标准合规性上大步前行。

**性能优化无处不在**：从元组解包的底层汇编级优化到 FK 批量检查、SIMD COPY 加速、LISTEN/NOTIFY 无效唤醒消除，性能提升渗透到多个关键路径，且对应用透明，升级即受益。

**安全加固稳步推进**：MD5 弃用警告、standard_conforming_strings 永久锁定、对象名特殊字符拦截，PG 19 在安全基线上向前迈了重要一步。

**升级注意事项清单：**
1. `max_locks_per_transaction` 默认值从 64 变为 128，若之前手动设置需评估是否同步翻倍
2. `standard_conforming_strings` 不再允许设为 OFF，检查应用层字符串转义处理
3. 数据库/角色/表空间名称不能包含 CR/LF，迁移前检查存量对象名
4. MD5 密码将持续收到警告，建议迁移计划中优先安排 SCRAM 切换
5. 充分利用 `vacuumdb --dry-run` 在维护窗口前预验证操作范围
