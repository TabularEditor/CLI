# Tabular Editor CLI - AI Agent Skill

A drop-in skill that teaches AI coding agents how to use the [Tabular Editor CLI (`te`)](https://github.com/TabularEditor/CLI) - a cross-platform single binary for managing Power BI and Analysis Services semantic models from the terminal on macOS, Linux, and Windows.

> [!IMPORTANT]
> **Preview.** The Tabular Editor CLI itself is in Limited Public Preview, with preview builds stopping after 2026-09-30. This skill tracks the preview surface and will evolve alongside the CLI.

## What this is

A skill folder ([`SKILL.md`](./SKILL.md) plus a [`references/`](./references/) directory) packed with the conventions, command reference, common workflows, gotchas, CI/CD patterns, and semantic modeling best practices Claude, GitHub CoPilot (and other AI agents) need to drive `te` productively. Once installed, the agent answers "how do I deploy this model?" or "add a measure that calculates margin" with idiomatic `te` invocations instead of guessing or hallucinating flags.

The skill covers:

- All `te` commands across all families (load, save, init, deploy, refresh, bpa, validate, query, script, format, etc.)
- Authentication patterns (interactive, SPN with secret or certificate, env-var, managed identity)
- Object path grammar (slash form, DAX form, wildcards)
- Staging model (`--save` / `--stage` / `--revert`)
- TE2 -> new CLI migration mapping
- CI/CD recipes for GitHub Actions and Azure DevOps
- Output formats, exit codes, environment variables, config keys
- Common `-q` properties cheatsheet
- The gotchas that trip up agents in practice

## What's a "skill"?

In the Anthropic ecosystem (Claude Code / Claude.ai / Claude Desktop), a **skill** is a Markdown file that an agent loads on demand based on the user's prompt. The skill's YAML frontmatter (`name`, `description`, `version`) tells the agent **when** to load it and **what** it covers. The plain-Markdown body teaches the agent **how** to do its job.

Other agents (GitHub Copilot, Cursor, Aider, generic `AGENTS.md` consumers) don't have an identical "skill" concept but can still benefit from the same content as a custom-instructions file. Installation per agent is covered below.

## Install as a Claude Code plugin (recommended)

This repo is a Claude Code plugin marketplace named `te-cli` with a single plugin, `te-cli-agentic-use`. Installing it pulls in `SKILL.md` and the whole `references/` directory at once:

```bash
claude plugin marketplace add TabularEditor/CLI
claude plugin install te-cli-agentic-use@te-cli
```

Run `/plugin` (or `claude plugin list`) to confirm it is enabled. For directory-based installs (below), copy the entire `skills/te-cli/` folder, not just `SKILL.md`, so the `references/` travel with it.

## Downloading the skill file

The skill ships as a single file: [**`SKILL.md`**](./SKILL.md).

To download:

1. Open [`SKILL.md`](./SKILL.md) on GitHub.
2. Click the **Download raw file** button (top-right of the file viewer, next to the **Raw** / **Copy** buttons).
3. Save the file somewhere convenient.

You'll move this file to a tool-specific location in the install steps below. To check what changed between versions before downloading a newer copy, see the [CHANGELOG](./CHANGELOG.md).

## Install for Claude Code

Claude Code loads skills from a named folder under `.claude/skills/`. The `description` field is matched against your prompts, so the skill only loads when relevant - it costs no tokens when you're working on unrelated code.

**Project scope** - the skill loads only inside this project:

1. In your project root, create the folder `.claude/skills/te-cli/`.
2. Place the downloaded `SKILL.md` inside that folder.

The final path is `<your-project>/.claude/skills/te-cli/SKILL.md`.

**User scope** - the skill loads in every project for the current user:

1. Create a `te-cli` folder inside your user-level Claude skills directory:
   - **macOS / Linux:** `~/.claude/skills/te-cli/`
   - **Windows:** `%USERPROFILE%\.claude\skills\te-cli\` (typically `C:\Users\<you>\.claude\skills\te-cli\`)
2. Place the downloaded `SKILL.md` inside that folder.

> [!NOTE]
> Claude Code watches skill directories and picks up new or edited skills within the current session - no restart needed. The exception is creating a `.claude/skills/` directory that did not exist when the session started: restart Claude Code once so it begins watching the new directory.

### Verify it loaded

Inside a Claude Code session, run:

```
/skills
```

You should see `te-cli` in the list. If it's missing, confirm the file path and that the file starts with `---` and has `name: te-cli` on the second line, then restart Claude Code.

For a functional smoke test, ask:

```
what does `te deploy --xmla` do?
```

Claude answers with the documented behavior - it generates a TMSL/XMLA script to stdout instead of deploying - which confirms the skill is loaded and in use.

## Install for GitHub Copilot

GitHub Copilot in VS Code supports the Agent Skills open standard natively - the same `SKILL.md` format Claude Code and Codex use. This is the recommended approach because the skill loads only when relevant. For Copilot setups that predate Agent Skills, fall back to the always-on custom-instructions file below.

### Agent Skills (VS Code)

Place the skill in a named folder under a skills directory. The folder name must match the `name` field in the frontmatter, so use `te-cli`, and keep the YAML frontmatter intact.

- **Workspace scope:** `.github/skills/te-cli/SKILL.md` (Copilot also reads `.claude/skills/` and `.agents/skills/`).
- **User scope:** `~/.copilot/skills/te-cli/SKILL.md` (Copilot also reads `~/.claude/skills/` and `~/.agents/skills/`).

Type `/` in Copilot Chat to confirm `te-cli` appears as a slash command, or open the Agent Customizations editor with **Chat: Open Customizations** from the Command Palette.


## Install for OpenAI Codex CLI

Codex CLI loads skills natively from a named folder under `.agents/skills/`, the same directory-based model as Claude Code. Keep the YAML frontmatter - Codex requires the `name` and `description` fields and uses the description to decide when to load the skill.

**Project scope** - the skill loads only inside this project:

1. In your project root, create the folder `.agents/skills/te-cli/`.
2. Place the downloaded `SKILL.md` inside that folder.

Codex scans upward from your working directory, so a skill committed at the repository root (`$REPO_ROOT/.agents/skills/te-cli/`) is shared across everyone working in the repo.

**Personal scope** - the skill loads in every project for the current user:

1. Create the folder `te-cli` inside your personal Codex skills directory: `~/.agents/skills/te-cli/`.
2. Place the downloaded `SKILL.md` inside that folder.

Run `/skills` in the Codex CLI or IDE to confirm `te-cli` is listed, and type `$` to mention a skill explicitly.

## Install for generic agents

For tools that follow the [`AGENTS.md` convention](https://agents.md) or accept an arbitrary instructions file - Aider, Continue, custom in-house agents:

1. Download `SKILL.md`.
2. Remove the YAML frontmatter block at the top (everything between the first and second `---` lines, including those lines).
3. Rename the file to `AGENTS.md` and place it at your project root, or wherever the tool expects its instructions file.
4. The next agent invocation in that project picks up the instructions.

## Updating

To pull a newer version:

1. Visit [`SKILL.md`](./SKILL.md) on GitHub.
2. Use the **Download raw file** button to grab the latest copy.
3. Replace the `SKILL.md` you previously placed in your skills folder (Claude Code) with the new copy, or re-upload it via the Skills UI (Claude.ai / Desktop), or re-paste the body into `.github/copilot-instructions.md` (Copilot).

See the [CHANGELOG](./CHANGELOG.md) for what changed between versions.

## Issues & feedback

Report skill bugs, gaps, or suggestions in the [Tabular Editor CLI issue tracker](https://github.com/TabularEditor/CLI/issues). Please apply the **skill** label (we add this label to issues that affect the skill rather than the CLI itself) so they're easy to triage.

For CLI behavior questions (not skill-content questions), the same tracker is the right place - just don't apply the `skill` label.

## License

[MIT](./LICENSE). Free to use, modify, and redistribute. Attribution appreciated but not required.

## Files in this folder

- [`SKILL.md`](./SKILL.md) - the lean skill entry point (when-to-use, critical rules, quickstart, command index, modeling checklist)
- [`references/`](./references/) - command reference, common workflows, gotchas, config/CI/CD/env, TE2 migration, and semantic modeling practices, loaded on demand
- [`README.md`](./README.md) - this file
- [`CHANGELOG.md`](./CHANGELOG.md) - version history
- [`LICENSE`](./LICENSE) - MIT license

---

For the CLI itself, see https://github.com/TabularEditor/CLI.
