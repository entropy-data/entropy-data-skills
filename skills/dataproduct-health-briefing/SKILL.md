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

Then list the products: page `entropy-data dataproducts list -p <n> -o json` into the scratch directory and filter on `team.name` with a script. If the user names a domain, include its sub-teams (teams whose `parent` is the domain).

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
entropy-data datacontracts get <contract-id> -o json
entropy-data test-results list --data-contract-id <contract-id> -p 0 -o json   # page until empty, keep the last 30 days
```

And once: `entropy-data policies list -o json`.

The briefing covers a handful of products, so per-product calls are fine. Write raw output to a scratch directory and summarize with a script; usage traces can contain query text with literal values, and example data contains raw values, so never print them. Join agreements to ports on `provider.outputPortId` = the port's `name`.

## Step 3: Analyze

**Quality**: latest result per contract; pass rate over the last 30 days; checks that fail repeatedly; time since the last run. With fewer than about five runs, report "no trend available" instead of a pass rate. A product whose tests stopped running is a finding even if the last run passed: the last run is older than a few times the contract's `frequency` SLA, or than two days if there is none. To explain a failure, hand off to `test-failure-root-cause`.

**Consumers**: active agreements by consuming team and data product; their purposes, grouped (for example "4 reporting, 2 ML features, 1 regulatory"); agreements ending soon; failed or pending provisioning (`provisioning.status` and `diagnostics`); purposes of active agreements that conflict with the contract's terms of use. Products of the same owner often depend on each other only through agreements (input ports are often empty), so a failure there can break the owner's own chain.

**Usage**: if usage traces are reported, who actually queries the product and how often. Compare with agreements: consumers with access but no usage (candidates for clean-up), and usage spans whose `access.id` matches no active agreement (worth investigating). Spans also carry `user`, `role`, and `rows`; never print `query.preview`.

**Cost**: recorded costs against the number of active consumers and usage. Say "not reported" when there is no data; do not estimate. When usage is not reported, recommend reporting it, since unused access cannot be found otherwise.

**Discoverability**: the product's `description.purpose` and the contract's `description.usage` and `limitations`, example data present, tags, stars. Rate them against the organization's policies on descriptions, if any.

**Pending access requests**: for each request, read the purpose and roles, the consumer, the contract's terms of use and field classifications, and the organization's policies. Recommend approve, ask for clarification, or reject, citing the clause of the contract or policy. These are recommendations only.

## Step 4: Report

Per product, a short block with status, quality, consumers, usage, cost, discoverability. Then one combined to-do list, ordered:

1. Failing or stopped contract tests, and failed provisioning, on products with active consumers.
2. Pending access requests, with the recommendation for each, and active agreements whose purpose conflicts with the terms of use.
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
