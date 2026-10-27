---
name: insights-dashboards
description: "Governance leads and data platform teams. Builds custom metrics and customizes the dashboards of Insights in Entropy Data: writes Apache Ossie semantic models with metrics on the Entropy Data Marketplace Analytics datasets, and dashboards in the Perses format with numbers, charts over time, bars, tables, pie charts, funnels, scatter charts, and text, then validates them and saves them after confirmation. Use when someone asks for a new KPI or governance metric, a custom or changed dashboard, a chart of some measure by team or domain, or how to configure a chart type in Insights."
---

# Insights Dashboards and Custom Metrics

**Persona:** governance lead, data platform team, anyone who owns governance KPIs
**Trigger:** on demand, when a KPI or dashboard is missing or needs to change
**Output:** a semantic model with the new metrics and a dashboard that shows them, validated, saved after confirmation, with the links to open them

Insights v2 is a preview. An organization owner switches it on under **Settings > Preview Features > Insights v2**; until then its pages and its API answer 404. The API is experimental and may still change.

How the pieces fit:

- **Datasets.** Every metric reads the tables of the Entropy Data Marketplace Analytics data contract: `fct_data_product`, `fct_data_product_output_port`, `fct_access_agreement`, `dim_domain`, and `dim_team` in the schema `dp_marketplace_analytics_v1`. Each holds the current state of the organization's own data products, output ports, and access agreements. The contract is in the UI under Insights > **Data Contract**.
- **Semantic models** define metrics on those datasets, as Apache Ossie documents with Entropy Data extensions. The built-in model `entropy_data_marketplace_analytics` holds the standard metrics and can't be changed: duplicate it, or write a model of your own.
- **Dashboards** are [Perses](https://perses.dev) `PersesDashboard` documents whose panels query metrics as `<model>.<metric>`. The built-in Overview has the id `overview`; saving a dashboard with that id replaces it for the organization, and deleting that saved copy brings the built-in one back.

The analysis is read-only. Saving a model or a dashboard is the only write: it needs the user's confirmation and a key with governance rights: an organization key that is not read-only, or the user key of an organization owner or a governance group member.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2, and note the organization's `vanityUrl`.

The CLI has no commands for Insights yet, so this skill calls the REST API with `curl`. It reads the key from the environment, never from the CLI's stored connections and never from the chat:

```bash
set -a; [ -f .env ] && . ./.env; set +a          # ENTROPY_DATA_API_KEY and ENTROPY_DATA_HOST
test -n "$ENTROPY_DATA_API_KEY" -a -n "$ENTROPY_DATA_HOST" || echo "missing"
insights() { curl -sS -H "x-api-key: $ENTROPY_DATA_API_KEY" -H "Accept: application/json" "$ENTROPY_DATA_HOST$1" "${@:2}"; }
insights /api/insights/dashboards -o /dev/null -w "%{http_code}\n"
```

If the variables are missing, ask the user to export them (or put them in a `.env` that is in `.gitignore`); do not ask for the key in the chat. `200` means ready. `404` means Insights v2 is off for the organization. `403` means the key may not read dashboards. Without API access, the skill still works: write the files and let the user paste them into the editors in the UI (Step 6).

## Step 1: Understand the request

Pin down, and ask where it is unclear:

- **The question each number answers**, for example "how many of our data products are well scored" or "how long access requests wait". Ask for the definition in business terms: what counts, what is excluded (drafts? retired?), and the unit.
- **Its shape**: one number, a course over time, a comparison by team or domain or by some property, shares of a whole, steps of a process, or one dot per data product.
- **Audience**: everyone in the organization, or only some teams (they see the dashboard; others do not).
- **Where**: a new dashboard, an existing one, or the Overview.

## Step 2: Read what exists

```bash
SCRATCH=$(mktemp -d)
insights /api/insights/semantic-models -o "$SCRATCH/models.json"
insights /api/insights/dashboards -o "$SCRATCH/dashboards.json"
insights /api/insights/semantic-models/entropy_data_marketplace_analytics/model.yaml -o "$SCRATCH/builtin.ossie.yaml"
insights /api/analytics/experimental/entropy-data-marketplace-analytics-v1 -o "$SCRATCH/datasets.json"
```

`semantic-models` needs an organization key or a governance user's key; `datasets.json` (every table with its columns, types, and descriptions) needs an organization key. If one is not allowed, read the data contract in the UI instead.

Summarize with a script, not by reading the raw files: per model its metrics (`name`, label, unit, whether it has a `time`), per dataset its fields and which are dimensions, and per dashboard its panels and the metrics they query. **Reuse before you build:** if a built-in metric already answers the question, use it on the dashboard instead of defining it again.

## Step 3: Define the metrics

Write a model of your own (or extend one the organization already has); never edit the built-in one. Start from `examples/governance_kpis.ossie.yaml` next to this skill, or from the downloaded built-in model, keeping only the datasets the new metrics read: a model is a complete document, and its metrics can read only the datasets it lists.

**Model.** `version: 0.2.0.dev0`, a `name` in lower case with digits and `_` (it becomes the metric reference prefix), a `description`, and the extension below. Optional `relationships` (`from`, `to`, `from_columns`, `to_columns`) document how datasets join.

**Dataset.** `name` (how metrics refer to it), `source: dp_marketplace_analytics_v1.<table>`, `primary_key`, the extension with `data_product`, `data_contract`, and `schema` (the table), and `fields`:

- A **column field** has the column name as its expression and the contract's `datatype` (`String`, `Integer`, `Decimal`, `Boolean`, `Date`, `DateTimeTz`).
- A **computed field** has a name of its own and an expression over the columns of one row, usually `CASE WHEN … THEN 'label' … END`.
- `dimension: {is_time: false}` makes a field a **category**: dashboards can group and stack by it. `dimension: {is_time: true}` marks a timestamp that metrics can be dated by. Fields without `dimension` are only measured.
- Primary key columns can be counted without listing them as fields.

**Metric.** `name`, `description`, and an expression in the `OSSIE_SQL_2026` dialect (dashboards run only this subset):

- Aggregates: `COUNT` (also `COUNT(*)`), `SUM`, `AVG`, `MIN`, `MAX`, each also with `DISTINCT`, and `MEDIAN` and `PERCENTILE(x, 0.9)`. Each aggregate reads exactly one dataset; a metric may combine aggregates of several.
- Inside and around them: arithmetic, comparisons, `AND`/`OR`/`NOT`, `IS NULL`, `IN`, `BETWEEN`, `LIKE`/`ILIKE`, `CASE`, numeric `CAST`, `NULLIF`, `COALESCE`, `ABS`, `ROUND`, `CEIL`, `FLOOR`, `GREATEST`, `LEAST`, `LOWER`, `UPPER`, `DAYS_BETWEEN(start, end)`, and `NOW()`.
- Refer to fields as `dataset.field`. No `FILTER (WHERE …)`: put a `CASE WHEN … THEN 1 END` inside the aggregate. No subqueries, window functions, or database-specific syntax.
- A metric always computes a number. A share is `100.0 * COUNT(CASE WHEN <condition> THEN 1 END) / NULLIF(COUNT(<all>), 0)` with the unit `%`.

**The `ENTROPY_DATA` extension.** Each element can carry one entry in `custom_extensions` with `vendor_name: ENTROPY_DATA` and `data`, a JSON object written as a string:

| Element | Keys |
|---|---|
| Model | `label` (its display name), `owner` (the owning team's id) |
| Dataset | `data_product`, `data_contract`, `schema` (which contract table it reads) |
| Computed field | `values` (the field's values in the order dashboards show them; a funnel needs them), `sequential` (`true` for a scale listed from the least to the most, `"descending"` from the most), `neutral` (values shown in grey, such as `not-scored`), `warning` (values shown as needing attention), `colors` (`{"value": "#rrggbb"}`) |
| Metric | `label`, `unit` (such as `%` or `days`), `time`, `cumulative`, `until`, `rating` |

The time keys decide what a metric can show:

- **No `time`**: the metric is measured today. It shows as a number, bars, a table, a pie, or a funnel, but not over time, not as a scatter chart, and without a trend under its number.
- **`time: ["dataset.field"]`**, one timestamp per dataset the metric reads: rows count in the period by that date, such as data products published in the last 90 days. The metric can be shown over time, by week or month.
- **`cumulative: true`** (with `time`): every row dated before each point in time counts, such as all data products published up to then.
- **`until: ["dataset.field"]`** (with `cumulative`): a row counts only until that timestamp, such as access requests that are still waiting.

`rating: "data-product-score"` shows a value with the organization's Data Product Score rating, such as Good.

Groupings a metric supports: `domain`, `top-level-domain`, and `team` when every dataset it reads has them (the analytics tables do), and `dataset.field` for each category field of the one dataset it reads.

## Step 4: Design the dashboard

Start from `examples/governance-board.dashboard.yaml`, which uses every panel kind, or from an existing dashboard (`insights /api/insights/dashboards/<id>/dashboard.yaml`).

**Dashboard.** `metadata.name` is its id (letters, digits, `.`, `_`, `-`). Annotations `entropy-data.com/owner` (a team id) and `entropy-data.com/visible-to` (team ids, comma-separated, including their subteams; leave it out for everyone). `spec.display.name` and `description`. `spec.duration` is the period it opens with, `30d` or `90d` (viewers can also choose this year); metrics without a time are measured today whatever the period. `spec.variables` adds filters every panel follows: a `ListVariable` with plugin `EntropyDataDomainVariable` (a domain and its subdomains) or `EntropyDataTeamVariable` (a team and its subteams), at most one of each.

**Panels.** Each panel has a `plugin.kind`, up to three `queries` of kind `TimeSeriesQuery` with plugin `EntropyDataMetricQuery` and `spec.metric: <model>.<metric>`, an optional `display.name` (otherwise the metric's label, plus "by team" for grouped charts), and optional `links`.

| `plugin.kind` | Shows | Needs | Configure |
|---|---|---|---|
| `StatChart` | one number; for a metric with a time, its change in a month and its course over the period | any metric | a second query is the total the number is a share of ("22 % of total"), in the same unit |
| `TimeSeriesChart` | the course over the period, by week or month | a metric with `time` | up to 3 metrics in the same unit, one line each; `querySettings`; `thresholds` |
| `BarChart` | one bar per group | `groupBy` (`domain` if left out) | `orientation: vertical` for columns (horizontal is the default); `stackBy` splits each bar by a second grouping; or up to 3 metrics in the same unit as the parts of each bar (not both); `querySettings`; `thresholds` |
| `Table` | one row per group | `groupBy` | up to 3 metrics, one column each, units may differ |
| `PieChart` | the shares of the groups | a count or sum, `groupBy` | |
| `FunnelChart` | the steps of a process: each row counts for the furthest step it reached and every step before | a single `COUNT` or `SUM` without `DISTINCT`, and `groupBy` a field whose model lists its `values` in step order (a computed field) | |
| `ScatterChart` | one dot per row, such as each data product, at its time | a metric with `time` on one dataset with a primary key, not cumulative | `groupBy` colours the dots (optional) |
| `Markdown` | text, such as the takeaways of a section | no queries | `plugin.spec.text` in Markdown |

Chart options (in `plugin.spec`):

- `querySettings`: `[{queryIndex: 0, colorMode: fixed, colorValue: "#4f46e5"}]` colours a query's series; `fixed-single` colours only a query with one series, so stacked parts keep their own colours.
- `thresholds`: `{mode: absolute, steps: [{value: 75, color: "#ef4444", name: Over 75 days}]}` tints the value axis from each step's value upwards.
- `calculation: last-number` is what Perses expects on numbers, bars, pies, and funnels; Entropy Data shows one value per series anyway.

Other Perses options are accepted and ignored: Entropy Data formats values from the metric's unit.

**Links** on a panel open pages with more detail: `links: [{name: Data Product Score, url: /governance/dataproducts/score}]`. A URL starting with `/` opens in the organization; `https://…` opens as it is; `targetBlank: true` opens a new tab. The title opens the first link.

**Layout.** `spec.layouts` is a list of `Grid`s, each an optional section with `display.title` and `display.collapse.open`, and `items` placing panels by `$ref: '#/spec/panels/<key>'` on 24 columns: `x`, `y`, `width`, and `height` in rows of about 30 pixels. Numbers fit `width: 6, height: 3` (four per row), charts and tables `width: 12, height: 8`, text `height: 2` to `3`. Every panel must be placed, and gaps close upwards as in Perses.

## Step 5: Validate

Check locally before saving:

- The YAML parses; every metric reference names an existing model and metric (from Step 2, or the model being written).
- Each panel fits its metric: over time and scatter charts need a `time`; series of a chart and a number's total share the metric's unit; a funnel's `groupBy` field lists `values`; `groupBy` and `stackBy` are groupings the metric supports.
- The dashboard validates against its JSON schema, which the YAML editor also uses: `curl -sS "$ENTROPY_DATA_HOST/schemas/dashboard-schema.json"`.

The server checks everything again when saving and answers `422` with every problem and its line. That is the authoritative check: fix and save again.

## Step 6: Save (only with confirmation)

Show the user the files and what changes (new or replaced model or dashboard, which panels), then ask. Save the model before the dashboard that uses it.

**With the API.** The body is the document as JSON, and the id in the path must match the document's `name` (`metadata.name` for a dashboard):

```bash
yaml2json() { uv run --with pyyaml python -c 'import json,sys,yaml; json.dump(yaml.safe_load(open(sys.argv[1])), sys.stdout)' "$1"; }
yaml2json governance_kpis.ossie.yaml > "$SCRATCH/model.json"
insights /api/insights/semantic-models/governance_kpis -X PUT -H "Content-Type: application/json" --data-binary @"$SCRATCH/model.json" -w "\n%{http_code}\n"
yaml2json governance-board.dashboard.yaml > "$SCRATCH/dashboard.json"
insights /api/insights/dashboards/governance-board -X PUT -H "Content-Type: application/json" --data-binary @"$SCRATCH/dashboard.json" -w "\n%{http_code}\n"
```

- `200`: saved; the `location-html` response header is the page in the UI.
- `422`: the `detail` lists the problems; fix them and save again.
- `409` on a model: the change removes or renames metrics that dashboards show, and the `detail` names them. Ask the user, then save again with `?removedMetrics=remove` (take those widgets off the dashboards) or `?removedMetrics=keep` (leave them, showing that the metric is gone).
- `403`: the key lacks governance rights.

**Without the API.** In the UI, **Insights > Semantic Models > Add semantic model** opens the model editor: paste the model and save. **Insights > Edit > New dashboard** (or **Edit** on an existing one) opens the dashboard editor, which switches between Visual and YAML: paste the dashboard into YAML. Both editors show problems and a live preview while typing.

Keep the YAML files in the user's repository if they want dashboards as code; the Perses format is the one Dash0 and other Perses tools exchange.

## Step 7: Hand over

Give the links, built from the `vanityUrl` and the UI host (ask for the host if unknown; the API host is not always the UI):

- `/<vanityUrl>/governance/insights-v2/<dashboard id>`
- `/<vanityUrl>/governance/semantic-models/<model name>`

Summarize what was saved: the metrics with their definitions in one line each, the dashboard's panels, and who sees it. Delete the scratch directory.
