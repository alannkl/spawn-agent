---
name: spawn-agent
description: Spawn a headless coding-agent CLI run (for example, `claude -p` or `agent -p`) as a subagent for a scoped subtask. Use when the user asks to delegate work to another agent instance, run an agent CLI non-interactively or headlessly, script or automate a headless agent run, add an agent to a build or CI step, or fan work out across isolated sessions; do not use when the host's own subagent, task, or agent tool already fits, or for interactive agent sessions.
---

# Spawn Agent

Shell out to a coding-agent CLI in headless mode so a separate agent process performs one self-contained subtask and returns its result. This file holds the harness-agnostic workflow. Everything tool-specific — flags, permission syntax, output shapes, auth, gotchas — lives in one reference file per harness, and that file must be read before composing a command.

## Supported harnesses

| Harness | CLI | Reference |
| ------- | --- | --------- |
| Claude Code | `claude` | `references/claude.md` |
| Cursor Agent | `agent` | `references/cursor.md` |

If the user asks for a harness with no reference file here (for example Codex), say this skill does not support it yet and offer a supported one. Do not improvise flags for an unsupported CLI from memory; headless flags differ across tools and releases.

## Dispatch

1. Pick the target harness: the one the user named; otherwise the host's own CLI if it is listed above; otherwise whichever listed CLI is installed.
2. Precheck availability with `command -v <cli>` before composing anything. If the CLI is missing, stop and tell the user how to install it; never substitute a different harness silently.
3. Read the harness's reference file and follow it for every tool-specific decision: flags, permissions, output parsing, resume, auth errors.

Check login reactively, not proactively: run the command and handle the harness's not-logged-in error as its reference file describes. Never inline a credential into a command, settings file, or committed script to make a run work.

## Workflow (all harnesses)

1. Decide whether to spawn at all.
   - Spawn when the subtask needs an isolated context window, a different working directory, repository, or git worktree, a scriptable step for CI or a build, or when the host agent has no subagent facility of its own.
   - Do not spawn when the host's built-in subagent or task tool fits: those share the host's context and cost less. Do not spawn work the host can simply do itself.
   - Do not spawn for anything requiring interactive approval or a terminal UI; headless runs cannot prompt a human.

2. Make the subtask self-contained.
   - Assume the spawned run inherits no conversation history: state the goal, the files or paths, the constraints, and the exact expected output in the prompt itself.
   - Pass large inputs as file paths rather than inline text.

3. Pass behavior-affecting settings explicitly.
   - A headless run inherits local configuration (settings, extensions, project memory), so the same command can act differently on another machine. Set model, permission mode, and directory access explicitly per the reference file instead of relying on local defaults.

4. Pre-authorize every tool the subtask needs.
   - A headless run never asks for approval. An unapproved tool call is denied or skipped, and the run can still report success with no work done — under-granting produces a false success, not a prompt.
   - Use the reference file's permission flags and syntax. Any fully-unattended bypass mode also allows unrestricted shell and network access: use it only in a sandbox or throwaway working copy, and say so when you do.

5. Bound the run before starting it.
   - Set the CLI's turn or iteration limit if it has one, and wrap the call in the host's timeout mechanism (for example `timeout 600 <cli> ...`).

6. Pick the output format the caller can actually parse.
   - Use the harness's JSON output mode for programmatic use; use its streaming mode for progress on long runs.

7. Run it, then verify the outcome instead of trusting the text.
   - Never treat exit 0 or a plausible-sounding result as success. Confirm the intended change exists on disk (`git status`, `git diff`, or read the file) before reporting success.
   - A non-zero status does mean the run failed; report stderr and the exit code. When the result says a tool was not approved, fix the grant from step 4 and re-run — do not escalate to a bypass mode to make the message go away.

8. Continue a spawned conversation only by session, using the harness's resume mechanism from the reference file.

## Gotchas

- Instruct the spawned run not to spawn further agent processes unless the user asked for that; nesting multiplies cost silently.
- Do not use a spawned run to launch a long-lived server; background work it starts is torn down when the run ends.
- Flags and behavior change between releases. When a flag errors or the output shape surprises you, check the installed CLI's `--help` and the official docs linked in the reference file rather than guessing — and update the reference file if it has drifted.
