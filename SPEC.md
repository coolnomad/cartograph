# cartograph - Specification

Status: DRAFT (greenfield) · Date: 2026-10-29

## 1. Purpose

Model an organization as a graph of **nodes** (the things it has) and **edges** (how those
things connect), keep that graph **current** by re-scanning, and attach **provenance** to
every assertion so the map can be audited and trusted.

"Organization" is a role, not a fixed target. v0 maps one person's `C:\GitHub` directory.
The same harness, pointed at a company's GitHub org or a data platform's asset catalog,
produces the same kind of graph. Nothing in the core model is personal-directory-specific;
the directory scanner is just the first *source adapter*.

## 2. Why it is its own repo

The inventory is not a document - it is a framework with a code harness that must:

- re-scan on demand and **diff** against the last snapshot,
- separate **auto-extracted** facts from **human-curated** knowledge so rescans never
  clobber judgment,
- carry **provenance** on every node and edge,
- be **improved over time** (new edge extractors, new source adapters, better heuristics).

That is a versioned software project, so it gets its own repo rather than a pile of scripts
at the top of `C:\GitHub`.

## 3. Core model

### 3.1 Node

A unit in the organization. v0 node kinds: `repo`. Extensible later to `data_asset`,
`service`, `doc`.

| Field | Source | Notes |
| --- | --- | --- |
| `id` | derived | stable slug, e.g. `mlwt-play-policy` |
| `kind` | scanner | `repo` in v0 |
| `path` | scanner | absolute or org-relative location |
| `classification` | scanner + override | `project` \| `experiment` \| `archive` \| `scratch` |
| `status` | scanner + override | `active` \| `dormant` \| `archived` \| `unknown` |
| `domains` | curated/LLM | theme tags, e.g. `mtg`, `knowledge-graph`, `causal` |
| `summary` | README/LLM | one line |
| `tech` | scanner | detected stack |
| `last_activity` | git/mtime | most recent commit or file write |
| `provenance` | engine | scan run id + extractor that asserted each field |

### 3.2 Edge

A directed, typed relationship. Every edge carries **evidence** and **provenance**; an edge
with neither is not admitted.

Mechanical edge types (auto-extracted, cite `file:line` or git data). Where a clean match
exists, align names to Backstage's well-known relations (see `docs/prior_art.md`):

- `shares-remote` / `fork-of` - git remote relationships
- `submodule-of` - git submodule
- `imports` / `references-path` (~= Backstage `dependsOn`) - source or doc points into a sibling node
- `naming-family` - shared prefix/stem signals lineage (`mlwt-*`, `graphomics*`,
  `kanban_os`/`kanban_practice`, `Operon`/`Operon-repr-eval`, `discovery_*`); the analogue of
  Nx's "implicit" dependency - real but not import-derivable
- `transferred-from` - a greenfield-transfer doc records a move between repos
  (e.g. `compact_greenfield_transfer*.md`)

Conceptual edge types (curated or LLM-proposed, cite a human/LLM pass):

- `consolidates` (composition, ~= Backstage `hasPart`) - A absorbs/assembles B (e.g.
  `discovery_os` <- `discovery_loop`, `the_committee`, `Compact`, `research_os`, `discovery_engine`)
- `consumes` - A uses B as a kit/dependency (e.g. `play_policy` <- `discovery_loop`)
- `supersedes` - A replaces B
- `shares-theme` - A and B belong to the same domain cluster
- `inspired-by` - softer intellectual lineage

Edge record: `{ source, target, type, evidence, confidence, provenance }`. Per prior art,
`evidence` is an OpenLineage-style facet (`{ kind, schemaURL, pointer }`) and `confidence` is
**graded by evidence source** (lockfile/import > manifest > name-inferred), not boolean.

### 3.3 Classification rubric (project / experiment / archive / scratch)

A scored heuristic, overridable by curation (no sourced precedent survived the prior-art scan;
to be grounded by a targeted CHAOSS follow-up - see `docs/prior_art.md` section 6):

- **project** - has README + commit history + is referenced by other nodes, or recent activity
- **experiment** - sparse history, no dependents, exploratory naming (`Test`, `*_practice`, `demo_*`)
- **archive** - explicit archive markers, `_backup_*`, long dormant, or `status: archived`
- **scratch** - temp/throwaway (`.tmp*`, `_wsldemo`, zips)

The heuristic proposes; a human (or LLM reviewer) confirms. Overrides live in the curated
layer and survive rescans.

## 4. Two-layer architecture

```
sources/        source adapters (v0: local git directory scanner)
   |
extracted/      auto-derived nodes + mechanical edges   [regenerated every scan]
   |
curated/        human + LLM knowledge: conceptual edges,  [persists across scans]
                classification overrides, domain tags
   |
graph/          merge(extracted, curated) -> graph.json + clustered HTML (/graphify)
```

Hard rule: **the scanner never writes to `curated/`.** A rescan replaces `extracted/`
wholesale and re-merges; curated knowledge is keyed by node/edge id so it re-attaches to the
new extraction. Orphaned curation (pointing at a node that vanished) is surfaced, not deleted.

The extraction rule mirrors Backstage: **derive edges from declared fields / parsed artifacts,
never hand-author edge objects** (see prior art).

## 5. Living behavior

- `cartograph scan` - run source adapters, write a new provenance-stamped snapshot to `extracted/`.
- `cartograph diff` - compare the new snapshot to the previous: nodes added/removed, edges
  changed, classifications shifted.
- `cartograph build` - merge layers and regenerate `graph/`.
- Each run appends to the provenance log; snapshots are retained so the org's evolution is
  itself queryable over time. Per OpenLineage, a rescan emits a new provenance facet that
  **replaces** the prior one while history is retained in snapshots.

## 6. Provenance & audit (via Compact)

cartograph is a Compact project: `registry.md` + `tasks/` + `audit/`. Work on cartograph
proceeds as task cards run through Compact's gate -> checks -> reviewer -> append-only audit
entry -> artifact registry. That covers provenance of *building cartograph*.

Provenance of *cartograph's own assertions* (which scan run asserted which node/edge, with what
evidence) is modeled inside the graph via the `provenance` field on every node and edge, and
mirrors Compact's philosophy: no assertion without a traceable origin.

## 7. v0 scope

In: local git directory source adapter; node extraction + classification heuristic;
mechanical edge extractors (naming-family, path-reference, git-remote, transfer-doc); curated
layer for conceptual edges seeded from existing knowledge (memory already records several);
merge -> `graph.json` -> `/graphify` HTML; `scan`/`diff`/`build`; provenance on every record.

Out (later): non-repo source adapters (data assets, services); hosted GitHub-org adapter;
automatic conceptual-edge inference at scale; web UI beyond the graphify artifact.

## 8. Open questions

- Node granularity: is `graphomics` + `graphomics_materials` one node or two linked nodes?
  (Default: two nodes + `naming-family` edge; curation may merge.)
- ~~Confidence scale for edges (boolean vs graded).~~ Resolved by prior art: **graded**,
  keyed to evidence source.
- Where the seed conceptual edges come from: hand-authored vs an LLM pass over READMEs vs
  harvesting the existing auto-memory graph (which already encodes consolidates/consumes).
- Whether to align the provenance schema to W3C PROV-O / SPDX (prior-art evidence gap -
  needs a targeted follow-up scan before task 002 finalizes schemas).
