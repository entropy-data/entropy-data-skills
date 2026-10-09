---
name: access-review
description: "Governance. Quarterly access review (recertification) over all data usage agreements in Entropy Data: flags agreements without a clear purpose, long-pending requests, access to deprecated products, individual users on restricted data, duplicates, and provisioning that never completed, ranked by risk and grouped by provider team. Use when a governance lead, security officer, or auditor asks for an access review, a recertification campaign, or who has access to what and why, organization- or team-wide across many providers. For one owner's own data products, use dataproduct-health-briefing instead."
---

# Access Review

**Persona:** governance lead, security officer, auditor
**Trigger:** quarterly, or before an audit
**Output:** ranked risky agreements grouped by provider team, plus the full list of active agreements

Access in Entropy Data is granted through access agreements (data usage agreements) between a provider data product output port and a consumer (a team, a user, or another data product), each with a purpose. This skill reviews them in bulk. It is read-only until the user decides on actions.

Approving individual incoming requests is the data product owner's job: see `dataproduct-health-briefing`.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2. The API key needs read access to all access agreements in scope (an organization-level key, or one key per team).

## Step 1: Scope

Ask the user: whole organization, or a set of provider data products or teams? Which time window counts as "stale" (default: pending longer than 14 days; ending within 30 days)?

## Step 2: Collect

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `access,dataproducts,datacontracts,policies,classification-schemes` into a scratch directory. That gives every agreement, the provider products' status and owning team, the contracts' field classifications, and the policies.

Agreements move through these lifecycle states: requested, upcoming, provisioning pending, active, deprovisioning pending, expired, rejected. Inspect one agreement first (`entropy-data access get <id> -o yaml`) to learn the field names.

For the "never used" check, fetch usage once per distinct provider product in scope that has active agreements:

```bash
entropy-data usage list --data-product-id <id> -o json
```

## Step 3: Checks

| Finding | Why it matters |
|---|---|
| Requested for longer than the stale window | Consumers are blocked; owners may be unresponsive |
| Active, but the purpose is empty, very short, or generic ("analytics", "test") | Purpose limitation cannot be demonstrated |
| Active on a provider product whose status is `deprecated` or `retired` | Consumers rely on data that is going away |
| Active with no end date, where a policy requires one | Policy violation |
| Ending within the window | Consumer may need to renew |
| Provisioning pending or deprovisioning pending for a long time | The grant in the data platform does not match the agreement |
| Consumer is an individual user on a product whose contract carries restricted classifications | Higher risk, review explicitly |
| Same consumer with several active agreements for the same output port | Duplicates to clean up |
| Active, but no usage by that consumer in the review period, if usage traces identify the caller | Strongest signal that access is no longer needed |

Where an organization policy defines the rule (for example a maximum duration), cite the policy by name in the finding.

## Step 4: Report

Write `access-review-<date>.md` with a summary (count individual users rather than naming them in chat), findings grouped by provider team (so each owner gets their list), and a full table of active agreements (provider product, output port, consumer, purpose, start, end, state).

## Step 5: Actions (only with confirmation)

Offer, per finding, and execute only what the user explicitly confirms:

```bash
entropy-data access cancel <id>      # end an active or requested agreement
entropy-data access reject <id>      # reject a pending request
```

Cancelling access affects real consumers and their pipelines. List the exact agreement ids, consumers, and impact in the confirmation question, and ask for each batch. Prefer handing the list to the provider team over cancelling on their behalf.
