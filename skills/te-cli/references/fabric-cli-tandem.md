# `te` + `fab` tandem workflows

`te` (Tabular Editor CLI) and `fab` (Fabric CLI) split cleanly along a single seam. `te` owns the semantic model itself: it parses TMDL/BIM locally, edits objects (`set`, `add`, `remove`, `move`, `script`), runs validation, BPA, and VertiPaq, generates TMSL, deploys over XMLA, and runs DAX queries, refreshes, and tests against a live XMLA endpoint. `fab` owns everything around the model in the service: discovering workspaces and items, resolving display names and GUIDs, exporting and importing item definitions over the Fabric REST API, managing permissions and capacities, driving deployment pipelines, and triggering refreshes via the Power BI REST API. Neither tool crosses into the other's half: `fab` has no DAX parser, no BPA engine, and no XMLA/TOM layer, so it cannot validate or BPA-check a model; `te` has no Fabric REST client, so it cannot enumerate workspaces, resolve item GUIDs, or trigger a REST refresh. The two share Azure AD identity but cache credentials independently, so authenticate both at the start of any tandem session.

Before each workflow: `fab auth status` and `te auth status`. If either is unauthenticated, ask the user to run `fab auth login` / `te auth login`. During preview, run `te <command> --help` and `fab <command> --help` the first time a command is composed; both surfaces are still moving.

## 1. Export, edit, deploy back (the common round-trip)

Pull a production model to local TMDL with `fab`, edit and quality-gate with `te`, push it back over XMLA with `te deploy --execute`.

```bash
# 0. Confirm the item path is real before exporting
fab exists "Production.Workspace/Sales.SemanticModel"

# 1. Pre-create the output dir (fab export will not create parents)
mkdir -p ./sales-export

# 2. Download the model definition as a TMDL folder
fab export "Production.Workspace/Sales.SemanticModel" -o ./sales-export -f
#    -> ./sales-export/Sales.SemanticModel/definition/  holds model.tmdl, tables/, etc.

# 3. Confirm te parses the exported TMDL (model root properties); settle on the path that loads
te get . -m ./sales-export/Sales.SemanticModel/definition

# 4. Baseline gates BEFORE editing (separate pre-existing issues from yours)
te validate -m ./sales-export/Sales.SemanticModel/definition --errors-only
te bpa run --fail-on error -m ./sales-export/Sales.SemanticModel/definition

# 5. Edit (each mutation prints a before/after diff and needs --save to persist)
te set "_Measures/Revenue" -p FormatString="#,0.00" -m ./sales-export/Sales.SemanticModel/definition --save
te set "_Measures/Revenue" --format Expression -m ./sales-export/Sales.SemanticModel/definition --save          # format one expression
te script --inline "Model.AllMeasures.FormatDax();" -m ./sales-export/Sales.SemanticModel/definition --save     # format every measure

# 6. Re-gate: no new errors or violations introduced
te validate -m ./sales-export/Sales.SemanticModel/definition --errors-only
te bpa run --fail-on error -m ./sales-export/Sales.SemanticModel/definition

# 7. Deploy back over XMLA (BPA gate runs by default; dry run prints the TMSL until --execute; --force skips the prompt)
te deploy -m ./sales-export/Sales.SemanticModel/definition --target-server "Production" --target-database "Sales" --execute --force
```

Per-step purpose: `fab exists` fails fast on a wrong name; `mkdir -p` avoids `[InvalidPath]`; `te get .` confirms the parse and pins the path; the baseline gates separate pre-existing breakage from edits being introduced; `--save` persists each mutation (a dry run that prints the diff otherwise); the second gate is the quality bar before deploy; `te deploy --execute` writes the definition back through XMLA, re-running BPA on the way out.

