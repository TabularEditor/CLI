# Semantic modeling practices for `te`

Companion to the `te-cli` skill. The skill teaches how to drive the CLI; this file teaches what to build with it. Each practice pairs a modeling decision with the `te` command that applies it, and cites the authoritative source (Microsoft Learn, SQLBI, or the Tabular Editor blog). Apply these after `te add` creates an object, and gate every batch with `te validate` and `te bpa run`.

Three cross-cutting reminders that change every command below:

- Mutating commands are dry runs by default: they print a before/after diff of what would change and discard it. Pass `--save` to persist (or `--save-to <path>` to write elsewhere). Inside `te interactive`, edits stage in memory until `save` instead.
- Each `Bash` call is a fresh shell, so `te connect` does not carry over. Pass `--model <path>` (or `-s`/`-d` for a deployed model) on every command, or set `TE_SESSION=<name>` first. The model is never a trailing positional path.
- Properties are set with `-p Name=Value` (repeatable; several `-p` on one command make one atomic change) and read with `te get <obj> -p Name`. Names are PascalCase as `te get` prints them and case-insensitive on input. `te get <obj>` ends with a `Settable:` line listing exactly what `te set -p` accepts on that object; it is the authoritative discovery tool. When a property is read-only there (for example `DataType` on a calculated-table column) or missing, fall back to `te script` (TOM).

## Dimensional design (star schema)

Build a star: one fact table per business process at a single grain, surrounded by conformed dimensions joined dimension-to-fact by one-to-many relationships. The VertiPaq storage engine and the DAX engine are both optimized for this shape; dimensions filter and group, facts summarize, which is exactly how visuals generate DAX. Reserve a flat single-table model for small prototypes or a deliberate DirectQuery denormalization.

