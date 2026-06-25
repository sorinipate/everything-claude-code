# Architecture — Everything Claude Code

Technical design of the **everything-claude-code** plugin. For the *why* see
[PRD.md](PRD.md); for the *how to develop* see [CONTRIB.md](CONTRIB.md).

---

## 1. System Overview

Everything Claude Code is a **configuration-as-a-plugin** for Claude Code. It has
no server and no build step. At runtime it contributes three things to a user's
Claude Code session:

1. **Declarative configs** (Markdown) — agents, skills, commands, rules,
   contexts — loaded by Claude Code from the plugin's directories.
2. **Event hooks** (`hooks/hooks.json`) — matchers bound to Claude Code
   lifecycle events that run inline `node -e` snippets or dispatch to scripts.
3. **Hook scripts** (`scripts/`) — dependency-free Node.js that implements the
   heavier hook logic and package-manager detection.

```
                ┌────────────────────────────────────────────┐
                │              Claude Code (host)              │
                │                                              │
   install ───► │  loads ► agents / skills / commands / rules  │
 (marketplace)  │          contexts                            │
                │                                              │
   events  ───► │  fires ► hooks.json matchers                 │
                │              │                               │
                └──────────────┼───────────────────────────────┘
                               ▼
                    scripts/hooks/*.js ──► scripts/lib/*.js
                               │
                               ▼
                  ~/.claude/sessions, ~/.claude/skills/learned,
                  ~/.claude/package-manager.json
```

---

## 2. Component Model

| Layer | Location | Format | Loaded/triggered by |
|-------|----------|--------|---------------------|
| Plugin manifest | `.claude-plugin/plugin.json` | JSON | Claude Code on install |
| Marketplace catalog | `.claude-plugin/marketplace.json` | JSON | `/plugin marketplace add` |
| Agents | `agents/*.md` | MD + frontmatter | Task delegation |
| Skills | `skills/*` | MD (+ optional `config.json`) | Commands/agents |
| Commands | `commands/*.md` | MD + frontmatter | `/<command>` |
| Rules | `rules/*.md` | MD | Always-on guidance |
| Contexts | `contexts/*.md` | MD | Mode injection |
| Hooks | `hooks/hooks.json` | JSON | Lifecycle events |
| Hook scripts | `scripts/hooks/*.js` | Node.js | Referenced by hooks |
| Shared libs | `scripts/lib/*.js` | Node.js | Imported by scripts |
| MCP configs | `mcp-configs/mcp-servers.json` | JSON | Copied to `~/.claude.json` |

`plugin.json` declares only `commands → ./commands` and `skills → ./skills`;
other directories are conventional and consumed by Claude Code or copied manually.

---

## 3. Directory Structure

```
everything-claude-code/
├── .claude-plugin/        # plugin.json, marketplace.json (distribution)
├── agents/                # 9 subagent definitions
├── commands/              # 15 slash commands
├── skills/                # 11 skills (dirs may hold config.json)
├── rules/                 # 8 always-follow rule files
├── contexts/              # dev / review / research modes
├── hooks/
│   ├── hooks.json         # event → matcher → command bindings
│   ├── memory-persistence/  # shell variants (session lifecycle)
│   └── strategic-compact/   # shell variant
├── scripts/
│   ├── lib/
│   │   ├── utils.js           # cross-platform fs/git/date helpers
│   │   └── package-manager.js # PM detection & config
│   ├── hooks/                 # session-start/end, pre-compact,
│   │   └── …                  #   suggest-compact, evaluate-session
│   └── setup-package-manager.js
├── tests/                 # test suite (run-all.js)
├── mcp-configs/           # MCP server definitions
├── examples/              # sample CLAUDE.md, statusline, sessions
└── docs/                  # PRD, ARCHITECTURE, CONTRIB, RUNBOOK
```

---

## 4. Distribution Architecture

The plugin is delivered straight from the git repo — there is no packaging step.

