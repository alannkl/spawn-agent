# spawn-agent

An agent skill for delegating a scoped subtask to a headless coding-agent CLI. It supports multiple harnesses through one shared workflow and one tool-specific reference per CLI.

The host—Claude Code, Cursor, Codex, or a shell script—decides when to launch a separate agent process, how to bound it, and how to verify its work. [`SKILL.md`](SKILL.md) is the thin dispatcher: it defines trigger policy, availability checks, permission requirements, run bounds, output handling, and protection against false success. Harness-specific flags, authentication, output contracts, and gotchas live under [`references/`](references/) and are loaded only for the selected CLI.

## Supported harnesses

| Harness | Reference | Official docs |
| ------- | --------- | ------------- |
| Claude Code (`claude -p`) | [`references/claude.md`](references/claude.md) | [headless mode](https://code.claude.com/docs/en/headless), [CLI reference](https://code.claude.com/docs/en/cli-reference) |
| Cursor Agent (`agent -p`) | [`references/cursor.md`](references/cursor.md) | [headless mode](https://cursor.com/docs/cli/headless), [CLI usage](https://cursor.com/docs/cli/using) |
| Codex CLI (`codex exec`) | [`references/codex.md`](references/codex.md) | [non-interactive mode](https://learn.chatgpt.com/docs/non-interactive-mode), [`codex exec` reference](https://learn.chatgpt.com/docs/developer-commands?surface=cli#cli-codex-exec) |

To support another harness, add its reference file and register it in `SKILL.md`.

## Installation

### Skills CLI

```bash
npx skills add alannkl/spawn-agent -g -y
```

Omit `-g` for a project-local install. Use `npx skills add alannkl/spawn-agent --list` to inspect the repo first, and `npx skills remove spawn-agent -g -y` to uninstall.

### Manual

This repository is the skill directory. Clone it, then copy or symlink it into your agent's skills path:

| Scope   | Universal (Cursor, Codex, Claude Code, and others) | Agent-specific (also supported)          |
| ------- | -------------------------------------------------- | ---------------------------------------- |
| Global  | `~/.agents/skills/`                                | `~/.cursor/skills/`, `~/.claude/skills/` |
| Project | `.agents/skills/`                                  | `.cursor/skills/`, `.claude/skills/`     |

```bash
git clone git@github.com:alannkl/spawn-agent.git ~/.agents/skills/spawn-agent
```

If your agent does not read `~/.agents/skills/` directly, link the skill into an agent-specific path:

```bash
ln -sfn ~/.agents/skills/spawn-agent ~/.claude/skills/spawn-agent
```

## Design notes

- **Thin dispatcher, per-harness references.** `SKILL.md` defines the shared sequence: write a self-contained prompt, pre-authorize the required tools, bound the run, and verify the artifact instead of trusting exit 0. Each harness reference owns its flags, permission syntax, authentication guidance, and gotchas.
- **Check installation first; handle login errors reactively.** `command -v` gates dispatch. Authentication is checked by running the command and handling a not-logged-in error; credentials are never inlined to force a run to work.
- **Reject unsupported harnesses explicitly.** If the requested CLI has no reference file, the skill offers a supported harness instead of improvising plausible but incorrect flags.

## License

[MIT](LICENSE)
