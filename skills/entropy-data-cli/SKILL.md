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

If `connection test` succeeds, tell the user which organization (from `organization get`) and which connection and host (from `entropy-data connection list`) they are using, and continue with the calling skill. Environment variables (`ENTROPY_DATA_API_KEY`, `ENTROPY_DATA_HOST`) override the stored connections; `connection list` names them in a notice line above the table. Trust that line and `organization get`, not a table row marked `(env)`, which can show a stored connection that is not in use.

If it fails, the user needs an API key. Ask them to create one in the Entropy Data UI (in the organization settings, or in a team's settings for a team-scoped key), then run **themselves**:

```bash
entropy-data connection add prod          # prompts for API key and host
entropy-data connection set-default prod
```

Do not ask the user to paste the API key into the chat, and do not write it into files that may be committed. Alternatives the user can choose:

- Environment variables `ENTROPY_DATA_API_KEY` and `ENTROPY_DATA_HOST` (good for CI).
- A `.env` file with the same variables in the current directory, which the CLI loads. Make sure `.env` is in `.gitignore`. Version 0.3.24 and older do not find it: export it first with `set -a; . ./.env; set +a`.

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

   The tree has one folder per resource kind and one YAML file per resource. A kind with no resources gets no folder, so scripts must not assume every folder exists. Add `example-data` only when a skill needs it, and treat it as personal data (see above). The export is also the way to find a contract's product and a team's products: `dataproducts list` items carry no `outputPorts`.
3. Analyze the tree with a script (Python or `jq`/`yq`) that writes compact summaries, for example one line per product with team, status, ports, contract ids. Read the summaries, not the raw files. If Python has no YAML library (also in isolated mode `-I`), run it with `uv run --with pyyaml python …`. Certifications and classification schemes carry large inline SVG `icon` fields: drop them before reading (`jq 'del(..|.icon?)'`).
4. Fetch org-wide lists once and group locally. Prefer one `access list` over one call per provider.
5. Test results, lineage, and usage are not part of the export. Fetch each once, organization-wide and bounded by the analysis period, into the scratch directory and group locally (see "Time ranges"). Filter per product or contract only when the scope is small. `costs list` has no organization-wide form and needs `--data-product-id`.
6. Delete the scratch directory at the end unless the user wants to keep it.

If `export dir` is not permitted for the API key, fall back to paged `list` calls written to files in the scratch directory.

### Reading the CLI

**Structured output.** Use `-o json` when you parse output and `-o yaml` when you show a resource to the user. Table output is for humans only.

**Learn field names from data.** Field names differ between versions and installations. Before relying on a field, inspect one real item (`<group> get <id> -o yaml`) instead of assuming.

**Errors.** A failed command exits with 1 and writes to stderr, never stdout. With the global option before the subcommand (`entropy-data -o json …`), stderr carries `{"error": {"type", "message", "status", "url"}}`; with `-o json` after the subcommand, and in CLI 0.3.24 and older, it is plain text. Check the exit code before parsing stdout, and keep stderr separate. An unknown `--data-product-id` on `usage list` or `lineage list`, or `--data-contract-id` on `usage list`, returns 404 (filters combine), but `access list` and `example-data list` return `[]` for unknown ids: confirm the id exists (in the export or with `get`) before reading an empty list as "none".

**Time ranges.** `test-results list`, `lineage list`, and `usage list` take `--since` and `--until` (an ISO date or date-time, or relative like `7d` or `2w`), `--page/-p`, `--limit/-l`, and `--all`; `test-results list` also takes `--server/-s`. Without `--all`, `usage list` (100 spans) and `test-results list` (10 results) truncate and print "More results available…" on stderr only, which a script redirecting stdout misses. For complete data, always pass `--all` and bound it with `--since`, for example `entropy-data usage list --since 90d --all -o json`. State the period the data actually covers (earliest to latest timestamp); it can be shorter than requested. The help says newest first, but `test-results list` is not sorted that way: always sort by `timestampStart` yourself.

**Paging.** Other lists are paged and 0-indexed (`-p 0`, `-p 1`, …): `teams`, `dataproducts`, `datacontracts`, `access`, `sourcesystems`, `definitions`, `certifications`, `assets`, `tags`, `organization members`. They print no "more results" hint, and page sizes differ by command, so request pages until one is empty. `policies`, `classifications`, `example-data`, `costs`, and `connectors` are not paged; do not pass `-p`. Ordering is not guaranteed: sort by timestamp yourself when you need "latest".

**Find before you ask.** When the user gives an id, try `get <id>` first: search does not return contracts by id. Enumerate from the export, not from search. Use search only for fuzzy lookup when the user names something loosely ("the orders product"), one query per type: `entropy-data search query "orders" --resource-type DATA_PRODUCT -o json`, then `DATA_CONTRACT`. Unfiltered results are capped at 10 and can leave out whole resource types. Search matches keywords: use single entity words and synonyms, not sentences, and ignore the chat-style `answer` field next to `results`. Each query takes several seconds. Results carry no `status`; take it from the export. Contract results have `url: "null"` (a string). Search can return HTTP 500 when a semantic concept or a tag without owner team ranks in the results: retry once, then narrow the type. For concepts, do not use `--resource-type SEMANTIC_CONCEPT` (it fails); use `semantics namespaces list` and `semantics search <namespace> <term>`, whose results include each concept's `classification`. If several candidates match, list them and ask which one.

**Where things live** (verify on one item, versions differ):

- Data product (ODPS): `team.name` (the owning team's id, a string inside the `team` object), `domain` (the id of a team of type `Domain`), `status` (for example `active`, `draft`, `proposed`, `deprecated`, `retired`), `description.purpose` (text; `description` can be empty), `context.instructions` and `synonyms[]` (often the most descriptive text), `tags`, `support[]`, `outputPorts[]` with `name`, `type`, `contractId`, `tags`, and `customProperties` (for example `status`, `autoApprove`), `inputPorts[]` (often empty: dependencies then show only in access agreements and lineage). Output ports have no `id`: every `outputPortId` (agreements, example data, lineage) is the port's `name`. The product type is the top-level `type` (ODPS 1.1.0); ODPS 1.0.0 documents keep it in `customProperties` as `type`. Read `type` first, then fall back to the custom property. `settings dataproduct-types get` lists the types the organization defines.
- Data contract (ODCS v3): `status`, `team.name` (a string) and `team.members[]` (owner and contacts), `description.purpose|usage|limitations` (the terms of use), `servers[]`, `slaProperties[]` with `property` such as `freshness`, `frequency`, or `retention`, `customProperties` (for example `noticePeriod`), `tags`, and `schema[].properties[]` (nested through `properties` and `items`) with `primaryKey`, `required`, `quality`, `tags`, `examples`, `classification`, `authoritativeDefinitions`. A contract does not name its product: find it through `outputPorts[].contractId`. Some contracts are referenced by no product.
- Policies: free-text Markdown in `content`, with a `status` such as `Accepted` or `Draft` (compare case-insensitively; a draft policy is not in force yet). Read every policy's `content` and extract the rules relevant to the analysis. Policies may name fields of the older Data Contract Specification; map them to ODCS (`info.contact` to `team.members[]` or `support`, `terms.usage|limitations` to `description.usage|limitations`, `info.status` to `status`) and state the mapping in the report.
- Classification: a `classification` field on contract properties whose value matches a scheme classification's `name` or `externalId`, case-insensitively (it may also sit in the property's `customProperties` under the scheme's `customProperty` name). The scheme gives each value a `level` (higher is more sensitive). Many organizations classify only a few fields and mark personal data with property tags instead (for example `pii:true` or `pii:false`) or contract tags. Check all of these. When no contract in scope is classified, report a classification check as "not applicable", not as "no findings".
- Business concepts: `authoritativeDefinitions` on products, input ports, and contract properties. Entries with `type: semantics` link to semantic concepts. The URL usually ends in `/semantics/<namespace>/<concept>`, but can also be an ontology IRI with another path: match concepts on the last path segment, case-insensitively. Other types (for example `implementation`) are not concepts. The `definitions` kind is often empty.
- Tags: on products, output ports, contracts, and contract properties; registered tags are in the `tags` kind.
- Certification: a product tag `certification/<name>`; each certification declares its `tag`, `rank`, and free-text `criteria`.
- Access agreement: `id` (use it as is; it can carry a prefix such as `access-`), `info.status` (requested, approved, rejected, …), `info.active`, `info.purpose`, `info.startDate`, `info.endDate` (there is no request or creation timestamp); `provider.dataProductId|outputPortId|teamId`; `consumer.teamId` or `consumer.userId`, plus `consumer.dataProductId`, which can be a placeholder such as `unknown` (treat it as "no data product"); `custom.platformRole`; optional `provisioning` with `status` (for example `provisioning`, `provisioned`, `provisioning-failed` with `failedAt` and `diagnostics`, `deprovisioned`) and timestamps such as `provisioningStartedAt` and `deprovisioningStartedAt`.
- Teams: `type` (for example `Domain`), `parent`, and `members[]`, so domains are teams with their sub-teams. Domain teams often have no members by design.
- Test result: `result` (`passed`, `failed`, `warning`, `error`), `dataContractId`, `dataContractVersion`, `server` (a server name; a contract can be tested against several, so group by server), `timestampStart`, `logs[]`, `checks[]` with `type`, `name`, `result`, `model`, `field`, `engine`, `qualityCategory`, `qualityCheckDefinition` (the executed check, useful to match it to a contract rule), and, mostly on failing checks only, `reason` (the explanation, including thresholds), `failedNumber`, `totalNumber`, and `failedSamples` (a list of values of the checked column: treat as personal data unless the column clearly is not). Checks carry no `severity`: take it from the matching contract quality rule. `test-results list` already returns full `checks[]` and `logs[]`. Results carry no branch field: `test-results list` without `--branch` returns only results on the contract itself.
- Usage: an OTLP object `{"resourceSpans": [...]}`, not a list. Spans are under `resourceSpans[].scopeSpans[].spans[]`, and attributes are lists of `{key, value: {stringValue|intValue}}`. The resource carries `dataproduct.id` and `outputport.name`, each span `user`, `role`, `access.id`, `query.type`, `query.preview` (query text, may contain personal data), `result`, `rows`. The resource's port can differ from the port of the agreement named by `access.id`: report such mismatches instead of resolving them silently. Spans carry no contract attribute, so `--data-contract-id` can return no spans: filter by `--data-product-id` and group by `outputport.name`.
- Lineage: OpenLineage events with `eventType`, `eventTime`, `inputs`/`outputs` (physical datasets with `namespace` and `name`), job facets `sql` and sometimes `sourceCodeLocation` (repository, path, version), column lineage in `outputs[].facets.columnLineage.fields.<column>.inputFields[]`, and a run facet `entropy_data` with `dataProductId`, `outputPortId`, and sometimes `dataContractId` of the job's data product. `--data-product-id` finds only that product's own jobs; readers of its data appear in other events' `inputs`, so search the organization-wide list. The facet's ids and `…Href` paths may not match current ids or the organization's slug: match on `outputPortId` and dataset names, not on the facet's contract id or hrefs. Dataset names can name another database than the contract's `servers`: match on schema and table names, treat the producing job's `outputs` as the port's real location, and state the mismatch as a caveat.
- Example data: entries with `dataProductId`, `outputPortId`, `dataContractId`, `schemaName`, and `data`, usually a list of rows, sometimes a single object; values can be nested.
- Events (`events poll`): CloudEvents with `type` such as `…TestResultsCreatedEvent`. `data` holds only the result's `id`, `status`, and `timestamp`, and the event `time` is when it was ingested: look up the result with `test-results get <id>`.

**Specifications.** Data contracts follow ODCS and data products follow ODPS. `entropy-data schemas get odcs|odps [--custom]` returns the JSON Schema, including the organization's custom fields with `--custom`. 

**Links.** UI paths are `/<vanityUrl>/dataproducts/<id>` and `/<vanityUrl>/datacontracts/<id>`, with `vanityUrl` from `organization get`. Prefix them with the UI host only when you know it (for example from absolute links in the organization's own data); `api.entropy-data.com` is the API, not the UI, and the `host` in `organization get` can be an API host too. Otherwise give relative paths. Never invent URLs.

**Long operations.** Large documents can exceed the 30 second default; retry with the global `--timeout 120`. `--debug` prints HTTP requests when a command fails without a clear message.

## Command map

| Area | Commands |
|---|---|
| Discovery | `search query`, `dataproducts list/get`, `datacontracts list/get/yaml`, `assets list/get`, `semantics search` |
| Quality and telemetry | `test-results list/get`, `lineage list`, `usage list` (all with `--since --all`), `costs list`, `events poll` |
| Access | `access list/get` (read), `access request/approve/reject/cancel` (write) |
| Governance | `policies`, `classifications`, `definitions`, `certifications`, `tags`, `semantics …` |
| Organization | `teams`, `organization get`, `organization members`, `sourcesystems`, `settings dataproduct-types get` |
| Bulk | `export dir` |

Run `entropy-data <group> --help` for exact options.