- **`marketplace.json`** lists the plugin with `source: "./"` (the repo root is
  the plugin). Users run `/plugin marketplace add affaan-m/everything-claude-code`.
- **`plugin.json`** declares metadata and the `commands`/`skills` paths.
- **No `version` field** is present in either manifest — intentional, so Claude
  Code always resolves the latest `main` and users get **automatic updates**
  (see commits `4ec7a6b`, `5230892`).

Manual installation is an equivalent path: copy `agents/`, `rules/`, `commands/`,
`skills/` into `~/.claude/`, merge `hooks.json` into settings, and merge desired
MCP servers into `~/.claude.json`.

---

## 5. Hook Lifecycle

`hooks/hooks.json` binds matchers to Claude Code events. Hooks are either inline
`node -e` one-liners or dispatch to `${CLAUDE_PLUGIN_ROOT}/scripts/hooks/*.js`.

| Event | Matcher(s) | Behavior | Impl |
|-------|-----------|----------|------|
| `SessionStart` | `*` | Load recent sessions (≤7d), list learned skills, detect PM | `session-start.js` |
| `PreToolUse` | Bash `*dev` | **Block** dev servers outside tmux | inline |
| `PreToolUse` | Bash install/test/build | Remind to use tmux | inline |
| `PreToolUse` | `git push` | Remind to review | inline |
| `PreToolUse` | Write `.md`/`.txt` (non-allowlisted) | **Block** stray docs | inline |
| `PreToolUse` | Edit/Write | Suggest `/compact` past threshold | `suggest-compact.js` |
| `PostToolUse` | Bash `gh pr create` | Log PR URL + review cmd | inline |
| `PostToolUse` | Edit `.ts/.tsx/.js/.jsx` | Prettier auto-format | inline |
| `PostToolUse` | Edit `.ts/.tsx` | `tsc --noEmit` check | inline |
| `PostToolUse` | Edit JS/TS | Warn on `console.log` | inline |
| `Stop` | `*` | Scan modified JS/TS for `console.log` | inline |
| `PreCompact` | `*` | Append to `compaction-log.txt`, annotate session | `pre-compact.js` |
| `SessionEnd` | `*` | Write dated session log | `session-end.js` |
| `SessionEnd` | `*` | Flag long sessions for pattern extraction | `evaluate-session.js` |

**Design principles:**
- **Fail-soft.** Every script `process.exit(0)` on error so a broken hook never
  blocks a session.
- **Latency-aware.** Expensive analysis runs on `Stop`/`SessionEnd`, not per
  message.
- **stderr for signal.** Hooks log to stderr (visible to the user); stdout is
  passed back through to Claude.

---

## 6. Scripts & Shared-Library Architecture

```
scripts/hooks/*.js  ─┬─►  scripts/lib/utils.js          (fs, git, date, platform)
                     └─►  scripts/lib/package-manager.js (PM detection)
scripts/setup-package-manager.js ─► lib/package-manager.js
```

### `lib/utils.js`
Cross-platform primitives so scripts never shell out to Unix-only tools:
- Paths: `getClaudeDir`, `getSessionsDir`, `getLearnedSkillsDir`, `getTempDir`,
  `ensureDir`.
- Files: `readFile`, `writeFile`, `appendFile`, `findFiles` (glob+age, replaces
  `find`), `replaceInFile` (replaces `sed`), `countInFile`, `grepFile`.
- System: `commandExists` (`where`/`which`), `runCommand`, `isGitRepo`,
  `getGitModifiedFiles`.
- I/O: `readStdinJson`, `log` (stderr), `output` (stdout).

### `lib/package-manager.js`
Defines `PACKAGE_MANAGERS` (npm/pnpm/yarn/bun with their `install`/`run`/`exec`/
`test`/`build`/`dev` commands and lock files) and the resolution engine.

---

## 7. Package-Manager Resolution

