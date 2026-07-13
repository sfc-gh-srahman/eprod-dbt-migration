---
name: eprod-pbi-repointing
description: "Repoints Power BI PBIP reports from Dremio Gateway DSN to Snowflake. Updates expressions.tmdl parameters, rewrites M queries, and optionally rebinds field references via pbir-cli. Called by eprod-dbt-migration after dbt conversion is confirmed. Triggers: power bi repointing, pbi repoint, tmdl update, dremio to snowflake pbip."
parent_skill: eprod-dbt-migration
---

# EProd Power BI Repointing Sub-Skill

Repoints Power BI PBIP reports from Dremio Gateway DSN to Snowflake.

Scope: `.SemanticModel` files only. Visual pages (`.Report`) are untouched.
Always works on a **copy** of the original report — the original is never modified.

---

## Inputs (from parent Gate P1)

- Report name (confirmed at Gate 0)
- Mapping table: PBI table name → Dremio view path → Snowflake REPORTING table
- Snowflake account URL, warehouse, and database (confirmed at Gate P1)

---

## Step 3.1 — Copy the PBIP report

Create Snowflake-targeted copies of both the `.Report` and `.SemanticModel` directories.

On macOS/Linux:
```bash
ReportName="<confirmed from Gate 0>"
cp -r "PowerBI/${ReportName}.Report" "PowerBI/${ReportName}-Snowflake.Report"
cp -r "PowerBI/${ReportName}.SemanticModel" "PowerBI/${ReportName}-Snowflake.SemanticModel"
```

On Windows (PowerShell):
```powershell
$ReportName = "<confirmed from Gate 0>"
Copy-Item -Path "PowerBI\$ReportName.Report" -Destination "PowerBI\$ReportName-Snowflake.Report" -Recurse
Copy-Item -Path "PowerBI\$ReportName.SemanticModel" -Destination "PowerBI\$ReportName-Snowflake.SemanticModel" -Recurse
```

---

> **STOP — Gate 4**: Confirm the report was copied.
> Ask: "What are the Snowflake account URL, warehouse name, and database name to inject?"
> Example: account = `myorg-myaccount.snowflakecomputing.com`, warehouse = `COMPUTE_WH`, database = `SANDBOX_DB`

---

## Step 3.2 — Update expressions.tmdl

In the **copied** `PowerBI/<ReportName>-Snowflake.SemanticModel/definition/expressions.tmdl`:

Replace the entire Gateway expression block:

```
// BEFORE (Dremio)
expression Gateway = "dsn=DremioArrowRight-tst" meta [IsParameterQuery=true, ...]
```

With Snowflake parameter expressions (use confirmed values from Gate 4):

```
expression SnowflakeAccount = "<account>.snowflakecomputing.com" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]
expression SnowflakeWarehouse = "<warehouse>" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]
```

---

## Step 3.3 — Rewrite M queries

For each `tables/<TableName>.tmdl` in the **copied** SemanticModel, using the
mapping table from Phase 0:

Replace the Dremio M query pattern (see `references/powerbi-repointing-template.md`)
with the Snowflake equivalent, substituting the correct REPORTING table name from
the mapping table.

**Dremio M query pattern (before):**
```m
let
    Source = Odbc.DataSource("dsn=DremioArrowRight-tst", [HierarchicalNavigation=true]),
    Commercial = Source{[Name="Commercial"]}[Data],
    NaturalGas = Commercial{[Name="NaturalGas"]}[Data],
    ...
    Data = ...{[Name="<DremioViewName>"]}[Data]
in
    Data
```

**Snowflake M query pattern (after):**
```m
let
    Source = Snowflake.Databases(SnowflakeAccount, SnowflakeWarehouse),
    Database = Source{[Name="<DATABASE>"]}[Data],
    Schema = Database{[Schema="REPORTING"]}[Data],
    Table = Schema{[Name="<SNOWFLAKE_TABLE_NAME>", Kind="Table"]}[Data]
in
    Table
```

The target schema is always `REPORTING` and the database is the value confirmed at Gate 4.
The `<SNOWFLAKE_TABLE_NAME>` comes from the mapping table column "Snowflake REPORTING table."

---

## Step 3.4 — Update .pbip file references

In `PowerBI/<ReportName>-Snowflake.pbip`, update the `SemanticModel` reference
path to point to `<ReportName>-Snowflake.SemanticModel`.

---

## Step 3.5 — Remove unused Gateway parameter

Delete the `expression Gateway = ...` line from `expressions.tmdl` in the copied
SemanticModel (it was replaced in Step 3.2).

---

## Step 3.6 — Field rebinding (conditional)

**Skip this step** if column names are preserved exactly between the Dremio view
and the Snowflake REPORTING table (the default goal is drop-in replacement).

If column names changed, use `pbir-cli`:

```bash
# List all current field bindings
pbir fields list "PowerBI/<ReportName>-Snowflake.Report"

# Dry-run each changed field
pbir fields replace "PowerBI/<ReportName>-Snowflake.Report" \
  --from "OldDremioTable.OldColumnName" \
  --to "SnowflakeTable.NewColumnName" \
  --dry-run

# Apply after confirming
pbir fields replace "PowerBI/<ReportName>-Snowflake.Report" \
  --from "OldDremioTable.OldColumnName" \
  --to "SnowflakeTable.NewColumnName"
```

---

## Step 3.7 — Validate

```bash
pbir validate "PowerBI/<ReportName>-Snowflake.Report" --fields --all
```

Report any validation failures to the user.

---

## On completion

Return to the parent skill (`eprod-dbt-migration`). The migration is complete
when `pbir validate` passes without errors.
