# PostgreSQL 19 对 DBA 和应用开发者最有价值的 50 个特性

> 本文基于 `markdown/gitlog_filtered.md` 中的 PostgreSQL 19 commit log 归纳整理。因为依据是开发分支提交日志，而不是最终 release note，特性名称、语法细节、默认值和限制条件仍应以 PostgreSQL 19 正式发布文档为准。

PostgreSQL 19 的变化可以概括为两条主线：一条是让 DBA 更容易做在线维护、复制切换、备份恢复、性能诊断；另一条是让应用开发者用更少 SQL 和更少应用层代码表达复杂业务逻辑。下面精选 50 个最值得 DBA 和用户重点关注的特性。

## 一、在线维护与 autovacuum

### 1. `REPACK` 统一表重写和物理整理入口

相关 commit：`ac58465e`

**选择原因**：表膨胀治理、空间回收、物理重排是 DBA 最常见也最危险的维护动作之一。PG19 引入 `REPACK`，把 `VACUUM FULL` 和 `CLUSTER` 背后的表重写能力统一到一个更清晰的命令模型。

**之前的问题**：`VACUUM FULL`、`CLUSTER` 命令入口分散，锁行为和语义不够统一；`CLUSTER` 这个词本身也容易和“数据库集群”混淆。

**之后的价值**：DBA 可以用 `REPACK` 作为“重写表/整理表”的统一心智模型。旧命令仍保留，但文档会弱化旧入口，为后续并发整理和更多表重写模式打基础。

**适用场景**：表膨胀治理、空间回收、按索引重排、维护命令标准化。

### 2. `REPACK CONCURRENTLY` 支持在线表重组

相关 commit：`28d534e2`, `e76d8c74`, `0d3dba38`

**选择原因**：这是 PG19 对生产 DBA 最有价值的能力之一。大表在线整理长期依赖外部工具或停机窗口，内核提供并发重组能力意义很大。

**之前的问题**：传统表重写需要较强锁，大表操作容易阻塞业务；DBA 常常只能等待低峰窗口，或者接受长时间锁表风险。

**之后的价值**：`REPACK CONCURRENTLY` 在创建新表副本期间使用较弱锁，通过 MVCC snapshot 和逻辑解码捕获并发变更，最后切换 relfilenode 时才短暂需要强锁。新增 `max_repack_replication_slots` 可以为并发 repack 单独预留 replication slot。

**适用场景**：高并发 OLTP 大表膨胀治理、SaaS/金融/电商等停机窗口很小的系统。

**注意**：commit log 明确提到仍有 replication slot、并发数量和最终锁升级等限制，生产使用前必须压测和演练。

### 3. autovacuum 可以使用并行 vacuum worker

相关 commit：`1ff3180c`

**选择原因**：autovacuum 是 PostgreSQL 长期稳定运行的核心后台机制。让 autovacuum 利用并行索引清理，对大表和多索引表非常实用。

**之前的问题**：手工 `VACUUM` 可以并行，但 autovacuum 过去总是禁用并行 vacuum。结果是最需要自动维护的大表，在后台维护时反而跑得慢。

**之后的价值**：PG19 允许 autovacuum 使用 parallel vacuum 基础设施，并提供实例级 `autovacuum_max_parallel_workers` 和表级 `autovacuum_parallel_workers` 控制。DBA 可以先对热点大表逐步启用，不必全局激进放开。

**适用场景**：多索引大表、频繁更新删除表、膨胀风险高的 OLTP 表、维护窗口有限的系统。

### 4. autovacuum 引入表优先级评分

相关 commit：`d7965d65`, `53b8ca68`

**选择原因**：autovacuum 不只是“是否触发”，还涉及“先处理谁”。PG19 开始把 autovacuum 从简单扫描推进到基于评分的调度。

**之前的问题**：autovacuum worker 收集待处理表后基本按列表顺序处理，某些更接近 wraparound 或更严重超过阈值的表，不一定能优先处理。

**之后的价值**：PG19 使用 score 对候选表排序，考虑 freeze 年龄、multixact 年龄、dead tuple、insert、analyze 等多个维度，并提供 `autovacuum_freeze_score_weight` 等权重参数。DBA 可以更主动地影响 autovacuum 的资源投向。

**适用场景**：库里有大量表、冷热表差异明显、wraparound 风险管理严格的环境。

### 5. `pg_stat_autovacuum_scores` 暴露 autovacuum 决策依据

相关 commit：`87f61f0c`

**选择原因**：这是 autovacuum 调优可观测性的关键补充。DBA 可以直接看到每张表的 autovacuum score。

**之前的问题**：判断“为什么 autovacuum 还没跑这张表”通常要 DBA 手工根据统计视图、阈值参数和 freeze 年龄推算，容易误判。

**之后的价值**：新视图显示当前数据库每张表的 autovacuum score，以及 autovacuum 是否会 vacuum/analyze 该表。调优从经验判断变成数据驱动。

**适用场景**：autovacuum 参数调优、wraparound 风险排查、表膨胀治理。

### 6. `log_autoanalyze_min_duration` 拆分 autoanalyze 日志阈值

相关 commit：`dd3ae378`

**选择原因**：统计信息维护和 vacuum 维护是两类问题，分开记录对 DBA 判断计划问题很有价值。

