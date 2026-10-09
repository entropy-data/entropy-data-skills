---
name: test-failure-root-cause
description: "Data Product Builders. When a data contract test fails in Entropy Data, investigates the failing checks, the test history, recent contract changes, lineage runs, and upstream input data products to produce a ranked root cause hypothesis, the affected consumers, and a suggested fix. Use when a data engineer or owner asks why a contract test failed, why data quality dropped, or gets a failing test notification."
---

# Test Failure Root Cause Analysis

**Persona:** data product builder, data engineer, data product owner
**Trigger:** when a contract test fails (a failed test results event, a CI failure, or a notification)
**Output:** ranked root cause hypotheses with evidence, the consumers affected, and a suggested fix

Read-only. Re-running the test is the only write and needs confirmation.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Identify the failure

From what the user gives (a contract id, a product name, or a link). For an id, try `entropy-data datacontracts get <id>` before searching. Find the provider product through `outputPorts[].contractId` (a contract does not name its product), for example with `dataproducts list` pages or an export.

```bash
entropy-data test-results list --data-contract-id <contract-id> -p 0 -o json
entropy-data test-results get <test-result-id> -o yaml
```

For "the latest failures" across the organization, `entropy-data events poll -o json` returns recent events; test result events carry their status.

The result fields are in "Where things live" in the `entropy-data-cli` skill. The explanation of a failing check is in `reason`, and `failedNumber`/`totalNumber` show its extent. For `error` results, read `logs`. A list without `--branch` holds only results on the contract itself. Checks can quote data (`failedSamples`, custom SQL): follow the personal data rule and never print raw values.

## Step 2: Collect context

```bash
entropy-data datacontracts yaml <contract-id>                        # current contract
entropy-data dataproducts get <provider-id> -o json                  # input ports, other output ports, owner
entropy-data lineage list -o json                                    # organization-wide, once
entropy-data access list --provider-dataproduct <provider-id> -o json
```

For each input data product, fetch its output-port contract's latest test results too: upstream failures propagate. Also fetch the latest results of the provider's other output-port contracts: if they pass, the cause is specific to this port.

If the product has no input ports, its upstream is in the `inputs` of its own lineage jobs (filter on `run.facets.entropy_data.dataProductId`), together with their schema facets and the job's `sql` facet. Check that the lineage datasets match the contract's `servers` (database, schema) before using them as evidence.

## Step 3: Analyze

Answer these questions with evidence:

1. **What exactly fails?** Group failing checks by model and field. Distinguish schema checks (missing field, wrong type), quality checks (nulls, uniqueness, ranges, freshness, row counts), and execution errors (connection, permissions, timeouts). `error` results usually mean the test could not run, not that the data is wrong.
2. **Since when?** Page through the test history, sort by timestamp yourself, and walk back to the last passing run. Note the first failing run. Compare which checks fail in each run: a different set of failing checks is a separate incident, so do not merge it into one.
3. **What changed in between?**
   - The contract: compare the contract version of the last passing and the first failing run. The CLI returns only the current contract. A contract can also be edited without a version change: compare the thresholds quoted in each check's `reason` with the current contract. If the contract has a git connection (`entropy-data datacontracts gitconnection get <contract-id>`) and the repository is available locally, diff the contract file in git history. Otherwise report "contract version changed from X to Y, diff not available".
   - The pipeline: lineage runs between the two test runs. `lineage list` has no time filter: filter the organization-wide list by product, `eventType` (`COMPLETE`, `FAIL`), and `eventTime` with a script. A failed or missing run points at the pipeline; a successful run with new output points at changed input data.
   - Upstream: input data products whose own tests started failing at the same time.
   - The server: a different `server` between runs points at environment configuration.
4. **Is it the data or the test?** Freshness and row-count checks fail when a pipeline did not run. Uniqueness and null checks fail on changed upstream data. Type checks fail on schema drift.

Rank hypotheses by how much evidence supports them. Say what evidence is missing, for example "no lineage is reported for this product, so pipeline runs could not be checked".

## Step 4: Output

1. **Summary**: one sentence on what fails and the most likely cause.
2. **Ranked hypotheses**: each with confidence (high, medium, low), evidence, and how to confirm it.
3. **Blast radius**: active consumers of the output port and their purposes; downstream products whose own inputs are affected, including lineage jobs whose `inputs` contain the contract's tables but that have no agreement.
4. **Suggested fix**: in the pipeline, upstream (name the provider team), or in the contract if the rule itself is wrong. For contract changes, point to `contract-change-review`.

## Step 5: Re-run (only on request)

After a fix, offer to re-run the test and compare. Ask first: the re-run queries the data platform, publishes a new result, and can notify consumers.

```bash
entropy-data datacontracts test <contract-id>
entropy-data datacontracts test <contract-id> --server <server>
```