Notes that bite:
- `fab export` nests TMDL under `<Model>.SemanticModel/definition/`. `te -m` accepts both a TMDL folder and a `.SemanticModel` container; run `te get .` once to confirm which path the export loads from, then use that exact path on every later `te` call.
- `te connect` state is per shell process (a session file keyed to the terminal). Tool harnesses that spawn a fresh shell per call do not share it, so pass `-m` (or `-s`/`-d` for remote) every time, or set `TE_SESSION=<name>` before the first call to share one named session across shells.
- The BPA gate is ON by default for `te deploy` (config `bpa.onDeploy`). If pre-existing violations block the deploy, use `--skip-bpa` once and log it, or `--fix-bpa` to auto-remediate; do not silently bypass.
- `te deploy` overwrites by default. Add `--create-only` to refuse if the model already exists (it errors if it does), to prevent clobbering.
- `-s`/`-d` on `te deploy` mean the model SOURCE, exactly as on every other command; the destination is always `--target-server`/`--target-database`. Remote-to-remote copy: `te deploy -s "Dev" -d "Sales" --target-server "Test" --target-database "Sales" --execute`.

### Variant: import back with `fab` instead of `te deploy`

Use this when XMLA write is blocked (Pro/PPU without XMLA, or tenant policy). `fab import` goes through the Fabric item-definition REST API and works on any SKU. It does NOT run BPA or validation, so the `te` gate before it is the only safety net.

```bash
te validate -m ./sales-export/Sales.SemanticModel/definition --errors-only
te bpa run --fail-on error -m ./sales-export/Sales.SemanticModel/definition
fab import "Production.Workspace/Sales.SemanticModel" -i ./sales-export/Sales.SemanticModel -f
```

`fab import` overwrites the whole item definition (no partial update) and does not refresh data. Pass the `.SemanticModel` folder (the one containing `.platform` and `definition/`), not the inner `definition/`. Trigger a refresh afterward if needed (workflow 5).

### Variant: `te`-only pull (no `fab`)

`te save-as` (alias `save`) can read a remote model and write it to local disk, so the round-trip does not strictly require `fab export` when XMLA is available:

```bash
te save-as -s "Production" -d "Sales" -o ./sales-export --serialization tmdl
te save-as -s "Production" -d "Sales" -o ./sales-export --supporting-files      # wrap in Sales.SemanticModel/ with .platform + definition.pbism
```

Use `fab export` when XMLA is blocked; use `te save-as` when an XMLA connection already exists. `--supporting-files` produces the `.SemanticModel` container layout, so the result can go back through `fab import` as well as `te deploy`. Pair with `--skip-validation` for the fastest passthrough when only the bytes are wanted.

## 2. Discover with `fab`, connect `te` by display name

When the exact workspace or model name is unknown or its casing is uncertain, resolve it with `fab` first, then feed the canonical display name into `te -s`/`-d`.

```bash
fab ls                                              # list visible workspaces
fab find 'sales' -P type=SemanticModel -l           # substring search; -l adds id + workspace_id
fab exists "Sales Analytics.Workspace/Sales Model.SemanticModel"   # confirm the exact path

te get . -s "Sales Analytics" -d "Sales Model"      # te takes the bare display name (no .Workspace suffix)
te bpa run --fail-on error -s "Sales Analytics" -d "Sales Model"    # gate the live model, no local export
```

Boundary: `fab` resolves and confirms names against the Fabric REST API and OneLake catalog; `te` connects to the XMLA endpoint using the display name directly. Path shapes: `fab` uses dot-extension syntax (`"Name.Workspace"`, `"Model.SemanticModel"`); `te -s` takes the bare workspace display name and `te -d` the bare model name, so strip the suffixes when handing off. `te -s` also accepts the `Name.Workspace` form and a full `powerbi://` URL, and one of those is required when the workspace display name itself contains a dot (a bare dotted name is read as an Analysis Services server, with a warning). Quote any name with spaces in both CLIs.

## 3. Extract GUIDs with `fab`, refresh after a `te` change

The Power BI REST API needs raw GUIDs that only `fab` can resolve from display names. `te` validates the model; `fab` triggers the refresh.

