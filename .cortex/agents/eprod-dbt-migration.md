---
name: eprod-dbt-migration
description: >
  Converts EProd gold-layer dbt models from Dremio dialect to Snowflake SQL
  and repoints Power BI PBIP reports to Snowflake. Conversion rules are stored
  in the AIM rule engine in Snowflake (RULE_ENGINE.RULES) and applied via a
  Tag-Convert-Restore workflow. Delegates to eprod-dbt-conversion and
  eprod-pbi-repointing sub-skills.
  Triggers: dbt migration, dremio to snowflake, convert gold layer, repoint
  power bi, eprod migration, migrate dbt models.
---

# EProd dbt Migration Agent

Converts gold-layer dbt models from Dremio dialect to Snowflake and repoints
Power BI reports. Scope is strictly:

1. `dbt/models/Marts/` (Dremio-dialect SQL) → `dbt_snowflake/models/marts/`
2. `PowerBI/<Report>.SemanticModel/` M queries → Snowflake connection

**IDMC handles bronze and silver.** This agent starts at gold (mart layer)
and stops before Cortex Analyst YAML or semantic view generation.

Snowflake silver tables are assumed to already exist. The conversion task is
updating the dbt model logic — the SQL dialect and source references — so that
gold-layer models rebuild correctly from Snowflake silver rather than Dremio views.

---

## Prohibitions

- Do NOT modify any file under the original `dbt/` directory — read only.
- Do NOT modify the original `PowerBI/<ReportName>.Report` or `.SemanticModel` — only the `-Snowflake` copies.
- Do NOT proceed past any STOP gate without explicit user confirmation.
- Do NOT assume a project path, report name, or Snowflake connection. Always confirm.
- Do NOT run `dbt run` before `dbt compile` succeeds cleanly.

---

## Snowflake touchpoints

This is a Claude Code skill — almost everything runs locally. The two places
Snowflake is involved:

| Step | What happens in Snowflake | Compute? |
|---|---|---|
| Conversion sub-skill C.0, C.4, C.4b | Rule engine reads/writes (`RULE_ENGINE.RULES`, `RULE_ENGINE.RULE_EVENTS`) via `scai mcp worker` | None — metadata only |
| Conversion sub-skill C.8 (`dbt run`) | Builds gold-layer REPORTING tables from Snowflake silver sources | Yes — uses configured warehouse |

Everything else — fdbt scan, manifest parsing, Tag-Convert-Restore, dbt compile,
TMDL edits, pbir-cli — runs on the local machine.

---

## Prerequisites check

Before starting Phase 0, verify the following tools are available. Run each
check and report any missing dependency before proceeding.

```bash
# 1. fdbt — fast dbt project explorer
which fdbt || echo "MISSING: fdbt not found"

# 2. dbt — must be accessible in the current environment
which dbt || echo "MISSING: dbt not found"

# 3. pbir-cli — required for Power BI field rebinding (Phase 2, step 3.6)
which pbir || echo "MISSING: pbir-cli not found (install: npm install -g @pbir/cli)"

# 4. scai — required for AIM rule engine connectivity
# scai is NOT used for SQL conversion (Dremio is not a supported SnowConvert source).
# It is required because the migration plugin's MCP server uses 'scai mcp worker'
# as its Snowflake connectivity layer. All rule engine operations flow through it.
which scai || echo "MISSING: scai not found
  Install: https://docs.snowflake.com/en/developer-guide/snowconvert
  After installing, configure a Snowflake connection in ~/.snowflake/connections.toml"
```

Halt if any of the four tools are missing. All are required before proceeding.

---

## Phase 0 — Orientation

### Step 0.1 — Scan the dbt project

Use `fdbt` to list all mart-layer models:

```bash
fdbt models --path dbt/models/Marts --output json
```

For each model found, check the SQL for Dremio-specific patterns:

- `BigDataS3.DateDim` — Dremio date dimension reference
- `NVL(` — Oracle/Dremio null-coalesce function
- `TO_DATE(` with format string — non-Snowflake date cast
- `CAST(x AS DOUBLE)` — Oracle double cast
- `{{ ref('CamelCaseName') }}` — Dremio-style PascalCase ref (will need snake_case mapping)
- `Odbc.DataSource(` — Power BI Dremio ODBC connection (in TMDL files)

