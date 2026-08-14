# Google Antigravity CLI (`agy -p`)

## Environment inheritance

- The current directory is the workspace. Cached auth and settings come from `~/.gemini/antigravity-cli/settings.json`. Put every task-critical instruction in the prompt.
- Run from the intended workspace. Repeat `--add-dir <path>` only for extra workspace roots; workspace reads and writes are auto-allowed, so each added directory broadens access.
- Pass `--model <model-id>` and `--effort low|medium|high` explicitly. Discover model IDs with `agy models`; do not copy an ID from an example.
- Pass `--agent <name>` only when the caller requests a configured agent. Discover agents with `agy agents`.
- Print mode expands slash commands and skills by default. Add `--disable-slash-commands` when automation must treat the prompt literally.

## Permissions

- Headless runs cannot prompt. An operation left at `ask` is soft-denied: the run can continue, exit 0, and write only a notice to stderr. Pre-authorize every required operation and inspect stderr even after apparent success.
- Prefer narrow `action(target)` entries under `permissions.allow` in `~/.gemini/antigravity-cli/settings.json`. Actions include `read_file`, `write_file`, `read_url`, `execute_url`, `command`, `unsandboxed`, and `mcp`. Deny overrides ask, which overrides allow.
- Do not silently rewrite a developer's persistent permission settings. Have the caller provision the smallest required rules, or use an automation environment that already has them.
- `--mode plan` only for a read-only plan. `--mode accept-edits` when the task must apply file edits unattended; shell, web, MCP, and non-workspace paths still need their own permission rules.
- There is no general-purpose read-only mode for Q&A. For filesystem immutability, use an external read-only mount, container, or disposable copy — not the prompt.
- `--sandbox` restricts terminal commands when the host supports it. It does not approve commands or replace fine-grained rules.
- `--dangerously-skip-permissions` approves every tool call, including writes and commands. Use it only in an externally isolated runner or throwaway working copy, and say so.

## Bounding

- Always set `--print-timeout <duration>`; otherwise print mode waits up to five minutes. Wrap the process in the host's timeout with a slightly larger ceiling so the CLI can emit its terminal result first.
- Keep the prompt and workspace roots narrow. Use a disposable worktree or container when the task is risky or should not affect the caller's checkout.

## Output

- Always pass `--output-format` explicitly. Use `text` only for human-readable output.
- `--output-format json` emits one JSON object after completion. Accept it only when the process exits 0, the object parses, and `.status == "SUCCESS"`; retain `.conversation_id` if continuation may be needed. Final text is `.response`; schema-constrained data is `.structured_output`.
- `--output-format stream-json` emits NDJSON. Require one initial `init` event and exactly one terminal `result` event whose `.result.status` is `SUCCESS`. Reject malformed streams, missing or duplicate terminal events, and every other terminal status.
- Read response deltas only from `step_update` events with `step_type` `agent_response`; concatenate `text_delta` with no extra separators. Ignore unknown event fields.
- Keep stdout and stderr separate: responses and machine-readable events on stdout; diagnostics, permission notices, and authentication errors on stderr.
- Treat every non-zero exit, timeout, `ERROR`, `CANCELED`, `INTERRUPTED`, `INVALID`, `WAITING`, or `RUNNING` terminal status as failure. Even after `SUCCESS`, inspect stderr for soft-denied tools and verify the artifact independently.
- Add `--json-schema '<schema-or-path>'` with JSON or streaming JSON when the caller needs typed output. In streaming mode, the schema applies to the terminal `result` event.

## Resume

- Capture `conversation_id` from JSON output or the streaming `init`/`result` event. Continue with `--conversation "$conversation_id"`; use `--continue` (`-c`) only for the most recent conversation.
- Resume from the intended workspace and repeat model, effort, additional directories, execution mode, sandbox, timeout, and output settings. A resumed run does not restore the caller's flags.

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

- Headless runs reuse cached credentials. If a run reports `authentication required`, stop and tell the user to launch `agy` interactively and complete browser or SSH authentication; do not authenticate inside the headless run.
- If the CLI is missing, point the user to <https://antigravity.google/docs/cli/install>. Do not run the remote installer without permission.
- Never inline, print, or commit credentials to make a run work.

## Antigravity-specific gotchas

- `agy` without `-p`, `--print`, or `--prompt` launches the interactive TUI. Always use print mode for a headless subtask.
- `--mode` selects `plan` or `accept-edits`; it is not a permission policy. Fine-grained rules still govern tool access.
- `--sandbox` restricts terminal execution; it does not make workspace file access read-only.
- A valid `SUCCESS` envelope is not proof that tools ran: an unapproved operation can be soft-denied while the run exits 0. Check stderr and validate side effects independently.
- Do not parse streaming output as one JSON document. Process it one line at a time and wait for the terminal `result` event.
- Consult `agy --help` or the docs only for uncovered behavior or after a flag or output failure: <https://antigravity.google/docs/cli/headless>, <https://antigravity.google/docs/cli/permissions>, and <https://antigravity.google/docs/cli/install>.
