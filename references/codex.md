# Codex CLI (`codex exec`)

Run `codex exec --help` before composing a command. For behavior not covered there, consult <https://learn.chatgpt.com/docs/non-interactive-mode> and the [`codex exec` command reference](https://learn.chatgpt.com/docs/developer-commands?surface=cli#cli-codex-exec).

## Environment inheritance

- Codex loads user and trusted-project configuration, project instructions, skills, plugins, hooks, and MCP servers. Put every task-critical instruction in the prompt, and pass the model, working directory, sandbox, and approval policy explicitly.
- Use `--cd <path>` for the primary workspace and repeat `--add-dir <path>` only for additional writable roots the task needs.
- Pass `--model <model-id>` explicitly. Use the model requested by the caller or one known to be available in the target environment; do not copy a model ID from an example.
- Add `--strict-config` so unknown configuration fields fail before the run starts.
- Use `--ignore-user-config` or `--ignore-rules` only in controlled automation that must exclude those inputs. `--ignore-rules` excludes execpolicy `.rules` files, not project instructions.

## Permissions

- Set `approval_policy="never"` with `-c` so a headless run fails closed instead of waiting for unavailable human approval. Never use an interactive approval policy in unattended automation.
- Use `--sandbox read-only` for analysis and `--sandbox workspace-write` for edits. Workspace-write does not grant network access by itself.
- Add `--add-dir` for a required writable path outside the main workspace instead of broadening the whole sandbox.
- Use `--sandbox danger-full-access` only in an externally isolated runner. Do not use `--dangerously-bypass-approvals-and-sandbox` merely to resolve a denied command.
- Do not use the deprecated `--full-auto` compatibility flag; select the sandbox explicitly.

## Bounding

- Check `codex exec --help` for a native turn or iteration limit. If none is available, wrap every run in the host's timeout mechanism, such as `timeout 600 codex exec ...`.
- Keep the prompt and writable roots narrow. Use a disposable worktree or container when the task is risky or should not affect the caller's checkout.

## Output

- Without `--json`, Codex writes progress to stderr and the final agent message to stdout. Use this mode only when a human or a simple pipeline needs the final text.
- Use `--json` for programmatic callers. It emits JSONL events, not one JSON object; ignore unknown event fields.
- Capture `thread_id` from `thread.started`. Accept the run only after `turn.completed`. Reject `turn.failed`, `error`, malformed streams, and streams that end without a terminal event.
- Use `--output-last-message <path>` when a caller needs the final message in a file. Use `--output-schema <path>` when the final response must conform to a JSON Schema.
- Treat every non-zero exit as failure and report stderr and the exit code. Even after a terminal success event, verify the requested artifact independently.

## Resume

- Run `codex exec resume --help` before composing a resume command. It rejects `--cd`, `--sandbox`, and `--add-dir`; set its sandbox with `-c 'sandbox_mode="workspace-write"'`.
- Continue a recorded session with `codex exec resume <session_id> "<follow-up>"`. Use `codex exec resume --last "<follow-up>"` only for the most recent session in the current working directory.
- Resume from the same working directory unless the caller intentionally targets another session scope. Add `--all` to `--last` only when it must consider sessions outside the current directory's history.
- Do not use `--ephemeral` when the run may need to resume; ephemeral runs do not persist session files.

## Templates

Set `codex_model_id` to a model available in the target environment.

```bash
codex_model_id='replace-with-an-available-model-id'

# One-off scoped task, read-only JSONL
timeout 300 codex exec --cd "$PWD" --model "$codex_model_id" \
  --sandbox read-only -c 'approval_policy="never"' --strict-config --json \
  "Summarize the public API of src/auth.ts in under 10 bullet points"

# Pipe context in while keeping the instruction explicit
git diff main | timeout 300 codex exec --cd "$PWD" --model "$codex_model_id" \
  --sandbox read-only -c 'approval_policy="never"' --strict-config --json \
  "List typos in this diff as filename:line — nothing else"

# Apply a bounded edit in the selected workspace
timeout 900 codex exec --cd "$PWD" --model "$codex_model_id" \
  --sandbox workspace-write -c 'approval_policy="never"' --strict-config --json \
  "Fix the failing tests in tests/unit; do not change public APIs and do not spawn other agents"

# Request a schema-constrained final response
timeout 300 codex exec --cd "$PWD" --model "$codex_model_id" \
  --sandbox read-only -c 'approval_policy="never"' --strict-config \
  --output-schema ./schema.json --output-last-message ./result.json \
  "Extract the exported function names from src/index.ts"

timeout 900 codex exec resume --model "$codex_model_id" \
  -c 'approval_policy="never"' -c 'sandbox_mode="workspace-write"' \
  --strict-config --json "$thread_id" "Continue the task"
```

After an editing run, require a terminal success event and an artifact check such as `git status --short`, `git diff --check`, the relevant tests, or direct file inspection.

## Authentication

- Let the first run use existing Codex authentication. If authentication is required, stop and tell the user to run `codex login`; do not attempt to authenticate on their behalf.
- In CI, let the existing secret manager expose `CODEX_API_KEY` only to the `codex exec` process. Never inline, print, or commit a credential to make a run work.
- If the CLI is missing, point the user to <https://learn.chatgpt.com/docs/codex/cli>. Do not install it without permission.

## Codex-specific gotchas

- `codex` without `exec` launches the interactive terminal UI. Always use `codex exec` for a headless subtask.
- Codex requires a Git repository by default. Use `--skip-git-repo-check` only when the caller intentionally selected a safe non-repository directory.
- If the prompt is omitted or is `-`, Codex reads the prompt from stdin. When stdin is piped alongside a prompt argument, Codex appends it as context.
- `codex mcp-server` exposes Codex as an MCP server; it does not run a subtask. Call `codex exec` directly.
