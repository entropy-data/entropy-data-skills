---
name: landscape-report
description: "Governance. Weekly governance landscape report over all data products and data contracts in Entropy Data: ownership, contract coverage, test health, glossary coverage, certifications, lifecycle, and the organization's own policies, returned as a scorecard per team with ranked findings. Use when a governance lead, data governance office, or auditor asks how compliant the data landscape is, needs a governance board report, or wants to know where to focus next."
---

# Governance Landscape Report

**Persona:** governance lead, data governance office, auditor
**Trigger:** weekly, or on demand before a governance board
**Output:** a scorecard per team and ranked findings, with the change since the last report

Read-only. Nothing in Entropy Data is changed.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Scope

Offer defaults and let the user adjust:

- Scope: whole organization, or one team and its sub-teams (via the teams' parent relationships).
- Thresholds: test results older than 7 days are stale; the test history window is the last 30 days.
- Output: `landscape-report-<date>.md` in the working directory.
- Comparison: if the user names an earlier report (or one is in the working directory), compare against it. In CI, restore the previous report as an artifact first.

## Step 2: Collect

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `dataproducts,datacontracts,access,teams,policies,certifications,definitions,tags` into a scratch directory, then a script that writes one summary line per product and per contract.

Test results have no bulk form. For each contract referenced by an output port in scope:

```bash
entropy-data test-results list --data-contract-id <contract-id> -p 0 -o json
```

Sort by timestamp yourself, keep only results on the contract itself (no `branch`), and stop paging once results are older than the history window. Cache the results in the scratch directory.

## Step 3: Analyze

Run each check per data product and record product, owning team, check, and evidence.

| Area | Check | Severity |
|---|---|---|
| Ownership | Owning team set and the team exists | high |
| Ownership | Description and status present; contract names an owner and terms of use | medium |
| Contracts | Every output port references a data contract that exists | high |
| Contracts | No draft contract behind an active product | high |
| Quality | Latest test result exists and passed | high |
| Quality | Latest test result within the staleness threshold | medium |
| Lifecycle | Deprecated or retired products with active access agreements | high |
| Certification | Active products carry a certification, if the organization defines certifications | low |
| Vocabulary | Share of contract fields with `authoritativeDefinitions`, per team | info |
| Vocabulary | Field names recurring across many contracts without `authoritativeDefinitions` (for example `customer_id` in 14 contracts): candidate glossary terms | info |
| Vocabulary | Tag hygiene: near-duplicates (`pii`, `PII`, `personal-data`), single-use tags | low |

Then the organization's own policies: for every policy in the export that states a checkable rule, translate it into a check over the collected data, describe how it was checked, and report violations by policy name. Policies that cannot be checked from the available data go under "not automatically checkable". Do not guess.

Each topic has one home. Report only headline numbers here for:

- access (pending requests, vague purposes): details in `access-review`,
- personal data: details in `pii-exposure-scanner`.

## Step 4: Output

1. **Headline**: products, contracts, teams, active agreements; findings by severity; the three biggest risks in three sentences; change since the last report.
2. **Scorecard by team**: one row per team, findings per area, so each team can be sent its row.
3. **Ranked findings**: high severity first, then by number of affected consumers. Each finding has product, team, and evidence, plus a UI link if the UI host is known.
4. **Policy coverage**: each policy, checked or not, violations.
5. **Top recommendations**: three to five actions with the biggest effect, for example "add contracts to 6 output ports of team X, all with active consumers".

Report only what the data shows. Do not notify teams or change anything from this skill. Delete the scratch directory at the end.