**之前的问题**：`log_autovacuum_min_duration` 同时控制 autovacuum 的 vacuum 和 analyze 日志，DBA 无法单独观察耗时 autoanalyze。

**之后的价值**：PG19 新增 `log_autoanalyze_min_duration`，可以单独记录自动 analyze 的慢操作，更清楚地分析统计信息刷新成本和 planner 计划波动。

**适用场景**：查询计划频繁变化、统计信息维护成本高、希望降低 vacuum 日志噪音但保留 analyze 观测的系统。

### 7. VACUUM/autovacuum 日志展示并行 worker 和 dead items 内存

相关 commit：`adcdbe93`, `736f754e`

**选择原因**：启用并行 vacuum 后，DBA 需要确认实际并行度；vacuum 慢时，也需要知道是否受 dead item 存储和内存限制影响。

**之前的问题**：日志不容易判断某次 vacuum 实际用了多少 worker，也不容易判断 dead items storage 是否频繁 reset 或内存不足。

**之后的价值**：PG19 在 `VACUUM VERBOSE` 和 autovacuum 日志中补充 parallel worker 使用情况、dead items storage 分配内存、reset 次数和配置上限。

**适用场景**：并行 autovacuum 上线验证、`maintenance_work_mem` 调优、大表删除后的 vacuum 排查。

### 8. vacuum WAL 量减少

相关 commit：`1252a4ee`, `a759ced2`, `add323da`, `d96f8733`

**选择原因**：vacuum 产生的 WAL 会影响主库 I/O、归档、复制延迟和备份链路。减少 WAL 是直接的运维收益。

**之前的问题**：vacuum 设置 all-visible/all-frozen 等可见性映射时，可能产生额外 WAL 记录，增加写放大和复制压力。

**之后的价值**：PG19 将部分 VM 设置合并进已有 prune/freeze WAL 记录，减少 vacuum WAL 量。对高 churn 表，复制链路和归档链路压力都会下降。

**适用场景**：归档量大、物理复制延迟敏感、更新删除频繁、vacuum 对 I/O 影响明显的生产库。

### 9. on-access pruning 可设置 all-visible，提升 index-only scan 机会

相关 commit：`b46e1e54`, `378a2161`, `01b7e4a4`, `4f7ecca8`

**选择原因**：index-only scan 是否能发挥作用，很大程度依赖 visibility map。PG19 让普通访问路径也能更积极维护 VM 状态。

**之前的问题**：很多页面只有 vacuum 或 `COPY FREEZE` 才会标记 all-visible/all-frozen，新插入或访问时发现可见性满足条件，也未必及时反映到 VM。

**之后的价值**：PG19 让 on-access pruning 能设置页面 all-visible，并增强 VM 损坏检测和修复。新数据更早受益于 index-only scan，vacuum 后续负担也可能降低。

**适用场景**：读多写少但持续有增量写入的表、依赖覆盖索引的查询、希望减少 heap fetch 的 OLTP/HTAP 场景。

## 二、复制、高可用与恢复

### 10. `WAIT FOR` 原生命令等待 WAL 到达指定状态

相关 commit：`447aae13`, `3b4e53a0`, `49a181b5`, `7a39f43d`

**选择原因**：读写分离里“写后读一致性”是高频业务问题。PG19 用原生命令解决脚本轮询的痛点。

**之前的问题**：应用在主库写入后，如果马上去 standby 读取，可能读不到刚写的数据。过去通常依赖轮询 `pg_stat_replication` 或 replay LSN 函数，逻辑分散且容易持有 snapshot 引发问题。

**之后的价值**：`WAIT FOR` 可以等待 standby replay/write/flush 或 primary flush 到指定 LSN，并通过共享内存等待基础设施高效唤醒等待者。

**适用场景**：读写分离一致性、主备切换前校验、发布系统等待副本追平、自动化运维脚本。

### 11. `pg_stat_recovery` 统一展示 standby 恢复状态

相关 commit：`01d485b1`

**选择原因**：standby 故障排查最怕信息分散。新视图把恢复状态整理成一致快照。

**之前的问题**：回放位置、暂停状态、promotion trigger、最后事务回放时间等信息分散在多个函数中，且不一定来自同一个一致状态点。

**之后的价值**：`pg_stat_recovery` 通过一次共享内存读取暴露恢复状态，包括最近成功回放 WAL、当前回放位置、当前 WAL chunk 时间、promotion trigger、最近 commit/abort 时间戳、recovery pause 状态等。

**适用场景**：standby 延迟排查、恢复卡住诊断、切换前健康检查、只读副本监控面板。

### 12. WAL receiver 增加 `CONNECTING` 状态，复制监控更准确

相关 commit：`a36164e7`

**选择原因**：高延迟网络或跨地域复制中，连接建立阶段和真正 streaming 阶段需要区分。

**之前的问题**：WAL receiver 新启动时很快标记为 streaming，但实际上复制连接可能还没建立完成，监控容易误判。

**之后的价值**：新增 `WALRCV_CONNECTING` 状态，DBA 可以区分“正在连接上游”和“已经开始流式接收”。

**适用场景**：跨地域容灾、网络抖动排查、复制连接建立慢、自动化故障诊断。

