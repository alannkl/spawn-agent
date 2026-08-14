# Google Antigravity CLI (`agy -p`)

Run `agy --help` before composing a command. For behavior not covered there, consult <https://antigravity.google/docs/cli/headless>, <https://antigravity.google/docs/cli/permissions>, and <https://antigravity.google/docs/cli/install>.

## Environment inheritance

- Antigravity uses the current directory as the active workspace and reuses cached authentication plus settings from `~/.gemini/antigravity-cli/settings.json`. Put every task-critical instruction in the prompt instead of relying on ambient configuration.
- Run from the intended workspace. Repeat `--add-dir <path>` only for additional workspace roots the task needs; workspace file reads and writes are auto-allowed, so every added directory broadens access.
- Pass `--model <model-id>` and `--effort low|medium|high` explicitly. Discover account-specific model IDs with `agy models`; do not copy a model ID from an example.
- Pass `--agent <name>` only when the caller requests a configured agent. Discover available agents with `agy agents`.
- Print mode expands slash commands and skills by default. Add `--disable-slash-commands` when controlled automation must treat the prompt literally and must not invoke those extensions.

## Permissions

- Headless runs cannot prompt for approval. An operation left at `ask` is soft-denied: the run can continue, exit 0, and write only a notice to stderr. Pre-authorize every required operation and inspect stderr even after apparent success.
- Prefer narrow `action(target)` entries under `permissions.allow` in `~/.gemini/antigravity-cli/settings.json`. Available actions include `read_file`, `write_file`, `read_url`, `execute_url`, `command`, `unsandboxed`, and `mcp`. Deny rules override ask rules, which override allow rules.
- Do not silently rewrite a developer's persistent permission settings. Have the caller provision the smallest required rules, or use a dedicated automation environment with those rules already configured.
- Use `--mode plan` only when the requested deliverable is a read-only plan. Use `--mode accept-edits` when the task must apply file edits unattended; shell commands, web access, MCP tools, and non-workspace paths still require their own permission rules.
- Antigravity has no general-purpose read-only mode for arbitrary Q&A. When filesystem immutability is required, enforce it with an external read-only mount, container, or disposable copy rather than relying on the prompt.
- Add `--sandbox` to restrict terminal commands when the host supports its sandbox. It does not approve commands or replace fine-grained permission rules.
- `--dangerously-skip-permissions` approves every tool call, including file writes and commands. Use it only in an externally isolated runner or throwaway working copy, and say so when you do.

## Bounding

- Always set `--print-timeout <duration>`; otherwise print mode waits up to five minutes. Also wrap the process in the host's timeout with a slightly larger ceiling so the CLI can emit its terminal result first.
- Keep the prompt and workspace roots narrow. Use a disposable worktree or container when the task is risky or should not affect the caller's checkout.

## Output

- Always pass `--output-format` explicitly. Use `text` only for human-readable output.
- `--output-format json` emits one JSON object after completion. Accept it only when the process exits 0, the object parses, and `.status == "SUCCESS"`; retain `.conversation_id` if continuation may be needed. Read the final text from `.response` and schema-constrained data from `.structured_output`.
- `--output-format stream-json` emits NDJSON progress. Require one initial `init` event and exactly one terminal `result` event whose `.result.status` is `SUCCESS`. Reject malformed streams, missing or duplicate terminal events, and every other terminal status.
- Read response deltas only from `step_update` events whose `step_type` is `agent_response`; concatenate `text_delta` without adding separators. Ignore unknown event fields for forward compatibility.
- Keep stdout and stderr separate: responses and machine-readable events go to stdout, while diagnostics, permission notices, and authentication errors go to stderr.
- Treat every non-zero exit, timeout, `ERROR`, `CANCELED`, `INTERRUPTED`, `INVALID`, `WAITING`, or `RUNNING` terminal status as failure. Even after `SUCCESS`, inspect stderr for soft-denied tools and verify the requested artifact independently.
- Add `--json-schema '<schema-or-path>'` with JSON or streaming JSON when the caller needs typed output. In streaming mode, the schema applies to the terminal `result` event.

## Resume

- Capture `conversation_id` from JSON output or the streaming `init`/`result` event. Continue that conversation with `--conversation "$conversation_id"`; use `--continue` (`-c`) only for the most recent conversation.
- Resume from the intended workspace and repeat the model, effort, additional directories, execution mode, sandbox, timeout, and output settings. Do not assume a resumed run restores the caller's command-line flags.

## Templates

Set `agy_model_id` to an ID returned by `agy models`. These examples assume the required fine-grained permission rules are already present.

```bash
agy_model_id='replace-with-a-current-model-id'

# Produce a bounded read-only implementation plan
timeout 330 agy -p "Plan a minimal fix for the failing authentication tests; do not edit files" \
  --mode plan --model "$agy_model_id" --effort medium \
  --output-format json --print-timeout 5m

# Apply a bounded edit and stream progress; allow required test commands separately
timeout 930 agy -p "Fix the failing tests in tests/unit; do not change public APIs and do not spawn other agents" \
  --mode accept-edits --sandbox --model "$agy_model_id" --effort medium \
  --output-format stream-json --print-timeout 15m

# Request a schema-constrained result without prompt expansion
timeout 330 agy -p "Parse semantic version v2.14.3 into major, minor, and patch integers" \
  --disable-slash-commands --model "$agy_model_id" --effort low \
  --output-format json --print-timeout 5m \
  --json-schema '{"type":"object","properties":{"major":{"type":"integer"},"minor":{"type":"integer"},"patch":{"type":"integer"}},"required":["major","minor","patch"]}' \
  | jq '.structured_output'
```

After an editing run, require both a valid terminal `SUCCESS` result and an artifact check such as `git status --short`, `git diff --check`, the relevant tests, or direct file inspection.

## Authentication

- Headless runs reuse cached credentials. If a run reports `authentication required`, stop and tell the user to launch `agy` interactively and complete browser or SSH authentication; do not attempt to authenticate inside the headless run.
- If the CLI is missing, point the user to <https://antigravity.google/docs/cli/install>. Do not run the remote installer without permission.
- Never inline, print, or commit credentials to make a run work.

## Antigravity-specific gotchas

- `agy` without `-p`, `--print`, or `--prompt` launches the interactive TUI. Always use print mode for a headless subtask.
- `--mode` selects `plan` or `accept-edits`; it is not a permission policy. Fine-grained rules still govern tool access.
- `--sandbox` restricts terminal execution; it does not make workspace file access read-only.
- A valid `SUCCESS` envelope is not proof that tools ran: an unapproved operation can be soft-denied while the run exits 0. Check stderr and validate the filesystem or other side effects independently.
- Do not parse streaming output as one JSON document. Process it one line at a time and wait for the terminal `result` event.
