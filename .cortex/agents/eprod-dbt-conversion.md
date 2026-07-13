---
name: eprod-dbt-conversion
description: "Converts EProd gold-layer dbt models from Dremio to Snowflake SQL via Tag-Convert-Restore. Bootstraps rules into the AIM rule engine, applies them, and writes to dbt_snowflake/models/marts/. Called by eprod-dbt-migration after Gate 0. Triggers: dbt conversion, convert gold layer, dremio to snowflake models."
parent_skill: eprod-dbt-migration
---

# EProd dbt Conversion Sub-Skill

Converts gold-layer dbt models from Dremio dialect to Snowflake SQL.

This sub-skill implements a **Tag-Convert-Restore** workflow inspired by the
PS DBT Migration Accelerator's `sc-dbt-automator` — without requiring any of
its tools. Claude Code performs each phase directly, using the AIM rule engine
in Snowflake as the source of conversion rules.

**Important:** `scai` is required for this sub-skill, but NOT for the SQL
conversion itself. Dremio is not a supported SnowConvert source dialect, so
`scai code convert` is never invoked. `scai` is required because the migration
plugin's MCP server uses `scai mcp worker` as its Snowflake connectivity layer —
all rule engine operations (creating rules, querying rules, recording
applications) flow through that subprocess.

---

## Inputs (from parent at Gate 0)

- Ordered model list (topologically sorted from Phase 0 step 0.1b)
- Resolved ref mapping (from `outstanding-questions.md` — all rows confirmed)
- Snowflake migration database (where `RULE_ENGINE` schema lives)
- Snowflake connection name and schema for dbt target

---

## Step C.0 — Rule engine bootstrap (idempotent)

### C.0.1 — Check scai

```bash
which scai || echo "MISSING"
```

If `scai` is not found, halt with:
> **scai is required.** Install from https://docs.snowflake.com/en/developer-guide/snowconvert
> then configure a Snowflake connection in `~/.snowflake/connections.toml`.
> Note: scai is not used for SQL conversion — it provides Snowflake connectivity
> for the rule engine (scai mcp worker).

### C.0.2 — Check for existing rules

Call `mcp_mcp_list_rules(detailed=false)`.

- If rules exist with `created_from = "eprod-dbt-migration"`: print
  "Rule engine already seeded with N rules. Skipping bootstrap." and continue to Step C.1.
- If the result is empty or errors with a schema-not-found message:
  proceed to bootstrap.

### C.0.3 — Bootstrap from seed file

Read `references/dremio-to-snowflake-rules.md`. Create each rule via
`mcp_mcp_create_rule` with the parameters below. All rules get:
- `source_platforms = ["Dremio"]`
- `created_from = "eprod-dbt-migration"`

| Rule name | match_pattern | mode | Notes |
|---|---|---|---|
| `eprod-nvl-to-coalesce` | `\bNVL\s*\(` | regex | `NVL(` → `COALESCE(` |
| `eprod-nvl2-to-iff` | `\bNVL2\s*\(` | ai | See dremio-to-snowflake-rules.md for IFF pattern |
| `eprod-decode-to-case` | `\bDECODE\s*\(` | ai | Multi-argument CASE rewrite |
| `eprod-to-date-format` | `TO_DATE\s*\([^,)]+,\s*'[^']+'` | ai | Format-string TO_DATE → DATE literal or TRY_TO_DATE |
| `eprod-cast-as-double` | `CAST\s*\([^)]+\bAS\s+DOUBLE\b` | ai | → TRY_TO_DOUBLE(x::VARCHAR) |
| `eprod-trunc-to-date-trunc` | `\bTRUNC\s*\(` | ai | → DATE_TRUNC('DAY', ...) |
| `eprod-sysdate` | `\bSYSDATE\b` | regex | → CURRENT_DATE() |
| `eprod-systimestamp` | `\bSYSTIMESTAMP\b` | regex | → CURRENT_TIMESTAMP() |
| `eprod-instr` | `\bINSTR\s*\(` | ai | → POSITION(substr IN str) |
| `eprod-bigdatas3-datedim` | `BigDataS3\.DateDim` | ai | → inline CTE_DATE_SPINE (see rules reference) |
| `eprod-ref-pascal-to-snake` | `ref\s*\(\s*'[A-Z][a-zA-Z]+'` | ai | CamelCase ref → confirmed snake_case equivalent |
| `eprod-single-letter-alias` | `\bAS\s+[a-z]\b(?!\w)` | ai | → qualify with table/CTE name |

