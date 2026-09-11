# Changelog

All notable changes to the `te-cli` skill are documented in this file.

## [0.4.0] - 2026-09-11

Aligns the skill with CLI 0.7.0, which removed six commands, changed the property and model-source syntax on every command, and made deploy and refresh dry runs by default. Every file was re-verified against the 0.7.0 binary and the current docs.

### Changed

- `te refresh --execute` asks for confirmation like `te deploy`; every unattended refresh example (SKILL.md, `command-reference.md`, `config-cicd-env.md`, `workflows.md`, `fabric-cli-tandem.md`) now passes `--execute --force`, and the exit-code list names an unattended `--execute` without `--force` as a failure.
- `te deploy` documented as failing (non-zero exit, JSON `success: false`) when the server accepts the metadata but parks objects with errors (`command-reference.md`, `config-cicd-env.md`, `gotchas.md`, `fabric-cli-tandem.md`).
- A partition's body is `-p Expression` on `te get` and `te set` for every partition kind; the `MExpression` asymmetry entry in `gotchas.md` and the `MPartition` row in the `-p` property table were rewritten, and `workflows.md` examples use `Expression` (`MExpression` / `Query` noted as aliases).
- `-p Name=null` clears text properties too, and a piped value is verbatim; `--unset <Name>` added to `te set` in SKILL.md, `command-reference.md` and `gotchas.md`.
- `--semicolons` described as an input dialect that exists only on `te util format-dax` and is refused on `te set --format`; the `formatOptions.useSemicolons` config key removed from `config-cicd-env.md` (`command-reference.md`, `workflows.md`, `gotchas.md`).
- `te get` object-path rules: a wildcard or container path lists every match without `--ls`; `-p`, `--deps` and `--properties` need one object (`command-reference.md`, `gotchas.md`).
- `te find` documented as a literal, case-insensitive substring match unless `--regex`; `te get --where` with no path filters the top-level tables; empty results name the scope and matching mode and JSON prints `[]` (`command-reference.md`, `gotchas.md`).
- `te get <object>` JSON leads with `objectPath` (was `path`); `te deps` JSON entries carry `objectPath`, `object`, `objectType`; `te diff` lists the contents of an object present in only one model (`command-reference.md`).
- `--ci` accepted values closed: `vsts`/`azdo`/`azure-devops`, `github`/`gh`, `none`; anything else is rejected instead of silently emitting nothing (`command-reference.md`, `config-cicd-env.md`, `gotchas.md`).
- `--update-schema` and `te add -t Table --source-table` infer the connection from the model's own data source, with `--data-source <name>` to choose among several; explicit connection flags always win (`command-reference.md`, `workflows.md`).
- `te move` refuses to rename relationships, KPIs and table permissions instead of printing `No changes.` (`gotchas.md`).
- Interactive shell: leaving with unsaved staged edits asks to confirm, or warns and exits non-zero when piped; `exit --force`; Ctrl+C discards the half-typed line (SKILL.md, `command-reference.md`, `gotchas.md`).