### 13. standby 状态回复避免重复 WAL 位置

相关 commit：`400a790a`

**选择原因**：复制延迟监控依赖 standby status reply。重复位置会让延迟指标失真。

**之前的问题**：startup process 应用 WAL 后可能请求 walreceiver 发送状态，即使 WAL 位置没有前进，也会发出重复状态。这可能重置上游看到的 replication lag。

**之后的价值**：PG19 仅在 apply 位置前进时发送该类回复，减少无意义消息，避免延迟监控被“假进展”干扰。

**适用场景**：基于 `pg_stat_replication` 的监控告警、SLA 延迟看板、自动故障切换判断。

### 14. `wal_sender_shutdown_timeout` 限制 shutdown 等待复制的时间

相关 commit：`a8f45dee`

**选择原因**：数据库 shutdown 被复制链路拖住，是生产变更和故障处理中的真实痛点。

**之前的问题**：walsender 在关闭时会等待 pending 数据复制给 receiver。如果逻辑复制 apply worker 被锁阻塞，shutdown 可能长时间无法完成。

**之后的价值**：新增 `wal_sender_shutdown_timeout`，允许设置 walsender shutdown 最长等待时间。默认 `-1` 保持旧行为，DBA 可以按业务权衡停机速度和复制一致性。

**适用场景**：逻辑复制链路较多、停机窗口严格、应用锁阻塞可能拖慢 shutdown 的环境。

### 15. subscription 支持单独设置 `wal_receiver_timeout`

相关 commit：`fb80f388`, `8a6af3ad`

**选择原因**：不同订阅链路的网络质量、业务重要性不同，全局超时不够精细。

**之前的问题**：`wal_receiver_timeout` 是全局设置，无法对某个跨地域或低质量链路单独放宽，也无法对关键链路单独收紧。

**之后的价值**：PG19 允许在 `CREATE SUBSCRIPTION` 和 `ALTER SUBSCRIPTION` 中为订阅设置 `wal_receiver_timeout`，覆盖全局值。

**适用场景**：多订阅、多地域逻辑复制、不同 SLA 的数据分发链路、单独调优某条订阅的断链检测。

### 16. 逻辑复制支持 sequence 发布、刷新和同步

相关 commit：`96b37849`, `f0b3573c`, `5509055d`, `b93172ca`

**选择原因**：sequence 是逻辑复制和低停机迁移里长期容易遗漏的对象。PG19 把 sequence 纳入逻辑复制闭环。

**之前的问题**：逻辑复制主要围绕表，sequence 值需要额外处理。迁移或升级后 sequence 不一致，可能导致主键冲突或号段回退。

**之后的价值**：publication 支持 `FOR ALL SEQUENCES`；subscription 支持 `REFRESH SEQUENCES`；sequencesync worker 可批量同步 sequence 当前值和 page LSN，完成后标记为 `READY`。

**适用场景**：低停机升级、主从切换、逻辑复制迁移、sequence 密集型应用。

### 17. 逻辑复制保留冲突检测相关数据

相关 commit：`228c3708`

**选择原因**：多主、双向复制或复杂冲突处理方案最怕错误识别冲突。PG19 增强了冲突检测可靠性。

**之前的问题**：VACUUM 可能过早清理 subscriber 上被其他 origin 删除的 tuple 或 commit timestamp 元数据，使后续 UPDATE 被误判为 INSERT。

**之后的价值**：订阅可通过 `retain_conflict_info` 保留冲突检测需要的 dead tuple 和 origin 元数据，并由 `pg_conflict_detection` slot 维护必要 xmin。

**适用场景**：双向逻辑复制、多源写入、需要记录或解决复制冲突的高级复制拓扑。

### 18. `pg_stat_replication_slots` 增加 logical decoding 内存超限计数

相关 commit：`d3b6183d`

**选择原因**：逻辑解码内存超限以前很难量化频率，只能看到现象。

**之前的问题**：`logical_decoding_work_mem` 不足时可能导致 spill 或性能下降，但 DBA 不容易知道这是偶发还是频繁问题。

**之后的价值**：`mem_exceeded_count` 统计逻辑解码内存超过限制的次数，帮助判断是否需要调整 `logical_decoding_work_mem` 或优化订阅消费能力。

**适用场景**：逻辑复制延迟排查、解码吞吐优化、大事务复制、CDC 平台运维。

### 19. replication slot 保存 restart LSN，避免 WAL 被意外清理

相关 commit：`ca307d5c`

**选择原因**：WAL 被 slot 误清理会直接影响复制连续性和恢复能力。

**之前的问题**：checkpoint 开始后 slot 被推进、checkpoint 结束清理旧 WAL 时，可能因为 restart LSN 状态边界不够稳妥，导致旧 WAL 段被意外移除。

**之后的价值**：新增 `last_saved_restart_lsn` 内存状态，用 slot 最后持久化的 restart LSN 参与 WAL 保留判断，降低 checkpoint/restart 边界条件下的 WAL 丢失风险。

**适用场景**：逻辑复制、物理 slot、频繁 checkpoint、复制链路需要强可靠性的环境。

## 三、监控、诊断与可观测性

### 20. `EXPLAIN (IO)` 观察扫描节点 I/O 和预取行为

