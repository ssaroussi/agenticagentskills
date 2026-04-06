---
name: opencode
description: Use this skill whenever the user wants to delegate a task to opencode, run opencode on their codebase, or mentions "opencode" in the context of coding tasks. Triggers on: "use opencode to...", "ask opencode to...", "run opencode on...", "let opencode handle this". Use this even if the user just says "opencode" without further detail.
---

# opencode CLI

opencode is an AI coding agent. Use it to delegate coding tasks or get a second-agent perspective on code.

## Preflight checks

```bash
command -v opencode >/dev/null 2>&1 || { echo "opencode not found"; exit 1; }

# For write tasks — warn on dirty git state (don't block)
git -C /path/to/project status --porcelain 2>/dev/null | grep -q . && \
  echo "Warning: working directory has uncommitted changes. opencode will write into this state."
```

**No sandbox mode.** Unlike Codex, opencode has no built-in sandbox — it runs with full shell permissions. This makes the permission confirmation step even more important.

## Permission policy

**Always tell the user what opencode will be able to do before running.**
"opencode has no sandbox — it will run with full write access to the filesystem. OK to proceed?"

There is no technical enforcement of read-only behavior — user confirmation is the only gate. A cheaper model does not mean read-only; opencode can still write regardless of model choice.

**Never exceed the orchestrating agent's own permissions** — as a principle. This is policy, not a guarantee.

## Choosing the right mode

### Run (non-interactive) — well-defined tasks

```bash
PROMPT="your prompt here"
OUTPUT=$(mktemp /tmp/opencode_XXXXXX)
echo "OUTPUT: $OUTPUT"
opencode run "$PROMPT" \
  --dir /path/to/project \
  --title "task-slug" \
  > "$OUTPUT" 2>&1
EXIT_CODE=$?
if [ -s "$OUTPUT" ]; then
  sed -n '/## Done/,$p' "$OUTPUT"
else
  echo "(no output)"
fi
exit $EXIT_CODE
```

Omit `--model` to use the user's configured default. If the user mentions a model in their request (e.g. "use GLM", "run with minimax", "use anthropic/claude-sonnet-4-5"), pass it as `--model <provider/model>`. You can ask if unsure, but don't ask unnecessarily — if they said it, use it.

Run with `run_in_background: true`. Parse `OUTPUT: /tmp/...` from the task notification.

For tasks that need specific files, attach them directly instead of mentioning paths in the prompt:
```bash
opencode run "your prompt" \
  --file /abs/path/to/file1.py \
  --file /abs/path/to/file2.py \
  --dir /path/to/project
```

After a successful run, summarize in 3-5 bullet points. Then triage git if available:
```bash
git -C /path/to/project diff --shortstat
```
If >50 files or >500 lines changed, show file list only.

### Interactive — open-ended or iterative work

opencode's TUI can't be driven remotely. Give the user the exact command:
```
opencode /path/to/project
opencode run "starting prompt" --continue   # continue last session
```

### Continue or fork a session

```bash
opencode run "follow-up message" --continue           # continue last session
opencode run "follow-up message" --session <id>       # specific session
opencode run "follow-up message" --session <id> --fork  # fork it
```

Use this when the task is naturally iterative — don't start a fresh session for every follow-up.

## Key flags

| Flag | Purpose |
|------|---------|
| `--dir <path>` | Working directory |
| `-m, --model <provider/model>` | Model to use — omit to use the user's configured default. Good options: `opencode-go/minimax-m2.7` (fast, cost-effective), or any provider/model the user has set up |
| `--variant <level>` | Reasoning effort: `minimal`, `high`, `max` — use `minimal` for light tasks |
| `--file <path>` | Attach file(s) to the prompt — use instead of inlining content |
| `--format json` | Raw JSON events (useful for scripting) |
| `--continue` | Continue the last session |
| `--session <id>` | Target a specific session |
| `--fork` | Fork before continuing |
| `--title <slug>` | Label the session for easier export/reference later |

## Crafting a good prompt

- Keep prompts short and scoped — attach files with `--file` rather than describing their contents
- Always request a structured, short response:
  ```
  Respond in this format only:
  ## Done
  one sentence summary
  ## Changed
  list of files, or "none"
  ## Issues
  anything that needs attention, or "none"
  ```
- Pass only what opencode needs — not Claude's surrounding context

## Exporting session output

If the run output is truncated or you need the full history:
```bash
opencode export          # exports last session as JSON
opencode export <id>     # specific session
```

Use this sparingly — the export can be large. Read only the final assistant message unless you need the full trace.

## Workflow

1. **Preflight** — verify opencode is installed
2. **Confirm permissions** — warn user there is no sandbox, get confirmation for write tasks
3. **Craft a precise prompt** — short, scoped, structured output format, attach files with `--file`
4. **Run in background** — `run_in_background: true`, print the output path
5. **On completion** — check exit code, read first 50 lines, summarize, triage git diff
6. **Let the user decide** — don't commit or further modify opencode's changes without being asked
