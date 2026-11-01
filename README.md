# cartograph

`cartograph` is a **living inventory of an organization's holdings and how they connect** -
directly (shared code, git lineage, cross-references) and conceptually (shared ideas,
consolidation, supersession).

The first organization it maps is a single developer's `C:\GitHub` directory (~57 repos).
But cartograph is a *reusable pattern*: point it at any organization's repository or data
infrastructure and it produces the same artifact - a provenance-stamped graph of what
exists and how it relates.

cartograph is a conceptual framework with a code harness, not a pile of scripts. It is built
as a [Compact](../control_plane) project, so every artifact it produces carries a
traceable origin: which scan run produced it, what the inputs were, and what signed off.

## What it produces

- **Node inventory** - every unit in the org (repo, project, data asset), classified as
  `project` / `experiment` / `archive` / `scratch`, with status, domain tags, tech, and
  last activity.
- **Edge graph** - typed, directed relationships between nodes, each with *evidence* and
  *provenance* (a mechanical edge cites `file:line` or git data; a conceptual edge cites a
  human or an LLM pass).
- **Graph artifacts** - `graph.json` plus a clustered HTML visualization (via `/graphify`).
- **Provenance log** - an append-only record of every scan run and every curation.

## Status

`PLANNED` - greenfield spec stage. See [SPEC.md](SPEC.md) for the conceptual model and
[docs/prior_art.md](docs/prior_art.md) for the survey of how others approach this.