相关 commit：`681daed9`, `3b1117d6`, `e157fe6f`, `61c36a34`

**选择原因**：过去 `BUFFERS` 只能告诉你读了多少 buffer，不能解释底层 I/O 请求和预取行为。PG19 开始补上这层观测。

**之前的问题**：查询慢时，DBA 很难判断扫描节点是否有效预取、AIO 行为如何、I/O 请求是否合理。

**之后的价值**：`EXPLAIN (ANALYZE, IO)` 可显示基于 ReadStream 的扫描节点的预取距离和 I/O 请求信息，覆盖 BitmapHeapScan、SeqScan、TidRangeScan。`auto_explain.log_io` 允许自动记录慢 SQL 的 I/O 细节。

**适用场景**：I/O 型慢 SQL 分析、顺序扫描/Bitmap 扫描调优、AIO 效果验证、存储层瓶颈定位。

### 21. `pg_stat_lock` 提供锁等待统计

相关 commit：`4019f725`

**选择原因**：锁问题是 DBA 日常排障最高频的问题之一，单看 `pg_locks` 只能看到当前，不容易做历史统计。

**之前的问题**：锁等待次数、累计等待时间、fast-path lock slot 不足等信息缺少统一统计。DBA 只能靠日志采样或外部监控近似推断。

**之后的价值**：PG19 新增 `pg_stat_lock`，记录锁等待次数、等待时间、fastpath_exceeded 次数。只在超过 `deadlock_timeout` 且成功拿锁后计入，避免严重影响热路径。

**适用场景**：锁争用热点分析、事务设计评估、DDL/DML 冲突排查、`max_locks_per_transaction` 容量判断。

### 22. `pg_stat_statements` 增加 generic/custom plan 计数

相关 commit：`3357471c`

**选择原因**：prepared statement 性能问题常常来自 generic plan 与 custom plan 的选择。

**之前的问题**：`pg_prepared_statements` 只能看当前会话，无法按 query 维度全局观察某类 SQL 到底用了多少 generic/custom plan。

**之后的价值**：`pg_stat_statements` 增加 `generic_plan_calls` 和 `custom_plan_calls`，DBA 可以从全局视角判断 prepared statement 计划缓存行为。

**适用场景**：ORM 参数化 SQL 调优、计划缓存误判、`plan_cache_mode` 评估、慢 SQL 偶发性分析。

### 23. `EXPLAIN` 显示 Memoize 估算细节

相关 commit：`4bc62b86`

**选择原因**：Memoize 计划是否合理，取决于缓存条目数、唯一 key 数、lookup 次数和命中率估算。PG19 让这些更透明。

**之前的问题**：看到 planner 选择 Memoize 但不知道为什么，难以判断是统计信息问题还是成本模型合理选择。

**之后的价值**：`EXPLAIN` 显示 Memoize 估计可存 cache entries、unique lookup keys、lookups 和 hit ratio。调优方向更明确，比如补扩展统计或调整 n_distinct。

**适用场景**：嵌套循环 + Memoize 计划诊断、关联查询性能不稳定、统计信息偏差分析。

### 24. 进度和统计视图补充关键字段

相关 commit：`deb67445`, `0d789520`, `ab40db38`, `a5b54325`, `b71bae41`, `8fe315f1`, `723619ea`

**选择原因**：监控数据必须说明任务类型、来源和统计窗口，否则容易误判。

**之前的问题**：base backup 进度不易区分全量/增量；vacuum/analyze 进度不易区分来源；多个统计视图缺少 `stats_reset`。

**之后的价值**：`pg_stat_progress_basebackup` 增加 `backup_type`；`pg_stat_progress_vacuum/analyze` 增加 `mode`/`started_by`；多类统计视图补充 `stats_reset`。

**适用场景**：备份监控、维护任务看板、性能报表、故障复盘。

### 25. 日志显示终止 backend 的来源 PID/UID

相关 commit：`55890a91`

**选择原因**：生产中“谁 kill 了我的连接”是典型审计问题。

**之前的问题**：backend 被 `pg_terminate_backend()` 或外部 SIGTERM 终止后，日志里不容易定位发起者。

**之后的价值**：PG19 在支持 `SA_SIGINFO` 的平台上把 sender PID 和 UID 放进 errdetail，帮助识别来源。

**适用场景**：连接异常断开排查、运维审计、多用户共享数据库平台、误操作追踪。

### 26. checkpoint 命令和日志更可控

相关 commit：`a4f12651`, `2f698d7f`, `8d33fbac`, `5b93a598`

**选择原因**：checkpoint 是 I/O 抖动、停机、备份和性能分析的重要节点。

**之前的问题**：手工 `CHECKPOINT` 缺少更细模式控制；completion log 缺少 request flags，DBA 需要回看 start log 才能还原上下文。

**之后的价值**：PG19 为 `CHECKPOINT` 增加选项列表，支持 `MODE FAST/SPREAD` 和 `FLUSH_UNLOGGED`，并在 checkpoint completion 日志中包含 request flags。

**适用场景**：手工 checkpoint 控制、变更前刷盘、unlogged 表风险管理、I/O 抖动诊断。

### 27. WAL 工具错误信息更详细并支持 tar WAL

