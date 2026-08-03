# Cursor Agent (`agent -p`)

Before composing a command, run `agent --help` and confirm it identifies Cursor Agent. Use <https://cursor.com/docs/cli/headless>, <https://cursor.com/docs/cli/using>, and the linked CLI reference pages to resolve behavior that the help output does not explain.

## Environment inheritance

- Cursor loads project rules from `.cursor/rules`, root `AGENTS.md` and `CLAUDE.md`, and configured MCP servers. Treat all of these as ambient behavior and pass task-critical instructions in the prompt.
- Use `--workspace <path>` to select the primary working directory and repeat `--add-dir <path>` only for additional roots the task actually needs. Pass `--trust` only after the caller has decided that the workspace is trusted.
- Pass `--model <model-id>` explicitly. Discover account-specific IDs with `agent models`; do not assume an ID from an example is available.
- `--plugin-dir` adds local plugins. MCP servers may still require approval; use `--approve-mcps` only when the task needs every configured MCP server and the workspace is trusted.

## Permissions

- For read-only work, use `--mode ask` for questions or `--plan` for planning. These modes prevent edits, and neither needs `--force`.
- In print mode, a run without `--force` proposes changes but does not apply them. Add `--force` only when the subtask must edit files or run commands unattended.
- `--force` (also exposed as `--yolo`) broadly auto-allows commands unless explicitly denied; Cursor has no equivalent of Claude Code's per-tool `--allowedTools` list. Use it only in a trusted, tightly scoped workspace. Prefer a disposable worktree, container, or supported Cursor sandbox for risky tasks.
- `--sandbox enabled` requests Cursor's shell sandbox, but support depends on the host. Verify it with a harmless run before relying on it. Inspect the output for sandbox startup errors and treat one as failure even if the process exits 0; never disable the sandbox silently.
- `--auto-review` can still require human approval and is therefore not a substitute for `--force` in a fully unattended run.

## Bounding

- Check `agent --help` for a native turn limit. When none is available, wrap the run in the host's timeout mechanism, for example `timeout 600 agent ...`.
- Keep the prompt and workspace scope narrow. For edits that should not touch the caller's checkout, use `--worktree [name]` and optionally `--worktree-base <ref>`; verify the result in the generated worktree, not the original directory.

## Output

- Always pass `--output-format` explicitly instead of depending on its default.
- Use `--output-format json` for a caller that waits for completion. Accept it only when the response includes `type: "result"`, `subtype: "success"`, `is_error: false`, `result`, and `session_id`.
- Use `--output-format stream-json` for NDJSON progress. Add `--stream-partial-output` only when the caller wants text deltas. Require a terminal success event before accepting the stream.
- Use `--output-format text` only for human-readable output; it is intentionally weaker as a machine contract.
- Reject missing, malformed, or non-success terminal JSON even when the process exits 0. Then verify the requested artifact independently.
- Ignore unknown JSON fields for forward compatibility. For a completed JSON run, extract `.result` and retain `.session_id` if continuation may be needed.

## Resume

- Capture `session_id` from JSON output and continue it with `--resume "$session_id"`. Use `--continue` or `agent resume` for the most recent session.
- Resume from the same workspace and pass the same explicit model, trust, directory, and permission settings. Do not assume the resumed run inherits the caller's flags.
- Use `agent ls` to inspect saved sessions when the intended ID is unknown.

## Templates

Set `cursor_model_id` to an ID returned by `agent models`.

```bash
cursor_model_id='replace-with-a-current-model-id'

# One-off scoped task, read-only
timeout 300 agent -p --mode ask --trust \
  --workspace "$PWD" --model "$cursor_model_id" --output-format json \
  "Summarize the public API of src/auth.ts in under 10 bullet points"

# Read-only plan in human-readable form
timeout 300 agent -p --plan --trust \
  --workspace "$PWD" --model "$cursor_model_id" --output-format text \
  "Plan a minimal fix for the failing authentication tests; do not edit files"

# Apply a bounded edit in the selected workspace
timeout 900 agent -p --force --trust \
  --workspace "$PWD" --model "$cursor_model_id" --output-format json \
  "Fix the failing tests in tests/unit; do not change public APIs and do not spawn other agents"

# Stream progress for a long edit
timeout 900 agent -p --force --trust \
  --workspace "$PWD" --model "$cursor_model_id" --output-format stream-json \
  "Implement the scoped change described in /tmp/task.md and verify its tests"
```

After an editing run, require both a valid terminal success event and an artifact check such as `git status --short`, `git diff --check`, the relevant tests, or direct file inspection.

## Authentication

- Cursor supports browser login and API-key authentication. Let the first run fail reactively; if it reports that authentication is required, stop and tell the user to run `agent login`. `agent status` is available for troubleshooting.
- In CI, an existing secret manager may provide `CURSOR_API_KEY`. Never inline a key with `--api-key`, print it, or commit it to make a run work.
- If the CLI is missing, point the user to <https://cursor.com/docs/cli/installation>. Do not install it without permission.

## Cursor-specific gotchas

- `cursor` is the editor launcher; the Cursor Agent CLI executable is `agent`. After `command -v agent`, confirm `agent --help` identifies Cursor Agent so a different executable with the generic name is not dispatched.
- `--print` is the non-interactive boundary. Do not use an interactive Cursor session for a headless subtask.
- A plausible final message is not proof that commands ran or files changed. Validate the filesystem and tests independently.
- `--force` does not imply that every MCP server is approved; add only the MCP approval needed by the task.
