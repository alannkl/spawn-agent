---
name: spawn-agent
description: Spawn a headless coding-agent CLI run (such as `agy -p`, `claude -p`, `agent -p`, or `codex exec`) for one scoped subtask. Use when the user asks to delegate work to another agent instance, run an agent CLI non-interactively, automate a headless agent in a script, build, or CI step, or fan work out across isolated sessions. Do not use for interactive agent sessions or when the host's own subagent, task, or agent tool already fits.
---

# Spawn Agent

Use a supported coding-agent CLI in headless mode for one self-contained subtask. This file is the shared workflow; each harness reference owns that CLI's flags, permissions, output, authentication, and gotchas.

## Supported harnesses

| Harness                | CLI      | Reference                   |
| ---------------------- | -------- | --------------------------- |
| Google Antigravity CLI | `agy`    | `references/antigravity.md` |
| Claude Code            | `claude` | `references/claude.md`      |
| Cursor Agent           | `agent`  | `references/cursor.md`      |
| Codex CLI              | `codex`  | `references/codex.md`       |

If the user names a harness with no reference here, such as Gemini CLI, say it is unsupported and offer a supported harness. Never invent CLI flags from memory; headless interfaces vary by tool and release.

## Dispatch

1. Select the harness: the one the user named; otherwise the host's CLI if supported; otherwise an installed supported CLI.
2. Check availability with `command -v <cli>` before composing a command. If it is missing, stop and give installation guidance; never substitute another harness silently.
3. Read that harness's reference. Follow it for flags, permissions, output parsing, session resumption, and authentication errors.

Check authentication reactively: run the command, then handle a not-logged-in error as the reference directs. Never inline credentials in a command, settings file, or committed script to make a run work.

## Workflow (all harnesses)

1. Decide whether to spawn at all.
   - Spawn when the subtask needs an isolated context window; a different working directory, repository, or Git worktree; a scriptable CI or build step; or an agent host with no suitable built-in subagent.
   - Do not spawn when the host's built-in subagent or task tool fits, or for work the host can do directly; it keeps the host's context and usually costs less.
   - Do not spawn for anything that needs interactive approval or a terminal UI; headless runs cannot prompt a human.

2. Make the subtask self-contained.
   - The spawned run has no conversation history. Put the goal, relevant paths, constraints, and exact expected output in the prompt.
   - Pass large inputs as file paths, not inline text.

3. Pass behavior-affecting settings explicitly.
   - Headless runs inherit local settings, extensions, and project memory, so identical commands can differ across machines. Set the model, permission mode, and directory access per the harness reference.

4. Pre-authorize every tool the subtask needs.
   - A headless run cannot ask for approval. An unapproved tool call may be denied or skipped while the run still reports success, so under-granting can look like success.
   - Use the reference's permission flags. Fully unattended bypass also allows unrestricted shell and network access: use it only in a sandbox or throwaway working copy, and say so.

5. Bound the run before starting it.
   - Set the CLI's turn or iteration limit when available, and wrap the call in the host's timeout, such as `timeout 600 <cli> ...`.

6. Pick the output format the caller can parse.
   - JSON for programmatic use; streaming for progress on long runs.

7. Run it, then verify the outcome instead of trusting the text.
   - Exit 0 or a plausible result is not proof. Before reporting success, confirm the artifact with `git status`, `git diff`, or file inspection.
   - Treat every non-zero status as failure and report stderr with the exit code. If a tool lacked approval, correct the grant from step 4 and rerun; do not switch to a bypass mode just to silence the error.

8. Continue a spawned conversation only through the harness's documented session-resume mechanism.

## Gotchas

- Tell the spawned run not to launch further agents unless the user requested nested delegation; nesting can multiply cost without being visible to the caller.
- Do not use a spawned run to launch a long-lived server; background work it starts is torn down when the run ends.
- Flags and behavior change between releases. Do not run `--help` as a preflight. Consult `--help` and the official documentation linked from the harness reference only for uncovered behavior or after a flag or output failure; then update the reference if it has drifted.