相关 commit：`1c162c96`, `b15c1513`, `b3cf461b`, `f8a0cd26`

**选择原因**：WAL 诊断直接关系到恢复、备份校验和故障分析。

**之前的问题**：WAL record 查找失败时错误泛泛；tar 格式备份里的 WAL 分析也不方便。

**之后的价值**：`pg_waldump`、`pg_walinspect`、WAL summarizer 可以报告更具体的 WAL 解析错误；`pg_waldump` 支持从 tar archive 读取 WAL，`pg_verifybackup` 对 tar 格式备份启用 WAL parsing。

**适用场景**：WAL 损坏分析、备份校验、归档文件排查、恢复演练。

### 28. wraparound 警告更早且带可用 ID 百分比

相关 commit：`48f11bfa`, `e646450e`

**选择原因**：事务 ID 和 multixact wraparound 是 PostgreSQL 最严重的运维风险之一。

**之前的问题**：警告阈值和提示信息不够直观，DBA 不容易快速判断剩余风险空间。

**之后的价值**：PG19 将 transaction/multixact ID 警告限制提高到 100M，并在 wraparound 警告中加入可用 ID 百分比，风险表达更清楚。

**适用场景**：长事务治理、autovacuum 失效排查、wraparound 风险值班告警。

## 四、性能优化与优化器增强

### 29. 外键检查 fast path 和批量 index probe

相关 commit：`2da86c1e`, `b7b27eb4`

**选择原因**：外键检查是很多 OLTP 写入和批量导入的核心成本。PG19 直接优化 RI trigger 路径，价值很高。

**之前的问题**：外键检查通过 SPI 路径逐行查询 PK 表，批量插入时重复 permission check、snapshot、CCI、index descent，成本高。

**之后的价值**：PG19 为外键 referential check 增加 fast path，直接探测 referenced table 的唯一索引；进一步把 FK 行缓存在 batch 中，单列 FK 可用 `SK_SEARCHARRAY` 一次批量查索引。commit 中的基准显示相对未打补丁代码可达到约 2.9 倍加速。

**适用场景**：批量导入、订单/明细类强外键模型、高并发插入、ETL 装载到带 FK 的事实表。

### 30. `COPY FROM` text/csv 使用 SIMD 优化

相关 commit：`e0a3a3fd`, `dc592a41`

**选择原因**：COPY 是 PostgreSQL 数据装载主力。解析层性能提升会直接影响导入吞吐。

**之前的问题**：text/csv COPY 解析大量字符时，分隔符、换行、转义扫描成本高，大文件导入 CPU 容易成为瓶颈。

**之后的价值**：PG19 对 `COPY FROM (FORMAT text/csv)` 使用 SIMD，并补充函数内联优化，减少解析开销。

**适用场景**：大批量 CSV 导入、离线数仓装载、日志落库、数据迁移。

### 31. AIO worker pool 自动伸缩

相关 commit：`d1c01b79`, `a9ee6688`, `999dec9e`, `6e648e35`

**选择原因**：异步 I/O 是 PostgreSQL 未来 I/O 路径的重要方向，自动伸缩降低了 DBA 调参成本。

**之前的问题**：`io_method=worker` 的 worker 数固定，不同负载下很难手工选出合适值；太少会排队，太多会浪费。

**之后的价值**：PG19 改为 `io_min_workers`、`io_max_workers`、`io_worker_idle_timeout`、`io_worker_launch_interval`，根据近期需求自动扩缩 worker pool；io_uring 路径也持续优化大 I/O 和后台完成检测。

**适用场景**：I/O 密集查询、大表扫描、存储延迟波动明显、希望启用新 AIO 路径但不想过度手工调参的环境。

### 32. streaming read 优化多个访问路径和扩展

相关 commit：`213f0079`, `d841ca2d`, `6c228755`, `4c910f3b`, `a6eac227`, `ae58189a`, `bfa3c4f1`

**选择原因**：大对象扫描、索引维护和统计类扩展经常受 I/O 效率影响。

**之前的问题**：部分扩展和索引清理路径没有充分使用 streaming read，I/O 访问不够顺滑。

**之后的价值**：PG19 将 streaming read 用到 `pgstattuple`、btree/hash index 函数、bloom vacuum/bulk deletion、GIN vacuum cleanup、BRIN vacuum scan、hash index bulk deletion 等路径，提高大对象扫描和维护效率。

**适用场景**：使用 `pgstattuple` 评估膨胀、GIN/BRIN/hash/bloom 索引维护、大表索引清理。

### 33. 默认关闭 JIT，避免短查询性能悬崖

相关 commit：`7f8c88c2`

**选择原因**：很多生产 OLTP 系统已经手工关闭 JIT，PG19 将默认行为向主流实践靠拢。

**之前的问题**：JIT 对大分析查询可能有收益，但对短查询可能编译成本远超执行收益。轻微成本估算变化可能触发 JIT，导致性能突然变差。

**之后的价值**：PG19 默认关闭 JIT，减少短查询性能悬崖。需要 JIT 的分析型系统仍可按需打开。

**适用场景**：高并发 OLTP、延迟敏感应用、ORM 生成大量短 SQL 的业务；分析型查询可针对性评估开启。

### 34. planner 支持 Eager Aggregation