- Property syntax everywhere is `-p Name=Value` (repeatable; bare `Name=Value` assignments also accepted on `te set`; `te get <path> -p <Name>` projects one property; `te add ... -p Expression="..."`; same on `te macro set` and `te bpa rules set`). Every `-q <name> -i <value>` example in SKILL.md and all references was rewritten; property names are shown PascalCase as `te get` prints them.
- The model is always given with `--model` (or `-s`/`-d`, `--local`, `--recent`, or the active `te connect`), never as a trailing path. All examples that passed a model positionally were rewritten across SKILL.md and every reference file.
- `te deploy` and `te refresh` are documented as dry runs that print TMSL; `--execute` performs the action, and CI examples now pass `--execute --force --non-interactive`. The deploy target is `--target-server`/`--target-database`; `-s`/`-d` on deploy mean the model source. `te deploy --xmla <file>` became `te deploy > <file>`; `te refresh --dry-run` became plain `te refresh`.
- `te save` is documented as `te save-as` (alias `save` in the shell; inside the REPL `save` commits staged edits and `save-as` re-serializes).
- `te format` was replaced: `te set <path> --format <Property>` formats a stored expression (with `--semicolons`, `--long`, `--no-space-after-function` on DAX), `te util format-dax` / `te util format-m` format loose expressions, and a whole-model DAX sweep is `te script --inline "Model.AllMeasures.FormatDax();" --save`. `te migrate` moved to `te util migrate` (`te2-migration.md`, `config-cicd-env.md`).
- `te incremental-refresh` was replaced by ordinary property access: `te get <table>/RefreshPolicy`, `te set <table>/RefreshPolicy -p Prop=Value`, `te set <table> -p RefreshPolicy=null`, and `te refresh --apply-refresh-policy <table> --execute` (`command-reference.md`, `workflows.md`).
- `te script` sources are `--file <path>` (or a bare `.cs`/`.csx` positional), `--inline "<code>"` and `--validate`; `-S`, `-e`, `--script`, `--expression`, `--dry-run` are gone. `te script` exits non-zero when a script calls `Error(...)`.
- Object path grammar rewritten in `command-reference.md` and SKILL.md: `{ } * ?` are reserved and must be quoted, `KPIs`/`Sets`/`Functions` are container keywords (a table literally named `KPIs` is `"'KPIs'"` in filter paths), relationships are `Relationships/{guid}`, calendars `<table>/Calendars/<name>`, sets `<table>/Sets/<name>`, and every path the CLI prints is quoted so it pastes back.
- `-t/--type` documented with its two meanings: disambiguation of a path on `get`/`set`/`remove`/`move`/`deps` (PascalCase values), filtering on `te list` and `te get --ls`/`--where` (lower-case values, now including `kpi`, `set`, `function`, `calculationitem`).
- BPA sections now describe exactly Tabular Editor 3's built-in rule set; the six `VPA_*` rules are no longer built-ins and `--vpa-rules` is gone (`--vpax` still feeds your own VPA-aware rules); `bpa.builtInRules`, `bpa.disabledBuiltInRuleIds`, `te bpa rules disable/enable` documented.
- `config-cicd-env.md` config-key table reconciled with the docs: `autoFormat` is scoped to the objects a mutation changed and always uses the built-in formatter, `formatOptions.shortFormat` and `launchInteractiveMode` defaults corrected, `bpa.onSave` gates `te save-as`; `--output-format` no longer lists a nonexistent `auto` value and `tmdl` is `te get` only; `--ci` no longer claimed on `te script`; runner install rewritten around a committed binary with the public CDN archive as the scriptable alternative.
- `fabric-cli-tandem.md`: `te diff` cannot take a remote side (export with `te save-as -s/-d -o` first), `--ci azdo` corrected to `--ci vsts`, `te connect` persistence and `te deps --unused` caveats corrected, XMLA refresh described as synchronous.
- `pbir-cli-tandem.md`: guidance that leaned on `te replace` now uses `te find` to locate and `te set -p` / `te script --inline` to change, dry run first then `--save`.
- `semantic-modeling-practices.md`: calculation-group, RLS, relationship and bulk-metadata workflows rewritten with `-p` and container-form paths; `DiscourageImplicitMeasures` corrected to a model-level property; calc-item format property is `FormatStringExpression`.
- `testing.md`: JSON output described as the unified findings envelope (`summary`, `findings[]` with `TEST_FAIL`/`TEST_ERROR`/`TEST_SUITE_INVALID`, `testSummary`, `suites[]`, `invalidSuites[]`); a suite that fails validation exits 1.
- Preview cutoff moved to 2026-10-31 in SKILL.md, README and references; wording is "no license is required during preview".
- README rewritten around the folder-based install (`SKILL.md` plus `references/`), matching the docs page; smoke-test question updated to the dry-run deploy behaviour. Plugin manifests bumped to 0.4.0.

