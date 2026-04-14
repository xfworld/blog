# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is **digoal's personal technical blog** (德哥), primarily focused on PostgreSQL, database technologies, AI/ML, and personal insights ("德说" series). The repository contains ~5000+ markdown blog posts and has ~8.3k GitHub stars.

## Repository Structure

```
blog/
├── YYYYMM/              # Blog posts organized by year-month (e.g., 202604/)
│   └── YYYYMMDD_NN.md   # Blog files: date + optional sequence number (_01, _02...)
├── class/               # 36 category index files (1.md to 36.md)
│   ├── 1.md  (应用开发)
│   ├── 2.md  (日常维护)
│   ├── 7.md  (问题诊断与性能优化)
│   └── ... (35 categories total)
├── me/                  # Author biography
├── pic/                 # Images
├── pdf/                 # PDF files
├── backup/              # Backup files
├── old_blogs_from_163/  # Pre-2018 migrated articles from 163 blog
├── generate_readme.sh   # Regenerates README.md from all blog posts
└── README.md            # Auto-generated index of all posts
```

## File Naming Convention

- **Blog posts**: `YYYYMM/YYYYMMDD_NN.md`
  - Example: `202604/20260413_01.md`
  - Multiple posts on same day use `_02`, `_03`, etc.
- **Category files**: `class/N.md` (1-36, each representing a topic category)

## Blog Post Format

Standard header in each blog post:
```markdown
# Title

### 作者
digoal

### 标签
[tag1] [tag2] ...

### 分类
[category reference]

### 日期
YYYY-MM-DD
```

## Key Topics Covered

- PostgreSQL (PG) - features, optimization,内核源码, extensions
- PolarDB, Greenplum, OceanBase, DuckDB
- Vector databases (pgvector, Milvus, VectorChord)
- AI/ML integration with databases
- "德说" (德哥's personal insights on tech, economics, philosophy)
- Database security, HA, performance tuning
- Database source code learning series (源码入门学习)

## Adding New Content

1. **New blog post**: Create file in appropriate `YYYYMM/` folder
2. **New category**: Add to `class/N.md` following existing pattern
3. **Update README**: Run `./generate_readme.sh` (in blog root)

## Scripts

- `generate_readme.sh` - Scans all blog posts and regenerates README.md with chronological index
- `class/get_class.sh` - Extracts category information from posts
- `class/ins_tag.sh` / `class/del_tag.sh` - Tag management utilities

## Important Notes

- The `197001/` folder contains pre-2018 migrated articles (163 blog era)
- Most articles are in Chinese
- Posts from 2024+ cover PostgreSQL 19 previews, AI + database integration
- Source code learning series are stored as multi-article collections (e.g., `202510/20251016_11.md`)

# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