相关 commit：`8e118591`

**选择原因**：聚合下推到 join 之前，是减少中间行数、改善复杂查询计划的重要方向。

**之前的问题**：传统 planner 在 scan/join 阶段看不到后续聚合机会，可能先 join 大量明细行，再做聚合，造成无谓中间结果。

**之后的价值**：PG19 引入 eager aggregation，在可行时把部分聚合推到 join tree 更低层，先减少行数再 join，从而可能得到更优计划。

**适用场景**：星型模型、报表 SQL、join 后 group by、明细表很大但聚合后基数显著下降的分析场景。

### 35. `NOT IN` 和 NULL 相关表达式优化增强

相关 commit：`383eb21e`, `c95cd299`, `cf74558f`, `0a379612`, `0aaf0de7`, `f41ab515`, `cb7b7ec7`

**选择原因**：应用 SQL 中大量存在 `NOT IN`、外连接、NULL-safe 比较和 NOT NULL 约束。优化器越能理解这些语义，计划越稳定。

**之前的问题**：`NOT IN` 因 NULL 语义难以优化；`LEFT JOIN`、`IS DISTINCT FROM`、`BooleanTest`、`ROW(...) IS NULL` 等即使输入可证明非空，也未必能被简化。

**之后的价值**：PG19 在安全时把 `NOT IN` 转换为 anti-join，利用 NOT NULL 约束化简 outer join 和 NULL 判断，让查询更容易使用索引、hash join、merge join 和等价类推导。

**适用场景**：复杂 ORM SQL、反关联查询、外连接过滤、NULL-safe 比较大量使用的业务。

### 36. 虚拟生成列和 range 类型统计增强

相关 commit：`f7f4052a`, `307447e6`, `261f89a9`

**选择原因**：表达式列、range/multirange 在现代业务建模中越来越常见，统计质量直接影响计划。

**之前的问题**：virtual generated column 上不能建立扩展统计；range 表达式统计信息不够完整，planner 对长度分布、空 range 比例、边界分布估算有限。

**之后的价值**：PG19 允许对虚拟生成列及其表达式建立扩展统计，并在 `pg_stats_ext_exprs` 中增加 range length histogram、empty fraction、bounds histogram 等信息。

**适用场景**：JSON/文本派生列、业务状态计算列、时态有效期、价格区间、IP 段匹配。

### 37. 页面校验和和 CRC 利用硬件加速

相关 commit：`fbc57f2b`, `5e13b0f2`, `948ef7cd`

**选择原因**：checksum/CRC 是可靠性功能，但不能让 CPU 成本过高。

**之前的问题**：在大量页面校验或 CRC 计算中，通用实现可能占用明显 CPU。

**之后的价值**：PG19 在 ARM 上使用 Crypto Extension 计算 CRC32C，在 x86 上使用 AVX2 加速页面 checksum，并避免小输入走向量路径带来的回归。

**适用场景**：开启 data checksums 的生产库、大量备份校验、WAL/页面校验密集场景。

## 五、SQL 能力与应用开发体验

### 38. `GROUP BY ALL` 简化探索式聚合查询

相关 commit：`ef38a4d9`

**选择原因**：应用开发者和数据分析人员会频繁写临时聚合 SQL，减少样板代码能直接提升效率。

**之前的问题**：`SELECT` 列表里所有非聚合表达式都要手工写进 `GROUP BY`，查询改列时容易漏改、错改。

**之后的价值**：`GROUP BY ALL` 自动把目标列表中不含聚合和窗口函数的表达式加入分组列表。语法也已被 SQL 标准委员会接受，稳定性较好。

**适用场景**：BI 探索、临时报表、SQL 原型开发、宽表聚合。

### 39. 窗口函数支持 `IGNORE NULLS`/`RESPECT NULLS`

相关 commit：`25a30bbd`

**选择原因**：空值跳过是时间序列、用户行为、画像字段中非常常见的需求。

**之前的问题**：`lead`、`lag`、`first_value`、`last_value`、`nth_value` 遇到 NULL 时，用户要写复杂子查询或自定义逻辑模拟“找上一个非空值”。

**之后的价值**：PG19 为这些窗口函数加入 null treatment clause，默认 `RESPECT NULLS`，也可显式 `IGNORE NULLS`。实现中还缓存非空信息，减少重复求值。

**适用场景**：时间序列补值、行为流分析、最近一次非空状态、财务/传感器数据清洗。

### 40. `INSERT ... ON CONFLICT DO SELECT` 返回冲突行

相关 commit：`88327092`

**选择原因**：幂等创建资源时，应用最常见的需求是“插入成功返回新行，冲突则返回已有行”。

**之前的问题**：`ON CONFLICT DO NOTHING` 不返回冲突行；`DO UPDATE` 为了返回旧行可能需要写无意义更新，带来额外锁和 WAL。

**之后的价值**：PG19 新增 `ON CONFLICT DO SELECT [FOR UPDATE/SHARE]`，冲突时直接返回已存在行，并可选择加锁。语句需要 `RETURNING`。

**适用场景**：唯一资源创建、去重写入、任务幂等提交、用户注册、业务主键 upsert。

### 41. `UPDATE/DELETE FOR PORTION OF` 原生支持时态数据局部修改

