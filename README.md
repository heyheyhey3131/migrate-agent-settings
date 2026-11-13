# migrate-agent-settings

OpenCode skill that copies MCP servers, skills, slash commands, subagents, and
instruction files from other coding agents into **OpenCode V2** native config.

Supported sources:

- Claude Code
- Codex
- Cursor
- Gemini CLI
- Antigravity

This is not an OpenCode V1 → V2 migrator, and it does not create an OpenCode
plugin. OpenCode plugins are code packages; this skill writes skills, commands,
agents, instruction files, and `mcp.servers` entries.

## Install

Clone into the global OpenCode skills directory:

```bash
git clone https://github.com/heyheyhey3131/migrate-agent-settings.git ~/.config/opencode/skills/migrate-agent-settings
```

Windows PowerShell:

```powershell
git clone https://github.com/heyheyhey3131/migrate-agent-settings.git "$env:USERPROFILE\.config\opencode\skills\migrate-agent-settings"
```

Or copy `SKILL.md` into:

```text
~/.config/opencode/skills/migrate-agent-settings/SKILL.md
```

OpenCode picks the skill up on the next turn. No restart.

## Usage

Ask OpenCode something like:

```text
Migrate my Claude Code / Codex / Cursor setup into OpenCode.
```

The skill will:

1. Discover what exists in the other agents (read-only).
2. Show a plan and wait for confirmation.
3. Write into `~/.config/opencode/` (or this project's `.opencode/` if you ask).
4. Report what moved, what was skipped, and any env vars / OAuth sign-in still needed.

It never edits or deletes the other agents' source files.

## Destination layout

| Kind | Default global path |
| --- | --- |
| MCP | `~/.config/opencode/opencode.jsonc` → `mcp.servers` |
| Skills | `~/.config/opencode/skills/<id>/` |
| Commands | `~/.config/opencode/commands/<name>.md` |
| Agents | `~/.config/opencode/agents/<name>.md` |
| Instructions | `~/.config/opencode/instructions/migrated-*.md` listed from `instructions` |

Claude Code skills under `~/.claude/skills` already load in OpenCode. The skill
skips copying those unless you ask it to.

## License

MIT
