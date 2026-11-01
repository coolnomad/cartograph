# Prior Art - how others inventory holdings and model their connections

Status: COMPLETE (task 001) · Date: 2026-10-29 · Method: deep-research harness (5 search
angles -> 25 primary sources -> 113 claims -> 25 adversarially verified 3-0, 0 killed).

> Confidence note: every claim below is tagged. **Verified** = survived 3-vote adversarial
> verification against a primary/vendor source. **Lead** = source was fetched but no claim
> survived into the verified set, so it is a pointer for a follow-up scan, not a finding.
> Four of the six requested areas are only partially covered; see section 5, Evidence Gaps.

## 1. Summary

Across software catalogs, dependency graphs, and data-lineage systems the same architecture
recurs, and it is exactly cartograph's: a **typed node model** (fixed entity kinds), a
**directional typed edge model** (named relations), a separate **provenance/evidence layer**,
and an **automated extraction method** that *derives* edges from source artifacts rather than
hand-authoring them. The most reusable vocabulary comes from **Backstage** (nodes + relation
pairs) and **OpenLineage** (evidence/provenance via versioned facets + run events). The
strongest transferable principle: **never author edges directly - derive them from declared
fields or parsed artifacts, and record where each one came from.**

## 2. Comparison table

| System | Node model | Edge model | Provenance model | Extraction method |
| --- | --- | --- | --- | --- |
| **Backstage** (verified) | 9 fixed kinds: Component, API, Resource, System, Domain, Group, User, Template, Location - uniform `apiVersion/kind/metadata/spec` envelope | 7 bidirectional well-known relation **pairs**: `ownedBy/ownerOf`, `partOf/hasPart`, `dependsOn/dependencyOf`, `providesApi/apiProvidedBy`, `consumesApi/apiConsumedBy`, `parentOf/childOf`, `memberOf/hasMember`; explicit FROM->TO direction | Annotations, not edges: `backstage.io/managed-by-location` (origin of the definition) + auto-generated globally-unique `uid` on first ingest | Processors **derive** `relations[]` from spec fields (`spec.owner`->`ownedBy`, `spec.system`->`partOf`, `spec.dependsOn`->`dependsOn`); descriptor YAML is "not supposed to contain" relations |
| **Port** (verified) | User-defined **blueprints** (any asset) in a "context lake" - schema-flexible, not fixed | Explicit relations mapped from Backstage's types via declarative **JQ** transforms | (inherits source provenance via mapping) | Reads Backstage Entities API; JQ maps fields -> entities + relations |
| **Nx** (verified) | **Projects** (detected via `project.json` / workspace `package.json`) | Project->project deps; 3 types: **static** (imports), **dynamic** (`import()`), **implicit** (inferred/manual - outside the import graph) | - | Auto-derived from source imports, installed deps, TS config; project granularity, not file-level |
| **Go module graph** (verified) | **Module versions** (not packages) | Directed edges = minimum required version via `require` in `go.mod`; pruned to immediate requirements for go-1.17+ modules | `go.mod` is the declared source | Parse `require` directives |
| **GitHub dep graph** (verified) | Packages from manifests/lockfiles | **direct** (declared) vs **indirect/transitive**; lockfile needed for completeness | Edge source (manifest vs lockfile) determines completeness/confidence | Static parse of manifest + lock files; updates on default-branch change |
| **OpenLineage** (verified) | **Job**, **Run** (UUID + start/complete time), **Dataset** (name in a namespace) | `RunEvents` (START/COMPLETE/FAIL/ABORT) tie job->inputs->outputs; column-level lineage maps output cols -> `inputFields[]` with typed transformation (DIRECT/INDIRECT; IDENTITY/AGGREGATION/JOIN...) | **Facets**: named atomic blocks on any entity; replace-on-emit; each carries immutable `_schemaURL` -> **versions the assertion itself**; events carry `producer`, `eventTime` | Event-driven **push** at state transitions |
| **DataHub** (verified) | **Dataset** (tables/streams/files) keyed by `datasetKey` = platform + name + fabric/env | `DownstreamOf` relationship derived from `upstreamLineage` aspect | `@Relationship isLineage:true` in canonical PDL | Aspect-based ingestion |

## 3. Vocabulary: adopt vs. coin

