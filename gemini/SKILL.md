---
name: gemini
description: Use this skill whenever the user wants to delegate a coding task to Gemini, run Gemini on their codebase, get a code review from Gemini, or mentions "gemini" in the context of coding tasks. Triggers on: "use gemini to...", "ask gemini to...", "run gemini on...", "let gemini handle this", "what does gemini think". Use this even if the user just says "gemini" without further detail.
---

# Gemini CLI

Gemini is Google's AI coding agent. Use it to delegate coding tasks or get a second-agent perspective on code.

## Preflight checks

```bash
command -v gemini >/dev/null 2>&1 || { echo "Gemini not found"; exit 1; }

# For write tasks — warn on dirty git state (don't block)
git -C /path/to/project status --porcelain 2>/dev/null | grep -q . && \
  echo "Warning: working directory has uncommitted changes. Gemini will write into this state."
```

## Permission policy

Gemini has explicit approval modes — match them to the task:

| Task type | Approval mode | What it means |
|-----------|--------------|---------------|
| Review / analysis | `--approval-mode plan` | Read-only, no edits |
| Supervised edits | `--approval-mode auto_edit` | Auto-approves file edits only |
| Full autonomy | `--approval-mode yolo` | Auto-approves everything |

**Default to `--approval-mode plan` for any task that doesn't require writing.** Only escalate with user confirmation.

These are two separate control planes — use both for write tasks:
- `--approval-mode` governs auto-consent (whether Gemini asks before acting)
- `-s, --sandbox` governs execution containment (whether shell commands run sandboxed)

Always tell the user the approval mode before running: "I'm going to run Gemini in plan (read-only) mode. OK?"

**Never exceed Claude's own permissions** — as a principle. There is no technical enforcement between Claude's runtime constraints and Gemini's approval mode. This is policy, not a guarantee.

## Choosing the right mode

### Non-interactive — well-defined tasks

Pipe the prompt via stdin to avoid shell injection:

```bash
PROMPT="your prompt here"
OUTPUT=$(mktemp /tmp/gemini_XXXXXX)
echo "OUTPUT: $OUTPUT"
gemini -p "$PROMPT" \
  --approval-mode plan \
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

After a successful run, extract only the structured section from the output (`## Done`, `## Issues`) — don't read the full file. Triage git if available:
```bash
git -C /path/to/project diff --shortstat
```
If >50 files or >500 lines changed, show file list only. If no git, rely solely on the output file.

### Isolated run with git worktree

For write tasks where you want full isolation:
```bash
gemini --worktree feature-name --approval-mode auto_edit -p "your prompt"
```
Gemini creates a new git worktree, works there, and the main branch is untouched. Ideal for risky or experimental changes.

### Resume a session

```bash
gemini --resume latest -p "follow-up message"
gemini --resume 3 -p "follow-up message"    # by index
gemini --list-sessions                       # see available sessions
```

Use this for iterative tasks — cheaper than starting fresh.

### Interactive — open-ended work

Gemini's TUI can't be driven remotely. Give the user the exact command:
```
gemini                    # starts interactive session in current directory
gemini --resume latest    # resume last session
```

## Key flags

| Flag | Purpose |
|------|---------|
| `-p, --prompt` | Non-interactive mode with given prompt |
| `-m, --model` | Model to use |
| `--approval-mode` | `plan` (read-only), `auto_edit`, `yolo` |
| `-s, --sandbox` | Sandboxed execution |
| `-w, --worktree` | Run in a new isolated git worktree |
| `-o, --output-format` | `text`, `json`, `stream-json` |
| `--resume` | Resume a session (`latest` or index number) |
| `--include-directories` | Additional dirs to include in workspace |

## Crafting a good prompt

- Reference files by absolute path — don't inline contents
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
- Pass only what Gemini needs — not Claude's surrounding context
- Gemini supports multi-hop search natively — for research tasks, give it a multi-part question rather than splitting into separate calls

## Workflow

1. **Preflight** — verify gemini is installed
2. **Choose approval mode** — `plan` by default, escalate with user confirmation
3. **Craft a precise prompt** — short, scoped, structured output format, absolute paths
4. **Run in background** — `run_in_background: true`, print the output path
5. **On completion** — check exit code, read first 50 lines, summarize, triage git diff
6. **Let the user decide** — don't commit or further modify Gemini's changes without being asked
