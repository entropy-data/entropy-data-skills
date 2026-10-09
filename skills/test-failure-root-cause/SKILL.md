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

From what the user gives (a contract id, a product name, or a link):

```bash
entropy-data test-results list --data-contract-id <contract-id> -p 0 -o json
entropy-data test-results get <test-result-id> -o yaml
```

For "the latest failures" across the organization, `entropy-data events poll -o json` returns recent events; test result events carry their status.

Inspect one result to learn the field names. Typically a result has an overall result (passed, failed, warning, error), the server it ran against, the contract version, timestamps, a branch if it ran on a contract branch, and a list of checks with type, name, result, model, field, and message. Make sure the failure is on the contract itself, not on a branch.

## Step 2: Collect context

```bash
entropy-data datacontracts yaml <contract-id>                        # current contract
entropy-data dataproducts get <provider-id> -o json                  # input ports and owner
entropy-data lineage list --data-product-id <provider-id> -o json    # recent pipeline runs
entropy-data access list --provider-dataproduct <provider-id> -o json
```

For each input data product, fetch its output-port contract's latest test results too: upstream failures propagate.

## Step 3: Analyze

Answer these questions with evidence:

1. **What exactly fails?** Group failing checks by model and field. Distinguish schema checks (missing field, wrong type), quality checks (nulls, uniqueness, ranges, freshness, row counts), and execution errors (connection, permissions, timeouts). `error` results usually mean the test could not run, not that the data is wrong.
2. **Since when?** Page through the test history, sort by timestamp yourself, and walk back to the last passing run. Note the first failing run.
3. **What changed in between?**
   - The contract: compare the contract version of the last passing and the first failing run. The CLI returns only the current contract. If the contract has a git connection (`entropy-data datacontracts gitconnection get <contract-id>`) and the repository is available locally, diff the contract file in git history. Otherwise report "contract version changed from X to Y, diff not available".
   - The pipeline: lineage runs between the two test runs. `lineage list` has no time filter: fetch with `--event-type COMPLETE` and `--event-type FAIL` and filter by time with a script. A failed or missing run points at the pipeline; a successful run with new output points at changed input data.
   - Upstream: input data products whose own tests started failing at the same time.
   - The server: a different `server` between runs points at environment configuration.
4. **Is it the data or the test?** Freshness and row-count checks fail when a pipeline did not run. Uniqueness and null checks fail on changed upstream data. Type checks fail on schema drift.

Rank hypotheses by how much evidence supports them. Say what evidence is missing, for example "no lineage is reported for this product, so pipeline runs could not be checked".

## Step 4: Output

1. **Summary**: one sentence on what fails and the most likely cause.
2. **Ranked hypotheses**: each with confidence (high, medium, low), evidence, and how to confirm it.
3. **Blast radius**: active consumers of the output port and their purposes; downstream products whose own inputs are affected.
4. **Suggested fix**: in the pipeline, upstream (name the provider team), or in the contract if the rule itself is wrong. For contract changes, point to `contract-change-review`.

## Step 5: Re-run (only on request)

After a fix, offer to re-run the test and compare. Ask first: the re-run queries the data platform, publishes a new result, and can notify consumers.

```bash
entropy-data datacontracts test <contract-id>
entropy-data datacontracts test <contract-id> --server <server>
```
