---
name: entropy-data-cli
description: Core. Set up and use the entropy-data CLI, the core component every other Entropy Data skill builds on. Use when the user wants to install the CLI, connect to an Entropy Data organization, check which organization they are talking to, or when another skill's Step 0 says to verify the connection.
---

# Entropy Data CLI

Every skill in this library talks to Entropy Data through the [`entropy-data`](https://pypi.org/project/entropy-data/) CLI. This skill covers installing it, connecting it, and the conventions all other skills follow.

## Step 1: Check the CLI is installed

```bash
entropy-data --version
```

If the command is missing, propose one of these and let the user choose:

```bash
uv tool install entropy-data        # recommended
pipx install entropy-data
pip install entropy-data
```

Inside a project that pins it as a dev dependency, use `uv run entropy-data …` instead of a global install.

## Step 2: Check the connection

```bash
entropy-data connection test
entropy-data organization get -o yaml
```

If `connection test` succeeds, tell the user which organization (from `organization get`) and which connection and host (from `entropy-data connection list`) they are using, and stop here. If `ENTROPY_DATA_API_KEY` is set, it overrides the stored connections and `connection list` does not show it: then `organization get` is the only reliable source for the organization and host.

If it fails, the user needs an API key. Ask them to create one in the Entropy Data UI (in the organization settings, or in a team's settings for a team-scoped key), then run **themselves**:

```bash
entropy-data connection add prod          # prompts for API key and host
entropy-data connection set-default prod
```

Do not ask the user to paste the API key into the chat, and do not write it into files that may be committed. Alternatives the user can choose:

- Environment variables `ENTROPY_DATA_API_KEY` and `ENTROPY_DATA_HOST` (good for CI).
- A `.env` file with the same variables, exported into the shell before running the CLI: `set -a; . ./.env; set +a`. Make sure `.env` is in `.gitignore`. (The CLI's own `.env` loading does not find a file in the current directory in version 0.3.24, so export it.)

The default host is `https://api.entropy-data.com`. Self-hosted installations use their own host.

Behind a corporate proxy that inspects TLS (`CERTIFICATE_VERIFY_FAILED`), pass the global option before the subcommand, `entropy-data --system-truststore connection test`, or export `ENTROPY_DATA_SYSTEM_TRUSTSTORE=1`.

The analyses in this library only read. A read-only API key is enough for them; only explicit actions such as approving or requesting access, staging a contract branch, or running a contract test need write access.

## Step 3: Several organizations or environments

```bash
entropy-data connection list
entropy-data -c test dataproducts list     # one-off against another connection
```

Before any write, state which connection you are about to write to when more than one exists.

## Conventions every skill follows

### Safety

**Read-only commands** are only: `list`, `get`, `yaml`, `branches changes`, `search query`, `semantics search`, `star-status`, `events poll`, `schemas get`, `organization get`, `connection test`, `connection list`, and `export dir`. **Every other command needs the user's explicit confirmation first**, naming the exact resources (for example the agreement ids). That includes `put`, `delete`, `access request|approve|reject|cancel`, `datacontracts test` (runs queries against the data platform and publishes a result that can notify consumers), `test-results publish`, `lineage submit`, `usage submit`, `branches create|put|merge|rebase|delete`, `gitconnection push|push-pr|pull`, `integrations run`, `star`, `apply dir`, `sync`, and `import zip`. Run `--dry-run` first where it exists. Never pass `--yes` unless the user asked for it.

**Never reveal secrets.** Never run `connection get --show-api-key`, `integrations get` (returns decrypted configuration), or `git-credentials get|list`. Never pass `--api-key` on the command line (it ends up in shell history), and never ask the user to paste an API key into the chat.

**Never output personal data unmasked.** Example data and usage traces can contain real personal data. Scan them with a script that prints only masked matches (for example `j***@e***.com`); never print raw example data or raw traces into the conversation. Individual users appear as consumers by user id or email: in chat summaries, count them ("2 individual users") and put identities only in the report file.

**Do not notify people.** Draft messages and pull request comments; send or post them only after explicit confirmation.

### Collecting data at scale

Organizations can have hundreds of data products. Do not call `get` once per product and do not read raw API output into the conversation.

1. Make a scratch directory outside the user's repository, for example `SCRATCH=$(mktemp -d)`.
2. Pull the organization's state in one go:

   ```bash
   entropy-data export dir "$SCRATCH/org" --include dataproducts,datacontracts,access,teams,policies,certifications,definitions,tags,sourcesystems,classification-schemes
   ```

   The tree has one folder per resource kind and one YAML file per resource. A kind with no resources gets no folder, so scripts must not assume every folder exists. Add `example-data` only when a skill needs it, and treat it as personal data (see above).
3. Analyze the tree with a script (Python or `jq`/`yq`) that writes compact summaries, for example one line per product with team, status, ports, contract ids. Read the summaries, not the raw files. If Python has no YAML library, run it with `uv run --with pyyaml python …`.
4. Fetch org-wide lists once and group locally. Prefer one `access list` over one call per provider.
5. Test results, lineage, and usage are not part of the export, but each has an organization-wide form: `test-results list -p <n>` (paged, without `--data-contract-id`), `lineage list`, and `usage list`. Fetch them once into the scratch directory and group locally; filter per product or contract only when the scope is small. `costs list` has no organization-wide form and needs `--data-product-id`.
6. Delete the scratch directory at the end unless the user wants to keep it.

If `export dir` is not permitted for the API key, fall back to paged `list` calls written to files in the scratch directory.

### Reading the CLI

**Structured output.** Use `-o json` when you parse output and `-o yaml` when you show a resource to the user. Table output is for humans only.

**Learn field names from data.** Field names differ between versions and installations. Before relying on a field, inspect one real item (`<group> get <id> -o yaml`) instead of assuming.

**Paging.** These lists are paged and 0-indexed (`-p 0`, `-p 1`, …): `teams`, `dataproducts`, `datacontracts`, `access`, `sourcesystems`, `definitions`, `certifications`, `test-results`, `assets`, `tags`, `organization members`. With `-o json` there is no "more results" hint, so request pages until one is empty or shorter than the first (pages hold 10 items, for example). Other lists (`policies`, `classifications`, `example-data`, `lineage`, `usage`, `costs`, `connectors`) are not paged; do not pass `-p`. Ordering is not guaranteed: sort by timestamp yourself when you need "latest". `test-results list` has no date filter: page and filter by `timestampStart` locally. `usage list` has neither paging nor a time filter and can return only a capped set of recent spans: state the time range the spans cover and draw no conclusions for a longer period.

**Find before you ask.** When the user gives an id, try `get <id>` first: search does not reliably return contracts by id. When the user names something loosely ("the orders product"), resolve it with `entropy-data search query "orders" -o json`. Search matches keywords: use single entity words and synonyms, not sentences, and ignore the chat-style `answer` field next to `results`. Results carry no `status`; take it from the export. Retry once on an HTTP 500. To filter, take the `resourceType` values from an unfiltered result and pass one with `--resource-type`. If several candidates match, list them and ask which one.

**Where things live** (verify on one item, versions differ):

- Data product (ODPS): `team.name` (an object holding the team id, not a string), `domain` (the id of a team of type `Domain`), `status` (for example `active`, `draft`, `proposed`, `deprecated`, `retired`), `description.purpose` (an object, not a string), `tags`, `support[]`, `outputPorts[]` with `name`, `type`, `contractId`, `tags`, and `customProperties` (for example `status`, `autoApprove`), `inputPorts[]` (often empty: dependencies then show only in access agreements and lineage). Output ports have no `id`: every `outputPortId` (agreements, example data, lineage) is the port's `name`. The product type can appear both as top-level `type` and as a `type` custom property, and the two can disagree.
- Data contract (ODCS v3): `status`, `team.name` and `team.members[]` (owner and contacts), `description.purpose|usage|limitations` (the terms of use), `servers[]`, `slaProperties[]` with `property` such as `freshness`, `frequency`, or `retention`, `customProperties` (for example `noticePeriod`), `tags`, and `schema[].properties[]` (nested through `properties` and `items`) with `primaryKey`, `required`, `quality`, `tags`, `examples`, `classification`, `authoritativeDefinitions`. A contract does not name its product: find it through `outputPorts[].contractId`. Some contracts are referenced by no product.
- Policies: free-text Markdown in `content`, with a `status` (a draft policy is not in force yet). Read every policy's `content` and extract the rules relevant to the analysis. Policies may name fields of the older Data Contract Specification; map them to ODCS (`info.contact` to `team.members[]` or `support`, `terms.usage|limitations` to `description.usage|limitations`, `info.status` to `status`) and state the mapping in the report.
- Classification: a `classification` field on contract properties whose value matches a scheme classification's `name` or `externalId`, case-insensitively (it may also sit in the property's `customProperties` under the scheme's `customProperty` name). The scheme gives each value a `level` (higher is more sensitive). Many organizations classify only a few fields and mark personal data with property tags instead (for example `pii:true` or `pii:false`) or contract tags. Check all of these. When no contract in scope is classified, report a classification check as "not applicable", not as "no findings".
- Business concepts: `authoritativeDefinitions` on products, input ports, and contract properties. Entries with `type: semantics` link to semantic concepts (the URL ends in `/semantics/<namespace>/<concept>`; compare concepts case-insensitively). Other types (for example `implementation`) are not concepts. The `definitions` kind is often empty.
- Tags: on products, output ports, contracts, and contract properties; registered tags are in the `tags` kind.
- Certification: a product tag `certification/<name>`; each certification declares its `tag`, `rank`, and free-text `criteria`.
- Access agreement: `id` (use it as is; it can carry a prefix such as `access-`), `info.status` (requested, approved, rejected, …), `info.active`, `info.purpose`, `info.startDate`, `info.endDate` (there is no request or creation timestamp); `provider.dataProductId|outputPortId|teamId`; `consumer.teamId` or `consumer.userId`, plus `consumer.dataProductId`, which can be a placeholder such as `unknown` (treat it as "no data product"); `custom.platformRole`; optional `provisioning` with `status` (for example `provisioning`, `provisioned`, `provisioning-failed` with `failedAt` and `diagnostics`, `deprovisioned`) and timestamps such as `provisioningStartedAt` and `deprovisioningStartedAt`.
- Teams: `type` (for example `Domain`), `parent`, and `members[]`, so domains are teams with their sub-teams. Domain teams often have no members by design.
- Test result: `result` (`passed`, `failed`, `warning`, `error`), `dataContractId`, `dataContractVersion`, `server` (a server name; a contract can be tested against several, so group by server), `timestampStart`, `logs[]`, `checks[]` with `type`, `name`, `result`, `model`, `field`, `reason` (the explanation, including thresholds), `failedNumber`, `totalNumber`, and sometimes `failedSamples` (raw values: treat as personal data). Results carry no branch field: `test-results list` without `--branch` returns only results on the contract itself.
- Usage: an OTLP object `{"resourceSpans": [...]}`, not a list. Spans are under `resourceSpans[].scopeSpans[].spans[]`, and attributes are lists of `{key, value: {stringValue|intValue}}`. The resource carries `dataproduct.id` and `outputport.name`, each span `user`, `role`, `access.id`, `query.type`, `query.preview` (query text, may contain personal data), `result`, `rows`. The resource's port can differ from the port of the agreement named by `access.id`: report such mismatches instead of resolving them silently.
- Lineage: OpenLineage events with `eventType`, `eventTime`, `inputs`/`outputs` (physical datasets with `namespace` and `name`), a job facet `sql`, and a run facet `entropy_data` with `dataProductId`, `outputPortId`, and sometimes `dataContractId` of the job's data product. `--data-product-id` finds only that product's own jobs; readers of its data appear in other events' `inputs`, so search the organization-wide list. Ids in the facet and dataset names do not always match contract ids or the contract's `servers` (database, schema): match on schema and table names, and treat the producing job's `outputs` as the port's real location.
- Example data: entries with `dataProductId`, `outputPortId`, `dataContractId`, `schemaName`, and `data`, usually a list of rows, sometimes a single object; values can be nested.
- Events (`events poll`): CloudEvents with `type` such as `…TestResultsCreatedEvent`, whose `data.status` is the test result.

**Specifications.** Data contracts follow ODCS and data products follow ODPS. `entropy-data schemas get odcs|odps [--custom]` returns the JSON Schema, including the organization's custom fields with `--custom`. 

**Links.** Search results carry a `url`, a path such as `/<org>/dataproducts/<id>` in the Entropy Data UI. Lineage events carry similar `…Href` paths. Prefix them with the UI host only when you know it (for example from `organization get`, or from absolute links in the organization's own data). The API host `api.entropy-data.com` is not the UI. Never invent URLs.

**Long operations.** Large documents can exceed the 30 second default; retry with the global `--timeout 120`. `--debug` prints HTTP requests when a command fails without a clear message.

## Command map

| Area | Commands |
|---|---|
| Discovery | `search query`, `dataproducts list/get`, `datacontracts list/get/yaml`, `assets list/get`, `semantics search` |
| Quality and telemetry | `test-results list/get`, `lineage list`, `usage list`, `costs list`, `events poll` |
| Access | `access list/get` (read), `access request/approve/reject/cancel` (write) |
| Governance | `policies`, `classifications`, `definitions`, `certifications`, `tags`, `semantics …` |
| Organization | `teams`, `organization get`, `organization members`, `sourcesystems` |
| Bulk | `export dir` |

Run `entropy-data <group> --help` for exact options.
