# Changelog

All notable changes to the `te-cli` skill are documented in this file.

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
