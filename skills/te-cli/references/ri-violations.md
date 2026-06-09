# Referential integrity violations (`te`)

How to detect, interpret, and locate RI violations in a semantic model with the
`te` CLI. Companion to SKILL.md and `gotchas.md`.

## What an RI violation is

A row on the **many (fact)** side of a regular relationship whose foreign key has
no matching row on the **one (dimension)** side, or whose key is blank. The engine
routes those rows to an auto-added blank member on the one side, so grand totals
still reconcile, but the unmatched rows silently vanish from any slice by the
dimension's attributes. Silent wrong numbers, not an error.

## Seeing them: `te vertipaq`

`te vertipaq --output-format json` gives each table object a `riViolations`
integer. Always use json; the text render crashes when piped/non-TTY (see
`gotchas.md`).

```bash
te vertipaq --output-format json -s <ws> -d <model> \
  | jq '.tables[] | select(.riViolations > 0) | {dimension:.name, relationshipsWithViolations:.riViolations}'
```

**What the number means — read this carefully.** `riViolations` on a table counts
the number of **relationships pointing at that table** (the table acting as the
one/dimension side) that currently have at least one unmatched key. It is **not**
the count of orphaned rows, and **not** the count of distinct missing keys.

So `riViolations: 3` means three relationships into that dimension are violated.
Orphaning ten million fact rows can still read as a single-digit counter.
(Verified: dropping one category from a dimension orphaned ~10M fact rows across
two fact tables yet moved the counter only `2 -> 3` — the `+1` was a third
relationship, `Forecast[Type] -> Products[Type]`, newly losing its matches.)

The counter answers *which dimension* and *how many of its relationships*. It does
not tell you which keys or how many rows. For that, query.

## Finding the actual violations (DAX via `te query`)

First discover the candidate relationships (`te ls` cannot enumerate them):

```bash
te query -q "EVALUATE INFO.VIEW.RELATIONSHIPS()" -s <ws> -d <model>
```

Then pick a detector based on whether the two key columns share a data type.

### Same-type keys -> `EXCEPT` (simple and exact)

```dax
EVALUATE EXCEPT(VALUES('Fact'[Key]), VALUES('Dim'[Key]))
```

Returns the distinct fact key values absent from the dimension. 0 rows = clean. No
`UNION`/`ADDCOLUMNS` scaffolding needed; one relationship per query.

### Mismatched-type keys (e.g. Int64 fact <-> String dim) -> `RELATED`, not `EXCEPT`

`EXCEPT` refuses a type mismatch:

```
Function 'EXCEPT' does not support joining a column of type Integer with a column of type Text.
```

The relationship still works because the engine **coerces on the join**; `EXCEPT`
does not. Use the relationship via `RELATED`. Two traps to avoid:

- **Many->one does not propagate fact->dim.** With a default single-direction
  relationship, filtering the fact (many) side does **not** filter the dimension
  (one) side. So
  `FILTER(VALUES('Fact'[Key]), ISBLANK(CALCULATE(COUNTROWS('Dim'))))`
  always returns 0 — a **false negative** that looks like "no violations".
- **`RELATED` is direction-agnostic** from a fact row, so it is the reliable
  detector.

Orphaned fact **rows**:

```dax
EVALUATE
ROW("orphan rows", COUNTROWS(FILTER('Fact', ISBLANK(RELATED('Dim'[Key])))))
```

Distinct orphaned **key values** (the "what is missing" list):

```dax
EVALUATE SUMMARIZE(FILTER('Fact', ISBLANK(RELATED('Dim'[Key]))), 'Fact'[Key])
```

Write results to a file for a clean, noise-free dump:
`te query --file find-orphans.dax --output-file orphans.csv`.

## Common causes

- **Over-filtering in the semantic model.** A `Table.SelectRows` step in the
  dimension's Power Query, or a restriction that trims the dimension to a subset
  (e.g. "active products only") while the facts still carry the full key history.
  This is the most common self-inflicted case.
- **ETL / source drift.** Dimension and fact loads run on different cadences or
  filters: late-arriving dimensions, a category excluded upstream, an incremental
  window that trims the dimension but not the fact, a dim reload that fails while
  facts succeed.
- **Data quality.** Blank/null foreign keys; placeholder keys (`-1`, `0`) with no
  matching dimension row; format drift between systems (leading zeros, trailing
  spaces); and int-vs-text key columns that only match by coincidental coercion —
  a smell worth fixing regardless, since it disables a clean `EXCEPT` check and
  hides drift.

## Fixing

- Stop over-filtering the dimension, or add an explicit unknown/blank member so
  every fact key resolves.
- Or coalesce unmatched fact keys to a known "Unknown" member during ETL.
- Re-verify with the `EXCEPT` / `RELATED` query above. After a data fix,
  `riViolations` clears only on refresh + recalc (relationship indexes rebuild on
  `te refresh --type calculate`).