```bash
te validate -s "Production" -d "Sales Model" --errors-only      # don't refresh a broken model

WS_ID=$(fab get "Production.Workspace" -q "id" | tr -d '"')
MODEL_ID=$(fab get "Production.Workspace/Sales Model.SemanticModel" -q "id" | tr -d '"')

fab api -A powerbi "groups/$WS_ID/datasets/$MODEL_ID/refreshes" -X post -i '{"type":"Full"}'
fab api -A powerbi "groups/$WS_ID/datasets/$MODEL_ID/refreshes?\$top=1" -q "value[0].{status:status,started:startTime}"
```

`fab get -q "id"` returns a quoted JSON string; always pipe through `tr -d '"'` or the GUID carries literal quotes that break URL construction. `te refresh --type full -s ws -d model --execute --force` also triggers a refresh over XMLA and blocks until it completes (without `--execute` it only prints the refresh TMSL; `--execute` alone asks for confirmation, so unattended runs need `--force`; `--output-format json` adds per-table `progress` and, with `vertipaqOnRefresh` enabled, `vertipaq` arrays); use `fab api` when the GUIDs are already in scope or when XMLA refresh is not available. The REST refresh is async; the `?$top=1` check is a sanity probe, not a completion wait.

## 4. Promote dev to test/prod with quality gates

Export from dev, diff against the target, gate, then deploy. The local round-trip is deliberate: `fab cp` can copy workspace-to-workspace faster but skips every `te` gate.

```bash
mkdir -p ./promote ./prod-export
fab export "Dev.Workspace/Sales.SemanticModel" -o ./promote -f

# Structural diff against the live prod model: te diff compares two local paths, so pull prod to disk first
te save-as -s "Production" -d "Sales" -o ./prod-export --skip-validation
te diff ./promote/Sales.SemanticModel/definition ./prod-export --output-format json   # exit 0 identical, 1 differ, 2 error

te validate -m ./promote/Sales.SemanticModel/definition --errors-only
te bpa run --fail-on error -m ./promote/Sales.SemanticModel/definition

# Deploy, including RLS roles and members
te deploy -m ./promote/Sales.SemanticModel/definition \
  --target-server "Production" --target-database "Sales" \
  --deploy-roles --deploy-role-members --execute --force

fab exists "Production.Workspace/Sales.SemanticModel"           # confirm it landed
```

`--deploy-roles` / `--deploy-role-members` are opt-in; omit them when RLS is managed independently in prod. `te diff` takes two local model paths (TMDL folder, `.bim`, `database.json` folder, or `.SemanticModel` folder) and never a `-s`/`-d` side; if XMLA is blocked, `fab export` prod separately instead of `te save-as` and diff the two folders the same way. The JSON `changes[]` array names each changed object by `objectPath` and `objectType`, with `changeKind` (`created`, `deleted`, `modified`, `moved`) and `properties[]` of `{property, before, after}`; the same shape every mutating `te` command prints.

### Variant: governed promotion via deployment pipeline

`te` provides the quality gate and an optional TMSL audit artifact; `fab` drives the Fabric deployment pipeline (which preserves item IDs across stages, so thin reports do not need rebinding).

```bash
te bpa run --fail-on error --ci vsts --non-interactive -m ./dev-export/Sales.SemanticModel/definition

# Optional: emit the TMSL a direct XMLA deploy WOULD run, as an audit artifact (dry run is the default; nothing executes)
te deploy -m ./dev-export/Sales.SemanticModel/definition --target-server "Dev" --target-database "Sales" > ./audit/deploy.tmsl

PIPELINE_ID=$(fab api "deploymentPipelines" -q "value[?displayName=='Sales Pipeline'].id | [0]" | tr -d '"')
DEV_STAGE=$(fab api "deploymentPipelines/$PIPELINE_ID/stages" -q "value[?order==\`0\`].id | [0]" | tr -d '"')
TEST_STAGE=$(fab api "deploymentPipelines/$PIPELINE_ID/stages" -q "value[?order==\`1\`].id | [0]" | tr -d '"')

# Promote; capture the LRO id from the response header
fab api -X post "deploymentPipelines/$PIPELINE_ID/deploy" \
  -i "{\"sourceStageId\":\"$DEV_STAGE\",\"targetStageId\":\"$TEST_STAGE\",\"note\":\"BPA-gated\"}" \
  --show_headers

# Poll the LRO to completion
until s=$(fab api "operations/$OPERATION_ID" -q "status" | tr -d '"'); [ "$s" = "Succeeded" ] || [ "$s" = "Failed" ]; do sleep 30; done
```

