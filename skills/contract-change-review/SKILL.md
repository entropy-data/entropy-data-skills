---
name: contract-change-review
description: "Data Product Owners. On a pull request or before a planned change to a data contract or data product in Entropy Data, classifies each change as breaking or not, finds the affected consumers, downstream data products, and lineage, and returns a green/yellow/red verdict with a rollout plan and notification drafts. Use when an owner or engineer plans to change a schema, rename or remove a field, tighten quality rules, change an output port, or deprecate or retire a data product, and asks what breaks, who to tell, whether a change is breaking, or for an impact analysis."
---

# Contract Change Review

**Persona:** data product owner, data engineer
**Trigger:** on a pull request that touches a data contract, or before a planned change or retirement
**Output:** a green/yellow/red verdict, affected consumers, rollout plan, notification drafts


Answers "if I make this change, what breaks and who is affected?" before anything changes. Read-only unless the user explicitly asks to stage the change on a branch.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Understand the planned change

Identify the data product, output port, and data contract (`entropy-data search query "<name>"` if unclear). Then get the change in one of these forms:

- a pull request or local edit of an ODCS file: get the changed file from the working tree or the PR (`git diff`, `gh pr diff <n>`), and compare it with the published contract:
  ```bash
  entropy-data datacontracts yaml <contract-id> --file "$SCRATCH/published.odcs.yaml"
  ```
- an existing contract branch:
  ```bash
  entropy-data datacontracts branches changes <contract-id> <branch> -o json
  ```
- a description in words ("drop `customer_email`", "retire the product").

## Step 2: Classify the change

For each changed element, decide breaking or non-breaking for consumers:

| Breaking | Non-breaking |
|---|---|
| Removing or renaming a schema, field, or output port | Adding an optional field |
| Changing a field's type to an incompatible one, or narrowing it | Widening a type (int to bigint, longer varchar) |
| Making a field required, or removing allowed values | Relaxing a constraint |
| Tightening quality rules consumers rely on, or changing semantics without changing the name | Improving descriptions |
| Changing the server or location of an output port | Adding a new output port |
| Deprecating or retiring the product | |

Semantic changes (same name, different meaning or unit) are breaking even though no schema tool catches them. Ask the user about them explicitly.

Recommend the version bump: major for breaking, minor for additive, patch for documentation only.

## Step 3: Find who is affected

```bash
entropy-data access list --provider-dataproduct <id> -o json
entropy-data usage list --data-product-id <id> -o json
```

Consider active agreements and also requested and upcoming ones: they are consumers too, soon.

For each consumer:

- Who it is (team, user, or data product) and their stated purpose.
- For a consumer data product: its lineage shows whether it reads the affected output port or dataset. A provider's own lineage only shows its own jobs, so look at the consumer's:
  ```bash
  entropy-data lineage list --data-product-id <consumer-id> -o json
  ```
  Check whether the input datasets of its jobs reference the provider's output dataset and, if column-level lineage is present, the affected fields. Then fetch that consumer's own consumers (`access list --provider-dataproduct <consumer-id>`) for the second level of impact.
- Whether usage shows that they read the affected fields. Usage traces can contain query text; search it for the field names with a script and never print raw queries.
- Usage by callers without any agreement, if traces identify the caller: these are unknown consumers.

If there is no lineage or usage data for a consumer, say so: its impact is then "possible", not "confirmed".

## Step 4: Report

1. **Verdict**:
   - **green**: no breaking change, or breaking changes with no known consumers (no active, requested, or upcoming agreements, and no usage).
   - **yellow**: breaking changes whose impact on consumers is possible but not confirmed by usage or lineage, or a deprecation with a migration path.
   - **red**: breaking changes that usage or lineage shows active consumers rely on, or retirement with active consumers.
2. **Change summary**: each change, breaking or not, recommended version.
3. **Affected consumers**: table with consumer, owning team, purpose, impact (confirmed by usage or lineage, or possible), second-level dependents.
4. **Rollout recommendation**: for breaking changes, the safer path, for example add the new field, mark the old one deprecated in the contract, give consumers a deadline, then remove it in a new major version or a new output port. For retirement: set status to `deprecated`, notify consumers, wait for agreements to end, then retire.
5. **Notification draft**: a short message per consuming team listing exactly what changes for them and when. Do not send it.

On a pull request, the PR comment contains only the verdict, the breaking changes, and counts ("affects 4 consumers in 3 teams, 2 confirmed by lineage"). Consumer names and purposes stay in the local report, because everyone with access to the repository can read the comment. Post it only if the user asks, with their git host's CLI (for example `gh pr comment`), after showing it.

## Step 5: Stage the change (only on request)

If the user wants to move ahead, the change can be staged on a contract branch and tested without touching the published contract:

```bash
entropy-data datacontracts branches create <contract-id> <branch>
entropy-data datacontracts branches put <contract-id> <branch> --file changed.odcs.yaml
entropy-data datacontracts test <contract-id> --branch <branch>
```

Each command needs the user's confirmation. The test runs queries against the data platform, which costs compute and publishes a result.

Merging is a separate decision for the owner: `entropy-data datacontracts branches merge <contract-id> <branch> --version <x.y.z>`, only with explicit confirmation.
