# Claude Code (`claude -p`)

Harness specifics for spawning a headless Claude Code run. Official source of truth: <https://code.claude.com/docs/en/headless> and <https://code.claude.com/docs/en/cli-reference>. Grounded in live `claude -p` runs, not docs alone.

## Environment inheritance

- A plain `claude -p` run auto-discovers hooks, skills, plugins, MCP servers, auto memory, and `CLAUDE.md` from the working directory and `~/.claude`, and it authenticates with the existing interactive login. Do not add `--bare`: it skips OAuth and keychain reads, so it cannot authenticate without a separate API key.
- Pass what the subtask depends on explicitly instead of assuming it: `--permission-mode`, `--append-system-prompt`, `--add-dir`, `--model`.
- Add `--strict-mcp-config` with `--mcp-config` when the run must see only the MCP servers you name.
- Piped stdin is capped at 10MB; reference a file instead of piping beyond that.

## Permissions

- A tool call that is not pre-approved is skipped silently, not queued for approval: the run can still exit 0 with `is_error` false and no work done.
- Grant coarsely. The run's full toolset depends on the MCP servers and plugins configured locally, so do not try to enumerate exact tool names; name the built-in tools and shell commands the task needs and let the rest go unused.
- Default to `--permission-mode acceptEdits` plus one `--allowedTools` entry per shell command the task needs. `acceptEdits` covers file writes and common filesystem commands, but other shell commands and network access still need an explicit entry.
- Always pass the permission mode explicitly. Inheriting whatever `defaultMode` the machine has set makes the run's autonomy depend on local settings.
- Keep the space before `*` in a permission rule: `Bash(git diff *)` prefix-matches, while `Bash(git diff*)` also matches `git diff-index`.
- `--permission-mode bypassPermissions` (or `--dangerously-skip-permissions`) is the only fully unattended grant, and it also allows unrestricted shell and network access. Use it only in a sandbox or throwaway working copy, and say so when you do.

## Bounding

- Set `--max-turns` for any autonomous run; it exits with an error when the limit is reached.
- Wrap the call in the host's timeout mechanism (for example `timeout 600 claude ...`); the CLI has no timeout flag.
- Set `--fallback-model` for unattended runs so an overloaded primary model does not fail the run.

## Output

- `--output-format json` for programmatic use: `.result`, `.session_id`, `.total_cost_usd`. Add `--json-schema` for typed fields, returned in `.structured_output`.
- `--output-format stream-json --verbose` for progress on long runs; add `--include-partial-messages` for token deltas.
- Exit codes: never treat exit 0 as success — a run whose tool calls were all denied still exits 0 with `is_error` false and a plausible-sounding result; verify the artifact on disk. A non-zero status does mean failure; SIGTERM-terminated runs exit with code 143.

## Resume

- Capture the session ID from JSON output and pass `--resume "$session_id"`, or use `--continue` for the most recent conversation.
- Run resume commands from the same directory: session lookup is scoped to the project directory and its git worktrees.

## Templates

```bash
# One-off scoped task, read-only
claude -p "Summarize the public API of src/auth.ts in under 10 bullet points" \
  --allowedTools "Read,Grep,Glob" --permission-mode default --max-turns 5

# Pipe context in, get JSON out, parse the result
git diff main | claude -p "List typos in this diff as filename:line — nothing else" \
  --output-format json --max-turns 3 | jq -r '.result'

# Let the run edit files, bounded
timeout 900 claude -p "Fix the failing tests in tests/unit; do not change public APIs" \
  --allowedTools "Read,Edit,Bash(npm test *)" --permission-mode acceptEdits --max-turns 30

# Typed output for a caller that needs fields
claude -p "Extract the exported function names from src/index.ts" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  | jq '.structured_output'
```

## Authentication

- Spawned runs reuse the existing interactive login, so the work bills to the account already signed in. Never set `ANTHROPIC_API_KEY` or an `apiKeyHelper` to get a run working: that switches to separate pay-per-token billing.
- If a run reports `Not logged in`, stop and tell the user to sign in with `claude` in a terminal. Do not attempt to authenticate.

## Claude-specific gotchas

- Background Bash tasks the spawned run starts are terminated a few seconds after the final result; background subagents are waited on with a default cap.
- `claude mcp serve` exposes this Claude Code's tools to another MCP client; it does not run a subtask. Third-party `claude-code-mcp` wrappers are unnecessary — call the CLI directly.
- For callbacks, tool-approval hooks, or native message objects, point the user at the Python or TypeScript Agent SDK instead of stretching the CLI.
