# agentdots-sync usage

After install you can:

1. **Ask an agent** in Claude Code, Codex, Grok, OpenCode, and similar tools to sync, back up, restore, or roll back.
2. **Run the CLI** yourself against `scripts/sync.py` inside the installed skill directory.

The tool copies files. It does not convert `AGENTS.md` into `CLAUDE.md` or rewrite platform config formats.

Replace `<skill-root>` with the directory that contains this skill’s `SKILL.md`:

| Install method | Typical skill root |
| --- | --- |
| `npx skills add ferbylv/agentdots-sync` (user-level) | `~/.claude/skills/agentdots-sync`, `~/.codex/skills/agentdots-sync`, `~/.grok/skills/agentdots-sync`, … |
| `npx skills add` (project-level) | `<project>/.claude/skills/agentdots-sync` or `<project>/.agents/skills/agentdots-sync` |
| `gh skill install ferbylv/agentdots-sync` | The host agent’s skills directory |
| `git clone` | `<clone>/skills/agentdots-sync` |

If unsure, ask the agent to use the loaded skill directory, or search for `agentdots-sync/SKILL.md`.

---

## 1. What to say to the agent

Examples:

- “List platforms supported by agentdots-sync.”
- “Doctor my local Claude Code skills. Do not write files.”
- “Plan pushing `~/.claude` skills to `git@github.com:me/agent-assets.git` on `main`.”
- “After I approve the plan, push Codex `AGENTS.md`, `rules`, and `skills`.”
- “Restore my Claude skills from the remote. Back up first.”
- “Roll back the latest Claude Code backup.”

The agent should:

1. Run `--list-platforms`, `--doctor`, or `--plan` first (read-only).
2. Show platform, root, assets, remote, and branch.
3. Add `--yes` only after you approve that exact plan.
4. Treat “sync it” as insufficient permission to pick a direction or overwrite files.

---

## 2. CLI: inspect, then mutate

Needs Python 3.9+. `--push` / `--apply` / `--restore` also need Git access to the remote.

```bash
python3 <skill-root>/scripts/sync.py --list-platforms
python3 <skill-root>/scripts/sync.py --platform claude-code --doctor
python3 <skill-root>/scripts/sync.py --platform claude-code --plan \
  --remote git@github.com:YOUR/agent-assets.git --branch main
```

`--plan` prints the plan and writes nothing. With no mutate flag, the default is inspect/plan.

Check `platform`, `root`, `assets`, `backup_dir`, `remote`, and `branch` before `--yes`.

---

## 3. Write operations

`--apply` and `--restore` copy **remote → local** (backup, then replace). `--push` copies **local → remote**.

### 3.1 Pull from Git (new machine / restore)

```bash
python3 <skill-root>/scripts/sync.py --platform claude-code --restore --yes \
  --remote git@github.com:YOUR/agent-assets.git --branch main
```

`--apply` does the same copy; the backup reason is recorded as `apply`:

```bash
python3 <skill-root>/scripts/sync.py --platform claude-code --apply --yes \
  --remote git@github.com:YOUR/agent-assets.git --branch main
```

Existing local assets are backed up first. A first run with nothing local has nothing to back up.

### 3.2 Push local changes to Git

```bash
python3 <skill-root>/scripts/sync.py --platform grok-build --push --yes \
  --remote git@github.com:YOUR/agent-assets.git --branch main \
  --message "Update Grok skills"
```

Only selected paths are staged. Unchanged assets print `PUSH skipped`. Push is never forced.

### 3.3 Roll back the latest local backup

```bash
python3 <skill-root>/scripts/sync.py --platform claude-code --rollback --yes
```

No `--remote`. Restores only assets that are both in the backup manifest and selected on this run.

---

## 4. Platforms

`--platform` defaults to **`codex`**. Pass the real host or you will touch `~/.codex`.

| `--platform` | Aliases | Default root | Default assets | Backup dir |
| --- | --- | --- | --- | --- |
| `codex` | `openai-codex` | `~/.codex` | `AGENTS.md`, `rules`, `skills` | `.codex_backups` |
| `claude-code` | `claude` | `~/.claude` | `CLAUDE.md`, `rules`, `skills` | `.meta-sync-backups` |
| `opencode` | — | `~/.config/opencode` | `skills` | `.meta-sync-backups` |
| `grok-build` | `grok` | `~/.grok` | `skills` | `.meta-sync-backups` |
| `codebuddy-code` | `codebuddy` | `~/.codebuddy` | `rules`, `skills` | `.meta-sync-backups` |
| `workbuddy` | — | **`--root` required** | `skills` | `.meta-sync-backups` |
| `custom` | — | **`--root` required** | **`--asset` required** | `.meta-sync-backups` |

Select OpenCode / Grok config files explicitly; they may contain environment-specific values.

For WorkBuddy, `--root` must be an unpacked local skill/package workspace. UI import is a separate step.

---

## 5. Overrides

```bash
python3 <skill-root>/scripts/sync.py --platform custom \
  --root /path/to/agent-config \
  --asset instructions.md --asset skills --plan

python3 <skill-root>/scripts/sync.py --platform opencode \
  --root /path/to/your-project \
  --asset AGENTS.md --asset .opencode/skills --plan
```

`--backup-dir` must stay a child path inside the selected root. One platform per run. Do not apply one platform’s defaults onto another platform’s files.

---

## 6. Safety

- Listing, planning, and `--doctor` are read-only.
- Mutations require `--yes`.
- apply / restore / rollback back up existing selected local assets first.
- Mutations on one root are serialized with `.meta-sync-lock`.
- Selected assets cannot escape the root or use outbound symlinks.
- Push never force-pushes. Stop on Git or lock failures.
- Error text redacts common URL credentials and tokens.

Create the remote and authenticate Git yourself. This skill will not run `gh auth`, create repositories, or guess branches.

---

## 7. Example: back up Claude Code skills

```bash
python3 <skill-root>/scripts/sync.py --platform claude --doctor

python3 <skill-root>/scripts/sync.py --platform claude-code --push --plan \
  --remote git@github.com:YOUR/agent-assets.git --branch main

python3 <skill-root>/scripts/sync.py --platform claude-code --push --yes \
  --remote git@github.com:YOUR/agent-assets.git --branch main \
  --message "Backup Claude Code skills"

python3 <skill-root>/scripts/sync.py --platform claude-code --restore --yes \
  --remote git@github.com:YOUR/agent-assets.git --branch main

python3 <skill-root>/scripts/sync.py --platform claude-code --rollback --yes
```

`claude` and `claude-code` are the same profile.

---

## 8. Troubleshooting

**`mutating actions require --yes`** — inspect with `--plan`, then add `--yes`.

**`platform 'workbuddy' requires explicit --root`** — WorkBuddy and `custom` have no default home path.

**`PUSH skipped`** — selected assets already match the remote.

**Agent ignores the skill** — confirm it is installed in that host’s skills directory, and name agentdots-sync or ask to sync/backup/restore. Browser-only chat cannot run local Python.

**Convert CLAUDE.md into AGENTS.md** — out of scope; treat as a separate rewrite.
