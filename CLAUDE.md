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
