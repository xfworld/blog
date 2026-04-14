# Workflows

Use these repeatable workflows to produce stable, verifiable output.

## PostgreSQL/PolarDB Technical Answer

1. Restate the scenario: workload type, data size, concurrency, SLA, deployment shape, and current symptom.
2. Search blog evidence by category and keyword.
3. Explain the mechanism before the fix: storage, index, optimizer, MVCC, lock, WAL, replication, memory, IO, or network.
4. Provide the practical path:
   - immediate mitigation
   - root-cause confirmation
   - durable design or configuration change
   - regression guard
5. Include SQL/commands where possible.
6. State assumptions and when the recommendation fails.

Output skeleton:

```markdown
## 结论
## 判断依据
## 排查/实施步骤
## SQL或命令
## 风险与回滚
## 验收标准
## 参考
```

## Performance Diagnosis

Order checks by probability and reversibility:

1. Confirm symptom: latency, QPS/TPS, CPU, IO, locks, bloat, checkpoint, WAL, autovacuum, network, connection count.
2. Find top SQL: `pg_stat_statements`, slow log, `auto_explain`, `EXPLAIN (ANALYZE, BUFFERS)`.
3. Check plan mismatch: wrong row estimate, missing/unused index, correlation, stale stats, parameter sensitivity.
4. Check memory and temp files: `work_mem`, hash/sort spill, parallelism, connection fan-out.
5. Check MVCC maintenance: dead tuples, autovacuum lag, freeze, bloat.
6. Check contention: row locks, relation locks, LWLock, IO queue, WAL sync.
7. Convert diagnosis into a minimal change and verify with before/after metrics.

Never prescribe a parameter blindly; bind it to workload, memory budget, and failure mode.

## Evidence Pack

Before giving a non-trivial answer, collect enough evidence to avoid overfitting to one post:

1. Search the exact keyword and at least one synonym, for example `pgvector`, `向量`, `HNSW`.
2. Prefer a series summary or category index when the user asks for a learning path or broad overview.
3. Prefer the newest dated posts for recent feature previews, but verify current external status with official sources when the answer depends on today's release state.
4. For code or commit claims, inspect the actual code/diff in addition to blog posts.
5. Cite repository-relative paths in the final answer.

Use JSON output when the search result is passed into another tool:

```bash
python3 skills/digoal/scripts/search_blog.py "pgvector" --json --limit 5
```

## Architecture and Selection

Start from first principles:

- What value must the database protect or accelerate?
- What is the dominant access pattern: OLTP, OLAP, HTAP, search, vector, graph, GIS, time-series, stream?
- What are scale dimensions: data volume, write rate, read rate, retention, tenant count, region count?
- What are hard constraints: RPO/RTO, consistency, compliance, cost, team skill, ecosystem?
- What failure mode is unacceptable?

Compare options with a table: `方案`, `适合场景`, `代价`, `风险`, `验证方法`.

## Article Writing in digoal Style

Use when asked to create public-account articles, project interpretations, course notes, or technical explainers.

1. Pick a sharp title that names the consequence, not just the feature.
2. Open with a concrete pain point or surprising observation.
3. State the core judgment early.
4. Explain from mechanism or first principles.
5. Add evidence: official docs, code, benchmark, case, or blog precedent.
6. Give a practical method or checklist.
7. Add caveats: what assumptions must hold and what happens if they fail.
8. Close with a short takeaway or "你怎么看" interaction when publishing style is requested.

Sections for PG commit/new feature interpretation:

```markdown
# 标题
## 摘要
关键词:
背景痛点:
核心价值:
## 原理介绍
## 应用场景及最佳实践
## 风险、限制与版本注意
## 小结和思考
## 参考
```

## Source Code or Commit Interpretation

1. Inspect the exact commit, diff, tests, docs, and nearby code.
2. Separate "what changed" from "why it matters".
3. Translate internals for the target reader: DBA, developer, architect, or product manager.
4. Avoid unsupported performance claims unless benchmark or code path proves them.
5. Verify article claims against code before final output.
6. If writing multiple files, keep each topic in an independent context and avoid cross-contamination.

## AI Skill/Digital Employee Distillation

Use the framework from `202603/20260317_03.md`, `202603/20260319_05.md`, and `202603/20260324_06.md`.

For each skill, define:

- Trigger: when should it be used?
- Input contract: what information is required?
- Decision path: how does it choose a workflow?
- Tools: what files, MCPs, CLIs, scripts, or source systems are used?
- Validation: how is correctness checked?
- Fallback: what happens when evidence is missing or tool calls fail?
- Output template: what artifact is produced?
- Responsibility boundary: what the skill must not do.

Strong skill criteria:

- 可验证: output can be checked.
- 可复用: procedure works repeatedly.
- 可规模化: not dependent on one-off genius.
- 可追责: inputs, steps, and boundaries are clear.

## Community and Ecosystem Strategy

Use digoal's community/product background when asked about open-source growth, database product promotion, or developer ecosystem.

Frame with:

- User pain and adoption barrier.
- Product core value and differentiated scenario.
- Content system: tutorials, labs, cases, benchmarks, migration guides, best practices.
- Community mechanism: contributors, users, partners, training, events, support channels.
- Flywheel: content -> trust -> usage -> feedback -> cases -> ecosystem.
- Metrics: users, deployments, contributors, issues, stars, courses, conversions, enterprise references.

Avoid vanity metrics without tying them to adoption or business value.
