---
name: spawn-agent
description: Spawn a headless coding-agent CLI run (such as `agy -p`, `claude -p`, `agent -p`, or `codex exec`) for one scoped subtask. Use when the user asks to delegate work to another agent instance, run an agent CLI non-interactively, automate a headless agent in a script, build, or CI step, or fan work out across isolated sessions. Do not use for interactive agent sessions or when the host's own subagent, task, or agent tool already fits.
---

# Spawn Agent

Use a supported coding-agent CLI in headless mode to run one self-contained subtask in a separate process. This file defines the shared workflow. Each harness reference owns its flags, permission syntax, output contract, authentication guidance, and gotchas; read that reference before composing a command.

## Supported harnesses

| Harness | CLI | Reference |
| ------- | --- | --------- |
| Google Antigravity CLI | `agy` | `references/antigravity.md` |
| Claude Code | `claude` | `references/claude.md` |
| Cursor Agent | `agent` | `references/cursor.md` |
| Codex CLI | `codex` | `references/codex.md` |

If the user requests a harness that has no reference file here, such as Gemini CLI, explain that the skill does not support it yet and offer a supported harness. Never improvise unsupported CLI flags from memory; headless interfaces vary across tools and releases.

## Dispatch

1. Select the target harness: use the one the user named; otherwise use the host's CLI if supported; otherwise use an installed supported CLI.
2. Check availability with `command -v <cli>` before composing a command. If the CLI is missing, stop and provide installation guidance; never substitute another harness silently.
3. Read the selected harness's reference file. Follow it for every tool-specific choice, including flags, permissions, output parsing, session resumption, and authentication errors.

Check authentication reactively: run the command, then handle any not-logged-in error as the harness reference directs. Never inline credentials in a command, settings file, or committed script to make a run work.

## Workflow (all harnesses)

1. Decide whether to spawn at all.
   - Spawn when the subtask needs an isolated context window; a different working directory, repository, or Git worktree; a scriptable CI or build step; or an agent host with no suitable built-in subagent facility.
   - Do not spawn when the host's built-in subagent or task tool fits; it retains the host's context and usually costs less. Do not delegate work the host can perform directly.
   - Do not spawn for anything requiring interactive approval or a terminal UI; headless runs cannot prompt a human.

2. Make the subtask self-contained.
   - Assume the spawned run has no conversation history. Put the goal, relevant files or paths, constraints, and exact expected output in the prompt.
   - Pass large inputs as file paths rather than inline text.

3. Pass behavior-affecting settings explicitly.
   - Headless runs inherit local configuration, such as settings, extensions, and project memory, so identical commands can behave differently across machines. Set the model, permission mode, and directory access explicitly according to the harness reference.

4. Pre-authorize every tool the subtask needs.
   - A headless run cannot ask for approval. An unapproved tool call may be denied or skipped while the run still reports success, so under-granting can produce false success rather than a prompt.
   - Use the reference file's permission flags and syntax. Any fully-unattended bypass mode also allows unrestricted shell and network access: use it only in a sandbox or throwaway working copy, and say so when you do.

5. Bound the run before starting it.
   - Set the CLI's turn or iteration limit when available, and wrap the call in the host's timeout mechanism, such as `timeout 600 <cli> ...`.

6. Pick the output format the caller can actually parse.
   - Use the harness's JSON output mode for programmatic use; use its streaming mode for progress on long runs.

7. Run it, then verify the outcome instead of trusting the text.
   - Never treat exit 0 or a plausible result as sufficient proof. Before reporting success, confirm the requested artifact or change exists with `git status`, `git diff`, or direct file inspection.
   - Treat every non-zero status as failure and report stderr with the exit code. If a tool lacked approval, correct the grant from step 4 and rerun; do not switch to a bypass mode merely to suppress the error.

8. Continue a spawned conversation only through the harness's documented session-resume mechanism.

## Gotchas

- Tell the spawned run not to launch further agents unless the user requested nested delegation; nesting can multiply cost without being visible to the caller.
- Do not use a spawned run to launch a long-lived server; background work it starts is torn down when the run ends.
- Flags and behavior change between releases. If a flag fails or an output shape is unexpected, consult the installed CLI's `--help` and the official documentation linked from its reference. Update the reference if it has drifted.
