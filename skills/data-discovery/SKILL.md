---
name: data-discovery
description: "Data Product Builders. For a use case described in plain language, searches Entropy Data for candidate data products and evaluates each one's fitness: schema match, semantics, quality track record, freshness, terms of use, sensitive data, existing consumers, and owner, returning a ranked shortlist with gaps and the next step to get access. Use when a data engineer, analyst, or data scientist asks which data to use for a use case, whether a data product is good enough, or where to find data about a topic. For a plain lookup of a known product by name, just run a search and answer."
---

# Data Discovery and Fitness Check

**Persona:** data product builder, analyst, data scientist
**Trigger:** on demand, at the start of a new data product or analysis
**Output:** a ranked shortlist of data products with a fitness verdict, the gaps, and how to get access

Read-only, except an access request the user explicitly asks for.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Understand the need

If the user only looks up a known product by name ("where is the orders product?"), run `entropy-data search query "<name>"`, answer, and stop.


Ask briefly, or extract from the request:

- The question or use case, and the business entities involved (customers, orders, shipments).
- Required fields or metrics, the grain (per order, per customer per day), the time range, and the freshness needed.
- Constraints: where the data must be consumed (platform, region), and whether personal data is acceptable.

## Step 2: Search

Search several ways and merge the results:

```bash
entropy-data search query "<use case in words>" -o json
entropy-data search query "<entity>" --resource-type DATA_PRODUCT -o json
entropy-data search query "<field or metric>" --resource-type DATA_CONTRACT -o json
entropy-data search query "<entity>" --resource-type SEMANTIC_CONCEPT -o json
```

For semantic concepts found, read them (`entropy-data semantics concepts get <namespace> <id> -o json`) and use their names, labels, and descriptions as further search terms. Within a known namespace, `entropy-data semantics search <namespace> "<term>"` finds related concepts.

Keep products with status `active`, and mention `draft` or `proposed` ones separately. Exclude `deprecated` and `retired` ones, but say if the best match is deprecated.

Rank the merged results by relevance and status, and evaluate only the top 5 to 8 in Step 3.

## Step 3: Evaluate each candidate

```bash
entropy-data dataproducts get <id> -o json
entropy-data datacontracts yaml <contract-id>
entropy-data test-results list --data-contract-id <contract-id> -o json
entropy-data example-data list --data-product-id <id> -o json      # use column names and row shape only, never print values
entropy-data access list --provider-dataproduct <id> -o json
```

Score each output port:

| Criterion | Question |
|---|---|
| Coverage | Which required fields or metrics are present, which are missing? |
| Grain and history | Does the grain match (key fields; the shape of the example data, if present)? Is the needed time range available? |
| Semantics | Do field descriptions and linked definitions mean what the use case needs? |
| Quality | Pass rate of contract tests over the history; recurring failures |
| Freshness | Freshness or SLA in the contract versus the need; date of the last test |
| Terms of use | Do the usage terms and limitations allow this purpose? |
| Sensitivity | Classified or personal fields that would need justification |
| Adoption | Number and kind of existing consumers: a widely used product is battle-tested |
| Ownership | Owning team and support channel present |
| Location | Server type and location match where the data must be consumed |

## Step 4: Output

A ranked shortlist, best fit first. For each candidate: product and output port, owning team, verdict (good fit, usable with gaps, not a fit), the gaps, and the evidence (for example "passed 28 of the last 30 tests, failures were freshness on weekends").

If no candidate covers the need, say so plainly and describe the gap. That gap is input for a new data product or a request to the owning team.

## Step 5: Request access (only on request)

If the user picks a candidate and wants access:

```bash
entropy-data access request <data-product-id> <output-port-id> --purpose "<specific purpose>" \
  [--consumer-team <team-id> | --consumer-dataproduct <id> | --consumer-user <user-id>]
```

Draft a specific purpose from Step 1 (what, why, for how long), show the full command, and wait for explicit confirmation: the request notifies the provider team. Vague purposes are often rejected.