Set `category = "syntax"` for direct substitutions (`eprod-nvl-*`, `eprod-sysdate`,
`eprod-systimestamp`). Set `category = "structural"` for rewrites that change
the SQL structure (`eprod-decode-to-case`, `eprod-bigdatas3-datedim`, `eprod-cast-as-double`).

> **Snowflake (metadata write):** This step writes N rows to `<migration_db>.RULE_ENGINE.RULES`
> via `scai mcp worker`. No warehouse compute. Idempotent — skipped if rules already exist.

Report: "Rule engine bootstrapped: N rules created in `<migration_db>.RULE_ENGINE.RULES`"
Provide the Snowsight query so the user can verify:
```sql
SELECT id, name, replacement_mode, category, priority
FROM <migration_db>.RULE_ENGINE.RULES
WHERE source_platforms LIKE '%Dremio%'
ORDER BY priority;
```

---

## Step C.1 — Dependency sort

If `dbt/target/manifest.json` exists, read it and build a topological sort of
the in-scope mart models based on `depends_on.nodes`. Models with no mart-layer
upstream dependencies convert first.

If no manifest exists:
```bash
dbt compile --profiles-dir <confirmed_path> --select tag:reporting --project-dir dbt/
```
Then read the generated manifest.

If compile fails (no source DB connection): ask the user to specify model
order manually.

Output: numbered list of model files in conversion order.

---

## Step C.2 — Ref scan (feeds Gate 0 in parent)

For each in-scope model, extract every `{{ ref('CamelCaseName') }}` call.

For each unique CamelCase ref found:
1. Try to auto-match: look for a snake_case equivalent under
   `dbt_snowflake/models/intermediate/` and `dbt_snowflake/models/marts/`
   using `fdbt` or by listing files.
2. If a match is found: note the mapping (will be applied during Restore phase).
3. If no match is found: add a row to `outstanding-questions.md` under
   "Ref mapping" with status **Pending**.

Print the complete ref list showing resolved and unresolved refs.

> **STOP if any Pending rows exist.** Do not proceed until the EProd team
> fills in `outstanding-questions.md` and confirms at Gate 0 in the parent skill.

---

## Steps C.3–C.5 — Tag, Convert, Restore (per model)

Process each model in the order from Step C.1.

### C.3 — Tag

Read `dbt/models/Marts/<path>/<model>.sql`.

Extract all Jinja blocks (`{{ ... }}`, `{% ... %}`), recording each block's
exact text and its position. Replace each block in a working copy with a
positional marker: `-- __JINJA_1__`, `-- __JINJA_2__`, etc.

The result is a pure-SQL skeleton with no Jinja.

### C.4 — Convert

> **Snowflake (metadata read):** `mcp_mcp_search_rules_regex` sends the model SQL to
> Snowflake for `REGEXP_LIKE` matching and Cortex semantic search against `RULE_ENGINE.RULES`.
> No warehouse compute — this is a small metadata table query routed via `scai mcp worker`.

Pass the skeleton SQL to `mcp_mcp_search_rules_regex` with a brief description
of the patterns visible (e.g., `"Dremio mart model — contains NVL, CAST AS DOUBLE,
BigDataS3.DateDim reference, PascalCase ref"`). This runs regex + Cortex semantic
search and returns matched rules sorted by priority.

Apply returned rules following the apply skill workflow:

