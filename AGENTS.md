# Strata Semantic Model Project

You are authoring a **semantic model** — version-controlled YAML in this repo (`models/tbl.*.yml`, `models/rel.*.yml`, datasources). That is not one-off SQL, warehouse ETL, or a standalone script.

| Term | Meaning in Strata |
|------|-------------------|
| **Semantic model** | What you edit in git — tables, fields, relationships, migrations |
| **Semantic layer** | What exists on the server **after deploy** — the governed dimensions, measures, and joins users query |

After **`strata deploy`**, this project’s model becomes (or updates) the **semantic layer** on the Strata server for that branch. The **Strata web app** charts and reports run against that layer, not against raw warehouse tables.

## What this model powers (end-to-end)

| Stage | What happens |
|-------|----------------|
| **You (agent + human)** | Edit the **semantic model** in `models/`; run `strata audit` |
| **Human** | `strata deploy` publishes the model to the **Strata server** (per git branch) |
| **Strata server** | Builds join universes, plans queries, runs SQL against datasources |
| **Strata web app** | Analysts query the deployed **semantic layer** — charting, reporting, filters, exploration by field **name** |

Field and table **names**, joins, and measure definitions affect **live dashboards and saved reports**, not only CLI or git. Breaking naming, duplicates, or joins can break production content after deploy. YAML is consumed by deploy on the server — it is **not** ad-hoc warehouse SQL run in isolation.

Read this file end-to-end before creating or editing `models/**/*.yml`. Do not mirror warehouse DDL or ad-hoc SQL style; follow the contracts below.