| Practice | Why | `te` command |
|---|---|---|
| Model a star, not a flat table | flat models measured up to ~2.65x larger in RAM and up to 2x slower on complex queries; where flat wins it is marginal | inspect topology first: `te list` (tables), `te list Relationships` (or DAX `EVALUATE INFO.VIEW.RELATIONSHIPS()` for the friendly cross-filter view), `te deps "Sales/Revenue"` |
| One consistent grain per fact table; never join fact-to-fact (header/detail) directly | mixing grains gives wrong aggregations; fact-to-fact becomes a limited relationship (SQLBI's 1.4B-row test: ~15x CPU, 17 minutes vs seconds for distinct count) | add a shared dimension and relate both facts to it rather than to each other |
| Flatten snowflake sub-dimensions into one dimension per business entity | extra hops cost CPU on every filter pass and a hierarchy cannot span tables; run-length encoding makes the duplication cheap. Keep a sub-dimension separate only when it is large and reused across parents | do the join in Power Query upstream, then relate the single dimension |
| Keep degenerate dimensions (order/invoice number) as hidden fact columns | a one-column table for a high-cardinality attribute adds size and clutter for no gain | `te set Sales/OrderNumber -p IsHidden=true --save` |
| Host model measures on a dedicated hidden measure table; prefix to sort it to the top | separates calculations from data and keeps the field list clean (organizational, not an engine requirement; below ~20 measures, display folders inside the source table are equally valid) | `te add _Measures -t Table --columns "_Measures:String" --partition-expression 'let Source = #table({"_Measures"}, {{""}}) in Source' --save`, then `te set _Measures -p IsHidden=true --save`, then `te move Sales/Revenue "_Measures/Revenue" --save` |
| Use plain business names (Customer, Sales); drop DIM_/FACT_ prefixes | a table's role comes from cardinality, not its name; prefixes degrade Q&A and Copilot interpretation (organizational, bends to warehouse naming constraints) | `te move FACT_Sales Sales --save` (renaming a table also reports the relationships that pointed at it) |

Nuance to preserve: SQLBI shows a flat model can beat star for some simple low-cardinality groupings, so "star" is the default rather than a universal absolute; when flat wins it is marginal, when it loses it is far worse.

Sources: [Understand star schema (MS Learn)](https://learn.microsoft.com/power-bi/guidance/star-schema), [The importance of star schemas (SQLBI)](https://www.sqlbi.com/articles/the-importance-of-star-schemas-in-power-bi/), [Star schema or single table (SQLBI)](https://www.sqlbi.com/articles/power-bi-star-schema-or-single-table/), [Header/Detail vs Star Schema (SQLBI)](https://www.sqlbi.com/articles/header-detail-vs-star-schema-models-in-tabular-and-power-bi/), [Naming conventions (Tabular Editor)](https://tabulareditor.com/blog/naming-conventions-for-power-bi-semantic-models).

## Relationships

Default every relationship to one-to-many cardinality and single cross-filter direction, flowing from the dimension to the fact. This is predictable, lets the storage engine pre-build join indexes, and avoids ambiguous filter paths.

| Practice | Why | `te` command |
|---|---|---|
| Add the dimension-to-fact relationship before authoring measures that cross it | measures using `RELATED()` or cross-table `CALCULATE()` validate at save time and the gate rejects with `DAX0002` if the relationship is missing | `te add "Sales[ProductKey]->Product[ProductKey]" --save` |
| Keep single cross-filter direction; never set model-level bidirectional just to sync slicers | bidirectional on a model with 3+ related tables creates two propagation paths and subtly wrong numbers; scope the rare genuine need to one measure with `CROSSFILTER(..., BOTH)`. Since 2019 a visual-level "is not blank" measure filter replaces the slicer-sync use case | list relationships with `te list Relationships` (`--paths-only` prints the `Relationships/{guid}` paths; DAX `EVALUATE INFO.VIEW.RELATIONSHIPS()` gives the friendly cross-filter view), read one with `te get "Relationships/{guid}"` (the display name `te list` shows is also accepted; the `->` shorthand is for `te add` only), and fix a stray bidirectional one with `te set "Relationships/{guid}" -p CrossFilteringBehavior=OneDirection --save` (`IsActive`, `FromCardinality`, `ToCardinality`, `SecurityFilteringBehavior`, `RelyOnReferentialIntegrity` are settable the same way) |
| Treat many-to-many cardinality as a last resort; bridge dimension-to-dimension via two one-to-many relationships (exactly one leg bidirectional) | one-to-many gives a regular relationship with a blank row for RI violations; many-to-many cardinality gives a limited relationship with no join index, multiple passes, and silent drops of unmatched values | model the bridge with `te add` relationships rather than a direct many-to-many |
| When many-to-many is genuinely required, document that totals are non-additive | with shared membership the total counts each item once and will not equal the sum of visible rows; this is correct by design, the risk is silent misreading | add a `te set <measure> -p Description="..."` note on affected measures |
| Role-playing dimensions: duplicate the physical dimension with an active relationship per role; use one shared table with inactive relationships + `USERELATIONSHIP` only when simultaneous multi-role filtering is not needed | active relationships give correct drag-and-drop results with no DAX; inactive ones force a `USERELATIONSHIP` wrapper in every measure and break Q&A. Never use `USERELATIONSHIP` in a calculated column (use `LOOKUPVALUE`) | load the dimension twice in Power Query (Order Date, Ship Date), each with its own active relationship; deactivate a leg with `te set "Relationships/{guid}" -p IsActive=false --save` |
| Avoid one-to-one; merge the tables in Power Query and use display folders | one-to-one splits fields users expect together, blocks cross-table hierarchies, and adds blank rows | merge upstream, then `te set <col> -p DisplayFolder="..."` to regroup |
| Prefer regular over limited relationships; validate referential integrity before deploy | regular relationships propagate in one engine pass; limited relationships force multiple passes and degrade above low-cardinality joins | `te validate --model ./model` and check for unmatched keys before `te deploy` |

Sources: [Relationships (SQLBI)](https://www.sqlbi.com/articles/relationships-in-power-bi-and-tabular-models/), [Bi-directional guidance (MS Learn)](https://learn.microsoft.com/power-bi/guidance/relationships-bidirectional-filtering), [Bidirectional ambiguity (SQLBI)](https://www.sqlbi.com/articles/bidirectional-relationships-and-ambiguity-in-dax/), [Many-to-many guidance (MS Learn)](https://learn.microsoft.com/power-bi/guidance/relationships-many-to-many), [Regular and limited relationships (SQLBI)](https://www.sqlbi.com/articles/strong-and-weak-relationships-in-power-bi/), [Active vs inactive (MS Learn)](https://learn.microsoft.com/power-bi/guidance/relationships-active-inactive), [Ambiguous paths (Tabular Editor)](https://tabulareditor.com/blog/there-are-ambiguous-paths-in-power-bi).

## VertiPaq, cardinality, and data types

Column cardinality (distinct value count), not row count, is the primary driver of model size. VertiPaq stores a per-column dictionary plus a compressed per-row index, and compression is governed almost entirely by distinct values. Profile with `te vertipaq` and attack the highest "% of DB" columns first.

| Practice | Why | `te` command |
|---|---|---|
| Profile by size and fix the biggest columns first | a high-cardinality column's dictionary alone can exceed 90% of its storage | `te vertipaq --columns --detail --top 20 -s ws -d model` (stats live in the deployed database; offline: `--import stats.vpax`); scope with `te vertipaq Sales`; `--output-format json` for the full dump |
| Split a high-cardinality datetime into Date and Time parts (in Power Query, not a calculated column) | a sub-second datetime is near-unique; one real case went from 38.1% of DB to 0.3% (>99% reduction); a calculated column over the original reclaims nothing | fix in Power Query, then verify with `te vertipaq Sales/OrderDate` |
| Prefer narrow integer surrogate keys for relationship columns | the win is a smaller dictionary for the same distinct count, NOT an encoding switch (VertiPaq always hash-encodes relationship columns); SQLBI measured a relationship dropping from 4 MB to under 50 KB | `te set Sales/CustomerKey -p DataType=Int64 --save` (data columns only; a calculated column's type follows its expression) |
| Use a Date data type (not integer YYYYMMDD) for the date key | SQLBI's 2B-row test found storage and scan essentially identical, so usability decides: Date enables native arithmetic and classic time intelligence | `te set Date/Date -p DataType=DateTime --save` |
| Use Fixed Decimal (Currency) for monetary values; avoid Double/Single for aggregated columns | VertiPaq scans segments in parallel and floating-point addition is non-associative, so Double sums can vary between runs; Fixed Decimal is a deterministic scaled integer | `te set Sales/Amount -p DataType=Decimal --save` (the diff lists the dependent measures the retype cascades to) |
| Remove unused columns before deployment, not after | every imported column costs dictionary, data-segment, and attribute-hierarchy memory; removing one 50M-value identity column cut a model from 1.3 GB to 490 MB | `te deps --unused --hidden` to find candidates, `te deps Sales/SomeCol --downstream` to check one, then `te remove Sales/SomeCol --dry-run` and `te remove Sales/SomeCol --if-exists --save` |
| Set `IsAvailableInMDX` = false on high-cardinality hidden columns (keep it true for Sort By Column targets when MDX/Excel clients exist) | the per-column attribute hierarchy is pure overhead on hidden columns unreachable from MDX (one case: 1.1 GB of hierarchy) | `te set Sales/Key -p IsAvailableInMDX=false --save` |
| Reduce numeric precision to what analysis needs when the business agrees | fewer distinct values means smaller dictionaries; rounding loses auditability, so validate with owners and consider keeping a hidden full-precision copy | round in Power Query, then re-profile with `te vertipaq` |

Nuance to preserve: integer-key superiority is about dictionary size, not a value-vs-hash encoding switch (the Tabular Editor blog rightly pushes back on "integer keys are mandatory" as relational dogma); state the mechanism correctly. The Date-vs-integer-key choice is usability-driven, not a performance win.

Size rules are not part of the built-in BPA set. To gate on cardinality or "% of DB" in CI, write the rule yourself, point `te bpa run --rules` at it, and feed it statistics with `--vpax stats.vpax` (exported by `te vertipaq --export stats.vpax`).

Sources: [Optimizing high cardinality columns (SQLBI)](https://www.sqlbi.com/articles/optimizing-high-cardinality-columns-in-vertipaq/), [Data reduction techniques (MS Learn)](https://learn.microsoft.com/power-bi/guidance/import-modeling-data-reduction), [Optimizing semantic model size (Tabular Editor)](https://tabulareditor.com/blog/a-comprehensive-guide-to-optimizing-semantic-model-size-in-power-bi-and-fabric), [Date or Integer for dates (SQLBI)](https://www.sqlbi.com/articles/choosing-between-date-or-integer-to-represent-dates-in-power-bi-and-tabular/), [Choosing numeric data types (SQLBI)](https://www.sqlbi.com/articles/choosing-numeric-data-types-in-dax/), [Costs of relationships (SQLBI)](https://www.sqlbi.com/articles/costs-of-relationships-in-dax/).

## Usability and AI-readiness metadata

A model is consumer-ready and Copilot-ready only once measures and columns carry format strings, folders, descriptions, and the right summarization and visibility flags. Copilot and data agents read this metadata to ground DAX, so the same work that helps humans helps AI.

| Practice | Why | `te` command |
|---|---|---|
| Set a format string on every visible measure and numeric/date column | unformatted values render raw and inconsistently; Copilot reads format strings when grounding DAX | `te set "_Measures/Revenue" -p FormatString="#,0.00" --save`; find the gaps with `te get Measures --where FormatString= --ls --paths-only` |
| Hide surrogate/foreign-key columns and set `SummarizeBy` = none on numeric columns that should not aggregate | keys are scaffolding, not attributes; `none` removes the implicit-aggregation sigma so a key cannot be summed into nonsense, and Copilot/Q&A exclude hidden columns | `te set Sales/CustomerKey -p IsHidden=true -p SummarizeBy=None --save` |
| Apply Sort By Column to display columns with non-alphabetical order (month, weekday, fiscal period) and hide the helper | without it "Month Name" sorts April, August, December; the sort column must be 1:1 with the display column | `te set Date/MonthName -p SortByColumn=MonthNumber --save` then `te set Date/MonthNumber -p IsHidden=true --save` |
| Add descriptions to visible tables, columns, and measures; front-load the key guidance in the first 200 characters | descriptions feed Copilot/data-agent DAX (which reads the first 200 chars) and surface as service tooltips for humans | `te set "_Measures/Revenue" -p Description="Net revenue after returns and discounts" --save` |
| Use human-readable, unique, business-aligned names; organize measures into display folders; avoid duplicate column names across tables | ambiguous or duplicated names make Copilot/Q&A misroute; folders keep a large field list navigable (purely cosmetic) | `te set "_Measures/Revenue" -p DisplayFolder="Revenue" --save`; qualify clashes as "Customer Name" vs "Store Name" with `te move` |
| Mark `IsKey` on dimension primary keys, then hide them | signals the unique grain the engine builds filters from, and keeps the field list clean | `te set Product/ProductKey -p IsKey=true -p IsHidden=true --save` |
| Set data categories on geographic, URL, and image columns | enables map visuals, clickable links, and image rendering, and Copilot reads them for grounding | `te set Geo/City -p DataCategory=City --save`, `te set Customer/Photo -p DataCategory=ImageUrl --save` (confirm category values with `te get`) |

For bulk metadata across many objects, prefer one `te script` pass over N `te set` calls; the model loads once, avoiding the ~1-2s startup per invocation. Without `--save` the script runs as a dry run and prints the diff:

```bash
te script --inline 'foreach (var m in Model.AllMeasures) if (string.IsNullOrEmpty(m.DisplayFolder)) m.DisplayFolder = "Uncategorized";' --model ./model --stat --save
```

Sources: [Star schema - Measures (MS Learn)](https://learn.microsoft.com/power-bi/guidance/star-schema#measures), [Prepare a model for Copilot (MS Learn)](https://learn.microsoft.com/power-bi/create-reports/tutorial-copilot-power-bi-prepare-model), [Semantic model best practices for data agent (MS Learn)](https://learn.microsoft.com/fabric/data-science/semantic-model-best-practices), [Sort By Column side effects (SQLBI)](https://www.sqlbi.com/articles/side-effects-in-dax-of-the-sort-by-column-setting/), [Built-in BPA rules (Tabular Editor)](https://docs.tabulareditor.com/en/features/built-in-bpa-rules.html), [Naming conventions (Tabular Editor)](https://tabulareditor.com/blog/naming-conventions-for-power-bi-semantic-models).

## Date tables and time intelligence

Time intelligence needs one dedicated, shared Date dimension that all facts relate to, marked as a date table, with a contiguous date column at day grain spanning complete years.

| Practice | Why | `te` command |
|---|---|---|
| Build one shared Date dimension; never drive time intelligence off a fact datetime column | classic time-intelligence needs a separate date spine to apply `REMOVEFILTERS` against, and one shared table lets a single slicer drive every fact | relate each fact date to `Date[Date]` with `te add "Sales[OrderDate]->Date[Date]" --save` |
| Mark the Date table (`DataCategory` = Time) and designate the date key | marking designates the date spine so time-intelligence functions (`DATESYTD`, `SAMEPERIODLASTYEAR`) know which column to clear and enumerate against, and it suppresses auto date/time; without it those functions intersect the current filter context and return silent wrong results | `te set Date -p DataCategory=Time --save` then `te set Date/Date -p IsKey=true --save` |
| Remove auto date/time tables (`LocalDateTable_*`, `DateTableTemplate_*`) | auto date/time creates a hidden calculated table per date column, inflating size and refresh, cannot be shared, and is invisible to Excel/paginated/XMLA clients | find them with `te list "LocalDateTable_*"` and `te list "DateTableTemplate_*"`, remove with `te remove <table> --save`; disable the toggle at the report/source so they do not regenerate on the next author save |
| Keep blank/null dates in the fact rather than inflating the Date table; guard comparisons with `NOT ISBLANK()` | blanks on the many side add a `(Blank)` slicer row and distort totals, and comparison operators treat `BLANK` as a pre-1900 date | author guarded measures with `te add ... -p Expression="..."` / `te set ... -p Expression="..."` |
| Standardize one org-wide Date definition (warehouse DimDate, dataflow, or parameterized DAX) | a shared definition prevents fiscal-year, week-numbering, and naming drift across models | import the warehouse DimDate where one exists |

`te validate` checks structural and DAX validity but does not check date contiguity. Probe for gaps with a query against the deployed model (`te query` cannot execute DAX on a local `--model` path):

```bash
te query "EVALUATE ROW(\"Gap\", COUNTROWS(Date) - (MAX(Date[Date]) - MIN(Date[Date]) + 1))" -s ws -d model
```

A nonzero `Gap` means the date column is not contiguous.

Sources: [Date tables guidance (MS Learn)](https://learn.microsoft.com/power-bi/guidance/model-date-tables), [Mark as Date table (SQLBI)](https://www.sqlbi.com/articles/mark-as-date-table/), [Auto date/time guidance (MS Learn)](https://learn.microsoft.com/power-bi/guidance/auto-date-time), [Blank in date columns (SQLBI)](https://www.sqlbi.com/articles/blank-in-date-columns-and-dax-time-intelligence-functions/).

## Measures, calculated columns, and calculation groups

Default to explicit DAX measures for all aggregated logic. Push row-level computation upstream, and use calculation groups to kill measure sprawl.

| Practice | Why | `te` command |
|---|---|---|
| Author aggregations as explicit measures, not implicit sigma-column measures | measures are metadata-only and respond to filter context; implicit measures break MDX clients, let authors pick wrong aggregations, and cannot join calculation groups | `te add "_Measures/Total Sales" -t Measure -p Expression="SUM(Sales[Amount])" -p FormatString="#,0" --save` |
| Push row-level columns upstream: source/warehouse, then Power Query, then DAX calculated column, then measure | DAX calculated columns compress worse (SQLBI: ~4x larger in one case) because they skip the sort-order search and they extend refresh; Power Query columns can fold to the source | compute in Power Query; reserve DAX calculated columns for `RELATED`, `PATH`, or `COMBINEVALUES` keys |
| Remember calculated columns and tables are unsupported in DirectQuery and (as of mid-2026) Direct Lake | DirectQuery prohibits them (use `COMBINEVALUES` for multi-column keys); Direct Lake transcodes from Delta and does not materialize them, so push to the Lakehouse | for a calc column where unavoidable: `te add Sales/Bucket -t CalculatedColumn -p Expression="<DAX>" --save` |
| Use calculation groups for time-intelligence or currency-conversion families with `SELECTEDMEASURE()`; set `DiscourageImplicitMeasures` = true | N base measures x M variants becomes N + M objects; calculation groups only fire on explicit measures, so discouraging implicit measures keeps them consistent | see the worked example below; `DiscourageImplicitMeasures` is a model-level property: `te set . -p DiscourageImplicitMeasures=true --save` |
| Scope items with `ISSELECTEDMEASURE()` rather than `SELECTEDMEASURENAME()` string comparison | `ISSELECTEDMEASURE` participates in rename fixup; a string literal silently breaks on rename | author the item DAX accordingly |
| Set a Format String Expression on any item that changes a measure's meaning (percentages, currency) | a YOY% item returning 0.12 displays "0" under a `#,##0` base format | `te set "Time Intelligence/YOY%" -p FormatStringExpression="\"0.0%\"" --save` |
| Set Precedence deliberately when multiple calculation groups coexist, and test with a trivial measure first | active items apply in precedence order (highest is outermost), which also decides whose format string wins | `te set "Time Intelligence" -p CalculationGroupPrecedence=10 --save`; `te list CalculationGroups` shows the groups |
| Consider parameterized DAX UDF measures over calculation groups when calculations must be opt-in (tooltip pages, filter panes, flexible matrices) | calculation items apply to every measure in context and can corrupt non-numeric measures; UDF measures are called explicitly per measure | see [When DAX UDF measures beat calculation groups (Tabular Editor)](https://tabulareditor.com/blog/when-dax-udf-measures-beat-calculation-groups-for-time-intelligence) |

Worked example, a Time Intelligence calculation group:

```bash
te add "Time Intelligence" -t CalcGroup --model ./model --save
te add "Time Intelligence/YTD" -t CalcItem -p Expression="CALCULATE(SELECTEDMEASURE(), DATESYTD('Date'[Date]))" -p Ordinal=0 -p FormatStringExpression="SELECTEDMEASUREFORMATSTRING()" --model ./model --save
te get "Time Intelligence/YTD" --model ./model      # confirm; Settable: Description, Expression, Ordinal, FormatStringExpression, Name
```

`Expression`, `Ordinal`, `FormatStringExpression` and `Description` are the settable `CalculationItem` properties. Items are addressed as `<group>/<item>` (or `<group>/CalculationItems/<item>` when a column shares the name); `te list "Time Intelligence/CalculationItems" --paths-only` prints them. On the group itself, `te get` lists `CalculationGroupPrecedence`, `NoSelectionExpression`, `MultipleOrEmptySelectionExpression` and their format-string variants as settable.

Sources: [Calculated columns and measures (SQLBI)](https://www.sqlbi.com/articles/calculated-columns-and-measures-in-dax/), [DAX calculated columns vs Power Query (SQLBI)](https://www.sqlbi.com/articles/comparing-dax-calculated-columns-with-power-query-computed-columns/), [Data reduction - custom columns (MS Learn)](https://learn.microsoft.com/power-bi/guidance/import-modeling-data-reduction#preference-for-custom-columns), [Calculation groups (MS Learn)](https://learn.microsoft.com/analysis-services/tabular-models/calculation-groups), [Understanding calculation groups (SQLBI)](https://www.sqlbi.com/articles/understanding-calculation-groups/), [Controlling format strings in calculation groups (SQLBI)](https://www.sqlbi.com/articles/controlling-format-strings-in-calculation-groups/).

## Security and governance (RLS, OLS, BPA)

Define roles, set permissions, attach filters, then validate and deploy. Prefer dynamic RLS, assign Entra ID groups rather than individuals, and treat BPA as the governance gate.

```bash
te add Roles/RegionManagers -t Role -p ModelPermission=Read --model ./model --save
# Dynamic filter on the fact table; the container-form path is the one `te list Roles/RegionManagers/TablePermissions --paths-only` prints
te add "Roles/RegionManagers/TablePermissions/Sales" -t TablePermission -p FilterExpression="[Region] = USERPRINCIPALNAME()" --model ./model --save
te get "Roles/RegionManagers/TablePermissions/Sales" --model ./model   # confirm the Settable: list before scripting many roles
te validate --model ./model
te deploy --model ./model --target-server ws --target-database model --deploy-roles --deploy-role-members --ci github          # dry run: prints the TMSL
te deploy --model ./model --target-server ws --target-database model --deploy-roles --deploy-role-members --execute --force --ci github
```

| Practice | Why |
|---|---|
| Prefer dynamic RLS with `USERPRINCIPALNAME()` against a mapping table; assign Entra ID groups, not users | one dynamic role scales with master data; group membership changes in Entra ID without republishing. Use `USERPRINCIPALNAME` (not `USERNAME`, which returns `DOMAIN\user` in Desktop) |
| Filter on the dimension and let active relationships propagate; move the filter to the fact when the dimension exceeds ~131K rows or carries `USERELATIONSHIP` paths | small dimensions get a reusable bitmap index; above the threshold the cache never activates and dimension filtering becomes worse than fact filtering |
| One role per access tier; membership is additive (OR), not subtractive | a user in a `FALSE()` role and a `TRUE()` role sees everything; combining OLS and RLS for the same user across roles throws a query-time error, so keep them in one combined role |
| Give consumers only Viewer/Read; Admin/Member/Contributor and model-editor SPNs bypass RLS and OLS entirely | any Edit-level identity silently removes all security; keep ETL/XMLA identities separate and least-privileged |
| Avoid `LOOKUPVALUE` in RLS filters; propagate via relationships | `LOOKUPVALUE` in a security filter runs in the formula engine on every query and blocks storage-engine caching |
| Do not rely on `USERELATIONSHIP`/`CROSSFILTER` to override an RLS-carrying relationship; relocate the filter | RLS propagates only through active relationships, and the engine blocks `USERELATIONSHIP` on an RLS-carrying relationship; `TREATAS` workarounds need explicit semantic review |
| Set object-level security with `te`: per-role `MetadataPermission` on a table permission (`= None` hides both the data and the object name), e.g. `te set "Roles/RegionManagers/TablePermissions/Sales" -p MetadataPermission=None --save`. Use `te script` for TOM-level access (column-level OLS) if the property is not in the `Settable:` list | DAX security targets tables and columns, not measures, so hiding a measure needs a sentinel-table workaround. Confirm the property name with `te get <obj>` before scripting it |
| Run BPA as a governance gate (Microsoft Analysis Services rule set as baseline + org rules); remember BPA detects whether roles exist, not whether they are correct | `te bpa run` loads rules from a URL and gates deploy on error-severity violations. The built-in layer is exactly Tabular Editor 3's built-in rule set; skip one with `te bpa rules disable <id>` rather than editing it |

```bash
te bpa run --rules https://raw.githubusercontent.com/microsoft/Analysis-Services/master/BestPracticeRules/BPARules.json --fail-on error --ci github --model ./model
te bpa run --model ./model --output-format json      # findings envelope: summary + findings[] with code = rule ID, objectPath resolvable by te get
```

Sources: [RLS guidance (MS Learn)](https://learn.microsoft.com/power-bi/guidance/rls-guidance), [Security cost (SQLBI)](https://www.sqlbi.com/articles/security-cost-in-analysis-services-tabular/), [RLS with inactive relationships (SQLBI)](https://www.sqlbi.com/articles/dax-limitations-with-inactive-relationships-and-row-level-security-rls/), [Object-level security (MS Learn)](https://learn.microsoft.com/analysis-services/tabular-models/object-level-security), [OLS in Power BI (Tabular Editor)](https://tabulareditor.com/blog/ols-in-power-bi), [RLS in Power BI semantic models (Tabular Editor)](https://tabulareditor.com/blog/row-level-security-in-power-bi-semantic-models), [Using the BPA (Tabular Editor)](https://docs.tabulareditor.com/common/using-bpa.html).

The validate-then-BPA-then-format authoring loop and the per-save BPA speed knob are in SKILL.md; run those gates continuously, not only at deploy. Formatting a changed expression is `te set <path> --format Expression --save`; a whole-model sweep is `te script --inline "Model.AllMeasures.FormatDax();" --save`.

## Scope notes

These practices are grounded in the sources above. The research did not cover, and this file deliberately does not assert detailed guidance on: user-defined aggregation design (grain, agg-awareness precedence), incremental-refresh and partitioning strategy, composite/Direct Lake-plus-Import tuning, or absolute size thresholds at which star beats flat. For those, consult the live docs and verify against real VertiPaq measurements with `te vertipaq`.
