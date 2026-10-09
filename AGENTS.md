# entropy-data-skills: agent manifest

> This file is the plugin's routing manifest. Reference it from your project's own `AGENTS.md` instead of copying it, so it updates with the plugin.

This repository is a coding-agent plugin with ready-made analyses for [Entropy Data](https://entropy-data.com). Each capability is a **skill**: a markdown file under `skills/<name>/SKILL.md` that you read top to bottom and execute step by step.

When a user request matches a skill, **read the corresponding `SKILL.md` completely before acting.** Each skill's frontmatter `description` is authoritative for when to use it.

## Skills

| Category | When the user asks about… | Follow this skill |
|---|---|---|
| Governance | How compliant the data landscape is, a governance scorecard or board report | `skills/landscape-report/SKILL.md` |
| Governance | Duplicate, overlapping, or redundant data products | `skills/duplicate-dataproduct-finder/SKILL.md` |
| Governance | Where personal data is, PII or GDPR exposure, personal data in example data | `skills/pii-exposure-scanner/SKILL.md` |
| Governance | Access review, recertification, who has access to what and why | `skills/access-review/SKILL.md` |
| Data Product Owners | How their data products are doing, who uses them, pending access requests | `skills/dataproduct-health-briefing/SKILL.md` |
| Data Product Owners | What breaks if a contract or product changes or is retired; reviewing a contract pull request | `skills/contract-change-review/SKILL.md` |
| Data Product Builders | Which data to use for a use case, whether a data product is fit for purpose | `skills/data-discovery/SKILL.md` |
| Data Product Builders | Why a data contract test failed | `skills/test-failure-root-cause/SKILL.md` |
| Core | Installing or connecting the `entropy-data` CLI | `skills/entropy-data-cli/SKILL.md` |

Every skill verifies the CLI connection first (`entropy-data connection test`, see `skills/entropy-data-cli/SKILL.md`) and stops if it fails. Skills never ask for API keys in the chat.

## Conventions when running skills

- **Read-only by default.** Only `list`, `get`, `yaml`, `changes`, `search query`, `star-status`, `schemas get`, `organization get`, `connection test|list`, and `export dir` run without asking. Every other command needs the user's explicit confirmation, naming the exact resources.
- **Never reveal secrets.** Never run `connection get --show-api-key`, `integrations get`, or `git-credentials get|list`, and never pass `--api-key` on the command line.
- **Do not notify people** (comments, messages, emails) without explicit confirmation; draft instead.
- **Evidence for every finding**, and label heuristic findings as "possible".
- **Never output unmasked personal data.**
- **Collect in bulk** (`export dir` into a scratch directory outside the user's repository), analyze with scripts, read summaries instead of raw output, and delete the scratch directory at the end.
