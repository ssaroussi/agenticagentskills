# Strategy: Orchestrating AI Coding Agents via CLI

A reference for building skills (for Claude Code, Codex, Gemini, opencode, or any AI agent with a skill/plugin system) that delegate work to other AI coding agents via CLI.

The core idea: Claude acts as an orchestrator (god-model) that decides *what* to delegate and *to whom*, while sub-agents handle the actual execution in their own context windows.

---

## Permission Policy

This is the most important concern. When Claude invokes a sub-agent, it is making a trust decision on behalf of the user. The user approved Claude's permissions — they did not automatically approve the same permissions for any sub-agent.

- **Default to read-only.** Never grant write access without explicit user confirmation: *"I'm going to run Codex with write access to `/path/to/project`. OK?"*
- **Escalate consciously.** Each agent has its own permission model — don't treat them as equivalent:
  | Agent | Read-only | Write | Full access |
  |-------|-----------|-------|-------------|
  | Codex | `--sandbox read-only` | `workspace-write` | `danger-full-access` |
  | Gemini | `--approval-mode plan` | `auto_edit` (+ `-s`) | `yolo` |
  | Claude Code | `--permission-mode plan` | `acceptEdits` | `auto` / `bypassPermissions` |
  | opencode | — (no sandbox) | user confirmation only | user confirmation only |
- **Never exceed Claude's own permissions** — as a principle. There is no technical enforcement layer. This is policy, not a guarantee. Be honest about that with the user.
- **Warn on dirty git state before write operations.** Check `git status --porcelain` and flag it — but don't block. Dirty feature branches are the common case. Offering to auto-stash is risky; leave that decision to the user.

---

## When to Delegate

**Worth delegating:**
- Tasks that would pollute Claude's context window (large refactors, whole-codebase scans, test-fix loops)
- Tasks where a second thinking strategy adds value (use a different model/agent for a fresh perspective)
- Long-running tasks that would block Claude while executing

**Not worth delegating:**
- One-liners, simple edits, anything faster to do inline
- Tasks where the overhead of spawning a sub-agent exceeds the benefit

**Background by default.** Use `run_in_background: true` on the Bash tool so Claude isn't blocked. Parse the output path from the task notification when done.

---

## Token Efficiency

The point of delegating is twofold: get a different thinking strategy, and offload token-heavy work away from the god-model. Don't defeat this.

- **Don't inline file contents into prompts.** Give sub-agents absolute paths — let them read files themselves.
- **Request structured, short output.** Use a canonical template (see below). Don't let sub-agents write essays Claude has to read in full.
- **Extract structured sections, don't truncate blindly.** Use `sed -n '/## Done/,$p' "$OUTPUT"` rather than `head -50` or `grep -A 50` — agents often emit reasoning before the structured section, and a fixed line count will truncate long responses.
- **Separate stderr when using JSON output.** `> "$OUTPUT" 2>"${OUTPUT}.err"` keeps JSON clean; check the error file on failure.
- **Pass only what the sub-agent needs.** Don't leak Claude's conversation history. Give the minimum: task, file paths, constraints, output format.
- **Claude's role is orchestration.** Deciding what to run, confirming permissions, summarizing results. The sub-agent handles execution.

### Canonical output template

Use this across all agents — keeps Claude's read-back cheap and consistent:

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

## Prompting Sub-Agents

- **Use absolute paths.** Sub-agents may resolve relative paths from a different working directory.
- **Prefer `--file` attachment** over paths in the prompt text, for agents that support it (e.g., opencode).
- **Structure the prompt.** Wrap the user's raw request with explicit instructions: what to do, what format to return, what to skip.
- **Scope tightly.** Tell the agent exactly which files or directories to touch and which to leave alone.

---

## Shell Safety

- **Never interpolate user input into a shell command.** Prompts containing backticks, subshells, or redirections execute on the host. Always pipe via stdin: `printf '%s' "$PROMPT" | agent exec -`.
- **Use `printf '%s'` not `echo`** — `echo` has shell-specific newline behavior.
- **Quote all variables.**
- **Some agents don't support stdin** (e.g., opencode takes positional args only). In that case, construct the command carefully with proper quoting — never interpolate raw user input directly.

---

## Output Handling

- **Summarize, don't relay.** Read the output and return a short report: what happened, what changed, anything to flag.
- **Git diff as ground truth.** `git diff --stat` is more reliable than the agent's own description.
- **Diff triage.** Check `git diff --shortstat` first. If >50 files or >500 lines, show file-list only.
- **No git fallback.** Not all projects are git repos. Codex supports `--skip-git-repo-check`; for other agents, wrap the diff step with `git rev-parse --is-inside-work-tree >/dev/null 2>&1 && git diff ...` so it silently skips when there's no repo.

---

## Temp File Management

- **Unique output paths.** Use `mktemp /tmp/prefix_XXXXXX` — X's must be the last characters. On macOS, adding a suffix after the X's (e.g. `mktemp /tmp/name_XXXXXX.md`) creates a literal filename without substitution.
- **Echo the path for background runs.** `echo "OUTPUT: $OUTPUT"` lets Claude parse the path from the task notification.
- **Separate stderr for JSON.** Use `> "$OUTPUT" 2>"${OUTPUT}.err"`.
- **Don't delete immediately.** Keep output files for the session in case of follow-up questions.

---

## Reliability

- **Capture output regardless of exit code.** Don't use `&&` — use `; EXIT_CODE=$?` to always read the output even on failure.
- **Avoid `[ -s "$OUTPUT" ] && cmd || fallback`.** If `cmd` fails (e.g. `sed` finds no match), the `||` branch fires incorrectly. Use `if/else` instead.
- **Check exit codes.** Surface failures clearly rather than silently reading an empty file.
- **Check for empty output.** `[ -s "$OUTPUT" ]` before reading.
- **Auth failures.** Let the agent surface them — don't gate on a specific env var. Agents authenticate via env var, cached login, or config.
- **Rate limits.** Sub-agents can hit 429s independently. Treat as transient — report and suggest retry.
- **Binary check.** `command -v agent >/dev/null 2>&1` before running. Avoid `which` — it's not portable across shells.

---

## Concurrency

- **Multiple Claude instances, same project.** Two agents with write access to the same directory will race. No easy fix — warn the user and consider a lock file.
- **Multiple Claude instances, output files.** Solved by `mktemp`.

---

## Post-Run

- **Don't commit automatically.** Present what changed and let the user decide.
- **Offer to resume or fork.** Most agent CLIs support session continuity — mention it if the result is incomplete or the user wants to iterate.
