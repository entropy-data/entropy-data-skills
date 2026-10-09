---
name: dataproduct-health-briefing
description: "Data Product Owners. Weekly briefing for a data product owner on their data products in Entropy Data: contract test trends, consumers and their purposes, actual usage versus granted access, costs, discoverability gaps, and pending access requests with a recommendation each, returned as a ranked to-do list. Use when a data product owner asks how their data products are doing, who uses their data, whether to approve access requests, or prepares a stakeholder review."
---

# Data Product Health Briefing

**Persona:** data product owner
**Trigger:** weekly, or on demand
**Output:** a short status block per product and one ranked to-do list


A read-only analysis for data product owners: is my product working, who uses it and for what, is it worth what it costs, what needs my attention.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Find the owner's products

Ask for the team. If the user does not know it, ask for their email address (do not guess it) and look it up:

```bash
entropy-data teams list --member <email> -o json
```

Then list the products: page `entropy-data dataproducts list -p <n> -o json` into the scratch directory and filter by owning team with a script.

Confirm the list of products with the user.

## Step 2: Collect per product

```bash
entropy-data dataproducts get <id> -o json
entropy-data access list --provider-dataproduct <id> -o json
entropy-data usage list --data-product-id <id> -o json
entropy-data costs list --data-product-id <id> -o json
entropy-data example-data list --data-product-id <id> -o json      # check presence only, never print values
entropy-data dataproducts star-status <id> -o json        # skip on error
```

Per output port contract:

```bash
entropy-data datacontracts yaml <contract-id>
entropy-data test-results list --data-contract-id <contract-id> -p 0 -o json   # last 30 days, sort by timestamp
```

The briefing covers a handful of products, so per-product calls are fine. Write raw output to a scratch directory and summarize with a script; usage traces can contain query text with literal values, so never print them raw.

## Step 3: Analyze

**Quality**: latest result per contract; pass rate over the last 30 days (results on the contract itself, not on branches); checks that fail repeatedly; time since the last run. A product whose tests stopped running is a finding even if the last run passed. To explain a failure, hand off to `test-failure-root-cause`.

**Consumers**: active agreements by consuming team and data product; their purposes, grouped (for example "4 reporting, 2 ML features, 1 regulatory"); agreements ending soon.

**Usage**: if usage traces are reported, who actually queries the product and how often. Compare with agreements: consumers with access but no usage (candidates for clean-up), and, if traces identify the caller, usage without a matching agreement (worth investigating).

**Cost**: recorded costs against the number of active consumers and usage. Say "not reported" when there is no data; do not estimate.

**Discoverability**: description quality (purpose, contents, limitations), example data present, tags, stars.

**Pending access requests**: for each request, read the purpose and roles, the consumer, the contract's terms of use and field classifications, and the organization's policies (`entropy-data policies list -o yaml`). Recommend approve, ask for clarification, or reject, citing the clause of the contract or policy. These are recommendations only.

## Step 4: Report

Per product, a short block with status, quality, consumers, usage, cost, discoverability. Then one combined to-do list, ordered:

1. Failing or stopped contract tests on products with active consumers.
2. Pending access requests, with the recommendation for each.
3. Deprecated products that still have consumers (use `contract-change-review` before retiring).
4. Consumers with access but no usage.
5. Gaps in description, example data, or tags.

## Step 5: Act only on request

If the owner decides on access requests, run exactly what they confirm, one by one:

```bash
entropy-data access approve <id>
entropy-data access reject <id>
```

For "ask for clarification", draft a message for the owner to send. Do not send it.