The TMSL audit artifact describes a direct XMLA deploy from the local source, not what the pipeline promotion will do; treat it as a reference, not a contract. The dry run still connects read-only to the target to decide between create and alter, so the target must be reachable; informational output goes to stderr, so the redirected file holds only the script. Capture `OPERATION_ID` from the `x-ms-operation-id` header immediately; it is not reliably retrievable later. Pipelines copy definitions only; refresh separately (workflow 5).

## 5. Post-deploy: refresh, then DAX regression tests

A pipeline or XMLA deploy moves definitions, not data. Refresh with `fab`, wait, then run the `te` test suite against the live model.

```bash
WS_ID=$(fab get "Test.Workspace" -q "id" | tr -d '"')
MODEL_ID=$(fab get "Test.Workspace/Sales.SemanticModel" -q "id" | tr -d '"')
fab api -A powerbi "groups/$WS_ID/datasets/$MODEL_ID/refreshes" -X post -i '{"type":"Full"}'

# Wait for the async refresh before asserting (the REST refresh has no blocking wait; poll in a loop)
until s=$(fab api -A powerbi "groups/$WS_ID/datasets/$MODEL_ID/refreshes?\$top=1" -q "value[0].status" | tr -d '"'); [ "$s" = "Completed" ] || [ "$s" = "Failed" ]; do sleep 30; done

te test run --suite ./.te-tests/ --ci vsts --trx test-results.trx --non-interactive -s "Test Workspace" -d "Sales"
```

Do not run `te test run` before the refresh finishes; stale or empty data causes false failures. `te refresh --type full --execute --force -s "Test Workspace" -d "Sales"` is the XMLA alternative and returns only when the refresh is done, so it needs no polling loop. `te test run` needs a `.te-tests/` suite; scaffold one first with `te test init --example`. In CI, set `AZURE_CLIENT_ID`/`AZURE_CLIENT_SECRET`/`AZURE_TENANT_ID` and pass `--auth env`. Under `--output-format json`, `te test run` emits the same findings envelope as `te validate` and `te bpa run` (`summary` plus `findings[]`, with `TEST_FAIL` / `TEST_ERROR` codes) alongside `suites` and `testSummary`.

## 6. Governance: enumerate with `fab`, audit with `te`

Discovery only comes from `fab`; `te` has no workspace or item listing surface. Pair them for tenant-wide BPA audits, VertiPaq profiling, and unused-column sweeps.

```bash
# Enumerate semantic models across all visible workspaces
fab find '' -P type=SemanticModel -l --output_format json > /tmp/models.json
jq -r '.[] | "\(.workspace)/\(.name)"' /tmp/models.json

# Per model: export, then analyze locally
mkdir -p /tmp/audit
fab export "Production.Workspace/Sales.SemanticModel" -o /tmp/audit -f

te bpa run -m /tmp/audit/Sales.SemanticModel/definition --rules ./BPARules.json --fail-on error --ci github --output-format json
te deps --unused --hidden -m /tmp/audit/Sales.SemanticModel/definition --output-format json
te vertipaq -m /tmp/audit/Sales.SemanticModel/definition --columns --detail --top 20
```

For governance fields `fab find` does not expose (last refresh, storage mode, owner, capacity SKU), use the fabric-cli skill's `scripts/search_across_workspaces.py` (note its filter is `--type Model`, not `SemanticModel`). VertiPaq stats from an offline TMDL export give column/relationship structure but not live row counts or cardinality; for those, point `te vertipaq` at a live `-s`/`-d` XMLA endpoint with `--stats`. `te deps --unused` already excludes columns used in relationships, hierarchy levels, sort-by, variations, AlternateOf bases, and calendar time roles; what it cannot see is report-layer usage (a visual or filter bound to the column), so inspect with `te get` and check the reports before removing. `te bpa run --rules` replaces the user-rule layer for that run; the built-in rule set (the same rules as Tabular Editor 3 desktop) still applies unless `--no-defaults` or `bpa.builtInRules=false`.

