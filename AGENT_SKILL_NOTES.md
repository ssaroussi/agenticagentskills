# Notes: Designing CLI Agent Orchestration Skills

Considerations when building a skill that lets Claude invoke another AI agent via CLI.

---

## Permission Policy

This is the most important design concern. When Claude invokes a sub-agent, it is making a trust decision on behalf of the user. The user approved Claude's permissions — they did not automatically approve the same permissions for Codex, Gemini, or any other agent.

- **Default to read-only.** Never grant write access without explicit user confirmation. Tell the user what you're about to grant before running: "I'm going to run Codex with write access to `/path/to/project`. OK?"
- **Escalate consciously.** Each agent has its own permission model with different flag names and values — don't treat them as equivalent:
  - Codex: `--sandbox read-only` → `workspace-write` → `danger-full-access`
  - Gemini: `--approval-mode plan` → `auto_edit` → `yolo` (+ separate `-s/--sandbox` flag)
  - Claude Code: `--permission-mode plan` → `acceptEdits` → `auto` → `bypassPermissions`
  - opencode: no sandbox — user confirmation is the only gate
  
  In all cases: start at the most restrictive mode that allows the task, escalate only with user confirmation.
- **Never exceed Claude's own permissions** — as a principle. There is no technical enforcement layer between Claude's runtime constraints and a sub-agent's sandbox settings. This is policy, not a guarantee. Be honest about that with the user.
- **Warn on dirty git state before write operations.** Check `git status --porcelain` and flag it to the user — but don't block. Dirty feature branches are the common case. Offering to auto-stash is risky; leave that decision to the user.

---

## Output handling

- **Summarize, don't relay.** Agent output is verbose. Claude should read it and return a short report: what happened, what changed, anything to flag.
- **Git diff as ground truth.** If the project is a git repo, `git diff --stat` is more reliable than the agent's own description of what it did.
- **Diff triage.** Before reading a full diff, check `git diff --shortstat`. If it's large (>50 files or >500 lines), switch to a file-list summary rather than dumping the full diff.
- **No git fallback.** Not all projects are git repos. Use `--skip-git-repo-check` (or equivalent) and rely solely on the agent's output message.

---

## Temp file management

- **Unique output paths.** Use `mktemp /tmp/prefix_XXXXXX` — no suffix after the X's. On macOS, `mktemp /tmp/name_XXXXXX.md` does NOT substitute the X's; the file is created literally. X's must be the last characters.
- **Background runs: echo the path.** For background tasks, `mktemp` runs inside the subprocess so the path isn't known upfront. Print it explicitly: `echo "OUTPUT: $OUTPUT"` so Claude can parse it from the task notification.
- **Don't delete immediately.** Keep the output file for the duration of the session — the user may ask follow-up questions. Clean up at the start of the next run, not the end of the current one.
- **Variable scoping.** Shell variables don't persist across separate Bash tool calls. For blocking runs, chain exec + read in one call.

---

## Shell safety

- **Never interpolate user input into a shell command.** Prompts passed as raw strings can contain backticks, subshells, or redirections that execute on the host. Always pass prompts via stdin: `printf '%s' "$PROMPT" | agent exec -`. Use `printf '%s'` not `echo` — `echo` has newline and shell-specific behavior that can cause surprises.
- **Quote everything.** All variables in shell commands must be quoted.

---

## Reliability

- **Capture output regardless of exit code.** Don't use `&&` to chain exec and read — if the agent fails, you'll miss its error output. Use: `agent exec ...; EXIT_CODE=$?; cat "$OUTPUT"; exit $EXIT_CODE`
- **Check exit codes explicitly.** Surface failures clearly rather than silently reading an empty file.
- **Empty output.** Check the output file is non-empty before summarizing.
- **Auth failures.** Let the agent surface auth errors itself — don't gate on a specific env var. Agents can authenticate via env var, cached login, or config. Report the error clearly to the user when it happens.
- **Rate limits.** Sub-agents can hit 429s independently of Claude. Treat this as a transient failure — report it to the user and suggest retrying, not as a bug in the skill.
- **Sub-agent doesn't support stdin.** Some agents (e.g., opencode) only accept prompts as positional CLI args, not stdin. In that case, construct the command carefully with proper quoting — never interpolate raw user input directly into the shell string.
- **Binary check.** Verify the agent is installed and executable: `[ -x "$(which agent)" ]`

