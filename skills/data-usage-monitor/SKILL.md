---
name: data-usage-monitor
description: "Governance and security. Analyzes how data products are actually used in Entropy Data, from usage traces, and flags usage without a valid access agreement, usage after an agreement ended, bulk extraction from ports with personal data, queries on sensitive columns that the agreed purpose does not cover, and access by users outside the consuming team, ranked by risk. Use when a governance lead, privacy officer, or security team asks whether data is used as agreed, wants to detect purpose limitation breaches or bulk extraction, or wants a daily data usage check."
---

# Data Usage Monitor

**Persona:** governance lead, privacy officer, security team
**Trigger:** daily, or before an audit
**Output:** ranked usage incidents with the agreement, consumer, evidence, and a recommended action

Read-only. `access-review` checks who *may* access what; this skill checks what actually *happened*.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Scope and period

Offer defaults, and use them without asking in a scheduled or non-interactive run:

- Period: the last 24 hours for a daily run, the last 30 days on demand.
- Scope: all data products that report usage, or a named set.
- Bulk threshold: 10,000 rows per query. On request, 10 times the consumer's median row count instead, if at least 7 days of its history exist.

## Step 2: Collect

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `dataproducts,datacontracts,access,teams,policies,classification-schemes` into a scratch directory. Then the usage for the period:

```bash
entropy-data usage list --since <period start> --all -o json > "$SCRATCH/usage.json"
```

Usage is OpenTelemetry: each resource carries `dataproduct.id` and `outputport.name`; each span carries `user`, `role`, `access.id`, `query.type`, `query.preview`, `rows`, and `startTimeUnixNano`. A resource can repeat across entries, so flatten spans and group by attribute values (see "Usage" in the `entropy-data-cli` skill). `query.preview` is query text and may contain personal data: process it only with a script and never print it unmasked.

If the period actually covered is less than half of the requested period, say so in the first line of the answer. If no data product reports usage, say so, list the products with active agreements that do not report usage, and stop: this analysis needs usage traces. Recommend that the platform team reports usage (see the Entropy Data usage documentation).

## Step 3: Detect incidents

Join every span to its agreement through `access.id`, and the agreement to the provider product, its output port, and that port's contract. "Looks personal" and "active" are defined in the `entropy-data-cli` skill. Read the accepted policies about personal data and purpose limitation (for example one requiring the purpose to explain why personal fields are needed), apply them, and cite them in the evidence.

| Incident | How to detect | Default severity |
|---|---|---|
| Usage without an agreement | `access.id` missing, or not found among the agreements | high |
| Usage outside the agreement | span on a different product or port than the agreement's `provider` | high |
| Usage after the agreement ended | span time after `info.endDate` (a missing `endDate` is open-ended), or the agreement is not active, rejected, or cancelled | high |
| Bulk extraction from a port with personal data | `rows` above the bulk threshold on a port whose contract has personal-looking fields: high if the query reads them (or `SELECT *`), medium if it reads only other fields | high or medium |
| Sensitive columns outside the purpose | the query reads fields classified as sensitive (or personal-looking), and the agreement's `info.purpose` does not plausibly need them (for example a purpose about revenue reporting selecting `EMAIL`) | medium |
| Access by an unexpected user | `user` is a person or service not associated with the consuming team or data product, where that can be seen | medium |
| Purpose does not match the query pattern | without sensitive fields, the queries do something other than the purpose says (for example per-customer lookups for a payment calculation), or the purpose is copied word for word from another agreement | low |
| Columns not in the contract | the query reads columns the port's contract does not define | low |
| Unusual activity | a consumer whose daily query or row count is at least 3 times its median over at least 7 days; with less history, "not evaluable" | low |

Field matching: extract column names from `query.preview` with a script (upper- and lowercase, with or without table prefix) and compare them with the contract's field names. `SELECT *` on a port with sensitive fields counts as reading all of them.

Users in traces are often technical names (for example a service user named after the consuming product), while team members are email addresses. Match a user to the consumer team, consumer product, or members by name similarity and mark the match "possible". Do not compare the span `role` with the agreement's `custom.platformRole` or the port's role unless they use the same naming scheme; say when they do not.

Purpose judgement is interpretation. State the purpose, the sensitive fields read, and why they seem outside it, and mark it "possible".

## Step 4: Output

Ranked by severity, then by rows affected:

| # | Severity | Incident | Provider product / port | Consumer | Agreement | Evidence | Recommended action |
|---|---|---|---|---|---|---|---|

- **Evidence:** counts, time range, field names, and a masked query shape (for example `SELECT CUSTOMER_ID, EMAIL FROM …ORDERS WHERE …`), never literal values.
- **Recommended action:** for example "review agreement X with the consumer", "add the undocumented columns to the contract", "restrict the port to the non-PII contract", "revoke access granted outside an agreement in the data platform". Name the provider team's contact from the product's `support[]`.

Then a summary: spans analyzed, period actually covered (earliest to latest span), products with usage, incidents per severity, and products without usage reporting (some port types, for example streaming topics, may not report usage at all).

Report individual users as "user (n)" in chat; put identities only in the report file.

## Guardrails

- Never cancel access or contact consumers from this skill. Hand confirmed incidents to the provider team or to `access-review`.
- Never print `query.preview` raw.
