# CLAUDE.md

Guidance for AI assistants (Claude Code and others) working in this repository.

## What this repository is

This is the **public `anthropics/claude-code` repository**. It is **not** the source code of the Claude Code CLI — that product is distributed as the `@anthropic-ai/claude-code` npm package and is closed source. Do not look for the CLI's TypeScript/runtime implementation here; it does not exist in this repo.

Instead, this repository serves three purposes:

1. **Public issue tracker** for the Claude Code product, plus the GitHub Actions automation that triages, dedupes, and manages the lifecycle of those issues.
2. **Plugin marketplace** — a curated set of official example plugins (commands, agents, skills, hooks) under `plugins/`, published via `.claude-plugin/marketplace.json`.
3. **Examples and reference material** — devcontainer setup, hooks, settings, MDM deployment, and gateway (AWS/GCP) deployment assets under `examples/`.

There is **no application to build or run** here. There is no `package.json`, no bundler, and no test suite for a shipped product. The only executable code is a handful of automation scripts (TypeScript run with Bun, plus shell scripts) that back the GitHub workflows.

## Repository layout

```
.
├── README.md                 # Product overview + install instructions
├── CHANGELOG.md              # Release notes for the CLI (large, auto-updated)
├── SECURITY.md               # Security policy / vulnerability reporting
├── feed.xml                  # RSS feed of releases (auto-updated with CHANGELOG)
├── demo.gif                  # README demo animation
│
├── .claude/                  # This repo's own Claude Code config
│   └── commands/             # Repo-local slash commands (dedupe, triage-issue, commit-push-pr)
├── .claude-plugin/
│   └── marketplace.json      # Plugin marketplace manifest (lists all plugins/)
├── .devcontainer/            # Dev container definition for Claude Code
├── .github/
│   ├── ISSUE_TEMPLATE/       # Issue forms (bug, feature, docs, model behavior)
│   └── workflows/            # Issue-automation workflows (see below)
├── .vscode/                  # Editor settings
│
├── plugins/                  # Official example plugins (the marketplace contents)
│   ├── README.md             # Table of all plugins + plugin-structure docs
│   └── <plugin-name>/        # One directory per plugin
│
├── examples/
│   ├── hooks/                # Example hook scripts (e.g. bash command validator)
│   ├── settings/             # Example settings.json (strict / lax / bash-sandbox)
│   ├── mdm/                  # MDM deployment (macos, windows)
│   └── gateway/              # LLM-gateway deployment examples (aws, gcp)
│
├── scripts/                  # TypeScript (Bun) + shell automation backing workflows
│   └── gh.sh                 # Hardened wrapper around the gh CLI (allowlisted subcommands)
└── Script/                   # PowerShell helper (run_devcontainer_claude_code.ps1)
```

## Plugins

The `plugins/` directory is the heart of this repo's editable content. Each plugin is a self-contained directory registered in `.claude-plugin/marketplace.json`.

### Standard plugin structure

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json     # REQUIRED: metadata (name, version, description, author)
├── commands/           # optional: slash commands (*.md with frontmatter)
├── agents/             # optional: subagent definitions (*.md)
├── skills/             # optional: Agent Skills (skill dir with SKILL.md)
├── hooks/              # optional: event handlers (hooks.json + handler scripts)
├── .mcp.json           # optional: MCP server configuration
└── README.md           # plugin documentation
```

### Current plugins

| Plugin | Kind of content |
|--------|-----------------|
| `agent-sdk-dev` | command + agents (Agent SDK app scaffolding & verification) |
| `claude-opus-4-5-migration` | skill (model-string / prompt migration) |
| `code-review` | command + parallel review agents |
| `commit-commands` | commands (`/commit`, `/commit-push-pr`, `/clean_gone`) |
| `explanatory-output-style` | SessionStart hook |
| `feature-dev` | command + agents (7-phase feature workflow) |
| `frontend-design` | skill (auto-invoked for frontend work) |
| `hookify` | commands + agent + skill (author hooks from conversation) |
| `learning-output-style` | SessionStart hook |
| `plugin-dev` | command + agents + skills (build plugins) |
| `pr-review-toolkit` | command + specialized review agents |
| `ralph-wiggum` | commands + Stop hook (self-referential loops) |
| `security-guidance` | PreToolUse hook (security pattern warnings) |

Keep `plugins/README.md` (the human-facing table) and `.claude-plugin/marketplace.json` (the machine-readable manifest) in sync whenever you add, remove, or rename a plugin.

### Conventions when editing plugins

- **`plugin.json` is required** and must have at least `name`, `version`, `description`, `author`. The `name` must match the marketplace entry and the directory name.
- **Bump the `version`** in both `plugin.json` and the marketplace entry when you make a meaningful change to a plugin.
- **Commands** are Markdown files with YAML frontmatter (`description`, optional `allowed-tools`). See `.claude/commands/commit-push-pr.md` for the shape (frontmatter + `## Context` with `!`backtick`` command interpolation + `## Your task`).
- **Agents** are Markdown files with frontmatter describing the subagent; **skills** live in a `skills/<name>/SKILL.md` layout.
- Every plugin should ship a `README.md` documenting its commands/agents/skills and usage.
- Use `.claude-plugin/marketplace.json`'s `$schema` and each plugin's structure to validate; the `plugin-dev` plugin contains skills and a `plugin-validator` agent for this purpose.

