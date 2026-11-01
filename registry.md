---
project_id: cartograph
status: PLANNED
default_mode: managed
created_date: 2026-10-29
updated_date: 2026-11-01
cli: claude
timeout_minutes: 30
---
# Project Registry: cartograph - Organizational Inventory Graph

## Description

Living inventory of an organization's holdings and how they connect, directly (shared code,
git lineage, cross-references) and conceptually (consolidation, consumption, shared themes).
A reusable pattern: v0 maps one developer's `C:\GitHub` directory; the same harness points at
any organization's repository or data infrastructure. Built as a Compact project so every
produced artifact carries provenance and an audit trail.

## Repo Path

`C:/GitHub/cartograph`

## Tech Stack

- Python
- Pydantic
- SQLite
- uv
- ruff
- mypy
- pytest

## Dependencies

- control_plane

## Success Criteria

- Scans a directory of repos into provenance-stamped nodes with classification
- Extracts mechanical edges with `file:line` or git evidence
- Persists a curated layer of conceptual edges that survives rescans
- Produces `graph.json` and a clustered HTML view via `/graphify`
- `scan` / `diff` / `build` operate as a living loop with retained snapshots

## Default Checks

- uv run ruff check .
- uv run ruff format --check .
- uv run mypy src/
- uv run pytest
