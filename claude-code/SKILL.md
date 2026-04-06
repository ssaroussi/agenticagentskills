---
name: claude-code
description: Use this skill whenever the user wants to delegate a task to another Claude Code instance, spin up a Claude sub-agent, or use Claude Code non-interactively as part of an orchestration workflow. Triggers on: "use claude code to...", "ask claude to...", "spawn a claude agent", "run claude on...", "delegate to claude". Also use when the task is too large for the current context and would benefit from a fresh Claude instance with a clean context window.
---

# Claude Code CLI (sub-agent)

Use `claude -p` to spawn a Claude Code sub-agent non-interactively. This gives you a fresh context window, fine-grained tool control, and cost tiering — all from within the current session.

## Why use a Claude sub-agent?

- **Context isolation** — the sub-agent starts with a clean context, preventing pollution of the god-model's window
- **Cost tiering** — use `haiku` for cheap/fast subtasks, reserve `sonnet`/`opus` for complex ones
- **Tool restriction** — `--allowedTools` lets you give the sub-agent only the tools it needs
- **Budget cap** — `--max-budget-usd` prevents runaway spend on a delegated task

## Preflight checks

```bash
command -v claude >/dev/null 2>&1 || { echo "Claude Code not found"; exit 1; }

# For write tasks — warn on dirty git state (don't block)
git -C /path/to/project status --porcelain 2>/dev/null | grep -q . && \
  echo "Warning: working directory has uncommitted changes. Claude will write into this state."
```

## Permission policy

Claude Code has explicit permission modes — match to the task:

| Task type | Permission mode | What it means |
|-----------|----------------|---------------|
| Review / analysis | `--permission-mode plan` | Read-only, no edits |
| Supervised edits | `--permission-mode acceptEdits` | Auto-accepts file edits |
| Full autonomy | `--permission-mode auto` | Auto-approves most actions |

**Default to `--permission-mode plan`** for any task that doesn't require writing. Always tell the user before escalating: "I'm going to run a Claude sub-agent with edit access. OK?"

**Never exceed the parent session's own permissions** — as a principle. There is no technical enforcement between the parent Claude session's constraints and the sub-agent's permission mode. This is policy, not a guarantee.

For write tasks, also restrict tools explicitly:
```bash
--allowedTools "Read,Grep,Glob,Bash(git:*)"   # read-only example
--allowedTools "Read,Edit,Write,Bash"           # write-enabled example
```

## Non-interactive run

Pass the prompt via stdin to avoid shell injection:

```bash
OUTPUT=$(mktemp /tmp/claude_XXXXXX)
echo "OUTPUT: $OUTPUT"
printf '%s' "your prompt here" | claude -p \
  --permission-mode plan \
  --output-format text \
  --no-session-persistence \
  --add-dir /path/to/project \
  --max-budget-usd 0.50 \
  > "$OUTPUT" 2>"${OUTPUT}.err"
EXIT_CODE=$?
if [ -s "$OUTPUT" ]; then
  sed -n '/## Done/,$p' "$OUTPUT"
else
  echo "Run failed:"; cat "${OUTPUT}.err" 2>/dev/null
fi
exit $EXIT_CODE
```

Run with `run_in_background: true`. Parse `OUTPUT: /tmp/...` from the task notification.

After completion, extract the structured section (`## Done`, `## Issues`) — don't read the full output. Triage git if available:
```bash
git -C /path/to/project diff --shortstat
```

## Structured output

For machine-readable results, use `--output-format json` or enforce a schema:

```bash
printf '%s' "your prompt" | claude -p \
  --output-format json \
  --json-schema '{"type":"object","properties":{"summary":{"type":"string"},"files":{"type":"array","items":{"type":"string"}},"issues":{"type":"string"}},"required":["summary","files","issues"]}' \
  ...
```

This gives Claude a structured contract to fill — more reliable than hoping it follows a text template.

## Model and cost tiers

| Task | Model | Effort | Approx cost |
|------|-------|--------|-------------|
| Simple read/review | `haiku` | `low` | Very cheap |
| Medium complexity | `sonnet` | `medium` | Moderate |
| Hard reasoning | `sonnet` | `high` | Higher |
| Critical tasks | `opus` | `high` | Expensive |

Omit `--model` to use the default. Override down to `haiku` for cheap tasks, up to `opus` for critical ones.

## Resume or fork a session

```bash
claude --continue -p "follow-up prompt"          # continue last session
claude --resume <session-id> -p "follow-up"      # specific session
claude --resume <session-id> --fork-session -p "variant"  # fork it
```

Use for iterative tasks — much cheaper than re-running from scratch.

## Worktree isolation

For write tasks where you want the main branch untouched:
```bash
claude --worktree feature-name -p "your prompt" --permission-mode auto
```
Claude works in an isolated git worktree. Merge manually when satisfied.

## Key flags

| Flag | Purpose |
|------|---------|
| `-p, --print` | Non-interactive mode |
| `--model <model>` | `haiku`, `sonnet`, `opus` or full model ID |
| `--effort <level>` | `low`, `medium`, `high`, `max` |
| `--permission-mode <mode>` | `plan`, `acceptEdits`, `auto`, `bypassPermissions` |
| `--allowedTools <tools>` | Restrict which tools the sub-agent can use |
| `--disallowedTools <tools>` | Deny specific tools |
| `--max-budget-usd <n>` | Hard spend cap — calibrate to task size: `0.50` for reviews, `2.00` for refactors |
| `--output-format <fmt>` | `text`, `json`, `stream-json` |
| `--json-schema <schema>` | Enforce structured output |
| `--add-dir <path>` | Give sub-agent access to a directory |
| `--no-session-persistence` | Don't save session to disk |
| `--continue` | Continue last session |
| `--resume <id>` | Resume specific session |
| `--worktree [name]` | Run in isolated git worktree |
| `--fork-session` | Create new session ID when resuming (use with `--resume` or `--continue`) |
| `--bare` | Minimal mode — skip hooks, LSP, CLAUDE.md discovery |
| `--append-system-prompt` | Add instructions on top of the default system prompt |

## Crafting a good prompt

- Use absolute paths — don't inline file contents
- Scope tightly — specify exactly what to touch and what to leave alone
- Always request structured output:
  ```
  Respond in this format only:
  ## Done
  one sentence summary
  ## Changed
  list of files, or "none"
  ## Issues
  anything that needs attention, or "none"
  ```
- Pass only what the sub-agent needs — not the current conversation history

## Workflow

1. **Preflight** — verify `claude` is installed
2. **Choose permission mode** — `plan` by default, escalate with user confirmation
3. **Pick model + effort** — cheapest that fits the task
4. **Craft a precise prompt** — absolute paths, structured output format
5. **Run in background** — `run_in_background: true`, print the output path
6. **On completion** — check exit code, extract structured section, triage git diff
7. **Let the user decide** — don't commit or further modify changes without being asked