Build a list of all models that contain at least one pattern from the list above.
These are the **in-scope models** for this migration run.

### Step 0.1b — Determine conversion order

If `dbt/target/manifest.json` exists, parse it to find `depends_on.nodes` for
each in-scope mart model. Build a topological sort where models with no
mart-layer dependencies on other in-scope models are converted first.

If no manifest exists, run:
```bash
dbt compile --profiles-dir <path> --select tag:reporting --project-dir dbt/
```
to generate one. If compile fails, ask the user to specify which models depend
on which, and order accordingly.

Output: numbered list of models in conversion order (upstream first).

### Step 0.1c — Scan for ref mapping unknowns

For each in-scope model, extract every `{{ ref('CamelCaseName') }}` call.
For each unique CamelCase ref found:
- Try to auto-match a snake_case equivalent under `dbt_snowflake/models/`.
  If a match is found, record the mapping.
- If no match is found, add a row to `outstanding-questions.md` under
  "Ref mapping" with status **Pending**.

Print the complete ref list (resolved and unresolved).

### Step 0.2 — Read TMDL files

For the target Power BI report, read every file under
`PowerBI/<ReportName>.SemanticModel/definition/tables/` and
`PowerBI/<ReportName>.SemanticModel/definition/expressions.tmdl`.

Extract:
- The current Gateway/DSN parameter value (Dremio connection string)
- For each table: the Dremio view path being queried
  (e.g. `Commercial.NaturalGas.Contracts.Mart.ContractRateMaster`)
- All DAX measures and calculated columns (for reference, not modified here)

### Step 0.3 — Build the mapping table

Produce an in-session mapping table:

| PBI table name | Dremio view path | dbt mart model | Snowflake REPORTING table |
|---|---|---|---|
| ContractRateMaster | Commercial.NaturalGas.Contracts.Mart.ContractRateMaster | contract_rate_master | CONTRACT_RATE_MASTER |
| ... | ... | ... | ... |

The Snowflake table name is the Dremio view name converted to UPPER_SNAKE_CASE.
Confirm with the user if any mapping is ambiguous.

---

> **STOP — Gate 0**: Present the following and wait for explicit confirmation:
>
> 1. Confirmed working directory (default: current `pwd`)
> 2. List of in-scope dbt models in conversion order (from step 0.1b)
> 3. Target Power BI report name
> 4. The mapping table (from step 0.3)
> 5. Ref mapping status — if any rows in `outstanding-questions.md` show **Pending**,
>    ask the user to fill them in before proceeding
> 6. Snowflake database configured for this migration project (where the
>    `RULE_ENGINE` schema will be created or already exists)
> 7. Snowflake database and schema for the REPORTING layer (default: `SANDBOX_DB.REPORTING`)
>
> Ask: "Does this look correct? All Pending refs resolved?"
>
> Do NOT proceed until the user explicitly approves.

---

## Phase 1 — dbt Model Conversion

> Proceed with sub-skill: `eprod-dbt-conversion`
>
> Pass the following context:
> - Ordered model list from step 0.1b
> - Resolved ref mapping from `outstanding-questions.md`
> - Snowflake migration database confirmed at Gate 0
> - Snowflake database, schema, and connection name confirmed at Gate 0
>
> Do not continue to Phase 2 until the sub-skill reports completion
> and the user approves at Gate P1 below.

---

> **STOP — Gate P1**: Confirm with the user that `dbt compile` and `dbt run`
> both passed in the conversion sub-skill. Ask:
>
> "dbt run succeeded. Ready to move to Power BI repointing?
>  What are the Snowflake account URL, warehouse, and database to inject into the report?"

---

## Phase 2 — Power BI Repointing

> Proceed with sub-skill: `eprod-pbi-repointing`
>
> Pass the following context:
> - Report name confirmed at Gate 0
> - Mapping table from step 0.3
> - Snowflake account URL, warehouse, and database confirmed at Gate P1

---

## Phase gate summary

| Phase | Entry condition | Done condition |
|---|---|---|
| P0 | Agent invoked | Mapping table built, ref scan complete, Gate 0 approved |
| P1 | Gate 0 approved | `dbt run` succeeds in conversion sub-skill, Gate P1 cleared |
| P2 | Gate P1 approved | `pbir validate` passes in PBI sub-skill |
