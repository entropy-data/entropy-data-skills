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

- Scope: whole organization, one domain (a team of type `Domain` and its sub-teams via `parent`), or one team.
- Thresholds: a latest test result whose date is more than 7 days before the report date is stale; the test history window is the last 30 days.
- Output: `landscape-report-<date>.md` in the working directory.
- Comparison: if the user names an earlier report (or one is in the working directory), compare against it by finding key (check plus product or contract id). In CI, restore the previous report as an artifact first.

## Step 2: Collect

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `dataproducts,datacontracts,access,teams,policies,certifications,definitions,tags,classification-schemes` into a scratch directory, then a script that writes one summary line per product and per contract.

Test results are not in the export. Fetch them organization-wide, once:

```bash
entropy-data test-results list --since <start of the history window> --all -o json
```

Group by `dataContractId` and `server`, and sort by `timestampStart` yourself (the list is not sorted). Without `--branch`, the list holds only results on the contract itself. A `warning` result counts as not passed, with medium severity.

## Step 3: Analyze

Run each check per data product and record product, owning team, check, and evidence.

| Area | Check | Severity |
|---|---|---|
| Ownership | Owning team (`team.name`) set and the team exists | high |
| Ownership | Description and status present; contract names an owner and terms of use | medium |
| Contracts | Every output port references a data contract that exists (high if the port has active consumers, that is approved agreements with `info.active: true`, otherwise medium; ports such as reports or APIs often have none by design, so name the port type). Active products without output ports: info | high |
| Contracts | No `draft` or `proposed` contract behind an active product | high |
| Contracts | Contracts referenced by no product, listed with their status and team | info |
| Quality | Latest test result per server exists and passed. Servers are the contract's `servers[]`, or the one server a port names in its custom properties. A contract without servers cannot be tested: report it | high |
| Quality | Latest test result within the staleness threshold | medium |
| Lifecycle | Deprecated or retired products with active access agreements | high |
| Lifecycle | Draft or proposed products that provide data through active access agreements (as consumer: info) | medium |
| Certification | Active products carry a certification tag (`certification/<name>`), if the organization defines certifications; certified products meet that certification's `criteria` where checkable | low |
| Vocabulary | Share of contract fields with `authoritativeDefinitions` of `type: semantics` (or links to `definitions`), per team | info |
| Vocabulary | Field names recurring across several contracts (for example three or more, compared case-insensitively) without such links: candidate glossary terms | info |
| Vocabulary | Tag hygiene across product, port, contract, and field tags: near-duplicates that differ only in case, separators, or prefix, and several tags for the same meaning (for example `pii`, `PII`, `pii:true`, `personal-data`). Ignore technology labels | low |

Certification criteria are free text. Map them to data with the common mappings and state each one in the report: SLAs to `slaProperties`, quality checks to `quality` rules, classified fields to `classification` on every property, versioned ports to `outputPorts[].version`, automated tests to fresh test results. Criteria with no mapping are "not checkable". When a criterion conflicts with a policy (for example it requires classification that a policy calls optional), report the conflict instead of failing every product on it.

Then the organization's own policies (see "Where things live" in the `entropy-data-cli` skill for reading them and mapping older field names): for every policy in the export that states a checkable rule, translate it into a check over the collected data, describe how it was checked, and report violations by policy name, with medium severity unless the policy says otherwise. For soft wording ("meaningful", "avoid abbreviations"), state the heuristic you used. Report draft policies separately and do not count their violations. Team rules (members, channels) apply to teams of type `Domain` only if the policy names domains; "every team" alone does not. Policies that cannot be checked from the available data go under "not automatically checkable". Do not guess.

Each topic has one home. Report only headline numbers here for:

- access (pending requests, vague purposes): details in `access-review`,
- personal data: details in `pii-exposure-scanner`.

## Step 4: Output

1. **Headline**: products, contracts, teams, active agreements; findings by severity; the three biggest risks (highest severity, then most affected consumers) in three sentences; change since the last report.
2. **Scorecard by team**: one row per team, findings per area, so each team can be sent its row. Product findings go to the product's team, contract findings to the contract's `team.name`. Contracts without a team get a "(no team)" row.
3. **Ranked findings**: high severity first, then by number of affected consumers of the product (0 for contracts no product references). Each finding has product, team, and evidence, plus a UI link if the UI host is known.
4. **Policy coverage**: each policy, checked or not, violations.
5. **Top recommendations**: three to five actions with the biggest effect, for example "add contracts to 6 output ports of team X, all with active consumers".

Report only what the data shows. Do not notify teams or change anything from this skill. Delete the scratch directory at the end.
