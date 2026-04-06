---
name: codex
description: Use this skill whenever the user wants to delegate a task to Codex, run Codex on their codebase, get a code review from Codex, or mentions "codex" in the context of coding tasks. Triggers on: "use codex to...", "ask codex to...", "run codex on...", "codex review", "let codex handle this", "what does codex think", "delegate to codex". Use this even if the user just says "codex" without further detail.
---

# Codex CLI

Codex is OpenAI's AI coding agent. Use it to delegate coding tasks or get a second-agent perspective on code.

## Preflight checks

Before any exec run:

```bash
# 1. Verify installed
[ -x "$(which codex)" ] || { echo "Codex not found"; exit 1; }

# 2. For write tasks — warn on dirty git state (don't block)
git -C /path/to/project status --porcelain 2>/dev/null | grep -q . && \
  echo "Warning: working directory has uncommitted changes. Codex will write into this state."
```

Codex can authenticate via env var, cached login, or config — don't gate on `OPENAI_API_KEY` alone. If auth fails, Codex will surface the error itself.

## Permission policy

**Always tell the user what you're about to grant before running with write access.**
"I'm going to run Codex with write access to `/path`. OK?"

- Default to `--sandbox read-only` for review/analysis tasks. Use `--sandbox workspace-write` for tasks that require editing files (shown in the exec example above)
- Escalate to `workspace-write` only with user confirmation
- Only use `danger-full-access` (installs, system commands) when explicitly requested
- Never grant more than what Claude itself has been granted in this session

## Choosing the right mode

### Exec (non-interactive) — well-defined tasks

Pass the prompt via stdin to avoid shell injection:

```bash
OUTPUT=$(mktemp /tmp/codex_output_XXXXXX)
echo "OUTPUT: $OUTPUT"
printf '%s' "your prompt here" | codex exec - \
  -C /path/to/project \
  --sandbox workspace-write \
  -c model_reasoning_effort=medium \
  -o "$OUTPUT" 2>&1
EXIT_CODE=$?
[ -s "$OUTPUT" ] && grep -A 50 "## Done" "$OUTPUT" || echo "(no output)"
exit $EXIT_CODE
```

Run this with `run_in_background: true` so Claude isn't blocked. Parse `OUTPUT: /tmp/...` from the task notification to find the file when done.

After a successful run, read only what you need — check the file size first:
```bash
wc -l "$OUTPUT"   # if large, read the first 50 lines only
head -50 "$OUTPUT"
```

Summarize in 3-5 bullet points — don't relay verbatim. Then check git:
```bash
git -C /path/to/project diff --shortstat   # triage first
```
If >50 files or >500 lines changed, show file list only. If small, `git diff --stat` is enough.

If there's no git, use `--skip-git-repo-check` and rely on the output file alone.

### Review — feedback on changes

Pick one mode. `codex review` must run from the project directory — it doesn't accept `-C`:

```bash
OUTPUT=$(mktemp /tmp/codex_output_XXXXXX)
echo "OUTPUT: $OUTPUT"
cd /path/to/project
# Pick one:
codex review --uncommitted > "$OUTPUT" 2>&1
# codex review --base main > "$OUTPUT" 2>&1
# codex review --commit <sha> > "$OUTPUT" 2>&1
# codex review --uncommitted "focus on security" > "$OUTPUT" 2>&1
EXIT_CODE=$?
[ -s "$OUTPUT" ] && head -80 "$OUTPUT" || echo "(no output)"
exit $EXIT_CODE
```

Run with `run_in_background: true`. Summarize the findings — don't dump the full output.

### Resume a session

```bash
codex resume --last          # continue the most recent session
codex resume                 # pick from session list
```

Use this for iterative tasks — cheaper than starting fresh.

### Interactive — open-ended or iterative work

Codex's TUI can't be driven remotely. Give the user the exact command to run themselves:
```
codex "your starting prompt"
codex --full-auto "your starting prompt"   # auto-approves more actions, use with care
```

## Key flags

| Flag | Purpose |
|------|---------|
| `-C <dir>` | Working directory |
| `-m <model>` | Override model (config default: `gpt-5.4`) |
| `-c model_reasoning_effort=medium` | Faster runs for lighter tasks (default is `high`) |
| `--sandbox <mode>` | `read-only`, `workspace-write`, `danger-full-access` |
| `--full-auto` | Shorthand for `-a on-request --sandbox workspace-write` |
| `-o <file>` | Write Codex's final message to a file |
| `--ephemeral` | Don't persist the session to disk |
| `--skip-git-repo-check` | Allow running outside a git repo |
| `--uncommitted` | (review) Review staged + unstaged + untracked changes |
| `--base <branch>` | (review) Review changes against a branch |
| `--commit <sha>` | (review) Review changes introduced by a commit |

## Crafting a good prompt for Codex

Don't pass the user's raw request. Wrap it with context — and keep it lean:
- Use absolute file paths, never relative — Codex reads them itself, don't inline file contents
- State exactly which files or directories to touch and which to leave alone
- Always specify a short, structured output format to keep the response small:
  ```
  Respond in this format only:
  ## Done
  one sentence summary
  ## Changed
  list of files, or "none"
  ## Issues
  anything that needs attention, or "none"
  ```
- Keep scope tight — Codex with write access will do exactly what you ask, broadly interpreted

The goal is for Codex to do the heavy work with its own tokens, not Claude's. A well-scoped prompt + structured response means Claude reads back ~10 lines, not 200.

## Workflow

1. **Preflight** — verify codex is installed, warn on dirty git state for write tasks
2. **Confirm permissions** — tell the user the sandbox level you're about to use
3. **Craft a precise prompt** — absolute paths, clear scope, structured instructions
4. **Run in background** — `run_in_background: true`, print the output path
5. **On completion** — check exit code, summarize output, run diff triage if git is available
6. **Let the user decide** — don't commit or further modify Codex's changes without being asked
