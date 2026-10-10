# Entropy Data Skills

Ready-made analyses for [Entropy Data](https://entropy-data.com) that run in your coding agent (Claude Code, Codex, GitHub Copilot CLI, or any agent that supports [Agent Skills](https://agentskills.io)).

Each skill is one outcome for one persona: a governance landscape report, a duplicate data product finder, a health briefing for your data products. The agent collects what it needs from Entropy Data through the [`entropy-data` CLI](https://github.com/entropy-data/entropy-data-cli), analyzes it, and returns ranked findings with evidence. Skills are read-only unless you explicitly ask for an action, and they confirm every write.

## Catalog

### Top Picks

| Skill | What you get |
|---|---|
| **[Governance Landscape Report](skills/landscape-report/SKILL.md)** | Weekly scorecard per team across ownership, contract coverage, test health, glossary coverage, certifications, lifecycle, and your own policies, with ranked findings and the change since last week. |
| **[Duplicate Data Product Finder](skills/duplicate-dataproduct-finder/SKILL.md)** | Monthly clusters of duplicate and overlapping data products, found by comparing schemas, grain, sources, lineage, and business concepts, with a consolidation recommendation per cluster. |
| **[PII Exposure Scanner](skills/pii-exposure-scanner/SKILL.md)** | Scans contracts and example data for personal data, flags unclassified fields and real-looking personal values in example data, and lists who has access, with masked samples. |
| **[Contract Change Review](skills/contract-change-review/SKILL.md)** | On a pull request that changes a data contract, a green/yellow/red verdict: what breaks, which consumers are affected, and how to roll it out. |
| **[Test Failure Root Cause Analysis](skills/test-failure-root-cause/SKILL.md)** | When a contract test fails, a ranked root cause hypothesis from the test history, contract versions, pipeline runs, and upstream products. |

### Governance

| Skill | Trigger | What you get |
|---|---|---|
| **[Governance Landscape Report](skills/landscape-report/SKILL.md)** | weekly | Scorecard per team and ranked governance findings, including checks derived from your organization's policies. |
| **[Duplicate Data Product Finder](skills/duplicate-dataproduct-finder/SKILL.md)** | monthly | Ranked clusters of duplicate, overlapping, and related data products with evidence and a recommendation. |
| **[PII Exposure Scanner](skills/pii-exposure-scanner/SKILL.md)** | daily or weekly | Unclassified personal data in contracts, personal data leaking through example data, and who can access it. |
| **[Access Review](skills/access-review/SKILL.md)** | quarterly | Recertification of all access agreements: vague purposes, stale requests, access to deprecated products, provisioning gaps. |
| **[Governance Incident Triage](skills/governance-incident-triage/SKILL.md)** | daily or hourly | An always-on governance inbox: what changed since the last run, the governance conditions it caused, prioritized high, medium, low with owner and recommended action. |
| **[Data Usage Monitor](skills/data-usage-monitor/SKILL.md)** | daily | Usage without a valid agreement, after it ended, bulk extraction of personal data, and sensitive columns read outside the agreed purpose. |
| **[Semantic Namespace Review](skills/semantic-namespace-review/SKILL.md)** | on demand, quarterly | Proposed improvements to a business ontology (duplicates, descriptions, relationships, missing concepts), put on a namespace branch to review visually in the Entropy Data UI. |
| **[Unmanaged Data Finder](skills/unmanaged-data-finder/SKILL.md)** | monthly | How much of the imported data estate data products actually govern, ranked ungoverned sensitive tables, broken asset references, and stale imports. |

### Data Product Owners

| Skill | Trigger | What you get |
|---|---|---|
| **[Data Product Health Briefing](skills/dataproduct-health-briefing/SKILL.md)** | weekly | Quality trends, consumers and purposes, usage versus granted access, costs, and pending access requests with a recommendation each, as one to-do list. |
| **[Contract Change Review](skills/contract-change-review/SKILL.md)** | on pull request, before a change or retirement | Breaking changes, affected consumers and downstream products, a verdict, a rollout plan, and notification drafts. |

### Data Product Builders

| Skill | Trigger | What you get |
|---|---|---|
| **[Data Discovery and Fitness Check](skills/data-discovery/SKILL.md)** | at the start of a new data product or analysis | A ranked shortlist of data products for your use case with a fitness verdict, the gaps, and how to get access. |
| **[Test Failure Root Cause Analysis](skills/test-failure-root-cause/SKILL.md)** | when a contract test fails | Ranked root cause hypotheses, blast radius, and a suggested fix. |

### Core

| Skill | What it does |
|---|---|
| **[Entropy Data CLI](skills/entropy-data-cli/SKILL.md)** | Installs and connects the `entropy-data` CLI and defines the conventions every other skill follows. Used automatically by the other skills. |

To build data products, use the platform-specific builder plugins, for example [dataproduct-builder-dbt](https://github.com/entropy-data/dataproduct-builder-dbt) and [dataproduct-builder-databricks](https://github.com/entropy-data/dataproduct-builder-databricks).

## Install

### Claude Code

```
claude plugin marketplace add https://github.com/entropy-data/entropy-data-skills
claude plugin install entropy-data-skills@entropy-data-skills
```

### OpenAI Codex

```
codex plugin marketplace add https://github.com/entropy-data/entropy-data-skills
codex plugin add entropy-data-skills@entropy-data-skills
```

### GitHub Copilot CLI

```
copilot plugin marketplace add https://github.com/entropy-data/entropy-data-skills
copilot plugin install entropy-data-skills@entropy-data-skills
```

### Any other agent

Clone the repository and point your agent at `skills/`, or reference [`AGENTS.md`](AGENTS.md) from your project's own `AGENTS.md`.

## Connect to Entropy Data

The skills use the [`entropy-data` CLI](https://pypi.org/project/entropy-data/):

```bash
uv tool install entropy-data
entropy-data connection add prod     # prompts for API key and host
entropy-data connection test
```

Create the API key in Entropy Data. The analyses only read, so a read-only key is enough. Write access is only needed for explicit actions you confirm, such as approving or requesting access, staging a contract change on a branch, or running a contract test. Or ask your agent to "set up the Entropy Data CLI".

## Usage

Ask in plain language:

- "Create the governance landscape report for our organization."
- "Do we have duplicate data products?"
- "What needs governance attention since yesterday?"
- "Is our customer data used as agreed?"
- "Which tables in Snowflake are not covered by any data product?"
- "Review our global semantic namespace and put your proposals on a branch."
- "Scan for personal data exposure."
- "How are the data products of team Checkout doing?"
- "What breaks if I remove `customer_email` from the orders contract?"
- "Why did the shipments contract test fail?"
- "Which data should I use to predict customer churn?"

### Run on a schedule

Analyses with a weekly or daily trigger work well unattended, for example in a scheduled CI job that runs your agent headless and stores the report as an artifact. Nobody is there to confirm actions, so:

- use a **read-only API key** (`ENTROPY_DATA_API_KEY` and `ENTROPY_DATA_HOST` from your CI secrets), and
- allow only what the analysis needs.

```bash
claude -p "Run the landscape-report skill for the whole organization and write landscape-report-$(date +%F).md" \
  --allowedTools "Bash(entropy-data:*),Bash(python3:*),Bash(jq:*),Read,Write"
```

To compare with the previous run, restore the previous report from your CI artifacts into the working directory first.

## Principles

- **Analysis first.** Skills collect and analyze. Anything that changes Entropy Data or notifies people needs your explicit confirmation.
- **Evidence over verdicts.** Every finding names the data it is based on. Heuristics are labeled as "possible".
- **No personal data in output.** Sample values are masked.
- **Your rules.** Where your organization defines policies, classifications, or certifications in Entropy Data, the analyses use them.

## License

[MIT](LICENSE)
