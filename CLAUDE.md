# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this repository is (and is not)

This is the **public `anthropics/claude-code` repository**. It is the public
face of Claude Code — an agentic coding tool that runs in the terminal, IDE, and
via `@claude` on GitHub.

**Important:** This repository does **not** contain the Claude Code CLI source
code. The CLI ships as a compiled binary distributed through the install script,
Homebrew, WinGet, and the (deprecated) `@anthropic-ai/claude-code` npm package.
What lives here instead:

- **User-facing distribution & documentation** — the `README.md`, the public
  `CHANGELOG.md`, the RSS `feed.xml`, `SECURITY.md`, and `LICENSE.md`.
- **Official example plugins** (`plugins/`) plus the marketplace manifest that
  exposes them.
- **Deployment & configuration examples** (`examples/`) — LLM gateways,
  hooks, enterprise MDM policies, and settings presets.
- **Issue/PR automation** (`.github/workflows/`, `scripts/`, `.claude/`) that
  runs Claude Code itself to triage, deduplicate, and manage the issue tracker.

Because there is no application source here, there is **no build step, no test
suite, and no package manifest** at the repo root. Do not look for one or try to
"install dependencies."

## Repository layout

```
.
├── README.md                  # Install instructions & project overview
├── CHANGELOG.md               # Public release notes (newest version at top)
├── feed.xml                   # RSS feed generated from the changelog
├── SECURITY.md, LICENSE.md
├── .claude/
│   └── commands/              # Repo-local slash commands (used by maintainers/CI)
│       ├── commit-push-pr.md
│       ├── dedupe.md
│       └── triage-issue.md
├── .claude-plugin/
│   └── marketplace.json       # Plugin marketplace manifest (lists every plugin)
├── plugins/                   # Official example plugins (see below)
│   └── README.md              # Catalog of all bundled plugins
├── examples/
│   ├── gateway/               # AWS & GCP LLM gateway deploys (Terraform + Docker)
│   ├── hooks/                 # Example hook scripts (e.g. bash command validator)
│   ├── mdm/                   # Enterprise device-management policies (macOS/Windows)
│   └── settings/             # Example settings.json presets (strict/lax/sandbox)
├── scripts/                   # Issue-management automation (Bun + bash)
├── .github/
│   ├── workflows/             # GitHub Actions (issue lifecycle, Claude triage, etc.)
│   └── ISSUE_TEMPLATE/        # Bug/feature/docs/model-behavior issue forms
├── .devcontainer/             # Dev container (Dockerfile, devcontainer.json, firewall)
└── Script/                    # PowerShell helper to run Claude Code in the devcontainer
```

## Plugins (`plugins/`)

The bulk of the code in this repo is example plugins. Each demonstrates a facet
of the Claude Code plugin system. `plugins/README.md` is the authoritative
catalog; `.claude-plugin/marketplace.json` is the machine-readable manifest that
must stay in sync with it.

### Standard plugin structure