**Regex-mode rules** (apply automatically, no approval needed):
- Read current working copy
- Apply `replacement_find` → `replacement_replace` substitution
- Report: "Applied `<rule name>`: N replacements"
- Call `mcp_mcp_record_rule_application(rule_id=..., outcome="applied", code_unit_name=<model>)`

**AI-mode rules** (show diff, wait for approval before writing):
- Identify all instances of the pattern in the SQL
- Show a before/after diff
- Ask: "Apply `<rule name>` to `<model>`?"
- If approved: write the change and record application
- If rejected: skip this rule for this model

**New patterns** (patterns found that have no matching rule):
1. Apply the fix manually
2. Immediately call `mcp_mcp_create_rule` to add it to the engine:
   - `name = "eprod-<description>"`
   - `source_platforms = ["Dremio"]`
   - `created_from = "eprod-dbt-migration-extract"`
3. Call `mcp_mcp_record_rule_application` to record the application
4. On subsequent models, `mcp_mcp_search_rules_regex` will now return this rule

> **Snowflake (metadata write):** New rules and application records written to
> `RULE_ENGINE.RULES` and `RULE_ENGINE.RULE_EVENTS` via `scai mcp worker`. No warehouse compute.

**Unresolvable patterns** (structural changes requiring domain context):
- Insert `-- REVIEW: <description of what needs manual attention>` at the relevant location
- Append a row to `outstanding-questions.md` under "Manual review items"
- Continue — do not halt for review markers

### C.5 — Restore

Replace each `-- __JINJA_N__` marker with its original Jinja content, with
one substitution: any `{{ ref('CamelCaseName') }}` is replaced with the
confirmed Snowflake equivalent from the resolved ref mapping.

Apply the standard mart header config block at the top of the model:

```jinja
{{ config(
    materialized='table',
    tags=['reporting', '<pipeline>'],
    post_hook=[
        "COMMENT ON TABLE {{ this }} IS '<TableName>: Snowflake drop-in replacement for Dremio view <DremioPath>. Column names preserved.'"
    ]
) }}
```

Derive `<pipeline>` from the model's folder name or from EProd's existing
pipeline tag convention (lowercase snake_case, e.g. `contracts_mart`).
Derive `<TableName>` as the UPPER_SNAKE_CASE form of the model filename.
Derive `<DremioPath>` from the mapping table built in Phase 0.

Verify no single-letter column aliases remain (`AS a`, `AS b`, etc.).

---

> **STOP — Gate C1**: Show a before/after summary for all converted models.
>
> Report:
> - N models converted
> - M `-- REVIEW:` markers inserted (list them with model name + description)
> - K new rules added to rule engine during conversion
> - Any models that were skipped due to unresolved refs
>
> Ask: "Do the converted models look correct? Ready to write to `dbt_snowflake/models/marts/`?"
>
> Do NOT write any file until the user explicitly approves.

---

## Step C.6 — Write converted models

Write each converted model to `dbt_snowflake/models/marts/<logical_name>.sql`.
Use snake_case filenames matching the dbt model name convention.

Do NOT modify any file under the original `dbt/` directory — read only.

---

## Step C.7 — Compile

```bash
cd dbt_snowflake
dbt compile --profiles-dir <confirmed_path> --select tag:reporting
```

If compile fails: show the full error, halt. Ask: "Return to Step C.4 to
fix the failing model, or fix it manually?"

Do NOT auto-fix compile errors.

---

> **STOP — Gate C2**: Compile must pass cleanly.
> Ask: "Compile succeeded. Which warehouse should `dbt run` use? Proceed?"

---

## Step C.8 — Run

> **Snowflake (warehouse compute):** `dbt run` is the only step in this skill that uses
> Snowflake warehouse compute. It builds or replaces the REPORTING tables from Snowflake
> silver sources using the warehouse confirmed at Gate C2.

```bash
dbt run --profiles-dir <confirmed_path> --select tag:reporting
```

Report row counts for each model on completion.

---

## On completion

Return to the parent skill (`eprod-dbt-migration`). The parent will present
Gate P1 before proceeding to the Power BI repointing sub-skill.