`getPackageManager()` resolves in strict priority order; each result carries its
`source` for transparency:

```
1. CLAUDE_PACKAGE_MANAGER env var          → source: environment
2. .claude/package-manager.json (project)  → source: project-config
3. package.json "packageManager" field     → source: package.json
4. lock file  (pnpm→bun→yarn→npm)          → source: lock-file
5. ~/.claude/package-manager.json (global) → source: global-config
6. first installed (DETECTION_PRIORITY)    → source: fallback
   else npm                                → source: default
```

Configured via `/setup-pm` or `node scripts/setup-package-manager.js`
(`--detect` / `--list` / `--global <pm>` / `--project <pm>`).

---

## 8. Data & State

All persistent state is **user-local**, outside the repo:

| Path | Written by | Purpose |
|------|-----------|---------|
| `~/.claude/sessions/<date>-session.tmp` | `session-end.js` | Per-session state/log |
| `~/.claude/sessions/compaction-log.txt` | `pre-compact.js` | Compaction history |
| `~/.claude/skills/learned/*.md` | continuous-learning | Extracted patterns |
| `~/.claude/package-manager.json` | `setup-package-manager.js` | Global PM preference |
| `.claude/package-manager.json` | `setup-package-manager.js` | Project PM preference |
| `$TMPDIR/claude-tool-count-<id>` | `suggest-compact.js` | Per-session tool counter |

No data leaves the machine; there is no telemetry.

---

## 9. Cross-Platform Design

- **Pure Node.js**, built-ins only — no npm install, no native deps.
- Platform branches isolated in `utils.js` (`isWindows/isMacOS/isLinux`,
  `where` vs `which`).
- Filesystem operations via Node `fs`/`path`; no shell globbing or `sed`/`find`.
- Hooks reference scripts through `${CLAUDE_PLUGIN_ROOT}` for location
  independence.

Legacy shell variants under `hooks/memory-persistence/` and
`hooks/strategic-compact/` remain for reference; the Node scripts are canonical.

---

## 10. Configuration Surface

| Variable | Default | Effect |
|----------|---------|--------|
| `CLAUDE_PACKAGE_MANAGER` | unset | Force a package manager |
| `COMPACT_THRESHOLD` | `50` | Tool calls before suggesting `/compact` |
| `CLAUDE_SESSION_ID` | `ppid` | Per-session counter key |
| `CLAUDE_TRANSCRIPT_PATH` | set by host | Transcript scanned for learning |

`skills/continuous-learning/config.json` (optional) tunes `min_session_length`
and `learned_skills_path`.

---

## 11. Security Considerations

- **No secrets in repo** — MCP configs ship `YOUR_*_HERE` placeholders.
- **Docs consolidation hook** blocks creation of arbitrary `.md`/`.txt` files
  (allowlist: README/CLAUDE/AGENTS/CONTRIBUTING) to prevent doc sprawl.
- **Guardrail hooks** nudge safe practices (tmux for servers, review before
  push) without hard-blocking core work.
- **Context-window hygiene** — guidance to keep <10 MCPs / <80 tools active to
  avoid degraded sessions.

---

## 12. Extensibility

Adding a component is a file drop in the matching directory following the
documented frontmatter (see [CONTRIBUTING.md](../CONTRIBUTING.md)):

- **Agent** → `agents/<name>.md` with `name/description/tools/model`.
- **Command** → `commands/<name>.md` with `description`.
- **Skill** → `skills/<name>/` (optionally a `config.json`).
- **Hook** → new entry in `hooks/hooks.json`; heavy logic → a new
  `scripts/hooks/*.js` using `lib/utils.js`.

Because hooks are independent entries, one can be added or disabled without
touching the rest of the system.

---

*Source-of-truth files: `.claude-plugin/*.json`, `hooks/hooks.json`,
`scripts/lib/*.js`. Keep this document in sync when those change.*
