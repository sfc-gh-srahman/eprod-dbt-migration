# Dremio to Snowflake SQL conversion rules

Seed file for the AIM rule engine (`RULE_ENGINE.RULES` in Snowflake).

When the `eprod-dbt-conversion` sub-skill runs, it bootstraps these rules
into Snowflake via `mcp_mcp_create_rule`. They can then be viewed and queried
from Snowsight, and the rule set grows over time as new patterns are extracted
from manual fixes during conversion.

**Git version control:** this file is the authoritative seed. If the
`RULE_ENGINE` schema is dropped or reset, re-run the conversion sub-skill's
bootstrap step (Step C.0) to recreate everything from this file.

**View live rules from Snowsight:**
```sql
SELECT id, name, replacement_mode, priority, rule_applications
FROM <migration_db>.RULE_ENGINE.RULES
WHERE source_platforms LIKE '%Dremio%'
ORDER BY priority;
```

---

## Function substitutions

| Dremio / Oracle pattern | Snowflake equivalent | Notes |
|---|---|---|
| `NVL(a, b)` | `COALESCE(a, b)` | Direct swap |
| `NVL2(expr, val_if_not_null, val_if_null)` | `IFF(expr IS NOT NULL, val_if_not_null, val_if_null)` | No NVL2 in Snowflake |
| `DECODE(col, v1, r1, v2, r2, default)` | `CASE WHEN col = v1 THEN r1 WHEN col = v2 THEN r2 ELSE default END` | |
| `TO_DATE('2020-01-01', 'YYYY-MM-DD')` | `DATE '2020-01-01'` or `TRY_TO_DATE(...)` | |
| `CAST(x AS DOUBLE)` | `TRY_TO_DOUBLE(x::VARCHAR)` | Safer for Oracle sources where empty string != NULL |
| `x::FLOAT` (from Oracle RAW layer) | `TRY_TO_DOUBLE(x::VARCHAR)` | Oracle NUMBER cols may contain empty strings |
| `TRUNC(date)` | `DATE_TRUNC('DAY', date)` | TRUNC is Oracle-specific |
| `INSTR(str, substr)` | `POSITION(substr IN str)` | |
| `SUBSTR(s, 1, 3)` | `SUBSTR(s, 1, 3)` | Same — no change needed |
| `LISTAGG(x, ',') WITHIN GROUP (ORDER BY y)` | `LISTAGG(x, ',') WITHIN GROUP (ORDER BY y)` | Same — no change needed |
| `SYSDATE` | `CURRENT_DATE()` | |
| `SYSTIMESTAMP` | `CURRENT_TIMESTAMP()` | |

---

## Date spine replacement

Dremio references `BigDataS3.DateDim` as a shared date dimension table.
Replace any join or FROM reference to it with an inline CTE:

```sql
-- BEFORE (Dremio)
FROM BigDataS3.DateDim AS DateDim
WHERE DateDim.DATE_ID BETWEEN ...

-- AFTER (Snowflake)
-- Replace with this CTE added to the WITH block:
CTE_DATE_SPINE AS (
    SELECT
        DATEADD('month', SEQ4.INDEX, DATE_TRUNC('month', DATEADD('year', -2, CURRENT_DATE))) AS MONTH_START,
        DATEADD('day', SEQ4.INDEX, DATE_TRUNC('day', DATEADD('year', -2, CURRENT_DATE))) AS DATE_ID
    FROM TABLE(GENERATOR(ROWCOUNT => 730)) AS SEQ4(INDEX)
),
```

Adjust `ROWCOUNT` for the required date range (730 = 2 years of days).

---

## Cast conventions (Oracle GFLOW sources)

These apply specifically to staging and intermediate models that read from
Oracle GFLOW via the RAW layer. Apply to all models that reference
`{{ source('raw', 'QPTM__...') }}` or similar Oracle-sourced tables.

| Column pattern | Correct cast | Wrong cast | Why |
|---|---|---|---|
| `*_PKG_ID`, `*_HASH_ID`, `*_ID` | `::VARCHAR(50)` | `::NUMBER` | Often contains alphanumeric values like `'CPS'` |
| `*_CTR_NO`, `*_LOC_ID`, `LOC_ID`, `USER_ID` | `::VARCHAR(255)` | `::VARCHAR(20)` | May contain long descriptive strings |
| Numeric / float columns from Oracle views | `TRY_TO_DOUBLE(col::VARCHAR)` | `::FLOAT` | Oracle NULL stored as empty string `''` in RAW layer |
| Date strings | `TRY_TO_DATE(col::VARCHAR, 'YYYY-MM-DD')` | `::DATE` | |
| Timestamps | `TRY_TO_TIMESTAMP_NTZ(col::VARCHAR)` | `::TIMESTAMP` | |

---

## DBT ref casing

Dremio/original dbt used PascalCase model names. Snowflake dbt uses snake_case.

| Before | After |
|---|---|
| `{{ ref('ContractDim') }}` | `{{ ref('contract_dim') }}` |
| `{{ ref('ContractRateMaster') }}` | `{{ ref('contract_rate_master') }}` |
| `{{ ref('DateDim') }}` | Replace with date spine CTE (see above) |

Rule: convert every `{{ ref('CamelCaseName') }}` to `{{ ref('camel_case_name') }}`.

---

## SQL style rules

These apply to every model written to `dbt_snowflake/`. They match the
conventions in EProd's `copilot-instructions.md`:

- No single-letter aliases — qualify every column with its table or CTE name.
- CTE names must be descriptive (`CTE_DEDUP_<TABLE>`, `ContractLink`, etc.).
- Always qualify columns inside CTE bodies.
- No `SELECT *` in CTEs — list only consumed columns.
- Deduplication CTEs use prefix `CTE_DEDUP_<SOURCE_TABLE_NAME>`.

---

## Naming convention: Dremio view → Snowflake REPORTING table

Dremio view names are PascalCase. Snowflake REPORTING table names are UPPER_SNAKE_CASE.

Conversion rule: insert underscore before each uppercase letter that follows a
lowercase letter, then uppercase the whole string.

Examples:

| Dremio view name | Snowflake REPORTING table |
|---|---|
| `ContractRateMaster` | `CONTRACT_RATE_MASTER` |
| `ContractMeterMonthlyCapacityUtilization` | `CONTRACT_METER_MONTHLY_CAPACITY_UTILIZATION` |
| `GatheringSummary` | `GATHERING_SUMMARY` |
| `TransportationSummary` | `TRANSPORTATION_SUMMARY` |

Add all report-specific mappings to Phase 0's mapping table before starting Phase 1.
