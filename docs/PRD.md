# Product Requirements Document — Everything Claude Code

| | |
|---|---|
| **Product** | everything-claude-code |
| **Type** | Claude Code plugin + marketplace |
| **Owner** | Affaan Mustafa ([@affaanmustafa](https://x.com/affaanmustafa)) |
| **License** | MIT |
| **Status** | Released / actively maintained |
| **Last updated** | 2026-06-25 |

> This PRD describes an existing, shipped product. It documents the product as
> built — requirements are reverse-specified from the repository so future
> contributors share one source of intent. It is not a launch plan.

---

## 1. Overview

Everything Claude Code is a **distributable configuration package for Claude
Code**. It bundles battle-tested agents, skills, slash commands, rules, contexts,
hooks, and MCP server configurations — evolved over 10+ months of daily
production use — into a single plugin a developer can install in one step, or
copy from piecemeal.

It is **not an application**. There is no server, build artifact, or runtime
service. The deliverable is a curated set of Markdown configs plus a small layer
of dependency-free Node.js scripts that power lifecycle hooks.

---

## 2. Problem Statement

Claude Code is highly configurable, but configuring it well is hard:

- **High setup cost.** Agents, hooks, rules, and MCP configs each have their own
  format and placement. Newcomers face a blank `~/.claude/` directory.
- **Tribal knowledge.** Effective patterns (memory persistence, strategic
  compaction, continuous learning, verification loops) are scattered across
  threads and personal setups, not packaged.
- **Context-window waste.** Naïve setups enable too many MCP tools and burn the
  context window; few users know the practical limits.
- **No reuse path.** Good per-project workflows rarely become reusable assets.
- **Platform friction.** Shell-based hooks historically broke on Windows.

## 3. Goals & Non-Goals

### Goals

1. Provide a **one-command install** of a complete, opinionated Claude Code setup.
2. Offer **modular, copy-what-you-need** components for users who want control.
3. Encode **advanced workflows** (memory, compaction, continuous learning, evals)
   as ready-to-use automation.
4. Be **cross-platform** (Windows, macOS, Linux) with zero runtime dependencies.
5. Be a **community resource** — easy to read, fork, and contribute to.

### Non-Goals

- Not a standalone app, SaaS, or hosted service.
- Not a replacement for Claude Code itself — it configures it.
- Not prescriptive: users are expected to delete/modify what doesn't fit.
- No paid-service lock-in; configs requiring paid MCPs always have alternatives.

---

## 4. Target Users & Personas

| Persona | Need | How the product serves them |
|---------|------|------------------------------|
| **New Claude Code user** | A sane starting setup | One-command plugin install; the Shorthand Guide |
| **Power user** | Advanced automation (memory, evals, parallelization) | Hooks, skills, contexts; the Longform Guide |
| **Team lead** | Consistent standards across a team | Rules + agents shareable via the marketplace |
| **Contributor** | A clear path to add agents/skills | `CONTRIBUTING.md` + documented formats |
| **Multi-stack dev** | Works across npm/pnpm/yarn/bun + OSes | Package-manager detection + Node.js scripts |

---

## 5. Scope — Feature Catalog

### 5.1 Agents (`agents/`)
Specialized subagents for delegated, scoped tasks:
`planner`, `architect`, `tdd-guide`, `code-reviewer`, `security-reviewer`,
`build-error-resolver`, `e2e-runner`, `refactor-cleaner`, `doc-updater`.

### 5.2 Skills (`skills/`)
Workflow definitions and domain knowledge:
`coding-standards`, `backend-patterns`, `frontend-patterns`, `clickhouse-io`,
`tdd-workflow`, `security-review`, `continuous-learning`, `strategic-compact`,
`eval-harness`, `verification-loop`, plus `project-guidelines-example`.

### 5.3 Commands (`commands/`)
Slash commands for quick execution:
`/tdd`, `/plan`, `/e2e`, `/code-review`, `/build-fix`, `/refactor-clean`,
`/learn`, `/checkpoint`, `/verify`, `/eval`, `/orchestrate`, `/test-coverage`,
`/update-codemaps`, `/update-docs`, `/setup-pm`.

### 5.4 Rules (`rules/`)
Always-follow guidelines: `security`, `coding-style`, `testing`,
`git-workflow`, `agents`, `performance`, `patterns`, `hooks`.

### 5.5 Contexts (`contexts/`)
Mode-specific system-prompt injection: `dev`, `review`, `research`.

### 5.6 Hooks (`hooks/hooks.json` + `scripts/hooks/`)
Event-driven automations across `SessionStart`, `PreToolUse`, `PostToolUse`,
`Stop`, `PreCompact`, `SessionEnd`. See [ARCHITECTURE.md](ARCHITECTURE.md#hook-lifecycle).

### 5.7 MCP Configs (`mcp-configs/`)
Ready-to-use server definitions (GitHub, Supabase, Vercel, Railway, etc.) with
placeholder secrets.

### 5.8 Tooling (`scripts/`)
Cross-platform Node.js scripts powering hooks and package-manager setup.

---

## 6. Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-1 | The repo MUST be installable as a Claude Code plugin via `/plugin marketplace add` + `/plugin install`. |
| FR-2 | Every component MUST also be usable via manual copy into `~/.claude/`. |
| FR-3 | Hooks MUST run on Windows, macOS, and Linux without modification. |
| FR-4 | Hook scripts MUST fail soft — an error MUST NOT block the user's session (`exit 0` on error). |
| FR-5 | The package manager MUST be auto-detected via a documented 6-step priority order, overridable by env var, project config, or `/setup-pm`. |
| FR-6 | Session state MUST persist across sessions (write on `SessionEnd`, load on `SessionStart`). |
| FR-7 | Long sessions SHOULD be flagged for continuous-learning pattern extraction. |
| FR-8 | The plugin MUST auto-update for users (no version pinning in manifests). |
| FR-9 | Component formats (agent/skill/command/hook frontmatter) MUST be documented for contributors. |
| FR-10 | Scripts MUST have **zero npm dependencies** (Node built-ins only). |

## 7. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| **Portability** | No reliance on Unix-only tools (`find`, `sed`); use `lib/utils.js` equivalents. |
| **Reliability** | Hooks defensive by default; never break a session. |
| **Performance** | Heavy work runs on `Stop`/`SessionEnd`, not per-message, to avoid latency. |
| **Context economy** | Guidance to keep <10 MCPs and <80 tools active per project. |
| **Maintainability** | Modular, focused configs; consolidated docs (a hook blocks stray `.md` files). |
| **Security** | No committed secrets; MCP configs use `YOUR_*_HERE` placeholders. |
| **Discoverability** | README inventory + external Shorthand/Longform guides. |

---

## 8. Success Metrics

- **Adoption** — GitHub stars, marketplace installs, forks.
- **Time-to-first-value** — minutes from install to a working agent/command.
- **Contribution rate** — community-submitted agents/skills/configs merged.
- **Cross-platform success** — installs working on Windows/macOS/Linux without
  manual fixes.
- **Session continuity** — sessions resumed via persisted state.

---

## 9. Assumptions & Dependencies

- Users run **Claude Code** and have **Node.js** + **git** available.
- Optional MCP servers require the user's own API keys.
- External long-form documentation lives in the Shorthand/Longform guides
  linked from the README (out of repo).

## 10. Out of Scope

- Hosting, billing, telemetry, or any backend service.
- Bundling third-party MCP server binaries.
- Guaranteeing every config fits every stack (users adapt).

## 11. Future / Ideas

From the README contribution backlog:
- Language-specific skills (Python, Go, Rust).
- Framework configs (Django, Rails, Laravel).
- DevOps agents (Kubernetes, Terraform, AWS).
- Additional testing-strategy and domain (ML, data, mobile) packs.

---

*See [ARCHITECTURE.md](ARCHITECTURE.md) for the technical design and
[CONTRIB.md](CONTRIB.md) for the development workflow.*
