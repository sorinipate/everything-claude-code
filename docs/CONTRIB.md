# Development Guide

Developer reference for working on **everything-claude-code** — a Claude Code
plugin/marketplace, not a built application. There is no `package.json`,
`node_modules`, build step, or `.env` file. The repo ships Markdown configs
(agents, commands, skills, rules, contexts) plus a set of plain Node.js scripts
that power the hooks.

> For *what* to contribute and the required file formats, see
> [CONTRIBUTING.md](../CONTRIBUTING.md). This guide covers the *how* — workflow,
> scripts, environment, and testing.

---

## Repository Layout

| Directory | Contents |
|-----------|----------|
| `agents/` | Subagent definitions (Markdown with frontmatter) |
| `commands/` | Slash command definitions |
| `skills/` | Skills (single `.md` or directory with `config.json`) |
| `rules/` | Always-follow guideline files |
| `contexts/` | Context presets (`dev`, `research`, `review`) |
| `hooks/` | `hooks.json` event config + shell hook variants |
| `scripts/` | Node.js scripts powering the hooks (see below) |
| `mcp-configs/` | MCP server configurations |
| `examples/` | Example `CLAUDE.md`, statusline, session files |
| `.claude-plugin/` | `plugin.json` + `marketplace.json` manifests |

---

## Prerequisites

- **Node.js** — required to run the hook scripts under `scripts/`. They use only
  Node built-ins (`fs`, `path`, `os`, `child_process`); there are no npm
  dependencies to install.
- **Git** — several hooks shell out to `git`.
- A **package manager** (`npm`, `pnpm`, `yarn`, or `bun`) only if you are testing
  the package-manager detection logic. `bun` is the configured default for this
  repo (see [.claude/package-manager.json](../.claude/package-manager.json)).

There is **no install step**: `git clone` and you can run any script directly.

---

## Available Scripts

These are standalone Node entry points (no `package.json` `scripts` block exists).
Run them with `node <path>`.

| Script | Purpose |
|--------|---------|
| [scripts/setup-package-manager.js](../scripts/setup-package-manager.js) | Configure the preferred package manager (also invoked by the `/setup-pm` command) |
| [scripts/hooks/session-start.js](../scripts/hooks/session-start.js) | `SessionStart` — load recent session context, list learned skills, detect package manager |
| [scripts/hooks/session-end.js](../scripts/hooks/session-end.js) | `SessionEnd` — create/update a dated session log file |
| [scripts/hooks/pre-compact.js](../scripts/hooks/pre-compact.js) | `PreCompact` — record a compaction event and annotate the active session file |
| [scripts/hooks/suggest-compact.js](../scripts/hooks/suggest-compact.js) | `PreToolUse` — suggest manual `/compact` after a threshold of tool calls |
| [scripts/hooks/evaluate-session.js](../scripts/hooks/evaluate-session.js) | `SessionEnd` — flag long sessions for continuous-learning pattern extraction |

Shared helpers live in [scripts/lib/utils.js](../scripts/lib/utils.js)
(cross-platform fs/git/date helpers) and
[scripts/lib/package-manager.js](../scripts/lib/package-manager.js)
(package-manager detection and config).

### `setup-package-manager.js` usage

```bash
node scripts/setup-package-manager.js --detect          # show current selection + detection
node scripts/setup-package-manager.js --list            # list supported managers
node scripts/setup-package-manager.js --global pnpm      # save to ~/.claude/package-manager.json
node scripts/setup-package-manager.js --project bun      # save to .claude/package-manager.json
node scripts/setup-package-manager.js --help
```

---

## Environment Variables

There is no `.env.example`. The scripts read a small set of **optional**
environment variables (all have safe fallbacks):

| Variable | Used by | Default | Purpose |
|----------|---------|---------|---------|
| `CLAUDE_PACKAGE_MANAGER` | `lib/package-manager.js` | _(unset)_ | Highest-priority package-manager override (`npm` / `pnpm` / `yarn` / `bun`) |
| `COMPACT_THRESHOLD` | `suggest-compact.js` | `50` | Tool-call count before suggesting `/compact` |
| `CLAUDE_SESSION_ID` | `suggest-compact.js` | `process.ppid` | Disambiguates the per-session tool-count file |
| `CLAUDE_TRANSCRIPT_PATH` | `evaluate-session.js` | _(set by Claude Code)_ | Path to the session transcript scanned for pattern extraction |

Package-manager resolution order (see `getPackageManager` in
[lib/package-manager.js](../scripts/lib/package-manager.js)):

1. `CLAUDE_PACKAGE_MANAGER` env var
2. Project config `.claude/package-manager.json`
3. `package.json` `packageManager` field
4. Lock file (`pnpm` → `bun` → `yarn` → `npm`)
5. Global config `~/.claude/package-manager.json`
6. First installed manager / fallback to `npm`

---

## Development Workflow

1. **Fork & branch** — `git checkout -b add-<thing>` (see
   [CONTRIBUTING.md](../CONTRIBUTING.md#how-to-contribute)).
2. **Add your config** in the matching directory using the required frontmatter
   format.
3. **Validate manifests** if you touched `.claude-plugin/` — see Testing below.
4. **Test in Claude Code** — load the plugin locally and exercise the
   agent/command/hook.
5. **Commit & PR** — describe what, why, and how you tested it.

### Conventions

- File names: lowercase-with-hyphens, descriptive (`python-reviewer.md`).
- Keep configs focused and modular; follow existing patterns.
- Never commit secrets, API keys, tokens, or absolute machine paths.
- The repo enforces consolidated docs: a `PreToolUse` hook **blocks** creating
  arbitrary new `.md`/`.txt` files (only `README` / `CLAUDE` / `AGENTS` /
  `CONTRIBUTING` are exempt). Prefer extending existing docs.

---

## Testing Procedures

There is no automated test runner in this repo. Validation is manual:

**Hook scripts** — run directly and confirm exit code 0 and expected stderr logs:

```bash
node scripts/hooks/session-start.js
node scripts/hooks/session-end.js
node scripts/setup-package-manager.js --detect
```

All hook scripts are defensive: they `process.exit(0)` on error so a failing
hook never blocks a Claude Code session.

**JSON configs** — validate before committing:

```bash
node -e "JSON.parse(require('fs').readFileSync('hooks/hooks.json','utf8'))"
node -e "JSON.parse(require('fs').readFileSync('.claude-plugin/plugin.json','utf8'))"
node -e "JSON.parse(require('fs').readFileSync('.claude-plugin/marketplace.json','utf8'))"
```

**Full integration** — install the plugin in a local Claude Code instance and
verify the agent/command/hook behaves as intended before opening a PR.

---

## Cross-Platform Notes

All scripts are written to run on Windows, macOS, and Linux. They use the helpers
in [lib/utils.js](../scripts/lib/utils.js) (e.g. `findFiles` instead of `find`,
`replaceInFile` instead of `sed`, `commandExists` using `where`/`which`) rather
than shelling out to Unix-only tools. Keep new scripts dependency-free and
platform-neutral.