### Added

- `te get <path> --properties [--all]`: the authoritative per-object list of property names `-p` accepts, with type, writability and allowed values; unknown-property errors point at it and the `Settable:` line no longer omits `SortByColumn` (SKILL.md, `command-reference.md`, `gotchas.md`).
- `te validate` codes `TE0012` / `TE0013` (name collisions that block `te save-as` unless `--force`/`--skip-validation`) and `TE0014` (TMDL folder without `database.tmdl`); every finding shows its code in the text tables (`command-reference.md`, `gotchas.md`).
- Administrator policies on Windows (`DisableCSharpScripts`, `DisableMacros`, `DisableBpaDownload`, `DisableTelemetry`) and the registry keys they are read from (`config-cicd-env.md`, `gotchas.md`, `command-reference.md`).
- `--non-interactive` with no credentials fails immediately with the ways to sign in (`gotchas.md`).
- `te get` as the single show pipeline: `--where Prop=Value` (repeatable AND, `*` wildcard), `--ls`, `--deps[=upstream|downstream]` with `--deep`/`--max-depth`, `--unused` with `--hidden`, `--paths-only`; `te list` and `te deps` described as shortcuts over it.
- `te list KPIs`, `te list Sets`, `te list Functions`, `--type relationship --paths-only` (GUID form).
- Mutation output: every mutating command prints a before/after diff; `--stat`, `--name-only`, `--diff` flags and the `mutationOutput` config key; JSON `changes[]` with `objectPath`, `objectType`, `changeKind` (`created`/`deleted`/`modified`/`moved`), `properties[] {property, before, after}`, shared with `te diff`.
- Unified findings JSON shape for `te validate`, `te bpa run`, `te test run`, `te query` (`command`, `durationMs`, `summary`, `findings[]` with `severity`, `code`, `objectPath`, `expressionPosition`, `fixable`; `bpa run --fix` outcome under a `fix` key); CI annotations carry the code and info findings are notices.
- `te add <table>/<col> -t DataColumn -p SourceColumn=... -p DataType=...`; `te add "<table>" -t Table --source-table dbo.X` (and `--query`, `--data-source`) from the model's own data source; `te set <table> --update-schema [--drop-removed-columns]`; `te add` container-form paths (`Sales/Measures/Margin`, `Sales/Partitions/Q1`, `Roles/Admin/TablePermissions/Sales`).
- `te util` family (`format-dax`, `format-m`, `migrate`; refuses model flags); remote-to-remote deploy (`te deploy -s src -d m --target-server dst --target-database m2 --execute`); `te refresh` JSON `progress[]`/`vertipaq[]`; `te connect --local` covering Power BI Desktop, Visual Studio and SSAS instances; `te profile set` overrides; `fish` and `powershell` completions; REPL launch via bare `te` and prefix-less help.
- `config-cicd-env.md`: `mutationOutput`, `queryLog`, `profiles` keys; path-resolution precedence; `TE_INTERACTIVE`, `NO_SPINNER`, `CI`, `AZURE_CLIENT_CERTIFICATE_PATH`, `AZURE_AUTHORITY_HOST`; closed `--auth` value list; offline `te script --validate` lint step; stdin secret login idiom.
- `testing.md`: `te test spec --json-schema`, `te test init --path`, `te test list --tag`, bare `te test use` clears the active suite, JSON run example.
- `semantic-modeling-practices.md`: finding unformatted measures with `te get Measures --where FormatString= --ls --paths-only`, auto date tables with `te list "LocalDateTable_*"`, prune candidates with `te deps --unused --hidden`; note that size and cardinality rules are not built-in BPA rules.
- `te2-migration.md`: context-sensitive TE2 flag note, `-LOGIN`/`-SC` as not yet implemented, `te util migrate --output-format json` shape, condensed migration playbook.
- `gotchas.md`: a "syntax that no longer exists" section (`-q`/`-i`, trailing model path, removed commands and script flags, `--vpa-rules`), the reserved-character and `KPIs`/`Sets` quoting traps, the `-t` dual meaning, deploy/refresh doing nothing without `--execute`, `-s`/`-d` on deploy being the source, `mutationOutput none` hiding the diff, `autoFormat` reformatting every changed object, `te set <table> --format` reaching nothing (name the partition), translation for a missing culture exiting 0 with "No changes.", single-object perspective membership not being addressable.

