---
name: semantic-namespace-review
description: "Governance and data architects. Reviews a semantic namespace (the business ontology) in Entropy Data, finds improvements such as duplicate or overlapping concepts, missing or weak descriptions, isolated concepts, missing relationships, concepts no data product uses, and recurring contract fields without a concept, and puts the proposed changes on a branch of the namespace so they can be reviewed visually in the Entropy Data UI before anyone merges them. Use when a data architect, steward, or governance lead asks to review, improve, clean up, or extend a semantic namespace, ontology, or business glossary."
---

# Semantic Namespace Review

**Persona:** data architect, data steward, governance lead
**Trigger:** on demand, or quarterly per namespace
**Output:** a review with proposed changes, and (on request) a namespace branch holding them, with the link to review it in the Entropy Data UI

The analysis is read-only. Creating the branch is the only write: it needs the user's confirmation and a key that may edit the namespace. The branch never changes the namespace itself. Merging stays a decision for the namespace owner in the UI.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Pick the namespace

```bash
entropy-data semantics namespaces list -o json
```

The list has `namespace`, `name`, `description`, `read_only`, and `team`, but no size. If the user did not name one, list them with their owner and ask; get sizes from the export in Step 2 if needed. A `read_only` namespace cannot take a branch: review it, but do not offer Step 5.

## Step 2: Collect

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `semantic-ontology,dataproducts,datacontracts` into a scratch directory. The namespace's main branch is `semantic-ontology/<namespace>.yaml`, one Apache Ossie ontology document:

- Root: `name`, `description`, `custom_properties` (for example `display_name`, `owner`), and `ontology`, the list of concepts.
- Concept: `concept` (the name), `id` (the external id, stable), `type` (`EntityType`, `ValueType`, `MetricType`, `GroupType`), `description`, `extends`, `relationships`, `examples`, `enum`, `classification`, `iri`, `unit`, `pattern`, `group`, `shared`, `ai_context` (for example `synonyms`), metric fields such as `formula` and `better_when`, and `custom_properties` (for example translations `name@de`, `description@de`, `abbreviation`, `owl:equivalentClass`).
- Relationship: `id`, `name`, `roles` (with `concept` and optionally `role_name`), `verbalizes`, `multiplicity`, `description`. Property relationships carry a `type` (`hasProperty`, `derived_from`, `measures`); relationships between entities (for example "places" or "ships to") have no `type`, only `multiplicity`. Do not filter relationships on `type` alone.

Inspect the document first: the exact fields may differ between versions. Also check for open branches, so the proposal does not collide with work in progress:

```bash
entropy-data semantics branches list <namespace> -o json
entropy-data semantics branches changes <namespace> <open-branch> -o json      # what an open branch touches
```

If an open branch already changes a concept you want to propose a change for, leave that concept out and mention it: conflicts are detected per concept, so even unrelated edits to the same concept would conflict.

## Step 3: Analyze

Work through the document with a script; read concepts in full only where a finding needs judgement.

| Finding | How to detect |
|---|---|
| Duplicate or overlapping concepts | same or near-same names (case, plural, synonyms), very similar descriptions, or two value types with the same pattern and unit |
| Missing or weak descriptions | empty, shorter than one sentence, repeating the name, or technical instead of business language |
| Missing translations | the namespace uses `name@<lang>` or `description@<lang>` on most concepts, but not on these |
| Isolated concepts | entity types without relationships, and non-shared value types no entity has. Skip group types (they have no relationships by design) and `shared` value types that contracts link to directly |
| Missing relationships | entity pairs that contracts join on but that have no relationship. Contracts mostly link id fields (for example `customer_id`, `order_id`) to value types: map each value type to its entity through the entities' `hasProperty` relationships, then look for contracts or schemas that hold ids of two entities |
| Inconsistent structure | value types without `extends`, metrics without unit, sensitive-looking concepts (email, birth, iban) without `classification` |
| Unlinked concepts | concepts that no data product or contract field links to through `authoritativeDefinitions` with `type: semantics`. Match the link's last path segment to the concept `id`, case-insensitively and ignoring `_`, `-`, and camelCase. Usually the gap is in the contracts, not in the ontology: propose links to the owning teams, not removals |
| Missing concepts | contract field names that recur across several contracts (normalize case, `_`, and prefixes such as `customer_`), including contracts no data product references, with no concept in the namespace |

For each finding, decide on a concrete proposal: merge (name the surviving concept), add or rewrite a description, add a relationship, add a classification, add a concept, or mark for removal.

Rules for proposals:

- Propose removal only for concepts that are duplicates or wrong, never just because nothing links to them.

- Never change the `id` of an existing concept: links from data products and contracts depend on it. To merge, keep one concept and propose removing the other only when nothing links to it; otherwise describe the merge and leave both.
- Reuse the namespace's own conventions: naming, description style, language properties, `extends` targets. Where conventions are mixed, follow the majority and say which one you chose: id style (for example `entity.property` for entity-specific value types, plain ids for shared ones), shared or entity-specific value types, and whether new entities belong to a `group`. If the namespace has translations, add them for new and changed texts.
- New descriptions are business language, one or two sentences, and say what the concept is, not where it is stored.

## Step 4: Present the review

Show the proposals grouped by kind (merges, descriptions, relationships, classifications, new concepts, links for teams to add, removals), each with the reason and the evidence, and a count per kind. Ask which to apply: all (the default), by kind, or one by one. Links for teams to add go into the review only; they change contracts, not the namespace.

Stop here if the user only wanted the review.

## Step 5: Put the proposal on a branch (only with confirmation)

Name the branch after the review, for example `review-2026-10`. Show the commands and ask for confirmation before running them.

```bash
entropy-data semantics branches create <namespace> <branch>
entropy-data semantics branches yaml <namespace> <branch> --file "$SCRATCH/branch.yaml"
```

Apply the accepted proposals to `branch.yaml` as text edits (insert or replace the affected lines), not by loading and dumping the YAML, which would reformat quoting and block scalars throughout the file. Keep the root `name` as downloaded: it is the branch's own namespace id, not the namespace name. Validate that the YAML parses and that every `id` the main branch had is still present (unless a removal was accepted). Then upload and check what the branch changes:

```bash
entropy-data --timeout 120 semantics branches put <namespace> <branch> --file "$SCRATCH/branch.yaml"
entropy-data semantics branches changes <namespace> <branch> -o json
```

Compare the change list with the accepted proposals: every proposal should appear, and nothing else. The server may reorder relationships on upload; reordering alone is not a change. If a field was lost or something unproposed changed, fix `branch.yaml` and upload again. Keep `branch.yaml` in the scratch directory until the user has reviewed the branch.

## Step 6: Hand over for visual review

Give the user the links to the branch and its changes in the Entropy Data UI. The URL uses the branch's own namespace id, not the branch name:

```bash
entropy-data semantics branches get <namespace> <branch> -o json      # branch_namespace, for example branch-<namespace>-<branch>
```

- `<ui host>/<organization>/studio/semantics/<namespace>/branches/<branch_namespace>`
- `<ui host>/<organization>/studio/semantics/<namespace>/branches/<branch_namespace>/changes` (opens the first changed element, with the list of changes alongside)

Take `<organization>` from `entropy-data organization get` (`vanityUrl`). If the UI host is unknown, ask for it rather than guessing.

Then summarize: proposals applied, the branch name, and that nothing is merged. Do not run `branches merge`. If the user wants to discard the proposal, `entropy-data semantics branches delete <namespace> <branch>` removes the branch and leaves the namespace untouched; run it only after confirmation.
