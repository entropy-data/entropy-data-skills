---
name: duplicate-dataproduct-finder
description: "Governance. Monthly analysis that finds duplicate and overlapping data products in Entropy Data by comparing names, descriptions, contract schemas, source systems, lineage, and linked business concepts, and returns ranked clusters with evidence and a consolidation recommendation. Use when a governance lead or data platform owner asks about duplicate data products, redundant pipelines, overlapping data, or wants to rationalize the data product portfolio."
---

# Duplicate Data Product Finder

**Persona:** governance lead, data mesh enablement team, platform owner
**Trigger:** monthly, or before a portfolio review
**Output:** ranked clusters of overlapping data products, with evidence and a recommendation per cluster

Read-only.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Collect

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `dataproducts,datacontracts,access,sourcesystems` into a scratch directory, plus one organization-wide `entropy-data lineage list --all -o json` (add `--since` to bound it on large organizations). Skip products with status `retired`. Keep `draft` and `proposed` products: they show planned duplicates, so label them as such.

## Step 2: Build a fingerprint per product

- **Text**: name, `description.purpose`, `context.instructions`, `synonyms`, output port names, tags, and the contracts' `description.purpose`.
- **Schema**: per schema (table) of each output port contract, the set of normalized field names (lowercase, `_`/`-`/camelCase unified, singular, common prefixes like `dim_`, `fct_`, `stg_` removed, nested fields by leaf name) with their types. Drop technical and audit fields (for example `id`, `timestamp`, `created_at`, `processing_timestamp`) before comparing.
- **Grain**: the primary key or unique fields of each schema. If none are declared, infer the grain from the description only for candidate pairs (Step 3) that are not already "Related, keep", and mark it as inferred.
- **Sources**: input ports, source systems, and upstream datasets from lineage `inputs`. Lineage is often the only source signal; many organizations register no source systems (no `sourcesystems` folder in the export).
- **Concepts**: `authoritativeDefinitions` with `type: semantics` on the product, its input ports, and contract properties.
- **Type**: the product type (layer, see "Where things live"), only as a hint.

Products without any contract can match only on text and concepts: weak evidence, say so.

## Step 3: Score pairs

Do not compare every pair. Block first with a script: only pairs that share at least one cheap signal become candidates: a source system, input port, or lineage input dataset, a significant word in the name (ignore generic words such as "report", "service", "model", "latest", "history"), three or more normalized field names, or a linked concept together with one other signal (one broad concept alone links too many products). Score only the candidate pairs; judge description similarity by meaning only for those.

If one product of a pair consumes the other (lineage inputs match the other's outputs, an access agreement, or an input port), it is derived, not a duplicate: classify it as "Related, keep".

| Signal | Weight |
|---|---|
| Schema overlap: Jaccard similarity of normalized field names of the best-matching schemas | high |
| Same grain (same key fields) | high |
| Same upstream sources or source systems | high |
| Same linked business concepts | medium |
| Similar name and description (meaning, not only words) | medium |
| Same consumers reading both (from the access agreements; placeholder consumer products such as `unknown` do not count) | low |

Classify each pair:

- **Duplicate**: same grain, same sources, schema overlap above about 0.7. Likely built twice.
- **Overlap**: shared sources and partial schema overlap. One may be a subset or a variant of the other.
- **Related, keep**: same concept but different grain or purpose, for example daily snapshot versus event stream. Report these briefly so nobody "fixes" them by mistake.

Group pairs into clusters. Also compare contracts that no product references with the contracts of products: older contract versions left behind are duplicates too. Such contracts have no sources, so classify them on schema overlap, key, SLAs, and text alone. An unreferenced contract that matches a product's input port is more likely a missing `contractId` on the producer's port: report it as a metadata gap. Ports of the same product (for example with and without personal data, latest and history) are variants: "Related, keep".

For the differences in Step 4, read `slaProperties` from the contracts and fetch `test-results list --data-contract-id <id> --since <date> --all` only for contracts in clusters that have consumers.

## Step 4: Output

Start with a one-line answer, also when there are no duplicates ("no duplicates; 3 clusters are deliberate layers"). For each cluster, ranked by confidence, then by combined number of consumers:

- the products (or unreferenced contracts), owning teams (or "(no team)"), status, and number of active consumers each (approved and active agreements),
- the evidence: shared fields (list them), shared sources, same grain or not, text similarity,
- the differences that matter (extra fields, different freshness, different quality rules),
- a recommendation: consolidate into one (name the candidate to keep, usually the one with more consumers, a contract, and passing tests; if it currently fails tests, say so rather than switching candidates), turn one into a consumer of the other, or keep both and clarify the descriptions,
- an estimate of what consolidation touches: consumers that would need to move.

End with a summary: number of clusters per class, teams most involved, and the top three consolidation opportunities.

Present findings as hypotheses with evidence. Consolidation decisions belong to the owning teams. For a cluster the user wants to act on, `contract-change-review` analyzes the consumer impact of retiring one of the products.
