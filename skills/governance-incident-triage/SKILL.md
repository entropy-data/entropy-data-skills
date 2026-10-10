---
name: governance-incident-triage
description: "Governance. Always-on triage of governance events in Entropy Data: reads what changed since the last run (new or changed data products, contracts, access requests, failed provisioning, test results, deleted assets), detects governance conditions such as products without owner or contract, sensitive data without a policy, failing quality, or contracts changed without a new version, and returns a prioritized action inbox (high, medium, low) with the owner and the recommended action for each item. Use when a data steward or governance team wants a daily governance inbox, asks what needs governance attention now, or wants to run governance monitoring on a schedule."
---

# Governance Incident Triage

**Persona:** data steward, governance team
**Trigger:** daily or hourly on a schedule, or on demand ("what needs attention?")
**Output:** a prioritized action inbox: one item per governance condition, with severity, evidence, owner, and recommended action

Read-only. The skill detects and recommends; it does not act and does not notify anyone.

The weekly `landscape-report` looks at everything. This skill looks only at what changed since the last run, so it stays cheap and can run often.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Know where the last run stopped

Keep the position in a small state file, by default `.entropy-data-triage.json` in the current directory: `{"organization": "...", "lastEventId": "...", "lastRun": "...", "contractVersions": {"<id>": "<version>"}, "openItems": ["<condition>:<resource>"]}`. Ask once whether that location is fine; in a scheduled or non-interactive run, use the default and say so. In CI, restore it from the previous run's artifacts.

- With a state file: check that `organization get` returns the same organization, then read events after `lastEventId`.
- Without one (first run): `events poll` can only start at the organization's first event. Page through the whole history to reach the latest id, evaluate the organization's current state, and label the inbox a baseline. The 1,000-event guardrail does not apply to a baseline.

## Step 2: Read the new events

```bash
entropy-data events poll --last-event-id <lastEventId> -o json      # first run: without --last-event-id
```

Repeat with the id of the last event received until a poll returns `[]`. Paging, payloads, and the stale-id trap are described under "Events" in the `entropy-data-cli` skill. If a stored id returns `[]` but `entropy-data test-results list --since <lastRun> -o json` shows newer results, the id is stale: say so and run a baseline instead of reporting "nothing new".

Group the events by resource, so a product changed five times is evaluated once. Match on the name after the last dot of `type`:

| Events | Resource |
|---|---|
| `DataProduct…Event`, `OutputPort…Event` (`data.id` is the product, `data.outputPortId` the port) | data product |
| `DataContract…Event` | data contract |
| `Access…Event` (for example `AccessCreatedEvent`, `AccessActivatedEvent`, `AccessProvisioningStartedEvent`, `AccessProvisionedEvent`, `AccessProvisioningFailedEvent`, `AccessDeprovisionedEvent`) | access agreement |
| `TestResultsCreatedEvent` (with `data.status`) | test result |
| `Asset…Event` other than `AssetCreatedEvent` | asset, and what referenced it |
| `Policy…Event`, `Classification…Event` | policies, classification schemes |
| `…DeletedEvent` of products or contracts | the remaining references to it |

Ignore `TeamCreatedEvent`, `TagCreatedEvent`, `AssetCreatedEvent`, and events about stars and example data unless a policy makes them relevant.

The payload is thin, and several conditions span resources (ports, contracts, agreements). For a handful of changed resources, `get` each one plus `access list`; for more, follow "Collecting data at scale" in the `entropy-data-cli` skill with one `export dir` and one `test-results list --since <lastRun> --all`.

## Step 3: Evaluate governance conditions

For each changed resource, check the conditions below, using the related products, contracts, and agreements as context. "Active consumers" and "looks personal" are defined in the `entropy-data-cli` skill. Read the organization's policies (`entropy-data policies list -o yaml`) first: where a policy states a rule, use it and cite it; a policy may also raise or lower a severity. A violation of an accepted policy that no row covers is medium, high when personal data is involved.

| Condition | Default severity |
|---|---|
| Active data product without an owning team | high |
| Output port with consumers but without a data contract | high |
| Contract field that looks personal without a classification or `pii` tag (unless a policy makes classification optional), or classified sensitive while no policy covers it | high |
| Access provisioning or deprovisioning failed | high |
| Approved access on a port whose contract contains sensitive data, with an empty or generic purpose | high |
| Test result `failed` on a contract with active consumers | high |
| Contract changed without a new version: a `DataContractUpdatedEvent` whose contract `version` equals the one in `contractVersions` (not evaluable on a baseline; mark "possible", the update may be metadata only) | medium |
| Access provisioning started but neither provisioned nor failed after 24 hours, or deprovisioned while the agreement is still approved and active | medium |
| Test result `failed` without consumers, or `warning` | medium |
| Access request waiting longer than the organization's limit (default 14 days, measured from its `AccessCreatedEvent` `time`, not `info.startDate`) | medium |
| Product or contract deleted while agreements or lineage still reference it | medium |
| Asset deleted that a data product still references | medium |
| Asset import disabled automatically | medium |
| New or changed data product without description or tags | low |
| Active contract referenced by no output port (a candidate for a port without contract) | low |
| Policy or classification scheme changed: products that may need re-evaluation | low |

Only report a condition that holds now: a test failure followed by a later passing result is resolved, not an incident.

## Step 4: Triage

Order the inbox by:

1. severity,
2. blast radius: number of active consumers and whether sensitive data is involved,
3. age: oldest first within the same severity.

Merge items with the same cause (for example one contract change that breaks three checks) into one item, and say what the root cause most likely is. Merge items with the same condition and the same owner team into one item that lists the resources. For a failing test, `test-failure-root-cause` can dig deeper.

## Step 5: Output

```
## Governance inbox, <date> (events <first id> to <last id>)

| # | Severity | Condition | Resource | Owner | Evidence | Recommended action |
```

- **Owner:** the owning team of the product or contract, or the provider team of an agreement.
- **Recommended action:** concrete and small, for example "add a data contract to port X", "classify field Y as confidential", "re-run provisioning for agreement Z and check its diagnostics".
- End with counts per severity, and the items in `openItems` of the state file that no longer hold (resolved since the last run).

Save the inbox as `inbox-<date>.md` next to the state file. Then write `organization`, the new `lastEventId`, `lastRun`, the current `contractVersions`, and the keys of open items to the state file. Write it only after the inbox was produced, so a failed run is repeated rather than skipped.

## Guardrails

- Never act on an item (approve, reject, cancel, change a contract) from this skill.
- Mask personal data in evidence. Name fields, never print values.
- If more than 1,000 events arrived since the last run, say so and offer to switch to the `landscape-report` instead.
