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

If `connection test` succeeds, tell the user which organization (from `organization get`) and which connection and host (from `entropy-data connection list`) they are using, and stop here.

If it fails, the user needs an API key. Ask them to create one in the Entropy Data UI (in the organization settings, or in a team's settings for a team-scoped key), then run **themselves**:

```bash
entropy-data connection add prod          # prompts for API key and host
entropy-data connection set-default prod
```

Do not ask the user to paste the API key into the chat, and do not write it into files that may be committed. Alternatives the user can choose:

- Environment variables `ENTROPY_DATA_API_KEY` and `ENTROPY_DATA_HOST` (good for CI).
- A `.env` file in the working directory with the same variables. Make sure `.env` is in `.gitignore`.

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

**Read-only commands** are only: `list`, `get`, `yaml`, `changes`, `search query`, `star-status`, `schemas get`, `organization get`, `connection test`, `connection list`, and `export dir`. **Every other command needs the user's explicit confirmation first**, naming the exact resources (for example the agreement ids). That includes `put`, `delete`, `access request|approve|reject|cancel`, `datacontracts test` (runs queries against the data platform and publishes a result that can notify consumers), `test-results publish`, `lineage submit`, `usage submit`, `branches create|put|merge|rebase|delete`, `gitconnection push|push-pr|pull`, `integrations run`, `star`, `apply dir`, `sync`, and `import zip`. Run `--dry-run` first where it exists. Never pass `--yes` unless the user asked for it.

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

   The tree has one folder per resource kind and one YAML file per resource. Add `example-data` only when a skill needs it, and treat it as personal data (see above).
3. Analyze the tree with a script (Python or `jq`/`yq`) that writes compact summaries, for example one line per product with team, status, ports, contract ids. Read the summaries, not the raw files.
4. Fetch org-wide lists once and group locally. Prefer one `access list` over one call per provider.
5. Make per-item calls only for data without a bulk form (`test-results list`, `lineage list`, `usage list`), only for the items in scope, and cache the results in the scratch directory.
6. Delete the scratch directory at the end unless the user wants to keep it.

If `export dir` is not permitted for the API key, fall back to paged `list` calls written to files in the scratch directory.

### Reading the CLI

**Structured output.** Use `-o json` when you parse output and `-o yaml` when you show a resource to the user. Table output is for humans only.

**Learn field names from data.** Field names differ between versions and installations. Before relying on a field, inspect one real item (`<group> get <id> -o yaml`) instead of assuming.

**Paging.** These lists are paged and 0-indexed (`-p 0`, `-p 1`, …): `teams`, `dataproducts`, `datacontracts`, `access`, `sourcesystems`, `definitions`, `certifications`, `test-results`, `assets`, `tags`, `organization members`. With `-o json` there is no "more results" hint, so request pages until one is empty. Other lists (`policies`, `classifications`, `example-data`, `lineage`, `usage`, `costs`, `connectors`) are not paged; do not pass `-p`. Ordering is not guaranteed: sort by timestamp yourself when you need "latest".

**Find before you ask.** When the user names something loosely ("the orders product"), resolve it with `entropy-data search query "orders" -o json`. To filter, take the `resourceType` values from an unfiltered result and pass one with `--resource-type`. If several candidates match, list them and ask which one.

**Specifications.** Data contracts follow ODCS and data products follow ODPS. `entropy-data schemas get odcs|odps [--custom]` returns the JSON Schema, including the organization's custom fields with `--custom`. Links from contract fields to business definitions or concepts live in `authoritativeDefinitions`.

**Links.** Only include links to the Entropy Data UI when you know the UI host (ask the user once; the API host `api.entropy-data.com` is not the UI). Never invent URLs.

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
