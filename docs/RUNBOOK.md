# Operations Runbook

Operational reference for **everything-claude-code**. This is a Claude Code
plugin distributed through a marketplace manifest — there is **no application
server, container, or cloud deployment**. "Operating" it means publishing plugin
updates, understanding the hook lifecycle that runs inside users' Claude Code
sessions, and recovering when a hook misbehaves.

---

## 1. Distribution & "Deployment"

The plugin is consumed by pointing Claude Code at this repo's marketplace
manifest. There is no build artifact.

| Manifest | Role |
|----------|------|
| [.claude-plugin/marketplace.json](../.claude-plugin/marketplace.json) | Marketplace entry; lists the plugin and its source (`./`) |
| [.claude-plugin/plugin.json](../.claude-plugin/plugin.json) | Plugin manifest; declares `commands` → `./commands` and `skills` → `./skills` |

**Release procedure:**

1. Make changes on a branch and open a PR.
2. Validate manifests parse (see [CONTRIB.md → Testing](CONTRIB.md#testing-procedures)).
3. Merge to `main`. Because the manifests intentionally carry **no version
   field**, Claude Code picks up the latest `main` automatically — there is no
   version bump to perform.
4. Tag a release only if you want a named checkpoint (optional).

> The absence of `version` fields is deliberate (see commits
> `4ec7a6b` and `5230892`) — it enables automatic plugin updates for users.

---

## 2. Hook Lifecycle (the runtime surface)

The operational behavior of this plugin is the hooks defined in
[hooks/hooks.json](../hooks/hooks.json). They fire on Claude Code lifecycle
events inside a user's session:

| Event | Hook | Action |
|-------|------|--------|
| `SessionStart` | [session-start.js](../scripts/hooks/session-start.js) | Loads recent session files (≤7 days), lists learned skills, detects package manager |
| `PreToolUse` | inline | Blocks dev servers run outside `tmux`; reminds to use `tmux` for long commands; warns before `git push`; **blocks** stray `.md`/`.txt` creation |
| `PreToolUse` | [suggest-compact.js](../scripts/hooks/suggest-compact.js) | Suggests `/compact` after `COMPACT_THRESHOLD` (default 50) tool calls |
| `PostToolUse` | inline | Logs PR URL + review command after `gh pr create`; auto-formats JS/TS with Prettier; runs `tsc --noEmit`; warns on `console.log` |
| `PreCompact` | [pre-compact.js](../scripts/hooks/pre-compact.js) | Appends to `compaction-log.txt` and annotates the active session file |
| `Stop` | inline | Scans modified JS/TS for `console.log` |
| `SessionEnd` | [session-end.js](../scripts/hooks/session-end.js), [evaluate-session.js](../scripts/hooks/evaluate-session.js) | Writes the dated session log; flags long sessions for pattern extraction |

`${CLAUDE_PLUGIN_ROOT}` in `hooks.json` resolves to the installed plugin
directory.

---

## 3. Monitoring

There is no metrics/alerting stack. Observability is local to a session:

- **Hook output** — hooks log to **stderr** (visible in Claude Code). Look for
  prefixed lines: `[SessionStart]`, `[PreCompact]`, `[StrategicCompact]`,
  `[ContinuousLearning]`, `[Hook]`.
- **Session logs** — `~/.claude/sessions/<date>-session.tmp` (per-session state)
  and `~/.claude/sessions/compaction-log.txt` (compaction history).
- **Learned skills** — `~/.claude/skills/learned/` accumulates extracted patterns.
- **Package-manager state** — `~/.claude/package-manager.json` (global) and
  `.claude/package-manager.json` (project).

Health check: run any hook script directly and confirm it exits 0 (see
[CONTRIB.md → Testing](CONTRIB.md#testing-procedures)).

---

## 4. Common Issues & Fixes

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Dev server command blocked | `PreToolUse` tmux guard | Run inside tmux: `tmux new-session -d -s dev "<cmd>"`, then `tmux attach -t dev` |
| New `.md`/`.txt` write blocked | Docs-consolidation `PreToolUse` hook | Add content to an existing allowed doc, or name it `README.md` |
| Wrong package manager detected | Resolution order picked an unexpected source | `node scripts/setup-package-manager.js --detect`, then `--global`/`--project` to override, or set `CLAUDE_PACKAGE_MANAGER` |
| `/compact` suggested too often / rarely | `COMPACT_THRESHOLD` default (50) | Set `COMPACT_THRESHOLD` env var |
| Prettier/tsc PostToolUse hook errors | `npx` can't find the tool / no tsconfig | Hooks fail soft (swallow errors); ensure the target project has the tooling if you want it to run |
| No previous context loaded on start | No session files <7 days old | Expected; a new `<date>-session.tmp` is created on `SessionEnd` |
| Hook seems to do nothing | Script errored | All hooks `exit(0)` on error by design — run the script manually to see the real error |

---

## 5. Rollback

Rollback is a **git operation** plus plugin refresh — there is nothing to redeploy.

**Revert a bad change on `main`:**

```bash
git revert <bad-commit-sha>     # safe, preserves history
git push origin main
```

Users pick up the reverted `main` on their next plugin update (no version pin).

**Pin to a known-good commit (emergency, per-user):**
Point the Claude Code plugin/marketplace source at a specific commit or tag
instead of the branch tip until `main` is healthy.

**Disable a single misbehaving hook without a full rollback:**
Remove or comment out the offending entry in
[hooks/hooks.json](../hooks/hooks.json) and merge — hooks are independent, so
one bad matcher does not require reverting unrelated work.

**Recover local session state:**
Session/compaction files live under `~/.claude/sessions/`; they are user-local
and not affected by repo rollbacks.