### Fixed

- Removed every section for `te load`, `te open`, `te replace`, `te format`, `te incremental-refresh` and root-level `te migrate`; removed the `te3ExePath` config key and `TE3_EXE_PATH` variable; no license-related command or key remains.
- Removed the MPartition path asymmetry gotcha (`te add Sales/Partitions/X` works) and the two-JSON-document `bpa run --fix` gotcha (one document with a `fix` key); removed the `te move` rename-to-table-name gotcha (verified to work).
- Partition expressions: `te set <partition> -p Expression=` and `te get -p Expression` both error; the property is `MExpression`, while `te add -t MPartition -p Expression=` and `te set <partition> --format Expression` work (verified against the binary).
- `te init` default compatibility level corrected to 1705 (PowerBI mode) / 1500 (Analysis Services); `--auth` default corrected to `auto`; `--output-format` default is `text`, not TTY-dependent.
- `te refresh --partition` cannot be combined with `--table` (the old recipe did both); `te find ... | xargs te get {} -q expression` pipeline corrected to `-p Expression`; `te deploy -p` is `--profile`, not `--property`.
- `pbir-cli-tandem.md`: sort-by clear is `-p SortByColumn=null`; `te init` comment no longer states a compatibility level.
- `fabric-cli-tandem.md`: removed the obsolete "diff exit codes documented inconsistently" note (docs and binary agree: 0 identical, 1 differ, 2 error).
- `workflows.md`: single-measure perspective membership cannot be set through a path on the binary; recipe now uses a `te script` one-liner; dropped the false claim that `te list "Perspectives/X"` confirms membership.

[0.4.0]: https://github.com/TabularEditor/CLI/releases/tag/skill-v0.4.0

## [0.3.0] - 2026-07-02

Aligns the skill with CLI 0.6.0.

### Changed

- Rewrote every command example to use long-form canonical verbs (`te list`, `te remove`, `te move`, `te config list`), with a short callout that Unix aliases (`ls`, `rm`, `mv`) still work everywhere. `te move` also accepts `rename`. `te add` has no short alias.
- Replaced every `--serialization te-folder` occurrence with `database.json` (BREAKING CLI rename in 0.6.0). The mirror on `te connect --workspace-format` and the TE2 migration table (`-F <dir>`) both track the new spelling.
- Removed the "known gap" language for `te list Relationships`: the CLI now enumerates relationships correctly, so SKILL.md and `references/command-reference.md` list `Relationships` as an enumerable container next to `Tables`, `Measures`, etc. The gotcha entry now covers the residual issue (system-assigned relationship names) rather than the wiring gap.
- Dropped the "`--source-type m` on models with a provider data source" pre-validation error from `references/workflows.md`; the CLI now accepts mixed-partition models to match TE3 desktop.

### Added

