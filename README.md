# spawn-agent

An agent skill for spawning a headless coding-agent CLI run as a subagent for a scoped subtask, with multi-harness support — one shared workflow, one reference file per supported CLI.

The host agent — Claude Code, Cursor, Codex, or a shell script — decides when to shell out to a separate agent process, how to bound it, and how to verify what it did. [`SKILL.md`](SKILL.md) is a thin dispatcher holding the harness-agnostic workflow: trigger policy, availability precheck, permission doctrine, bounding, output parsing, and the failure modes that make a spawned run look successful when it did nothing. Everything tool-specific lives in one reference file per harness under [`references/`](references/), loaded only when that harness is targeted.

## Supported harnesses

| Harness | Reference | Official docs |
| ------- | --------- | ------------- |
| Claude Code (`claude -p`) | [`references/claude.md`](references/claude.md) | [headless mode](https://code.claude.com/docs/en/headless), [CLI reference](https://code.claude.com/docs/en/cli-reference) |
| Cursor Agent (`agent -p`) | [`references/cursor.md`](references/cursor.md) | [headless mode](https://cursor.com/docs/cli/headless), [CLI usage](https://cursor.com/docs/cli/using) |

Add another harness by creating its reference file and registering it in `SKILL.md`.

## Installation

### Skills CLI

```bash
npx skills add alannkl/spawn-agent -g -y
```

Omit `-g` for a project-local install. Use `npx skills add alannkl/spawn-agent --list` to inspect the repo first, and `npx skills remove spawn-agent -g -y` to uninstall.

### Manual

This repository is itself the skill directory. Clone it, then copy or symlink it into your agent's skills path:

| Scope   | Universal (Cursor, Codex, Claude Code, and others) | Agent-specific (also supported)          |
| ------- | -------------------------------------------------- | ---------------------------------------- |
| Global  | `~/.agents/skills/`                                | `~/.cursor/skills/`, `~/.claude/skills/` |
| Project | `.agents/skills/`                                  | `.cursor/skills/`, `.claude/skills/`     |

```bash
git clone git@github.com:alannkl/spawn-agent.git ~/.agents/skills/spawn-agent
```

If an agent does not read `~/.agents/skills/` directly, symlink from there:

```bash
ln -sfn ~/.agents/skills/spawn-agent ~/.claude/skills/spawn-agent
```

## Design notes

- **Thin dispatcher, per-harness references.** The shared doctrine (self-contained prompt → pre-authorize → bound → verify the artifact, never trust exit 0) lives once in `SKILL.md`; flags, permission syntax, auth, and gotchas are per-harness and load on demand.
- **Installed-check up front, login-check reactively.** `command -v` gates dispatch, but auth is probed by running the command and handling the harness's not-logged-in error — there is no verified cheap auth-status command to rely on, and credentials are never inlined to make a run work.
- **Unsupported harness → say so.** If no reference file exists for the requested CLI, the skill refuses to improvise flags from memory rather than producing plausible-but-wrong commands.

## License

[MIT](LICENSE)