**Do not infer Strata behavior from warehouse SQL or generic BI tools.** Rules in this file are authoritative. Use the [official documentation index](#official-documentation) only when you need depth beyond what is inlined here.

## Critical rules

These are non-negotiable. Violations break deploy, query planning, or production reports.

1. **Every field `name` is unique project-wide** — one name = one dimension or measure entity across all `tbl.*.yml` files. Strata has no `table.field` namespaces; bracket references like `[Total Revenue]` resolve by name alone.
2. **`many_to_many` join cardinality is not supported.** Use a junction/bridge table with two relationships (`one_to_many` + `many_to_one`) instead.
3. **Measures on unrelated detail facts must have distinct names** — e.g. `Store Gross Sales`, `Catalog Gross Sales`, not one `Gross Sales` on three channel facts. See [Naming conventions](#naming-conventions).
4. **Dimensions may share names** when the business role is the same; Strata picks among tables at query time. **Measures do not combine that way** — duplicate measure names attach more SQL to the same measure entity (double-count risk).
5. **Cross-fact totals use compound measures or blending** — not the same measure name on multiple facts. See [Combined metrics across facts](#combined-metrics-across-facts).

## Unsupported or impossible requirements

**If the user asks for something Strata does not support, say so clearly and do not edit `models/**/*.yml` to approximate it.** A broken or misleading semantic model is worse than no change.

### Before writing YAML

1. **Classify the request:** supported in Strata · supported with a different pattern · unsupported · unknown.
2. **If unsupported or unknown:** explain the limitation, why a workaround would fail (deploy, double-count, invalid cardinality, invented keys), and offer **compliant alternatives** if any exist.
3. **Wait for the user to choose** a supported approach (or to change the requirement) before proposing file changes.
4. **Never ship “best effort” YAML** that you expect `audit` or deploy to reject, or that violates [Critical rules](#critical-rules) — fix the design in conversation first.

### What to tell the user

Use plain language, for example:

- “Strata does not support X. Doing Y in YAML would not work because …”
- “Supported alternative: … (trade-off: …)”
- “I’m not sure Strata supports X; I won’t change the model until we confirm — see [official documentation](#official-documentation) or your Strata admin.”

### Do not do these when blocked

- Invent YAML keys or properties not in Strata’s schema (`custom_join`, `blend_measures`, etc.).
- Use duplicate measure names, fake dimensions, or `many_to_many` joins as workarounds.
- Paste full ad-hoc report SQL into `expression.sql` and call it a semantic model.
- Rename or migrate entities to “make” an unsupported shape fit.
- Silently implement something that only works in raw warehouse SQL, not in Strata’s planner.

### Common unsupported asks → response pattern

| User ask | Why it fails | Compliant direction (if any) |
|----------|--------------|------------------------------|
| `many_to_many` join between two tables | Not supported | Junction table + two `rel` entries |
| One measure name on store + catalog + web facts | One measure entity, double-count risk | Distinct measures + [compound measure](#combined-metrics-across-facts) |
| “Combine metrics by reusing the same name” | Not how Strata merges facts | Compound measure or blending on shared dimensions |
| Cross-datasource compound measure | Same datasource required | Model per datasource or separate metrics |
| Datasource swap migration | Not supported | Rename datasource; human migration plan |
| Snapshot-style metric on a flow fact without `snapshot_date` | Wrong measure type | Normal additive measure, or proper snapshot table setup |
| Requirement needs engine feature you cannot verify | Risk of invalid model | Stop; cite docs or ask human — do not guess |

When `strata audit all --agent` reports errors you cannot resolve within Strata’s rules, **stop and report the errors to the user** — do not keep patching YAML until the model drifts further from a valid design.

## Order of operations for agents

Follow this sequence. Do not skip steps or reorder deploy before audit.

| Step | Who | Action |
|------|-----|--------|
| 1 | Agent | `strata datasource list --agent` → `tables` → `meta` (discovery) |
| 2 | Agent | `strata table list --agent` (existing models, if any) |
| 3 | Agent | Classify tables; check request is [supported](#unsupported-or-impossible-requirements); propose field names and joins (or explain why not) |
| 4 | Human | Approve proposal (required before any file writes) |
| 5 | Agent | Write `models/**/*.yml` (and `migrations/*.yml` if renaming — see [Renames and swaps](#renames-swaps-and-production-branch)) |
| 6 | Agent | `strata audit all --agent` — **required after every model change** |
| 7 | Human | `strata deploy` on the intended git branch (deploy runs audits again) |

**Agents must not run:** `strata deploy`, `strata datasource add`, interactive `strata create table` / `strata create relation` / `strata create migration`, or writing secrets into `.strata`.

**Branches:** Model on feature branches; deploy to the matching Strata server branch for staging. Production deploys typically use `production_branch` in `project.yml` (default `main`). Renames and swaps that affect production query names should happen on the production branch — see [Renames and swaps](#renames-swaps-and-production-branch).

## How the semantic layer maps to the warehouse

| Concept | In your YAML | Not the same as |
|--------|----------------|-----------------|
| Table users query | `name` in `tbl.*.yml` | `physical_name` (warehouse table) |
| Column in a table | `expression.sql` on each field | Bare column string or `physical_name.column` |
| Join endpoints | `left` / `right` (logical table names) | Table names inside join `sql` |
| Join condition | `sql` using `left.` and `right.` | `store_sales.col = date_dim.col` |
| Second FK to same dimension | New logical `tbl` (role-playing) + join to that name | Two `rel` entries with same `left` + `right` |

## Deploy contracts (required on every change)

Every `tbl` field and every `rel` join must satisfy **all** of these before deploy will succeed:

1. **Field `expression`** is a non-empty SQL string shortcut (`expression: order_id`, `expression: sum(amount)`) or a mapping with non-empty `sql`. Use the mapping form when you need `lookup`, `primary_key`, or `array`.
2. **Join `sql` uses only `left.` and `right.`** — column names from field definitions on those tables. Never warehouse prefixes (`orders.`, `date_dim.`, `store_sales.`).
3. **At most one join per logical table pair** in a branch. If a fact has two FKs to the same physical dimension (e.g. sold date and ship date both → `date_dim`), create **separate logical tables** (role-playing dimensions) and join to each distinct `name`.
4. **`left` / `right` in rel files match `name` in `tbl.*.yml` exactly** (same spelling and casing as the table model).

Run `strata audit all --agent` after edits; a human runs `strata deploy`.

## CLI for agents

Append `--agent` to every `strata` command (structured output, no prompts):

```bash
strata datasource list --agent
strata datasource tables MY_DS --agent
strata datasource meta MY_DS TABLE_NAME --agent
strata table list --agent
strata audit all --agent
```

**Do not run:** `strata create table`, `strata create relation` (interactive generators). Write `models/tbl.*.yml` and `models/rel.*.yml` directly instead.

If multiple datasources exist, pass `DS_KEY` on `tables` and `meta`.

## Datasources — human-in-the-loop (secrets)

If the CLI returns `no_datasources`, ask the user to run `strata datasource add [ADAPTER]` in a terminal. Do not create `datasources.yml` entries or fill `.strata` credentials yourself.

## Human-in-the-loop before file writes

Do **not** silently bulk-write model files. Before any `models/**/*.yml` change:

0. **If the request is unsupported** — follow [Unsupported or impossible requirements](#unsupported-or-impossible-requirements); do not proceed to file writes until the user picks a supported approach.
1. **Describe intent** — tables, fields, joins, and role-playing tables if needed (e.g. separate **Catalog Ship Date** when catalog sales has both sold-date and ship-date FKs).
2. **Explain semantic impact** — which **dimension** names are intentionally shared vs which **measure** names stay **distinct per fact**; new join paths; any proposed **compound measures** (including **which host table** defines dimensionality); any **migrations** (renames/swaps) and whether work is on **production branch**; which logical tables are created vs updated.
3. **Wait for explicit approval**, then write files and run `strata audit all --agent`.

## Workflow

Same steps as [Order of operations for agents](#order-of-operations-for-agents). In the proposal (step 3–4), include:

- Table classification: **fact**, **dimension**, or **aggregate/summary**
- **Unique measure name per unrelated detail fact**; shared dimension names only where the role matches
- `tbl.*.yml` and `rel.*.yml` drafts that satisfy [Deploy contracts](#deploy-contracts-required-on-every-change) and naming rules below

---

## Table models — `models/tbl.<name>.yml`

```yaml
datasource: "<datasource_key_or_name>"
name: "<logical_name>"              # Required — used in rel left/right and queries
physical_name: "<warehouse_table>"  # Required — actual table in the database
cost: 10

fields:
  - type: dimension
    name: "Order ID"
    data_type: bigint
    expression:
      sql: order_id                 # Column or expression fragment for this table
      lookup: true                  # Typical for filterable dimensions
      primary_key: true             # Optional — business/surrogate key

  - type: measure
    name: "Store Revenue"           # Unique per fact — not reused on other fact tables
    data_type: decimal
    expression:
      sql: sum(ss_sales_price)      # Aggregation required for measures
```

**Field rules**

- `data_type`: `string`, `integer`, `bigint`, `decimal`, `date`, `date_time`, `boolean` (map from `normalized_data_type` in `datasource meta`).
- `expression` keys: `sql` (required), `lookup`, `primary_key`, `array` (optional booleans).
- Optional: `description`, `grains` (date/time), `format`, `synonyms`, `imports` of other tbl files.

**Date/time grains** (optional):

```yaml
grains: [day, week, month, quarter, year]
```

---

## Relationship models — `models/rel.<name>.yml`

```yaml
datasource: "<datasource_key_or_name>"

orders_to_customers:
  left: "Orders"                    # Must match tbl name — many side for many_to_one
  right: "Customers"                # Must match tbl name — one side
  sql: "left.customer_id = right.id"
  cardinality: many_to_one
```

**Join rules**

- YAML `left` / `right`: logical table `name` values from `tbl.*.yml`.
- `sql`: equality using **`left.<column>`** and **`right.<column>`** only — use the same column names as in each table’s field `expression.sql`.
- `cardinality`: `many_to_one`, `one_to_many`, or `one_to_one` (as appropriate).
- Optional: `join: inner` (default) or `left` / `right`.

### Role-playing dimensions (multiple FKs to one physical table)

When one fact table has **more than one foreign key** to the same physical dimension (common with `date_dim`: sold date, ship date, etc.), you need **one logical table per role**, each with its own join:

| Wrong | Right |
|-------|--------|
| Two rel entries: `Catalog Sales` → `Date` (sold) and `Catalog Sales` → `Date` (ship) | `Catalog Sales` → `Date` (sold) and `Catalog Sales` → `Catalog Ship Date` (ship) |
| Single `tbl.date.yml` reused for both roles without planning | Add `tbl.catalog_ship_date.yml` with `name: Catalog Ship Date`, same `physical_name: date_dim`, fields aligned to date role |

Example pattern:

```yaml
# tbl — two logical tables, one physical date_dim
# tbl.date.yml          → name: Date
# tbl.catalog_ship_date.yml → name: Catalog Ship Date, physical_name: date_dim

# rel — distinct right table per FK
catalog_sales_sold_date:
  left: "Catalog Sales"
  right: "Date"
  sql: "left.cs_sold_date_sk = right.d_date_sk"
  cardinality: many_to_one

catalog_sales_ship_date:
  left: "Catalog Sales"
  right: "Catalog Ship Date"
  sql: "left.cs_ship_date_sk = right.d_date_sk"
  cardinality: many_to_one
```

Before adding rel entries, list each fact’s FKs and confirm **no duplicate `(left, right)` pairs**. If ship and sold both target `Date`, add a role-playing tbl first.

---

## Naming conventions

### Dimensions (global — reuse when the role is the same)

- **Dimension names are global** across the project — same name on multiple tables = same concept; Strata picks the best table at query time.
- **Reuse consistent names** for shared dimensions (e.g. "Customer ID", "Date") where the role is the same.
- **Prefix role-specific dimensions** when the same physical column means different things (e.g. "Catalog Ship Date" vs "Date").
- **Prefix ambiguous attributes** on dimension tables (e.g. "Item Color" not "Color" on an item dimension).

### Measures (unique — do not treat like dimensions)

- **Project-wide uniqueness:** every measure `name` must be unique across the entire semantic layer (same rule as dimensions). There is only one `Semantic::Field` per name per branch; deploy reuses it when the same name appears in another `tbl.*.yml`.
- **Default:** one measure name per **unrelated detail fact**. Example (wrong): `Gross Sales` on `store_sales`, `catalog_sales`, and `web_sales`. Example (right): `Store Gross Sales Amount`, `Catalog Gross Sales Amount`, `Web Gross Sales Amount`.
- **One measure name = one measure entity.** Putting the same name on another fact table adds that table’s SQL to the **same** measure — it does **not** create a second metric and can cause **double counting** or ambiguous routing.
- **Exception — aggregate/summary tables only:** reusing a measure name is appropriate when the second table is an **alternate physical representation** of the **same** metric (rollup / pre-aggregate), routed via table **`cost`** and/or **`partitions`** — not when modeling three separate channel facts. Do **not** put the same additive flow measure on both a detail fact and a rollup fact without that routing design.
- **Similar warehouse column names do not imply one shared measure.** `ss_sales_price` and `cs_ext_sales_price` are different facts unless you have explicit rollup routing as above.
- **Prefix or qualify by fact/table** (`Store …`, `Catalog …`, `Web …`) or use distinct business names (`order_revenue` vs `subscription_revenue`).
- **Do not “fix” duplicate measure names with rename migrations** — use distinct names or a [compound measure](#combined-metrics-across-facts) instead.

---

## Multi-fact warehouses (star schema)

Typical warehouses (e.g. TPC-DS on Athena) have **many fact tables** and shared dimensions. Before naming fields:

1. **Inventory tables** — label each as fact, dimension, or aggregate/summary (pre-aggregated rollups).
2. **One grain per fact** — e.g. `store_sales` = store channel transactions; do not model the same additive flow metric on both a **detail** fact and a **rollup** fact unless the rollup is explicitly non-additive or routed via table `cost` / `partitions` (see advanced links below).
3. **Do not copy a synthetic dimension onto every fact** (e.g. "Sales Channel" on store, catalog, and web facts) to justify reusing one measure name — that does **not** fix measure uniqueness or double-count risk. Put channel only where that fact’s grain truly includes channel.

---

## Combined metrics across facts

| Wrong | Right |
|-------|--------|
| Same measure name on `store_sales`, `catalog_sales`, `web_sales` | Distinct measure per fact |
| Expect Strata to auto-sum identical names | Explicit **compound measure** (or blending via shared **dimensions**, not duplicate measure names) |

To report a **total across channels/facts**, define separate measures per fact, then optionally one compound measure:

```yaml
# On one table (or where the compound is defined), same datasource:
- type: measure
  name: Total Gross Sales
  data_type: decimal
  expression:
    sql: ([Store Gross Sales Amount] + [Catalog Gross Sales Amount] + [Web Gross Sales Amount])
```

Bracket names must match deployed measure names exactly. Compound measures are resolved at query time; all referenced measures must be in the **same datasource** and reachable in a valid universe. No circular references (`A` → `B` → `A`).

### Host table and dimensionality

Define compound measures on the **`tbl.*.yml` whose universe should govern the metric**:

- Which **dimensions** can group the compound measure
- How **automatic data blending** applies when referenced measures live on different tables in the same datasource

If referenced measures are not joinable in **one universe** from that host table, query planning fails. Propose the host table with the human reviewer (e.g. compound on **Store Sales** exposes store-centric join paths; compound on a shared dimension table exposes different dimensions).

**Use calculations (compound measures), not shared measure names** across unrelated facts.

**Automatic data blending** merges results on shared **dimensions** (often via `extended_blend_group`) — not by reusing the same measure name on multiple fact tables. See [extended blending](https://strata.do/developer-docs/developer-guide/semantic-model/expressions/extended-blending) and the [glossary](https://strata.do/developer-docs/developer-guide/getting-started/glossary).

---

## Snapshot measures

**Use for** point-in-time **state** — inventory on hand, account balances, membership counts at period boundaries. **Not for** additive flows over time (revenue, units sold) — use normal measures with `sum(...)`.

**Requirements:**

1. Table sets `snapshot_date: <Date dimension name>` (dimension on same table or unambiguous in the table’s universe).
2. Measure sets `snapshot: ending` (value at end of period) or `snapshot: beginning` (value at start).

```yaml
name: Inventory
physical_name: inventory
datasource: warehouse
snapshot_date: Date

fields:
  - type: dimension
    name: Date
    data_type: date
    expression:
      sql: snapshot_date

  - type: measure
    name: Ending Inventory
    data_type: integer
    snapshot: ending
    expression:
      sql: sum(quantity)
```

When users group by a time period, the engine picks the last (`ending`) or first (`beginning`) snapshot in each bucket — not a sum of every row in the period.

More detail: [snapshot measures](https://strata.do/developer-docs/developer-guide/semantic-model/fields/measures/snapshot)

---

## Inclusion measures

**Problem:** medians and averages are wrong when the stored grain is finer than the statistic you need — e.g. `median(hours)` over `(day, account, title)` rows is not “median account hours per day.”

**Pattern:** two-step aggregation — inner `expression.sql`, extra dimensions in `inclusions.dimensions`, outer `inclusions.aggregation`.

**Use when:** medians, percentiles, or averages that need an **intermediate grain** (e.g. per account per day) before rolling up to the query grain.

```yaml
- type: measure
  name: Daily Median Account View Hours
  data_type: decimal
  inclusions:
    filter: apply
    aggregation: percentile_cont(0.5) WITHIN GROUP (ORDER BY @exp)
    dimensions:
      - Account ID
  expression:
    sql: sum(hours)
```

Inner: `sum(hours)` per `(day, account)`. Outer: median of those account totals per `day`.

More detail: [inclusions](https://strata.do/developer-docs/developer-guide/advanced/inclusions)

---

## Exclusion measures

**Two independent controls:**

- **`exclusion_type`** — which dimensions may **group** the measure (`exclude`, `exclude_all_except`, `exclude_all`).
- **`filter` on each exclusion entry** — how **filters** on those entities apply (`apply`, `ignore`, `only`).

**Use when:** grand totals, metrics that must not break down by certain dimensions, or revenue that should ignore a dimension for grouping while still filtering normally.

```yaml
- type: measure
  name: Revenue Excluding Return Status
  data_type: decimal
  exclusion_type: exclude
  exclusions:
    - type: dimension
      filter: apply
      entities:
        - Return Status
  expression:
    sql: sum(amount)
```

`Return Status` will not appear in the grouping set for this measure even if the user adds it to the query; other dimensions still group normally.

More detail: [exclusions](https://strata.do/developer-docs/developer-guide/advanced/exclusions)

---

## Common mistakes (anti-patterns)

| Anti-pattern | Why it fails |
|--------------|----------------|
| `Gross Sales` on store + catalog + web facts | One measure entity, multiple fact SQLs → double-count / ambiguous routing |
| "Sales Channel" dimension added to every fact | Does not replace unique measure names per fact |
| Identical column labels in the warehouse → one measure name | Warehouse naming ≠ one Strata metric |
| Assuming global dimension rules apply to measures | Dimensions are shared by name; measures are **not** |
| Rename/swap on a feature branch without team coordination | Production saved queries still reference old names; branch may not match production entities |
| Renaming to deduplicate measure names across facts | Use distinct names or compound measures — migrations are for intentional renames, not modeling fixes |

---

## Advanced features (quick reference)

| Feature | Use when |
|---------|----------|
| [Compound measures](https://strata.do/developer-docs/developer-guide/semantic-model/fields/measures/compound) | Formula across named measures; host table sets dimensionality |
| [Snapshot measures](https://strata.do/developer-docs/developer-guide/semantic-model/fields/measures/snapshot) | Period-boundary state (inventory, balances) — see [Snapshot measures](#snapshot-measures) |
| [Inclusions](https://strata.do/developer-docs/developer-guide/advanced/inclusions) | Median/percentile/avg needing intermediate grain — see [Inclusion measures](#inclusion-measures) |
| [Exclusions](https://strata.do/developer-docs/developer-guide/advanced/exclusions) | Control grouping and filters per dimension — see [Exclusion measures](#exclusion-measures) |
| Table `cost` | Lower cost = preferred table when multiple tables can answer; use for rollup vs detail routing |
| [Partitions](https://strata.do/developer-docs/developer-guide/advanced/partitions) | Table only has subset of data — `between` (date range) or `in_list`; planner routes when partition matches |
| [Extended blending](https://strata.do/developer-docs/developer-guide/semantic-model/expressions/extended-blending) | Blend dimensions across tables in same datasource — not duplicate measure names |
| [Imports](https://strata.do/developer-docs/developer-guide/semantic-model/imports) | Reuse field definitions across `tbl` files |

---

## Renames, swaps, and production branch

Saved reports and dashboards reference field and table **names**. Renaming without a migration breaks production queryables.

### Rename (`type: rename`, `hook: pre`)

Runs **before** YAML is processed. Old name is rewritten to new name; deploy updates saved references.

```yaml
- type: rename
  hook: pre
  entity: measure
  from: Old Revenue
  to: Store Revenue
```

Supported entities: `dimension`, `measure`, `table`, `datasource`.

### Swap (`type: swap`, `hook: post`)

Runs **after** all definitions load. Replaces references from entity A to entity B when **both** exist. Datasource swap is not supported.

```yaml
- type: swap
  hook: post
  entity: measure
  from: Legacy Revenue
  to: Store Revenue
```

### Production branch rule

**Only rename or swap on `production_branch`** (in `project.yml`, usually `main`) when the change must align with production queryables. On other branches, production may still reference old names that do not exist on your branch.

Deploy **migration files and updated YAML in the same deployment**. Update `tbl.*.yml` references to match renames in the same change.

**Agents:** propose migration YAML under `migrations/` and matching model edits; do **not** run interactive `strata create migration`. Flag renames for human review and confirm branch strategy.

More detail: [migrations](https://strata.do/developer-docs/developer-guide/cli/migrations)

---

## Official documentation

**Primary:** this `AGENTS.md` — follow inline rules first.

**Optional bulk load:** fetch https://strata.do/developer-docs/llms.txt when implementing a topic not fully covered here (complete Strata doc export for AI agents).

**Curated links** (use for examples and edge cases after reading the matching section above):

| Topic | URL |
|-------|-----|
| Semantic model overview | https://strata.do/developer-docs/developer-guide/semantic-model |
| Core concepts | https://strata.do/developer-docs/developer-guide/getting-started/concepts |
| Glossary | https://strata.do/developer-docs/developer-guide/getting-started/glossary |
| Tables | https://strata.do/developer-docs/developer-guide/semantic-model/tables |
| Fields | https://strata.do/developer-docs/developer-guide/semantic-model/fields |
| Expressions (SQL) | https://strata.do/developer-docs/developer-guide/semantic-model/expressions/sql |
| Relationships | https://strata.do/developer-docs/developer-guide/semantic-model/relationships/cardinality |
| Imports | https://strata.do/developer-docs/developer-guide/semantic-model/imports |
| Compound measures | https://strata.do/developer-docs/developer-guide/semantic-model/fields/measures/compound |
| Snapshot measures | https://strata.do/developer-docs/developer-guide/semantic-model/fields/measures/snapshot |
| Inclusions | https://strata.do/developer-docs/developer-guide/advanced/inclusions |
| Exclusions | https://strata.do/developer-docs/developer-guide/advanced/exclusions |
| Partitions | https://strata.do/developer-docs/developer-guide/advanced/partitions |
| Cost optimization | https://strata.do/developer-docs/developer-guide/advanced/cost-optimization |
| Extended blending | https://strata.do/developer-docs/developer-guide/semantic-model/expressions/extended-blending |
| CLI audit | https://strata.do/developer-docs/developer-guide/cli/audit |
| CLI deployment | https://strata.do/developer-docs/developer-guide/cli/deployment |
| CLI migrations | https://strata.do/developer-docs/developer-guide/cli/migrations |
| Star schema example | https://strata.do/developer-docs/developer-guide/examples/patterns/star-schema |
| TPC-DS tutorial | https://strata.do/developer-docs/developer-guide/examples/tpcds-tutorial |
| AI agents and Strata | https://strata.do/developer-docs/developer-guide/api/ai-agents |