相关 commit：`8e72d914`, `5eed8ce5`

**选择原因**：合约、价格、雇佣关系等业务常用“有效期”建模。局部修改有效期历史很容易写错。

**之前的问题**：应用要自己判断 range/multirange 是否重叠、拆分左/右剩余区间、插入 leftover 行，代码复杂且并发一致性难保证。

**之后的价值**：PG19 允许 `UPDATE/DELETE ... FOR PORTION OF valid_at FROM ... TO ...`，只修改目标时间段内的历史；对部分重叠行自动截断并插入未受影响的 temporal leftovers。

**适用场景**：合约有效期、价格版本、员工任期、资费规则、主数据历史版本。

### 42. `jsonpath` 增加字符串方法

相关 commit：`bd4f879a`

**选择原因**：JSON 查询越来越常见，把字符串处理留在 jsonpath 内部能减少 SQL 拼接复杂度。

**之前的问题**：`jsonpath` 过滤中要做 lower/upper/trim/replace/split 等处理时，经常要退回 SQL 函数或写复杂表达式。

**之后的价值**：PG19 增加 `ltrim/rtrim/btrim`、`lower`、`upper`、`initcap`、`replace`、`split_part` 等 jsonpath 方法。

**适用场景**：JSON 文档搜索、半结构化数据清洗、大小写不敏感匹配、字符串规范化过滤。

### 43. SQL/PGQ 图查询进入 PostgreSQL

相关 commit：`2f094e7a`

**选择原因**：图关系查询是 SQL 标准演进方向之一，对关系型数据库内处理路径和关系网络有潜在价值。

**之前的问题**：层级、路径、网络关系通常要用递归 CTE 或应用层图计算，表达复杂。

**之后的价值**：PG19 引入 SQL Property Graph Queries 相关能力，为原生图模式匹配和路径表达打基础。

**适用场景**：关系网络、权限链路、供应链路径、社交/推荐关系、血缘分析。

### 44. 编码、类型和国际化函数增强

相关 commit：`497c1170`, `e1d91718`, `ba21f5bf`, `3b4c2b9d`, `905e4415`, `57ee3979`

**选择原因**：这些增强不一定是大功能，但能减少应用层转换代码，提高多语言和数据交换能力。

**之前的问题**：base64url/base32hex 需要应用层处理；`bytea` 与 `uuid` 转换不够直接；domain 上 `IS JSON` 支持不足；ICU tailoring 强度控制不够方便；Unicode 数据需要跟进标准。

**之后的价值**：PG19 增加 base64url/base32hex、bytea/uuid 显式转换、domain `IS JSON`、ICU collation strength、Unicode 17.0.0 数据。

**适用场景**：token/URL 编码、二进制 UUID 处理、JSON domain 校验、多语言排序和文本处理。

## 六、导入导出、迁移与备份恢复

### 45. `COPY FROM ON_ERROR SET_NULL` 增强脏数据导入韧性

相关 commit：`2a525cc9`

**选择原因**：现实数据接入经常是半脏数据，不应因为单个字段转换失败就整批失败。

**之前的问题**：字段类型转换失败通常导致 COPY 失败；跳过整行又可能丢失其他有效字段。

**之后的价值**：`ON_ERROR SET_NULL` 遇到类型转换错误时将该列置为 NULL，同时继续执行 `NOT NULL` 和 domain 约束。它提升导入韧性，但不绕过核心约束。

**适用场景**：CSV 接入、外部系统数据同步、数据湖落库、先导入后治理的 ETL 流程。

### 46. COPY 导出/导入格式能力增强

相关 commit：`4bea91f2`, `7dadd38c`, `4c0390ac`, `bc2f348e`, `26cb14ae`

**选择原因**：COPY 是数据迁移和批处理的基础工具，格式和分区支持越完善，应用层脚本越少。

**之前的问题**：分区表不能直接 `COPY TO`；导出 JSON 需要手写 `row_to_json` 或应用层转换；多行 header 文件需要预处理。

**之后的价值**：PG19 支持分区表直接 `COPY TO`，支持 `COPY TO` JSON format 和 `force_array`，`COPY FROM HEADER` 可跳过多行 header，`file_fdw` 也支持多行 header。

**适用场景**：分区历史数据导出、JSON 数据交换、第三方 CSV 导入、ETL 管道简化。

### 47. `pg_dump`/`pg_restore` 支持扩展统计数据导出恢复

相关 commit：`c32fb29e`, `0e80f3f8`, `302879bd`, `648a7e28`, `ba97bf9c`, `efbebb4e`, `c2e2589a`, `34eb2a80`

**选择原因**：迁移/升级后最容易出现的问题之一是统计信息缺失导致计划变差。

**之前的问题**：dump/restore 后需要重新 ANALYZE；大库升级后统计信息恢复慢，期间查询计划可能不稳定。外表统计和扩展统计覆盖也不完整。

**之后的价值**：PG19 增加 `pg_restore_extended_stats()` 及相关导出能力，可在指定 `--statistics` 时导出扩展统计数据，包括 ndistinct、dependencies、mcv、exprs 等；也补齐 foreign table statistics。`pg_dump/pg_dumpall` 默认改为 `--no-statistics`，避免无意导出统计，`pg_restore/pg_upgrade` 默认仍使用统计。