## Boundaries and gotchas

- **Path layout after `fab export`.** TMDL lands in `<Model>.SemanticModel/definition/`. `te validate`/`bpa run`/`vertipaq`/`deploy` take the model with `--model`, never as a trailing positional path (a stray path is rejected as an unrecognized argument); `fab import` wants the `.SemanticModel` folder (with `.platform`). Confirm the exact `te` target with `te get .` first.
- **Return path: XMLA vs REST.** `te deploy --execute` writes via XMLA and runs BPA inline but needs XMLA write (Premium/Fabric capacity, PPU with XMLA, or Trial). `fab import` writes via the Fabric REST API on any SKU but runs no gate. Pick by SKU and by whether the inline BPA gate is wanted.
- **Dry run is the default on `te deploy` and `te refresh`.** Without `--execute` both connect read-only and print the TMSL they would send; a pipeline that forgets `--execute` succeeds and changes nothing, and one that forgets `--force` stops at the confirmation prompt with an error. Redirect the dry-run output to capture the script (`te deploy ... > deploy.tmsl`). A deploy the server accepts with object errors exits non-zero.
- **`te diff` is local-to-local.** Two positional model paths, no remote side; pull the remote model with `te save-as -o` (XMLA) or `fab export` (REST) first. Exit codes: `0` identical, `1` differ, `2` error. Branch on all three (`case $? in 0) ...;; 1) ...;; *) ...;; esac`).
- **One findings JSON across `te validate`, `te bpa run`, `te test run`, `te query`.** Each emits a single document with `command`, `durationMs`, `summary` and a flat `findings[]` (`severity`, `source`, `code`, `message`, `object`, `objectType`, `objectPath`, `fixable`). `te bpa run --fix --output-format json` is also one document, with the fix outcome under a `fix` key; a single-document parser is the right choice.
- **CI annotation format names.** `--ci` takes `vsts` (Azure DevOps) or `github`; `azdo` is not accepted. Info-severity findings are emitted as notices, not warnings.
- **Quoted GUIDs from `fab get`.** `fab get -q "id"` returns a quoted string; always `| tr -d '"'` before interpolating into an API path.
- **`fab` REST refresh path shape.** `fab api -A powerbi` uses the Power BI `groups/<ws-id>/datasets/<model-id>` shape; the Fabric REST API uses `workspaces/<ws-id>/semanticModels/<model-id>`. Same GUIDs, different URL.
- **No native pipe between the CLIs.** `fab find` / `search_across_workspaces.py` produce paths; a shell intermediary (`jq`, `while read`) feeds them into a `te` loop. `te` has no multi-model loop and no service-discovery command of its own.
- **Always `mkdir -p` before `fab export`** (it does not create parents), and **always `-f`** on `fab export`/`import` (skips the sensitivity-label / overwrite prompt). If sensitivity labels or DLP policies are in play, confirm with the user before exporting; `-f` strips the label on export.
- **CI flags on `te`.** Pass `--execute --force --non-interactive` on `te deploy` and `--execute --non-interactive` on `te refresh`; without `--force` the deploy confirmation prompt defaults to `n` and is an error in non-interactive mode. Use `--auth env` with `AZURE_CLIENT_*` (valid `--auth` values: `auto`, `interactive`, `spn`, `env`, `managed-identity`); never put a secret on the command line.
- **Keeping a local copy and a workspace in step.** `te connect <ws> <model> -w ./local-copy` (or `te connect ./local-copy -w <ws> <model>`) mirrors every later `--save` to both sides, local first; it is an alternative to the export/deploy loop above for an interactive editing session, not for CI. Clear it with `te connect --clear`.
- **Thin-report rebinding is a `fab` job.** After a `fab import` or `fab cp` of a thin report to a new workspace, rebind with `fab set "<ws>/<Report>.Report" -q semanticModelId -i "<target-model-id>"`. Deployment pipelines preserve IDs and skip this; manual import/copy does not.