## GitHub Actions automation

The `.github/workflows/` directory automates issue management. Most workflows authenticate to the Claude API via **Workload Identity Federation** (OIDC token exchange, `id-token: write`) rather than a static API key.

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `claude.yml` | `@claude` mention in issue/PR/comment/review | Runs `claude-code-action` to respond |
| `claude-issue-triage.yml` | issue opened / comment | Auto-labels and triages issues |
| `claude-dedupe-issues.yml` | issue opened / manual | Detects duplicate issues with Claude |
| `auto-close-duplicates.yml` | daily cron / manual | Closes confirmed duplicates |
| `backfill-duplicate-comments.yml` | manual | Backfills dedupe comments |
| `issue-lifecycle-comment.yml` | scheduled | Posts lifecycle status comments |
| `remove-autoclose-label.yml` | label / activity | Removes the auto-close label on activity |
| `lock-closed-issues.yml` | scheduled | Locks long-closed issues |
| `sweep.yml` | cron (10:00, 22:00 UTC) | Periodic issue sweep |
| `issue-opened-dispatch.yml` | issue opened | Dispatches an event to an internal repo |
| `log-issue-events.yml` | issue events | Logging |
| `non-write-users-check.yml` | — | Guards actions from non-write users |

Backing logic lives in `scripts/` as TypeScript files run with **Bun** (`sweep.ts`, `auto-close-duplicates.ts`, `issue-lifecycle.ts`, `lifecycle-comment.ts`, `backfill-duplicate-comments.ts`) and shell scripts (`gh.sh`, `comment-on-duplicates.sh`, `edit-issue-labels.sh`). GitHub Actions are pinned to commit SHAs for supply-chain safety — preserve SHA pinning when editing workflows.

`scripts/gh.sh` is a **hardened wrapper** around the `gh` CLI that only permits an allowlist of read-oriented subcommands (`issue view`, `issue list`, `search issues`, `label list`) and flags, scoped to the current repo. Prefer it over raw `gh` in automation.

## Repo-local slash commands (`.claude/commands/`)

This repo configures its own Claude Code commands, distinct from the plugins:

- `/dedupe` — find duplicate GitHub issues
- `/triage-issue` — triage an issue by analyzing and applying labels
- `/commit-push-pr` — commit, push, and open a PR in a single message

## Conventions & guidance for changes

- **No build step.** There is nothing to compile or bundle. Do not add a `package.json` or build tooling unless explicitly asked.
- **Don't hand-edit `CHANGELOG.md` or `feed.xml`.** They are updated by automation (the many `chore: Update CHANGELOG.md and feed.xml` commits). Leave them alone unless the task is specifically about them.
- **Markdown is the primary format.** Plugins, commands, agents, skills, and docs are all Markdown; keep frontmatter valid and consistent with existing files.
- **Match existing style.** Mirror the structure and tone of neighboring plugins/commands rather than introducing new patterns.
- **When adding a plugin**, update all three touchpoints: the plugin directory (with `plugin.json` + `README.md`), `.claude-plugin/marketplace.json`, and the table in `plugins/README.md`.
- **Documentation links** point to the official docs at `https://code.claude.com/docs/` (and `https://docs.claude.com/en/docs/claude-code/` for plugin docs). Keep external links pointing there.
- **Security:** report vulnerabilities per `SECURITY.md`; never commit secrets. Workflow secrets are referenced via `${{ secrets.* }}` and federation `vars`.

## Where the actual product lives

- Install / usage / configuration docs: https://code.claude.com/docs/en/overview
- Plugin system docs: https://docs.claude.com/en/docs/claude-code/plugins
- Agent SDK docs: https://docs.claude.com/en/api/agent-sdk/overview
- The CLI itself: `@anthropic-ai/claude-code` on npm (installed via `curl -fsSL https://claude.ai/install.sh | bash`, Homebrew, or WinGet — npm install is deprecated).

To report a bug in the product, use `/bug` inside Claude Code or open a GitHub issue here using the issue templates in `.github/ISSUE_TEMPLATE/`.