**适用场景**：低停机升级、跨环境恢复、需要尽快恢复计划稳定性的核心业务库、FDW 统计迁移。

### 48. `pg_upgrade` 大对象、tablespace 和升级前检查增强

相关 commit：`161a3e8b`, `158408fe`, `b33f7536`, `412036c2`, `ce11e63f`, `77645d44`, `b380a56a`, `6429e5b7`

**选择原因**：升级窗口和升级后稳定性是 DBA 最关注的风险点。

**之前的问题**：海量 large object 元数据恢复慢；in-place tablespace 处理受限；不支持 encoding 或含 CR/LF 的对象名可能在升级中才暴露；`vacuumdb --analyze-only` 会漏掉分区表。

**之后的价值**：PG19 在 binary upgrade 中用 COPY 或文件转移优化 LO 元数据；支持 in-place tablespace；`pg_upgrade` 检查不支持 encoding 和非法对象名；`vacuumdb --analyze-only` 处理分区表。

**适用场景**：大版本升级、含大量 large objects 的历史系统、复杂 tablespace 布局、升级前健康检查。

### 49. `pg_createsubscriber`、`pg_rewind` 和备份进度工具增强

相关 commit：`6b5b7eae`, `d6628a5e`, `5173bfd0`, `deb67445`

**选择原因**：复制迁移、备份恢复和 rewind 是高可用体系中的关键运维动作。

**之前的问题**：`pg_createsubscriber` 失败后日志留存不够清晰；`pg_rewind` 可能复制分歧点之前生成的 WAL；base backup 进度不区分 full/incremental。

**之后的价值**：`pg_createsubscriber -l/--logdir` 保存工具和 standby server 日志；`pg_rewind` 跳过分歧点之前的 WAL；`pg_stat_progress_basebackup.backup_type` 标识 full/incremental。

**适用场景**：低停机迁移、主备重建、自动化复制平台、备份监控。

## 七、安全、权限、FDW 与扩展生态

### 50. 安全、权限、FDW、分区和扩展生态的一组高价值增强

相关 commit：`a1643d40`, `bc60ee86`, `1d92e0c2`, `b977bd30`, `99336811`, `dd1398f1`, `d9819760`, `28972b6f`, `fd366065`, `493f8c64`, `f2e4cc42`, `4b3d1736`, `a4f774cf`, `b99fd9fd`, `76e514eb`, `5883ff30`, `e8ec19aa`, `092f3c63`, `6b1c4d32`

**选择原因**：这些提交横跨安全基线、权限审计、FDW 计划质量、逻辑复制配置、分区管理和扩展生态，单项看不一定都排进前 50，但作为一组能力对生产平台价值很高。

**之前的问题**：RADIUS/MD5 等旧认证方式存在安全债务；密码过期缺少提前提示；OAuth 集成不够灵活；权限 grantor 不够可控；FDW 统计依赖本地采样；`FOR ALL TABLES` publication 难以排除少数表；分区拆分合并依赖手工流程；全局对象 DDL 抽取和计划建议缺少内建工具。

**之后的价值**：PG19 移除 RADIUS、MD5 成功认证发警告、增加密码过期提示、增强 OAuth/libpq；`GRANT/REVOKE ... GRANTED BY` 可指定 grantor，`pg_read_all_data`/`pg_write_all_data` 覆盖 large objects；`postgres_fdw` 可导入远端统计；publication 支持 `EXCEPT TABLE`；分区表支持 `MERGE PARTITIONS`/`SPLIT PARTITION`；新增 `pg_get_database_ddl`、`pg_get_tablespace_ddl`、`pg_get_role_ddl`；`pg_plan_advice`/`pg_stash_advice` 支持专家级计划控制；libpq/psql 增强 service file 可观测性。

**适用场景**：安全合规、权限 IaC、跨库查询、逻辑复制大库发布、分区生命周期管理、升级 DDL 审计、核心 SQL 计划稳定性治理、多环境连接防误操作。

## 升级验证优先级建议

如果从 DBA 视角排序，建议优先验证：`REPACK CONCURRENTLY`、parallel autovacuum、`pg_stat_autovacuum_scores`、`WAIT FOR`、`pg_stat_recovery`、sequence logical replication、`EXPLAIN (IO)`、`pg_stat_lock`、pg_dump/pg_upgrade 统计信息恢复。

如果从应用开发者视角排序，建议优先验证：`ON CONFLICT DO SELECT`、`GROUP BY ALL`、窗口函数 `IGNORE NULLS`、`FOR PORTION OF`、`jsonpath` 字符串方法、COPY JSON/SET_NULL、多种编码和类型转换增强。

## 结语

PostgreSQL 19 不是只新增几个 SQL 语法，而是在持续补齐生产数据库的关键能力：大表如何在线整理，autovacuum 如何更聪明，逻辑复制如何更完整，standby 如何更容易观测，升级后如何更快恢复稳定计划。

对 DBA 来说，PG19 的重点是在线运维、复制一致性、可观测性和升级恢复链路。对应用开发者来说，PG19 的重点是 SQL 表达力增强、导入导出更灵活、时态和 JSON 等复杂业务模型更容易落到数据库原生能力上。
