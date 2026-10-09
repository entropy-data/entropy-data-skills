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

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `dataproducts,datacontracts,access,sourcesystems` into a scratch directory. Skip products with status `retired`.

Lineage has no bulk form. Fetch it later, only for products that end up in a candidate pair (Step 3):

```bash
entropy-data lineage list --data-product-id <id> -o json
```

## Step 2: Build a fingerprint per product

- **Text**: name, description, output port names, tags.
- **Schema**: per output port, the set of normalized field names (lowercase, `_`/`-`/camelCase unified, common prefixes like `dim_`, `fct_`, `stg_` removed) with their types.
- **Grain**: the primary key or unique fields of each schema.
- **Sources**: input ports, source systems, and upstream datasets from lineage.
- **Concepts**: business definitions and concepts linked through `authoritativeDefinitions` in the contracts, if any.

## Step 3: Score pairs

Do not compare every pair. Block first with a script: only pairs that share at least one cheap signal become candidates: a source system or input port, a linked definition, a significant word in the name, or three or more normalized field names. Score only the candidate pairs; judge description similarity by meaning only for those.

| Signal | Weight |
|---|---|
| Schema overlap: Jaccard similarity of normalized field names of the best-matching output ports | high |
| Same grain (same key fields) | high |
| Same upstream sources or source systems | high |
| Same linked business concepts | medium |
| Similar name and description (meaning, not only words) | medium |
| Same consumers reading both (from the access agreements) | low |

Classify each pair:

- **Duplicate**: same grain, same sources, schema overlap above about 0.7. Likely built twice.
- **Overlap**: shared sources and partial schema overlap. One may be a subset or a variant of the other.
- **Related, keep**: same concept but different grain or purpose, for example daily snapshot versus event stream. Report these briefly so nobody "fixes" them by mistake.

Group pairs into clusters.

## Step 4: Output

For each cluster, ranked by confidence, then by combined number of consumers:

- the products, owning teams, status, and number of active consumers each,
- the evidence: shared fields (list them), shared sources, same grain or not, text similarity,
- the differences that matter (extra fields, different freshness, different quality rules),
- a recommendation: consolidate into one (name the candidate to keep, usually the one with more consumers, a contract, and passing tests), turn one into a consumer of the other, or keep both and clarify the descriptions,
- an estimate of what consolidation touches: consumers that would need to move.

End with a summary: number of clusters per class, teams most involved, and the top three consolidation opportunities.

Present findings as hypotheses with evidence. Consolidation decisions belong to the owning teams. For a cluster the user wants to act on, `contract-change-review` analyzes the consumer impact of retiring one of the products.