---

## Concurrency

- **Multiple Claude instances, same project.** Two agents with write access to the same directory will race on files. No easy fix — warn the user if parallelism is involved and consider a lock file.
- **Multiple Claude instances, output files.** Solved by `mktemp`.

---

## When to delegate vs. do it yourself

- **Worth delegating:** well-scoped tasks that would pollute Claude's context (large refactors, whole-codebase scans, test-fix loops).
- **Not worth delegating:** one-liners, simple edits, anything faster to do inline. Every subprocess invocation has token and latency overhead.
- **Background by default.** Use `run_in_background: true` on the Bash tool for exec tasks so Claude isn't blocked. Parse the output path from the task notification when done.

---

## Interactive vs. non-interactive

- **Non-interactive (exec mode):** use for discrete, well-defined tasks. Output is captured and summarized.
- **Interactive (TUI mode):** can't be driven remotely. Give the user the exact command to run themselves.

---

## Token efficiency

The whole point of delegating to a sub-agent is twofold: get a different thinking strategy, and offload token-heavy work away from the god model (Claude). Don't defeat this by loading everything into Claude's context.

- **Don't inline file contents into prompts.** Tell the sub-agent where to find files using absolute paths — let it read them itself. Inlining large files into the prompt burns Claude's tokens and defeats the purpose.
- **Request structured, short output.** Ask the sub-agent to respond in a fixed format (e.g. `## Summary`, `## Changed files`, `## Issues`) with a hard length cap. Don't let it write an essay that Claude then has to read in full.
- **Extract structured sections, don't truncate blindly.** Use `grep -A 50 "## Done" "$OUTPUT"` rather than `head -50` — agents often emit reasoning or preamble before the structured section, so head-truncation misses the actual result.
- **Separate stderr when using JSON output.** `> "$OUTPUT" 2>&1` mixes logs into JSON and breaks parsers. Use `> "$OUTPUT" 2>"${OUTPUT}.err"` instead and check the error file on failure.
- **Pass only what the sub-agent needs.** Don't leak Claude's conversation history or surrounding context into the prompt. Give the minimum: task, file paths, constraints, output format.
- **Claude's role is orchestration.** Deciding what to run, confirming permissions, summarizing results — that's Claude's job. The actual execution and reasoning lives in the sub-agent.

## Prompting sub-agents well

- **Use absolute paths.** Never refer to files by relative name in the prompt — the sub-agent may resolve them from a different working directory. For agents that support `--file` attachment (e.g., opencode), prefer that over mentioning paths in the prompt text — it's more reliable and avoids the agent having to discover files itself.
- **Structure the prompt.** Don't pass the user's raw request. Wrap it with explicit instructions: what to do, what format to respond in, what to skip.
- **Be explicit about scope.** Tell the agent exactly which files or directories to touch and which to leave alone.
- **Use this canonical output template** across all agents — keeps Claude's read-back cheap and consistent:
  ```
  Respond in this format only:
  ## Done
  one sentence summary
  ## Changed
  list of files, or "none"
  ## Issues
  anything that needs attention, or "none"
  ```

---

## Post-run

- **Don't commit automatically.** Present what changed and let the user decide. Never commit or push sub-agent changes without explicit instruction.
- **Offer to resume or fork.** Most agent CLIs support resuming or forking a previous session. Mention this if the result is incomplete or the user wants to iterate.
