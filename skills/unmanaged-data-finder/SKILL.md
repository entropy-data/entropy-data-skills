---
name: unmanaged-data-finder
description: "Governance and platform. Compares the data assets that Entropy Data imported from the data platforms with what data products and data contracts actually govern, and reports important or sensitive-looking tables that no data product covers, data products that reference assets which no longer exist, and asset imports that have not run recently or are failing. Use when a governance lead or platform owner asks how much of the data estate is governed, wants to find ungoverned or shadow data, or checks catalog coverage."
---

# Unmanaged Data Finder

**Persona:** governance lead, data platform owner
**Trigger:** monthly, or after connecting a new data platform
**Output:** coverage per platform, ranked ungoverned assets, broken references, and stale imports

Read-only.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Collect

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `assets,dataproducts,datacontracts` into a scratch directory, plus lineage for the "in active use" signal:

```bash
entropy-data lineage list --since 90d --all -o json > "$SCRATCH/lineage.json"
```

An asset has `id`, `info` (`source`, `type`, `name`, `qualifiedName`, `status`, and often `owner`, a team id), `properties`, `columns`, `assignedTags`, and `relationships[]` (`relationshipType` such as `parent`, `foreign_key`, `uses`, with `assetId`). Build the container hierarchy from `parent` relationships, not by parsing `qualifiedName`, whose format differs per source. Compare `status` case-insensitively (`active` and `ACTIVE` both occur). Types are platform-specific, for example `snowflake_table`, `databricks_table`, `databricks_schema`, `kafka_topic`, `powerbi_semantic_model`. Inspect a few assets first to learn the types in this organization. **Table-like** assets are those with `columns` (tables, views, topics, BI tables); count reports, dashboards, and semantic models separately.

The asset imports and their runs:

```bash
entropy-data integrations list -o json                           # externalId, name, source, enabled; no schedule
entropy-data integrations runs-latest <integration> -o json      # per integration
entropy-data integrations runs <integration> --limit 5 -o json  # recent history, if needed
```

Never run `integrations get`: it returns decrypted credentials.

If the organization has no assets, say so: coverage cannot be measured until the data platforms are connected through asset imports. Stop there.

## Step 2: Build the governed set

An asset counts as governed when a data product or contract points at it:

1. **Explicit links:** asset ids in the `customProperties` entry with `property: assets` (its `value` is a list of ids), on the data product and on each output port; most links sit on ports. Only `assets` counts: note other asset-looking properties (for example an internal asset id) without treating them as links.
2. **Location matches:** a contract table's location (server containers plus `schema[].physicalName` or `name`, see "Data contract" in the `entropy-data-cli` skill) matches an asset and its `parent` containers. Require at least the schema-level container plus the table name; a server with only a `location` path matches assets whose path starts with it, never by table name alone. Compare case-insensitively, and use every server of the contract.
3. **Containers:** upward, a schema or catalog counts as governed when a governed table lies inside it, but report schemas whose tables are mostly ungoverned. Downward, the children of an explicitly linked container (for example a BI semantic model) are governed too; say how many were counted this way.

Record for each governed asset which product and port govern it, and how (explicit or by location).

## Step 3: Analyze

**Coverage.** Per source and per asset type: number of table-like assets, governed, ungoverned, and the governed share.

**Ungoverned assets, ranked.** Score each ungoverned table-like asset:

| Signal | Weight |
|---|---|
| Columns that look personal (see the `entropy-data-cli` skill; recurse into nested columns), or a sensitivity tag | high |
| The asset appears in lineage `inputs` or `outputs` (match schema and table names), so it is in active use | high |
| The name suggests a shared business entity (customer, order, product, invoice, payment) | medium |
| It sits in a schema next to governed tables (a likely sibling of an existing product) | medium |
| The name suggests scratch or temporary data (`tmp`, `test`, `bak`, `_old`, a date suffix) | lowers the score |

**Broken references.** Asset ids in data products that no longer exist as assets, assets with a `status` other than active that a product still references, and contract locations that match no asset although their platform and catalog are imported (often a wrong catalog or schema in the contract).

**Not imported.** Lineage datasets that match no asset are platform data the catalog does not show. List them by schema with the jobs that read or write them.

**Stale or failing imports.** Integrations that are enabled but whose latest run is older than 7 days (the list shows no schedule), whose latest run failed, or that are disabled (the API does not say whether manually or automatically; report a recent run too). Sources with assets but no integration have unknown freshness: say so, and give a hint such as an update timestamp in `properties` when present. Assets from a stale or unknown import may no longer reflect the platform, so mention this next to coverage numbers.

## Step 4: Output

1. **Summary:** governed share per platform, number of ungoverned sensitive-looking tables, broken references, stale imports.
2. **Top ungoverned assets:** asset, source, why it matters (the signals), the most likely owning team (the asset's `info.owner`, else a team owning products about the same entity, else the schema's governed neighbors), and a suggestion: a new data product, an output port on an existing product, or "probably scratch data, confirm and clean up".
3. **Broken references:** product, port, missing asset id or unmatched contract location; then the not-imported lineage datasets.
4. **Imports:** integration, last run, status.

Name columns, never values. Coverage is a heuristic where governance is matched by location: say how many assets were matched explicitly and how many by location.
