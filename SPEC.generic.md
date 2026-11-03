# cartograph - Specification (generic)

Status: DRAFT · Org-agnostic framework spec.

> This is the reusable, organization-independent statement of the idea. It names no specific
> repositories, directories, or holdings. For a concrete instantiation (a particular
> organization, its actual holdings, and its seed conceptual edges), see `SPEC.md`.

## 1. Purpose

Model an organization as a graph of **nodes** (the things it has) and **edges** (how those
things connect), keep that graph **current** by re-scanning, and attach **provenance** to
every assertion so the map can be audited and trusted.

"Organization" is a role, not a fixed target. It may be one person's collection of projects, a
company's code estate, a research group's outputs, or a data platform's asset catalog. The core
model is identical across all of them; only the **source adapter** that reads a given
organization's holdings changes.

## 2. Why it is its own system

An inventory of this kind is not a document - it is a framework with a code harness that must:

- re-scan on demand and **diff** against the last snapshot,
- separate **auto-extracted** facts from **human-curated** knowledge so rescans never
  clobber judgment,
- carry **provenance** on every node and edge,
- be **improved over time** (new edge extractors, new source adapters, better heuristics).

That is a versioned software project, so it lives as its own system rather than a loose
collection of scripts.

## 3. Core model

### 3.1 Node

A unit the organization holds. Node kinds are extensible; a deployment enables the kinds
relevant to its sources (for example: a code repository, a service, a data asset, a document).

| Field | Source | Notes |
| --- | --- | --- |
| `id` | derived | stable slug unique within the organization |
| `kind` | scanner | the node type (e.g. repository, service, data asset, document) |
| `path` | scanner | location or address within the organization |
| `classification` | scanner + override | `project` \| `experiment` \| `archive` \| `scratch` |
| `status` | scanner + override | `active` \| `dormant` \| `archived` \| `unknown` |
| `domains` | curated/LLM | theme/subject tags |
| `summary` | source/LLM | one line |
| `tech` | scanner | detected stack or format |
| `last_activity` | source metadata | most recent change timestamp |
| `provenance` | engine | scan run id + extractor that asserted each field |

### 3.2 Edge

A directed, typed relationship between nodes. Every edge carries **evidence** and
**provenance**; an edge with neither is not admitted.

**Mechanical edge types** - auto-extracted, each citing a concrete artifact pointer
(`file:line`, a parsed manifest, or version-control metadata). Where a clean match exists,
align names to an established catalog vocabulary (e.g. a `depends-on` / `depended-on-by` pair)
so the graph interoperates with existing tooling:

- `shares-origin` / `fork-of` - version-control origin relationships
- `submodule-of` / `contains` - nesting relationships
- `imports` / `references` - one node's source or documents point into another
- `naming-family` - a shared naming prefix or stem signals lineage between siblings; the
  analogue of an "implicit" dependency - real, but not derivable from imports alone
- `transferred-from` - a recorded migration or hand-off of content between nodes

**Conceptual edge types** - curated or model-proposed, each citing a human or LLM pass:

- `consolidates` - one node absorbs or assembles one or more others (composition)
- `consumes` - one node uses another as a component, kit, or dependency
- `supersedes` - one node replaces another
- `shares-theme` - two nodes belong to the same domain cluster
- `inspired-by` - softer intellectual lineage

Edge record: `{ source, target, type, evidence, confidence, provenance }`. `evidence` is a
named facet (`{ kind, schema_ref, pointer }`); `confidence` is **graded by evidence source**
(a parsed/declared edge outranks a manifest-only edge, which outranks a name-inferred edge),
not boolean.

### 3.3 Classification rubric (project / experiment / archive / scratch)

A scored heuristic, overridable by curation. Signals are source-dependent; the intent of each
bucket is fixed:

- **project** - a maintained unit: has documentation and history, and/or is referenced by other
  nodes, and/or shows recent activity.
- **experiment** - exploratory: sparse history, no dependents, throwaway-style naming.
- **archive** - deliberately retired: explicit archive markers, backup copies, long dormant, or
  flagged as archived at the source.
- **scratch** - transient: temporary, generated, or clearly disposable content.

The heuristic proposes; a human (or LLM reviewer) confirms. Overrides live in the curated layer
and survive rescans. (The precise signals and thresholds should be grounded against an
established repository/asset health-metric standard rather than invented; see `docs/prior_art.md`.)

## 4. Two-layer architecture

```
sources/        source adapters (one per kind of holding)
   |
extracted/      auto-derived nodes + mechanical edges   [regenerated every scan]
   |
curated/        human + LLM knowledge: conceptual edges,  [persists across scans]
                classification overrides, domain tags
   |
graph/          merge(extracted, curated) -> graph data + clustered visualization
```

Hard rule: **the scanner never writes to `curated/`.** A rescan replaces `extracted/`
wholesale and re-merges; curated knowledge is keyed by node/edge id so it re-attaches to the
new extraction. Orphaned curation (pointing at a node that has vanished) is surfaced, not
deleted.

Extraction follows the catalog principle: **derive edges from declared fields or parsed
artifacts; never hand-author edge objects.**

## 5. Living behavior

- `scan` - run the enabled source adapters, write a new provenance-stamped snapshot to `extracted/`.
- `diff` - compare the new snapshot to the previous: nodes added/removed, edges changed,
  classifications shifted.
- `build` - merge the layers and regenerate the graph data and visualization.

Each run appends to the provenance log; snapshots are retained so the organization's evolution
is itself queryable over time. A rescan emits a new provenance facet that **replaces** the
prior assertion while history is preserved in snapshots.

## 6. Provenance & audit

Two distinct provenance concerns:

1. **Provenance of building the system** - every change to cartograph itself is produced through
   an audited task pipeline (contract -> gate -> deterministic checks -> reviewer sign-off ->
   append-only audit entry -> artifact registry). Each produced file has a traceable origin.
2. **Provenance of the graph's own assertions** - which scan run asserted which node or edge,
   from what evidence. This is modeled inside the graph via the `provenance` field on every
   record. The governing principle is shared: no assertion without a traceable origin.

## 7. Scope

**Core (any deployment):** at least one source adapter; node extraction + the classification
heuristic; mechanical edge extractors with artifact-pointer evidence; a curated layer for
conceptual edges and overrides; merge into graph data plus a clustered visualization; the
`scan` / `diff` / `build` loop; provenance on every record.

**Extensions:** additional source adapters for new kinds of holding; hosted/remote adapters;
automatic conceptual-edge inference at scale; richer query and UI surfaces.

## 8. Open questions

- Node granularity: when two closely related holdings exist, are they one node or two linked
  nodes? (Default: two nodes plus a `naming-family` edge; curation may merge them.)
- Where seed conceptual edges come from for a given organization: hand-authored, an LLM pass
  over node documentation, or harvested from an existing knowledge store.
- Which established provenance standard the graph's assertion model should align to, versus
  what must be coined.