- `te query "<dax>"` positional-DAX shorthand documented in the quickstart and `references/command-reference.md`; explicit `-q` still wins when both are supplied.
- `te macro set` now accepts repeated `-q <property> -i <value>` pairs in one call, with a worked example in `references/command-reference.md`.
- `te interactive` batch mode (redirected stdin) and its new flags (`--no-banner`, `--echo`, `--batch` / `--no-batch`) added to `references/command-reference.md`, with a batch-mode example block.
- New `launchInteractiveMode` config key (`Auto` / `Never` / `Always`) documented in `references/config-cicd-env.md`.
- `--serialization tmsl` documented as an alias for `bim` (canonical) on `te save`, `te init`, and `te connect --workspace-format`.
- Note that `te bpa run` text output now includes a `Rule ID` column, so IDs can be copied straight into `--fix --rule <id>`.
- Note that `te vertipaq` surfaces a clear error on unknown table/column filters (with up to 10 candidates), and its output is pipe-safe (`te vertipaq > report.txt`, `te vertipaq | less`).
- `references/testing.md`: test-suite authoring guide distilled from `te test spec` (`.test.yaml` anatomy, all assertion types, tolerance semantics, tags, matrix expansion), snapshot regression (`--save`/`--diff`), and A/B compare across two deployed models (`--source-a`/`--source-b`).
- Rebuilt the CI/CD section in `references/config-cicd-env.md` around the two-pipelines-one-artifact shape: PR validation (validate + BPA on the repo artifact, no secrets) vs deploy + post-deploy regression tests per environment; one-time Fabric service-principal setup (XMLA read-write, workspace roles), public-CDN runner install step, full GitHub Actions and Azure DevOps examples, and promotion/approval-gate patterns.
- Documented the public CDN download (`https://cdn.tabulareditor.com/files/cli/latest/te-<os>-<arch>.tar.gz`, `.zip` on Windows; GET only, HEAD returns 404) and corrected the macOS/Linux archive extension to `.tar.gz`.
- Documented the new global `--error-format text|json` flag, `te validate --server-only`, `te add` repeatable `-q <prop> -i <value>` pairs (set extra properties at creation), `te add --mode dual` and `--file`/`--connection-string`, and the short type aliases (`CalcTable`, `CalcColumn`, `CalcGroup`, `CalcItem`).
- Pinned the `te incremental-refresh set` flags (`--rolling-window-periods/-granularity`, `--incremental-periods/-granularity/-offset`, `--mode import|hybrid`, `--source-expression(-file)`, `--polling-expression(-file)`), replacing the "read them from the binary" placeholder.

### Fixed

- `te diff` exit codes were documented inverted; the binary exits 0 identical, 1 models differ, 2 error. Corrected the reference tables and the shell example that branched on exit 2.
- `te query` was shown executing against a local `-m` path; DAX execution needs a deployed model (`-s`/`-d` or an active connection). Same for `te vertipaq` (offline exception: `--import <file.vpax>`), `te refresh`, `te test run`/`snapshot`. Added a critical rule and a gotcha separating metadata commands (local OK) from DAX-executing commands.
- `te query -f` corrected to `--file` (no short form exists).
- `te macro set -q description` example replaced; the settable properties are `name`, `execute`, `enabled`, `tooltip`, `validContexts`.
- Removed the nonexistent `te script --timeout` flag.
- `te test snapshot`/`compare` were documented as a bare local snapshot-diff pair; snapshot takes `--save <file>`/`--diff <baseline>` and compare is A/B between two deployed models (`--source-a`/`--source-b`).
- The pbir-tandem rename workflows claimed `te move` does not rewrite DAX and made `te replace` a mandatory second step; on 0.6.0 `te move` cascades bracketed DAX references for both measure and table renames (verified against the binary). `te replace` is now scoped to what the cascade cannot see (string-literal name comparisons, descriptions, partition M). Cross-table moves still do not rewrite table-qualified refs; the save gate rejects them, which the workflow now explains.
- Softened the "bpa run --fix emits TWO JSON documents" gotcha: the fix-summary document is only emitted when fixes actually apply.

[0.3.0]: https://github.com/TabularEditor/CLI/releases/tag/skill-v0.3.0

## [0.2.0] - 2026-06-08

### Changed

- Restructured the single-file skill into a lean `SKILL.md` plus a `references/` directory for progressive disclosure: `command-reference.md`, `workflows.md`, `gotchas.md`, `config-cicd-env.md`, `te2-migration.md`.

### Added

