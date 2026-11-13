---
name: migrate-agent-settings
description: >-
  Migrates the user's MCP servers, skills, slash commands, subagents, and
  instruction files from other coding agents (Claude Code, Codex, Cursor,
  Gemini CLI, Antigravity) into OpenCode. Use when the user wants to bring
  over, import, or migrate their setup or config from another agent or tool.
  Do not use for OpenCode V1 to V2 migration (use the opencode skill),
  installing Marketplace plugins, or authoring a new OpenCode plugin from
  scratch.
---

# Migrate agent settings into OpenCode

Collect what the user set up in other coding agents and write it into **OpenCode
V2 native locations**. Prefer file-based definitions (`skills/`, `commands/`,
`agents/`, instruction files) plus a merge into `opencode.json(c)` for MCP.
Do **not** wrap the import in an OpenCode plugin: OpenCode plugins are code
packages, not settings bundles.

Work in four steps: discover, confirm, write, hand back. **Never write anything
before the user confirms the plan in Step 2.** Never modify or delete the source
files of the other agents.

Treat this as OpenCode V2. Fetch
[Skills](https://opencode.ai/docs/skills/),
[MCP servers](https://opencode.ai/docs/mcp-servers/),
[Agents](https://opencode.ai/docs/agents/),
[Commands](https://opencode.ai/docs/commands/),
[Instructions](https://opencode.ai/docs/instructions/), and
[Config](https://opencode.ai/docs/config/) when a field or path is uncertain.
Do not fall back to OpenCode V1 field names (`permission`, `bash`, `mcp` as a
flat server map, `plugin`, `command`, `agent`).

On Windows, `~` is `%USERPROFILE%`. OpenCode still accepts `~/` in config.
OpenCode's `shell` tool is PowerShell: call `npx.cmd` / `npm.cmd` from the
shell, but keep MCP `command` arrays as process argv (rewrite `npx` to
`npx.cmd` only when the source is being migrated on Windows).

## Destination map

Default to **global** OpenCode config so the import applies everywhere. Use
project files only when the user asks.

| Kind | Global | Project |
| --- | --- | --- |
| Config / MCP | `~/.config/opencode/opencode.jsonc` (or `.json`) | `<workspace>/opencode.jsonc` or `<workspace>/.opencode/opencode.jsonc` |
| Skills | `~/.config/opencode/skills/<id>/SKILL.md` | `<workspace>/.opencode/skills/<id>/SKILL.md` |
| Commands | `~/.config/opencode/commands/<name>.md` | `<workspace>/.opencode/commands/<name>.md` |
| Agents | `~/.config/opencode/agents/<name>.md` | `<workspace>/.opencode/agents/<name>.md` |
| Instructions | new files under `~/.config/opencode/instructions/`, listed in `instructions` | project `AGENTS.md` or `.opencode` `instructions` |

Prefer `opencode.jsonc` over `.json` when either exists. Preserve `$schema`,
comments, and every unrelated setting. Config reloads automatically; do not
tell the user to restart OpenCode.

OpenCode already discovers some foreign skill trees. Record that in Step 1
instead of copying blindly:

| Already loaded by OpenCode | Path |
| --- | --- |
| Claude Code skills (compat) | `~/.claude/skills`, `<workspace>/.claude/skills` |
| Codex / agents skills (compat) | `~/.agents/skills`, `<workspace>/.agents/skills` |
| OpenCode skills | `~/.config/opencode/skills`, `<workspace>/.opencode/skills` |

`~/.codex/skills` is **not** an OpenCode compatibility path. Copy those.

## Step 1: Discover

Check each location below and note what exists. Read only what you need: list
directories, and parse JSON or TOML for server names rather than dumping whole
files. Only look at the current workspace's project files if a workspace is
open. Skip any source that is absent; do not ask about it.

Resolve `~` against the user's home directory on this machine. On Windows also
accept `%USERPROFILE%`.

| Source | MCP servers | Skills / commands | Subagents | Instructions |
| --- | --- | --- | --- | --- |
| Claude Code | `~/.claude.json` top-level `mcpServers`; `~/.claude/settings.json`; `<workspace>/.mcp.json` | `~/.claude/skills/*/SKILL.md`; `~/.claude/commands/*.md` | `~/.claude/agents/*.md` | `~/.claude/CLAUDE.md`; `<workspace>/CLAUDE.md` |
| Codex | `~/.codex/config.toml` `[mcp_servers.<name>]` | `~/.codex/skills/*/SKILL.md`; `~/.agents/skills/*/SKILL.md`; `~/.codex/prompts/*.md` | — | `~/.codex/AGENTS.md` |
| Cursor | `~/.cursor/mcp.json`; `<workspace>/.cursor/mcp.json` | — | — | `<workspace>/.cursor/rules/*.mdc` |
| Gemini CLI | `~/.gemini/settings.json` `mcpServers` | `~/.gemini/commands/**/*.toml`; `~/.gemini/skills/*/SKILL.md` | — | `~/.gemini/GEMINI.md` |
| Antigravity | `~/.gemini/config/mcp_config.json`; `~/.gemini/config/plugins/*/mcp_config.json` | `~/.gemini/config/skills/*/SKILL.md`; `~/.gemini/config/plugins/*/skills/*/SKILL.md` | `~/.gemini/config/plugins/*/agents/*.md` | `~/.gemini/config/plugins/*/rules/*.md`; `~/.gemini/GEMINI.md` |

Also note what OpenCode already has, so the import does not duplicate it:

- Skill IDs under `~/.config/opencode/skills/` and project `.opencode/skills/`
- Command names under `~/.config/opencode/commands/` and `.opencode/commands/`
- Agent IDs under `~/.config/opencode/agents/` and `.opencode/agents/`
- Built-in agents: `build`, `plan`, `general`, `explore` (do not overwrite)
- MCP server names in the target `opencode.json(c)` under `mcp.servers`
- Global `~/.config/opencode/AGENTS.md` and existing `instructions` entries
- Skills already visible via Claude / `.agents` compatibility paths

Some things don't convert. Record them for the Step 4 report, don't migrate
them:

- Claude Code `hooks` and `permissions` in `settings.json`: tool names
  (`Bash`, `Edit`) and allow-list syntax differ from OpenCode `permissions`.
- Claude Code plugins under `~/.claude/plugins/` and Gemini CLI extensions
  under `~/.gemini/extensions/`: OpenCode has no importer for those packages.
- Cursor user rules and settings stored in its app database.
- Antigravity / Gemini `plugin.json` packages as plugins. Copy their
  **contents** (MCP, skills, agents, rules), not the plugin wrapper.
- MCP servers with the legacy SSE transport (`"type": "sse"`, a URL ending in
  `/sse`, or Gemini CLI `url` without `httpUrl`). If one has no Streamable HTTP
  endpoint, offer a stdio bridge instead:

  ```jsonc
  {
    "type": "local",
    "command": ["npx", "-y", "mcp-remote", "<url>"]
  }
  ```

  On Windows, use `npx.cmd` as the executable.

## Step 2: Confirm the plan

Show the user a short summary grouped by source: counts plus names (for
example, "Claude Code: 3 MCP servers (github, linear, postgres), 2 skills,
1 instruction file"). List what will be skipped and why, in one line each.
If you include anything from the open workspace (`.mcp.json`,
`.cursor/mcp.json`, `.cursor/rules/`, `CLAUDE.md`) and the destination is
global, say that it will apply in every workspace, so the user can leave it
out.
Never print secret values (tokens, API keys, passwords, `Authorization`
headers) in chat; refer to them by key name only.

Ask once, in a single message:

1.  Proceed with everything, or leave some items out?
2.  Write globally to `~/.config/opencode/` (default) or only into this
    workspace's `.opencode/` / `opencode.jsonc`?
3.  For Claude Code skills that OpenCode already loads from `~/.claude/skills`,
    skip them (default) or copy them into `~/.config/opencode/skills/` so they
    no longer depend on Claude Code?

Wait for the answer. Do not invent a plugin name.

## Step 3: Write into OpenCode

Create only the parts you need. Never overwrite an existing OpenCode skill,
command, agent, or MCP server of the same ID. On a clash, prefix with the
source (`claude-github`, `cursor-review`) or skip and report it.

### MCP servers

Merge into the target config's `mcp.servers`. V2 does **not** put server names
directly under `mcp`. Create `mcp` / `servers` if missing. Keep unrelated
servers.

Each server needs `type`. Convert as follows.

**Local / stdio**

Source `command` + `args` become one `command` array. Source `env` becomes
`environment`. Source `cwd` stays `cwd`.

```jsonc
{
  "type": "local",
  "command": ["npx", "-y", "example-mcp-server"],
  "cwd": ".",
  "environment": {
    "LOG_LEVEL": "info",
    "MCP_API_KEY": "{env:MCP_API_KEY}"
  }
}
```

- Codex TOML maps directly: `command`, `args`, `env`, `cwd`.
- Make `cwd` absolute when the source used `${workspaceFolder}`, `$HOME`,
  `~`, `%USERPROFILE%`, or a path that only made sense inside the old tool.
  Relative `cwd` in OpenCode resolves from the workspace.
- Resolve paths that pointed into the old tool's own directory (for example
  `${CLAUDE_PLUGIN_ROOT}`, `${extensionPath}`) to absolute paths, or skip the
  server if the path does not exist.
- On Windows, if `command` is `npx` / `npm` / `pnpm`, rewrite the executable
  to `npx.cmd` / `npm.cmd` / `pnpm.cmd`.
- Drop unknown keys (`trust`, `alwaysAllow`, `type` from Claude when it is
  not `sse`/`http`, and Antigravity `variables` objects).

**Remote / HTTP**

Claude Code and Cursor `url`, Codex `url`, Gemini CLI `httpUrl`, and
Antigravity `url` become OpenCode `url`. Keep `headers`.

```jsonc
{
  "type": "remote",
  "url": "https://mcp.example.com/mcp",
  "headers": {
    "Authorization": "Bearer {env:MCP_API_KEY}"
  },
  "oauth": false
}
```

- Leave `oauth` unset (enabled) when the server has no API-key header and
  looks like a normal Streamable HTTP MCP endpoint. Tell the user to sign in
  with `/mcps` after the write.
- Set `oauth: false` only when the source exclusively uses an API key or
  other header credential.
- Codex `http_headers` go into `headers`. Codex `bearer_token_env_var = "X"`
  becomes `"Authorization": "Bearer {env:X}"` and `oauth: false`.
- Codex `env_http_headers` entry `Header = "X"` becomes
  `"Header": "{env:X}"`.

**Secrets and placeholders**

OpenCode substitutes `{env:NAME}` inside JSON strings. `$NAME`, `${NAME}`,
and Cursor `${env:NAME}` are **not** expanded.

- A stdio `environment` entry that only passes a variable through
  (`"X": "${X}"`, `"X": "$X"`, `"X": "${env:X}"`, `"X": "{env:X}"`) is
  redundant: drop it, and the server inherits `X` from the process
  environment.
- Any other secret or placeholder in `command`, `environment`, `url`, or
  `headers`: rewrite to `{env:X}` and tell the user which environment
  variables they must set. Mention `${X:-default}` defaults in the Step 4
  report instead of baking the default into config.
- Keep a literal secret in the file only when the source already stored it
  that way and there is no env var name. Do not echo it. Prefer `{env:}`
  when a name exists.
- Never print secret values.

**Name clashes**

Prefix with the source, such as `claude-github`, when the target already has
that server name.

Prefer `opencode mcp add` only when it will not clobber unrelated config and
the user asked for the CLI. Otherwise edit the config file.

### Skills

Copy each skill **folder** whole (`SKILL.md`, `scripts/`, `references/`,
`assets/`). Paths inside a skill are relative to the directory that contains
`SKILL.md`.

- The skill **ID** is the directory name, not the frontmatter `name`. Use a
  unique lowercase kebab-case directory of 1–64 characters:
  `^[a-z0-9]+(-[a-z0-9]+)*$`.
- Frontmatter `name` is only a display label. Keep it or set it from the
  folder name.
- Add a non-empty `description` if missing (from the opening paragraph) so
  OpenCode will advertise the skill. Skills without a description are not
  shown to the model.
- Claude Code allows a `SKILL.md` without frontmatter; write one.
- Do not copy a skill whose ID already exists in OpenCode, unless the user
  asked to overwrite.
- Skip Claude Code skills that OpenCode already loads from `~/.claude/skills`
  unless Step 2 said to copy them.

### Commands

OpenCode has first-class slash commands. Convert prompts and slash commands
**to commands**, not to skills.

Write `~/.config/opencode/commands/<name>.md` (or project
`.opencode/commands/<name>.md`). Nested Gemini command paths become names
with `/` (`git/commit` → `commands/git/commit.md`, invoked as `/git/commit`).

Keep OpenCode argument placeholders: `$ARGUMENTS`, `$1`, `$2`, …. Do **not**
replace them with “Apply this to what the user asked for.”

- Claude Code `~/.claude/commands/*.md` and Codex `~/.codex/prompts/*.md`:
  copy the Markdown body. Keep YAML frontmatter `description` when present.
  Drop unknown frontmatter (`allowed-tools`, and so on) and mention it.
- Gemini CLI `commands/*.toml`: use `description` in frontmatter and the
  `prompt` string as the body. Rewrite `{{args}}` to `$ARGUMENTS`.
- Nested Gemini paths join with `/` for OpenCode, not `-`.
- Do not put `template` in Markdown frontmatter; the body is the template.
- Skip a command name that already exists.

### Agents (Claude Code and Antigravity)

Write `~/.config/opencode/agents/<name>.md`. The Markdown body is the system
prompt. Use native V2 frontmatter only (do not mix V1 fields such as
`tools`, `permission`, `prompt`, `disable`):

```markdown
---
description: Reviews changes for correctness
mode: subagent
---

Review the current changes. List findings in severity order.
```

- `name`: from the source frontmatter or file name. Prefix it (for example
  `claude-reviewer`) if it clashes with a built-in or existing agent:
  `build`, `plan`, `general`, `explore`, plus any custom ID already present.
  Hidden built-ins `compaction`, `title`, and `summary` must not be reused.
- `description`: required for subagents; write one from the body if missing.
- `mode: subagent` for Claude Code agents and Antigravity agents that were
  subagents (`subagent: true` / `mainAgent: false`). Use `mode: primary`
  only when the source was clearly a primary / mode agent.
- Drop source `model` unless it is already `provider/model` or
  `provider/model#variant` and that provider is configured. Claude-only
  short names (`sonnet`, `claude-sonnet-4-5`) do not belong here.
- Drop `color` unless it is already a six-digit hex value such as `#ff6b6b`.
- Do **not** write a `tools` list. Map tool grants to `permissions`.

If the source had **no** tool list (meaning all tools), omit `permissions`.

If the source listed tools, restrict the agent:

```yaml
permissions:
  - action: "*"
    resource: "*"
    effect: deny
  - action: read
    resource: "*"
    effect: allow
  - action: glob
    resource: "*"
    effect: allow
  - action: grep
    resource: "*"
    effect: allow
```

Map grants:

| Source | OpenCode `action` |
| --- | --- |
| `Read` | `read` |
| `Glob` | `glob` |
| `Grep` | `grep` |
| `Edit`, `MultiEdit`, `Write`, `NotebookEdit` | `edit` |
| `Bash` | `shell` |
| `WebSearch` | `websearch` |
| `WebFetch` | `webfetch` |
| `Agent`, `Task` | `subagent` |
| `Skill` | `skill` |
| Antigravity `read_file` | `read` |
| Antigravity `edit_file`, `write_file` | `edit` |
| Antigravity `run_command` | `shell` |
| Antigravity `manage_task` | `subagent` |
| Antigravity `search_web` | `websearch` |

Drop grants with no equivalent (`TodoWrite`, `NotebookEdit` if you choose not
to map it, MCP-specific tools) and tell the user.

### Instructions

Do **not** overwrite `~/.config/opencode/AGENTS.md`. Write one file per
source under `~/.config/opencode/instructions/`, then add those paths to the
target config `instructions` array (combine with existing entries; do not
replace the array).

```jsonc
{
  "instructions": [
    "~/.config/opencode/instructions/migrated-claude.md",
    "~/.config/opencode/instructions/migrated-codex.md"
  ]
}
```

- Claude `CLAUDE.md`, Codex `AGENTS.md`, Gemini `GEMINI.md`: copy the body
  into `migrated-<source>.md`. Remove lines that only make sense in the old
  tool (for example “use the Bash tool”, “call the Skill tool”) and tell the
  user what you removed.
- Cursor `.mdc` with `alwaysApply: true`: one instruction file per rule,
  named after the rule file.
- Cursor `.mdc` with globs and no `alwaysApply`: OpenCode has no glob
  trigger. List them as skipped unless the user asked to load them always.
- Cursor `.mdc` with only a `description` (on-demand): convert to a **skill**
  named after the file, not an always-on instruction.
- Antigravity `rules/` with `trigger: always_on`: instruction files.
  `trigger: glob` → skip (no glob trigger). `model_decision` / `manual` →
  skill.
- OpenCode V2 recognizes `AGENTS.md` only, not `CLAUDE.md`. Migrating Claude
  instructions is required if the user wants that guidance.

## Step 4: Hand back

1.  Re-read the edited `opencode.json(c)` and confirm it is valid JSONC.
    Re-read any new skill / command / agent files for obvious frontmatter
    errors.
2.  Tell the user, in a few lines: what was migrated, what was skipped and
    why, and anything they still need to do (set `{env:…}` variables, sign in
    to a remote MCP server with `/mcps`).
3.  Point them at the files you wrote. Skills are available from the next
    turn; config, MCP, commands, and agents reload on save. Do not ask them
    to restart.
4.  If a remote MCP server needs OAuth, say it is configured but awaiting
    sign-in until `/mcps` shows connected. Do not run
    `opencode mcp auth` through the shell tool.

## Common mistakes

- Do not write an Antigravity `plugin.json` / `mcp_config.json` tree.
- Do not put MCP servers directly under `mcp`; use `mcp.servers`.
- Do not use V1 names: `permission`, `bash`, `enabled`, `plugin`, `command`,
  `agent`, `prompt`, `tools` on agents.
- Do not convert OpenCode-compatible `$ARGUMENTS` / `$1` into prose.
- Do not append migrated instructions onto an existing `AGENTS.md`.
- Do not print secrets or put them in the Step 2 / Step 4 summary.
- Do not delete or edit Claude Code, Codex, Cursor, Gemini, or Antigravity
  source files.
- Do not treat this skill as OpenCode V1 → V2 migration.