**Adopt (don't reinvent):**
- **Backstage relation pairs + FROM->TO directionality** for cartograph's mechanical/ownership
  edges. Direct map: our `imports`/`references-path` -> `dependsOn/dependencyOf`; `consolidates`
  has composition semantics close to `hasPart/partOf`; ownership (if we add it) -> `ownedBy`.
  Reason: it is a stable, battle-tested, fixed vocabulary; using the same names makes cartograph
  interoperable with the largest catalog ecosystem.
- **Backstage's "derive edges from declared fields, never author edges directly"** as the core
  extraction rule - it *is* our two-layer `extracted/` design, already independently validated.
- **OpenLineage's facet pattern** for cartograph's "evidence + provenance per assertion"
  requirement: each node/edge carries named atomic facets, each with a schema pointer, so an
  assertion can be versioned and replaced on rescan without losing its origin. This directly
  satisfies SPEC's rule that no edge is admitted without evidence + provenance.
- **GitHub's direct-vs-transitive + "lockfile determines completeness"** -> model edge
  **confidence as a function of evidence source** (a lockfile/import edge is higher-confidence
  than a manifest-only or name-inferred edge).
- **Nx's static / dynamic / implicit** tri-split -> validates cartograph separating mechanical
  (deterministic provenance) from conceptual (curated/inferred) edges. Our `naming-family` edge
  is the analogue of an "implicit" dependency - real but not import-derivable.

**Coin (no confirmed standard):**
- The **conceptual edge taxonomy** (`consolidates`, `consumes`, `supersedes`, `shares-theme`,
  `inspired-by`). No surveyed system models intellectual lineage between projects; PKM backlinks
  were an evidence gap. Keep our own names but give them OpenLineage-style facets.
- The **classification rubric** (`project` / `experiment` / `archive` / `scratch`). No sourced
  health-maturity rubric survived verification (see section 5) - coin it, informed by a targeted
  CHAOSS follow-up.
- `transferred-from` (greenfield-transfer edge) - specific to this org's workflow.

## 4. Implications for cartograph

- **Edge taxonomy:** rename/align mechanical edges to Backstage pairs where a clean match exists;
  keep conceptual edges custom. Every edge record gets a facet-style `evidence` block
  (`{ kind, schemaURL, pointer }`) and a `confidence` graded by evidence source.
- **Provenance:** adopt OpenLineage semantics - a rescan **emits a new facet that replaces** the
  prior one; snapshots retain history. Node identity gets a stable key (cf. DataHub `datasetKey`
  = platform+name+env; ours ~= org + path + kind).
- **Extraction:** confirm the derive-don't-author rule in code - the scanner only writes
  `extracted/`; curated conceptual edges live in `curated/` and re-attach by node id.
- **Confidence model:** adopt a graded scale (not boolean) keyed to evidence source - resolving
  SPEC section 8's open question in favor of graded confidence.

## 5. Evidence gaps (feed a targeted follow-up scan)

These requested areas were fetched but produced **no verified claims**; treat as leads:

- **PKM / conceptual graphs** (area 3) - Obsidian/Roam/Logseq backlinks, Zettelkasten, emergent
  clusters. Entirely absent from verified set. Leads: InfraNodus PKM graphs, Obsidian-vs-Logseq
  comparisons. Needed to ground the conceptual-edge + clustering design.
- **Provenance standards** (area 5) - W3C **PROV-O** (`Entity`/`Activity`/`Agent` +
  `wasDerivedFrom`/`wasAttributedTo`/`wasGeneratedBy`) and **SPDX** relationship vocabulary were
  fetched but not verified. This is the canonical "who asserted this, from what evidence, when"
  model the SPEC should likely align to - worth a dedicated scan before finalizing the provenance
  schema (task 002).
- **Health/activity classification** (area 6) - GitHub archive flag, **CHAOSS** metrics
  (starter-project-health + community-activity models) fetched but unverified. The
  project/experiment/archive/scratch rubric currently has **no sourced precedent**.
- **Uncovered systems:** Bazel/Buck target graphs, OpenMetadata, Amundsen, Marquez, OpsLevel,
  Cortex produced no surviving claims - likely convergent with Backstage/OpenLineage/DataHub but
  unconfirmed.

## 6. Recommended next research (before/within task 002)

1. Targeted scan of **PROV-O + SPDX relationship types** -> decide adopt-vs-coin for the
   provenance schema.
2. Targeted scan of **CHAOSS** metrics -> ground the classification rubric's signals + thresholds.
3. Optional: **PKM clustering** methods -> inform `/graphify` community detection for conceptual edges.

## Sources (primary/vendor unless noted)

- Backstage descriptor format - https://backstage.io/docs/features/software-catalog/descriptor-format/
- Backstage well-known relations - https://backstage.io/docs/features/software-catalog/well-known-relations/
- Backstage well-known annotations - https://backstage.io/docs/features/software-catalog/well-known-annotations/
- Port <- Backstage sync - https://docs.port.io/build-your-software-catalog/sync-data-to-catalog/other/backstage/
- Nx explore graph - https://nx.dev/docs/features/explore-graph
- Nx DependencyType - https://nx.dev/nx-api/devkit/documents/DependencyType
- Go modules reference - https://go.dev/ref/mod
- GitHub dependency graph data - https://docs.github.com/en/code-security/concepts/supply-chain-security/dependency-graph-data
- OpenLineage spec - https://github.com/OpenLineage/OpenLineage/blob/main/spec/OpenLineage.md
- OpenLineage column-lineage facet - https://openlineage.io/docs/spec/facets/dataset-facets/column_lineage_facet/
- DataHub Dataset entity - https://docs.datahub.com/docs/generated/metamodel/entities/dataset
- DataHub Upstream.pdl - https://github.com/datahub-project/datahub/blob/master/metadata-models/src/main/pegasus/com/linkedin/dataset/Upstream.pdl
- (leads, unverified) W3C PROV-O - https://www.w3.org/TR/prov-o/ ; SPDX RelationshipType - https://spdx.github.io/spdx-spec/v3.0.1/model/Core/Vocabularies/RelationshipType/ ; CHAOSS starter-project-health - https://www.chaoss.community/kb/metrics-model-starter-project-health/ ; GitHub archiving - https://docs.github.com/en/repositories/archiving-a-github-repository/archiving-repositories