- Packaged as a Claude Code plugin (`te-cli-agentic-use`) with `.claude-plugin/plugin.json` and a one-entry `marketplace.json`, installable via `claude plugin install`.
- `references/semantic-modeling-practices.md`: semantic modeling best practices (star schema, relationships, VertiPaq and cardinality, usability and AI-readiness metadata, date tables, measures vs calculated columns, calculation groups, RLS/OLS/BPA), each tied to a `te` command and cited from Microsoft Learn, SQLBI, and the Tabular Editor blog.
- Worked authoring workflows for RLS roles, calculation groups, date tables, perspectives, translations, incremental refresh, and field parameters.
- `references/fabric-cli-tandem.md` and `references/pbir-cli-tandem.md`: tandem workflows for using `te` with the Fabric CLI (`fab`) and the pbir CLI, including model-to-report rename propagation.
- SKILL.md "Common operations" quick-reference: summarize, search, query (inline vs `.dax` file), make a change, make bulk changes, and validate/optimize, each with the most concise command.

### Fixed

- Promoted the `te connect` session-scope behavior and the MPartition path asymmetry to critical rules.
- Corrected the save-gate wording (blocks on newly introduced validation errors, not a diff against the loaded model), softened the `--force` framing, and hedged the bidirectional workspace-mirror claim.
- Documented that `te ls` cannot enumerate relationships (`te ls Relationships` / `--type relationship` error with `No objects match path 'Relationships'`, a CLI wiring gap); relationship discovery now uses DAX `EVALUATE INFO.VIEW.RELATIONSHIPS()` throughout, with a gotcha explaining the error tell.

[0.2.0]: https://github.com/TabularEditor/CLI/releases/tag/skill-v0.2.0

## [0.1.0] - 2026-06-04

### Added

Initial public preview release of the `te-cli` skill. Coverage:

- All `te` commands across all families: Model I/O (`load`, `save`, `init`, `open`), Editing (`set`, `add`, `rm`, `mv`, `replace`), Inspection (`ls`, `get`, `find`, `diff`, `deps`), Analysis & Quality (`validate`, `bpa`, `vertipaq`, `format`), Execution (`query`, `script`, `macro`), Deployment & Refresh (`deploy`, `refresh`, `incremental-refresh`), Testing (`test`), Connection & Auth (`connect`, `auth`, `profile`, `session`), Configuration (`config`, `license`, `migrate`), Shell (`interactive`, `completion`).
- Authentication patterns: interactive browser, service principal (secret + certificate), env-var, managed identity.
- Object path grammar: slash-form, DAX-form (quoted + bracket-suffix), wildcard filter paths.
- Staging model (`--save` / `--stage` / `--revert`) and the `interactiveEditMode` config.
- Common workflows: data-bound table creation with one-shot `--columns` + `--partition-expression`, TMDL/BIM/PBIP conversion, deploy with BPA gate, dry-run refresh, find-and-remove-unused, workspace mirroring.
- TE2 migration mapping table and `TE_COMPAT=te2` compat layer.
- CI/CD integration patterns for GitHub Actions and Azure DevOps Pipelines.
- Output formats (text / JSON / CSV / TMSL / TMDL) and CI annotation formats (vsts/azdo/azure-devops, github/gh).
- Environment variables (`TE_CONFIG`, `TE_DEBUG`, `TE_COMPAT`, `TE_SESSION`, `TE_MACROS_PATH`, `TE_BPA_RULES`, `TE_BPA_CONFIG`, `AZURE_CLIENT_*`).
- Configurable keys via `te config set` (BPA gates, format options, interactive edit mode, telemetry, etc.).
- Speed knobs for batch / demo / CI runs.
- Common `-q` properties cheatsheet for the most-used TOM property names per object type.
- Gotchas covering partition path asymmetries, `MExpression` vs `expression`, BPA JSON output shape, `--output-format` vs `--serialization` confusion, `te connect` session non-persistence, and more.

[0.1.0]: https://github.com/TabularEditor/CLI/releases/tag/skill-v0.1.0