Every plugin follows this convention:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Metadata: name, description, version, author (required)
├── commands/                # Slash commands as markdown files (optional)
├── agents/                  # Specialized subagents as markdown files (optional)
├── skills/                  # Agent Skills, each a dir with SKILL.md (optional)
├── hooks/                   # Event handlers + hooks.json config (optional)
├── .mcp.json                # MCP server configuration (optional)
└── README.md                # Plugin documentation (required)
```

### The bundled plugins

| Plugin | What it demonstrates |
|--------|----------------------|
| `agent-sdk-dev` | Command + verifier agents for scaffolding Claude Agent SDK apps |
| `claude-opus-4-5-migration` | A **skill** that automates model-string/beta-header migration |
| `code-review` | A `/code-review` command that fans out to parallel Sonnet review agents |
| `commit-commands` | `/commit`, `/commit-push-pr`, `/clean_gone` git workflow commands |
| `explanatory-output-style` | A **SessionStart hook** injecting educational context |
| `feature-dev` | Multi-agent 7-phase feature workflow (explorer/architect/reviewer) |
| `frontend-design` | An auto-invoked **skill** for high-quality frontend generation |
| `hookify` | Python hooks + rule engine to author custom hooks from markdown rules |
| `learning-output-style` | A **SessionStart hook** for interactive learning mode |
| `plugin-dev` | 7 expert **skills** covering every part of plugin authoring |
| `pr-review-toolkit` | Specialized PR-review agents (comments/tests/errors/types/…) |
| `ralph-wiggum` | A **Stop hook** that loops Claude on a task until completion |
| `security-guidance` | A **PreToolUse hook** warning about risky code patterns |

### Conventions when editing plugins

- **Commands and agents are markdown files with YAML frontmatter.** Commands use
  `description` and optional `allowed-tools`; agents describe their role in the
  body. Look at an existing file in the same plugin before adding a new one.
- **Skills** live in `skills/<skill-name>/SKILL.md` with frontmatter describing
  when the skill should trigger.
- **Hooks** are declared in `hooks/hooks.json` and implemented as scripts
  (Python is the norm here — see `hookify/` and `security-guidance/`). Python
  hooks in this repo generally target the standard library and read event JSON
  from stdin.
- **Keep three things in sync when adding/renaming/removing a plugin:**
  1. the plugin's own `.claude-plugin/plugin.json`,
  2. the table and details in `plugins/README.md`,
  3. the entry in `.claude-plugin/marketplace.json`.
- Bump the plugin's `version` in both `plugin.json` and `marketplace.json` when
  making a user-visible change.

## Issue & PR automation (`scripts/` + `.github/` + `.claude/`)

This repo runs Claude Code and helper scripts to manage its own busy issue
tracker. Understanding this subsystem matters if you touch automation.

- **`scripts/*.ts` run under [Bun](https://bun.sh)** (`#!/usr/bin/env bun`), not
  Node. They call the GitHub REST API directly and expect `GITHUB_TOKEN` in the
  environment. `issue-lifecycle.ts` is the single source of truth for lifecycle
  labels, timeouts, and messages; other scripts import from it.
- **`scripts/*.sh` are locked-down `gh` CLI wrappers.** `scripts/gh.sh` only
  permits `issue view/list`, `search issues`, and `label list`, and scopes every
  call to the current repo via `GH_REPO`/`GITHUB_REPOSITORY`. `edit-issue-labels.sh`
  and `comment-on-duplicates.sh` are similarly narrow. These constraints are
  intentional security boundaries — preserve them; do not broaden the allowed
  subcommands or flags without a deliberate reason.
- **`.claude/commands/`** holds the slash commands the automation invokes
  (`dedupe`, `triage-issue`, `commit-push-pr`). Their `allowed-tools` frontmatter
  deliberately restricts them to the wrapper scripts above.
- **`.github/workflows/`** wires it together: `claude.yml` responds to `@claude`
  mentions, `claude-issue-triage.yml` and `claude-dedupe-issues.yml` run the
  triage/dedupe flows, and the `issue-lifecycle`/`sweep`/`lock-closed-issues`
  workflows manage stale issues. Workflows authenticate to the Claude API via
  Workload Identity Federation (OIDC), not static API keys.

## Development workflow & conventions

- **Line endings:** `.gitattributes` enforces `eol=lf` for all files, and `*.sh`
  files are always LF. Keep it that way.
- **`.gitignore`** only excludes `.DS_Store`; there are no build artifacts to
  ignore.
- **Changelog & feed:** `CHANGELOG.md` lists the newest release at the top with
  `Added / Fixed / Improved / Removed / Changed` bullet groupings. `feed.xml` is
  generated from it — the two are updated together (typically by release tooling,
  not by hand).
- **No linter/formatter/test runner is configured at the repo root.** Match the
  style of the file you are editing. For the TypeScript scripts, follow the
  existing formatting; for Python hooks, follow the style within that plugin.
- **When documenting**, mirror the existing tone: concise, user-facing markdown
  with tables and code fences (see `README.md` and `plugins/README.md`).

## Git & contribution practices

- Do the work on a feature branch; never push directly to the default branch.
- Write clear, imperative commit messages. Commit and push only when the task
  calls for it.
- After pushing, open a **draft** pull request if no open PR exists for the
  branch. Check `.github/` for a PR template and mirror its structure if present.
- When posting on GitHub (comments, reviews, issues), be frugal — comment only
  when genuinely necessary — and end every post with the Claude Code attribution
  footer.

## Where to learn more

- Official docs: <https://code.claude.com/docs/en/overview>
- Plugins: <https://docs.claude.com/en/docs/claude-code/plugins>
- Agent SDK: <https://docs.claude.com/en/api/agent-sdk/overview>
- Claude Code on the web: <https://code.claude.com/docs/en/claude-code-on-the-web>
