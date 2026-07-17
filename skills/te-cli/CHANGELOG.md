# Changelog

All notable changes to the `te-cli` skill are documented in this file.

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
