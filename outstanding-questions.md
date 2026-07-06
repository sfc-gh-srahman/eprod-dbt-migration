# EProd dbt Migration: Outstanding Questions

This file is maintained by the `eprod-dbt-conversion` agent sub-skill and the EProd
migration team. The agent appends rows during Phase 0 orientation; the team fills in
answers before conversion proceeds.

---

## Ref mapping (blocking for conversion)

Each Dremio CamelCase `{{ ref('...') }}` found in gold-layer models must map to an
existing model in `dbt_snowflake/models/` before conversion can proceed for any model
that uses it.

The agent auto-matches where a snake_case equivalent exists. Rows with status **Pending**
require the EProd team to confirm the correct Snowflake target.

| Dremio ref | Found in model(s) | Snowflake dbt model | Status |
|---|---|---|---|
| *(populated by agent at Gate 0)* | | | Pending |

**How to resolve:** For each Pending row, confirm the `dbt_snowflake/models/` path of the
Snowflake intermediate or mart model that this Dremio view was replaced by. Once all rows
are filled in, reply to the agent at Gate 0 to proceed.

---

## Manual review items (non-blocking)

SQL patterns the conversion agent could not auto-convert. These are marked `-- REVIEW:`
inline in the converted model files. Each needs a human decision before the final `dbt run`.

| Model file | Pattern description | Line / context | Resolution |
|---|---|---|---|
| *(populated during conversion)* | | | |

**How to resolve:** Open the flagged model, read the `-- REVIEW:` comment, apply the fix
manually, then remove the comment. Re-run `dbt compile` to verify.

---

## Notes

- Add any other questions or open decisions here as the migration progresses.
- Once a question is resolved, mark the row's Status column as **Done** and add a brief
  note on what was decided.
