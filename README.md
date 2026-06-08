# Tabular Editor CLI (te)

> [!NOTE]
> 📢 **Read the [announcement blog post](https://tabulareditor.com/blog/introducing-the-tabular-editor-cli-limited-public-preview)** to learn what the CLI is, why we built it, and how to get started.

Public home for **bug reports, feature requests, and discussions** about the [Tabular Editor CLI](https://docs.tabulareditor.com/features/te-cli/te-cli.html) (`te`) — a cross-platform command-line tool for Power BI and Analysis Services semantic models.

> [!IMPORTANT]
> The Tabular Editor CLI is in **Limited Public Preview**. It is offered for evaluation with a Tabular Editor account; no license is required during preview. Commands, flags, and outputs may change before general availability. **The preview build stops functioning after 2026-09-30.** We recommend against using the CLI in production CI/CD pipelines during preview.

**📥 Download:** https://tabulareditor.com/download-tabular-editor-cli (requires account creation)

## 📦 What this repository is

- **[Issues](../../issues)** — report bugs, request features, and track known problems.
- **[Discussions](../../discussions)** — ask questions, share feedback, and swap usage tips with other early adopters.

This repository does **not** host the Tabular Editor CLI source code or binaries. It exists to provide a public place for feedback and issue tracking during preview.

## 🤖 AI agent skill

This repo also ships an **AI agent skill** that teaches Claude, GitHub Copilot, and other AI coding agents how to use `te` productively. It is packaged as a Claude Code plugin and also installs as a plain skill folder for other agents.

Add it with the **`claude` CLI** (installs the whole skill, `references/` included):

```bash
claude plugin marketplace add TabularEditor/CLI
claude plugin install te-cli-agentic-use@te-cli
```

Add it to the **`copilot` CLI** (GitHub Copilot CLI). Copilot has no install command; it auto-discovers skills from a folder, so drop the skill into one it reads:

```bash
git clone --depth 1 https://github.com/TabularEditor/CLI /tmp/te-cli
mkdir -p ~/.copilot/skills && cp -r /tmp/te-cli/skills/te-cli ~/.copilot/skills/te-cli
```

For workspace scope, copy into `.github/skills/te-cli/` instead. Copilot also reads `~/.claude/skills/` and `~/.agents/skills/`, so the same folder placed in one of those works for both tools.

See [`skills/te-cli/`](./skills/te-cli/) for the skill, its `references/`, and per-agent (Claude Code, Copilot, Codex, generic) install instructions.

## 🐛 Before you open an issue

Please take a minute to do these two things — it saves everyone time and helps your report get triaged faster.

1. **Make sure you're on the latest version.** Run `te --version` and compare it with the latest preview release on [tabulareditor.com](https://tabulareditor.com). Many reports turn out to be bugs that have already been fixed. Update and try to reproduce before reporting.
2. **Search existing issues and discussions** for anything similar to what you're seeing. If you find a match, please add a comment (with your repro, version, and environment) to the existing thread rather than opening a duplicate. This keeps conversation in one place and bumps priority based on reaction counts.

If you can't find a match, open a new issue and include:

- `te --version`
- OS and architecture (e.g., Windows 11 x64, macOS 14 ARM64, Ubuntu 22.04 x64)
- The exact command you ran and the output (redact any secrets first)
- What you expected to happen vs. what actually happened

## 🔒 Security

If you believe you have discovered a security vulnerability, please **do not open a public issue or discussion**.

See our [Security Policy](SECURITY.md) for responsible disclosure instructions and contact details.

You can also visit our Trust Center: https://trust.tabulareditor.com/

## 📚 Documentation

Full documentation lives at <https://docs.tabulareditor.com/features/te-cli/te-cli.html>.

Quick links:

- [Installation and Setup](https://docs.tabulareditor.com/features/te-cli/te-cli-install.html)
- [Authentication and Connections](https://docs.tabulareditor.com/features/te-cli/te-cli-auth.html)
- [Command Reference](https://docs.tabulareditor.com/features/te-cli/te-cli-commands.html)
- [CI/CD Integration](https://docs.tabulareditor.com/features/te-cli/te-cli-cicd.html)
- [Migrating from the TE2 CLI](https://docs.tabulareditor.com/features/te-cli/te-cli-migrate.html)
