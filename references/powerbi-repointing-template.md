# Power BI M-query repointing template

Reference for Phase 3, steps 3.2-3.3. All substitutions are deterministic
text replacements in the copied `.tmdl` files.

---

## Step 3.2 — expressions.tmdl connection parameter swap

### Before (Dremio ODBC DSN)

```
expression Gateway = "dsn=DremioArrowRight-tst" meta [IsParameterQuery=true, List={"dsn=DremioArrowFlight-tst","dsn=DremioArrowFlight-prd"}, Type="Text", IsParameterQueryRequired=true]
```

### After (Snowflake parameters)

Replace the single Gateway expression with two Snowflake parameters.
Substitute `<account>`, `<warehouse>`, and `<database>` with the confirmed
values from Gate 4.

```
expression SnowflakeAccount = "<account>.snowflakecomputing.com" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]
expression SnowflakeWarehouse = "<warehouse>" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]
expression SnowflakeDatabase = "<database>" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]
```

Delete the Gateway expression entirely after adding the Snowflake expressions.

---

## Step 3.3 — Per-table M query rewrite

### Before (Dremio ODBC + hierarchical navigation)

```m
let
    Source = Odbc.DataSource(Gateway, [HierarchicalNavigation=true]),
    Database = Source{[Name="SAMPLES"]}[Data],
    Schema = Database{[Name="Commercial.NaturalGas.Contracts.Mart"]}[Data],
    Table = Schema{[Name="ContractRateMaster", Kind="View"]}[Data]
in
    Table
```

### After (Snowflake native connector)

```m
let
    Source = Snowflake.Databases(SnowflakeAccount, SnowflakeWarehouse, [CreateNavigationProperties=true]),
    Database = Source{[Name=SnowflakeDatabase]}[Data],
    Schema = Database{[Name="REPORTING"]}[Data],
    Table = Schema{[Name="CONTRACT_RATE_MASTER", Kind="Table"]}[Data]
in
    Table
```

The only values that change per table are:
- The Dremio schema path (`Commercial.NaturalGas.Contracts.Mart`) — removed entirely
- The view name (`ContractRateMaster`) — replaced with the Snowflake REPORTING table name (`CONTRACT_RATE_MASTER`)
- `Kind="View"` → `Kind="Table"`

All other lines are identical across all tables.

---

## Mapping: Dremio view name → Snowflake table name in TMDL

Combine the Phase 0 mapping table with the template above.
For each `tables/<TableName>.tmdl` file:

1. Identify the Dremio view name from the original M query (`Schema{[Name="ViewName", Kind="View"]}`).
2. Look up the corresponding Snowflake REPORTING table from the Phase 0 mapping table.
3. Apply the template substitution above, inserting the correct `Kind="Table"` table name.

Example mapping for the Contracts report:

| TMDL file | Dremio view (original) | Snowflake table (replacement) |
|---|---|---|
| `ContractRateMaster.tmdl` | `ContractRateMaster` | `CONTRACT_RATE_MASTER` |
| `ContractMeterMonthlyCapacityUtilization.tmdl` | `ContractMeterMonthlyCapacityUtilization` | `CONTRACT_METER_MONTHLY_CAPACITY_UTILIZATION` |

---

## Step 3.4 — .pbip file update

In `PowerBI/<ReportName>-Snowflake.pbip`, find the `SemanticModel` reference:

```json
{
  "version": "1.0",
  "artifacts": [
    {
      "report": {
        "path": "<ReportName>.Report"
      }
    },
    {
      "semanticModel": {
        "path": "<ReportName>.SemanticModel"
      }
    }
  ]
}
```

Update both paths to the `-Snowflake` copies:

```json
{
  "version": "1.0",
  "artifacts": [
    {
      "report": {
        "path": "<ReportName>-Snowflake.Report"
      }
    },
    {
      "semanticModel": {
        "path": "<ReportName>-Snowflake.SemanticModel"
      }
    }
  ]
}
```

---

## Step 3.6 — When to run pbir fields replace

**Skip entirely** if column names are preserved between Dremio and Snowflake
(the default — EProd's goal is drop-in replacement).

**Run only if** a visual fails to load after opening the repointed report,
which indicates a column name mismatch. In that case:

```bash
# Identify all field bindings
pbir fields list "PowerBI/<ReportName>-Snowflake.Report"

# For each mismatched field (dry-run first):
pbir fields replace "PowerBI/<ReportName>-Snowflake.Report" \
  --from "DremioTableName.dremio_column_name" \
  --to "SNOWFLAKE_TABLE_NAME.SNOWFLAKE_COLUMN_NAME" \
  --dry-run

# Apply once confirmed:
pbir fields replace "PowerBI/<ReportName>-Snowflake.Report" \
  --from "DremioTableName.dremio_column_name" \
  --to "SNOWFLAKE_TABLE_NAME.SNOWFLAKE_COLUMN_NAME"

# Validate:
pbir validate "PowerBI/<ReportName>-Snowflake.Report" --fields --all
```

Note: Snowflake's native connector returns all column names in UPPERCASE.
If the original Dremio model used lowercase column names, add them to the
field replace list.
