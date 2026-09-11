# Tabular Editor CLI - AI Agent Skill

A drop-in skill that teaches AI coding agents how to use the [Tabular Editor CLI (`te`)](https://github.com/TabularEditor/CLI) - a cross-platform single binary for managing Power BI and Analysis Services semantic models from the terminal on macOS, Linux, and Windows.

> [!IMPORTANT]
> **Preview.** The Tabular Editor CLI itself is in Limited Public Preview, with preview builds stopping after 2026-10-31. No license is required during preview. This skill tracks the preview surface and will evolve alongside the CLI.

## What this is

A skill folder ([`SKILL.md`](./SKILL.md) plus a [`references/`](./references/) directory) packed with the conventions, command reference, common workflows, gotchas, CI/CD patterns, and semantic modeling best practices Claude, GitHub Copilot, and other AI agents need to drive `te` productively. Once installed, the agent answers "how do I deploy this model?" or "add a measure that calculates margin" with idiomatic `te` invocations instead of guessing or hallucinating flags.

The skill covers:

- All `te` commands across all families (save-as, init, deploy, refresh, bpa, validate, query, script, util, etc.)
- Authentication patterns (interactive, SPN with secret or certificate, env-var, managed identity)
- Object path grammar (slash form, DAX form, wildcards, quoting rules)
- The save model: dry run by default, `--save` to persist, and the interactive shell's `--stage` / `--revert`; `te deploy` and `te refresh` print TMSL until `--execute --force`
- TE2 -> new CLI migration mapping
- CI/CD recipes for GitHub Actions and Azure DevOps
- Output formats, the shared findings and changes JSON shapes, exit codes, environment variables, config keys
- Common property names for `-p Name=Value`
- The gotchas that trip up agents in practice

## What's a "skill"?

A skill is a folder with a `SKILL.md` entry point that an agent loads on demand based on the user's prompt. The skill's YAML frontmatter (`name`, `description`, `version`) tells the agent **when** to load it and **what** it covers. The Markdown body teaches the agent **how** to do its job, and larger skills - like this one - bundle extra reference files under `references/` that the agent reads only when needed.

Other agents (Cursor, Aider, generic `AGENTS.md` consumers) don't have an identical "skill" concept but can still benefit from the same content as a custom-instructions file. Installation per agent is covered below.

## Install as a Claude Code plugin (recommended)

This repo is a Claude Code plugin marketplace named `te-cli` with a single plugin, `te-cli-agentic-use`. Installing it pulls in `SKILL.md` and the whole `references/` directory at once:

```bash
claude plugin marketplace add TabularEditor/CLI
claude plugin install te-cli-agentic-use@te-cli
```

Run `/plugin` (or `claude plugin list`) to confirm it is enabled. For directory-based installs (below), copy the entire `skills/te-cli/` folder, not just `SKILL.md`, so the `references/` travel with it.

## Downloading the skill

The skill lives in this folder: `SKILL.md` plus its `references/` subfolder.

1. Clone the [TabularEditor/CLI](https://github.com/TabularEditor/CLI) repository, or download the repository ZIP (**Code > Download ZIP**) and extract it.
2. Copy the whole `skills/te-cli/` folder somewhere convenient, keeping `references/` next to `SKILL.md`.

You'll move this folder to a tool-specific location in the install steps below. To check what changed between versions before grabbing a newer copy, see the [CHANGELOG](./CHANGELOG.md).

## Install for Claude Code

Claude Code loads skills from a named folder under `.claude/skills/`. The `description` field is matched against your prompts, so the skill only loads when relevant - it costs no tokens when you're working on unrelated code.

**Project scope** - the skill loads only inside this project:

1. In your project root, create the folder `.claude/skills/te-cli/`.
2. Copy the contents of the downloaded `te-cli` folder (`SKILL.md` and `references/`) into that folder.

The final path is `<your-project>/.claude/skills/te-cli/SKILL.md`, with `references/` alongside it.

**User scope** - the skill loads in every project for the current user:

1. Create a `te-cli` folder inside your user-level Claude skills directory:
   - **macOS / Linux:** `~/.claude/skills/te-cli/`
   - **Windows:** `%USERPROFILE%\.claude\skills\te-cli\` (typically `C:\Users\<you>\.claude\skills\te-cli\`)
2. Copy the contents of the downloaded `te-cli` folder (`SKILL.md` and `references/`) into that folder.

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
what does `te deploy` do without `--execute`?
```

Claude answers with the documented behavior - it is a dry run that prints the TMSL deployment script to stdout without deploying anything - which confirms the skill is loaded and in use.

## Install for Claude.ai and Claude Desktop

Claude.ai (web and desktop) has a built-in **Skills** feature. Skills require code execution, and you upload them as a ZIP of the skill folder rather than the bare `SKILL.md`.

1. Enable code execution: go to **Settings > Capabilities** and turn on **Code execution and file creation**. On Team and Enterprise plans, an owner enables this in organization settings.
2. Compress the whole downloaded `te-cli` folder (including `references/`) into `te-cli.zip`.
3. Go to **Settings > Capabilities > Skills** (also reachable under **Customize > Skills**).
4. Click **+**, choose **Upload skill**, and select `te-cli.zip`. Claude reads the `SKILL.md` inside and shows a summary of the skill.
5. Toggle the skill on. It loads automatically when you mention `te` or a related concept.

Custom skills you upload are private to your account unless an owner enables organization-wide sharing on Team or Enterprise.

## Install for GitHub Copilot

GitHub Copilot in VS Code supports the Agent Skills open standard natively - the same `SKILL.md` format Claude Code and Codex use. This is the recommended approach because the skill loads only when relevant. For Copilot setups that predate Agent Skills, fall back to the generic `AGENTS.md` install below.

### Agent Skills (VS Code)

Place the skill folder contents (`SKILL.md` and `references/`) in a named folder under a skills directory. The folder name must match the `name` field in the frontmatter, so use `te-cli`, and keep the YAML frontmatter intact.

- **Workspace scope:** `.github/skills/te-cli/SKILL.md` (Copilot also reads `.claude/skills/` and `.agents/skills/`).
- **User scope:** `~/.copilot/skills/te-cli/SKILL.md` (Copilot also reads `~/.claude/skills/` and `~/.agents/skills/`).

Type `/` in Copilot Chat to confirm `te-cli` appears as a slash command, or open the Agent Customizations editor with **Chat: Open Customizations** from the Command Palette.

## Install for OpenAI Codex CLI

Codex CLI loads skills natively from a named folder under `.agents/skills/`, the same directory-based model as Claude Code. Keep the YAML frontmatter - Codex requires the `name` and `description` fields and uses the description to decide when to load the skill.

**Project scope** - the skill loads only inside this project:

1. In your project root, create the folder `.agents/skills/te-cli/`.
2. Copy the contents of the downloaded `te-cli` folder (`SKILL.md` and `references/`) into that folder.

Codex scans upward from your working directory, so a skill committed at the repository root (`$REPO_ROOT/.agents/skills/te-cli/`) is shared across everyone working in the repo.

**Personal scope** - the skill loads in every project for the current user:

1. Create the folder `te-cli` inside your personal Codex skills directory: `~/.agents/skills/te-cli/`.
2. Copy the contents of the downloaded `te-cli` folder (`SKILL.md` and `references/`) into that folder.

Run `/skills` in the Codex CLI or IDE to confirm `te-cli` is listed, and type `$` to mention a skill explicitly.

## Install for generic agents

For tools that follow the [`AGENTS.md` convention](https://agents.md) or accept an arbitrary instructions file - Aider, Continue, custom in-house agents:

1. Download the skill folder.
2. In a copy of `SKILL.md`, remove the YAML frontmatter block at the top (everything between the first and second `---` lines, including those lines).
3. Rename that file to `AGENTS.md` and place it at your project root, or wherever the tool expects its instructions file.
4. Copy the `references/` folder next to your `AGENTS.md` so its relative links keep working.
5. The next agent invocation in that project picks up the instructions.

## Updating

To pull a newer version:

1. Grab the latest `skills/te-cli` folder from GitHub (re-clone, pull, or re-download the repository ZIP).
2. Replace what you previously installed:
   - **Native skills (Claude Code, Codex, Copilot Agent Skills):** replace the whole skill folder contents (`SKILL.md` and `references/`).
   - **Claude.ai / Desktop:** re-zip the `te-cli` folder and re-upload it through the Skills UI.
   - **Instruction-file installs (AGENTS.md):** re-paste the body into `AGENTS.md` and refresh the copied `references/` folder.

See the [CHANGELOG](./CHANGELOG.md) for what changed between versions.

## Issues & feedback

Report skill bugs, gaps, or suggestions in the [Tabular Editor CLI issue tracker](https://github.com/TabularEditor/CLI/issues). Please apply the **skill** label (we add this label to issues that affect the skill rather than the CLI itself) so they're easy to triage.

For CLI behavior questions (not skill-content questions), the same tracker is the right place - just don't apply the `skill` label.

## License

[MIT](./LICENSE). Use, modify, and redistribute as you like. Attribution appreciated but not required.

## Files in this folder

- [`SKILL.md`](./SKILL.md) - the lean skill entry point (when-to-use, critical rules, quickstart, command index, modeling checklist)
- [`references/`](./references/) - command reference, common workflows, gotchas, config/CI/CD/env, TE2 migration, testing, semantic modeling practices, and the pbir/fab tandem guides, loaded on demand
- [`README.md`](./README.md) - this file
- [`CHANGELOG.md`](./CHANGELOG.md) - version history
- [`LICENSE`](./LICENSE) - MIT license

---

For the CLI itself, see https://github.com/TabularEditor/CLI.
