# Agentic Agent Skills

> Use one AI to orchestrate others.

Most AI coding workflows hit the same wall: one agent, one context window, one thinking strategy. This repository breaks that constraint.

These are skills (plugins) for **Claude Code**, **Codex**, **Gemini**, **opencode**, and any AI agent with a plugin system — that let your AI act as a **god-model**: an orchestrator that delegates coding tasks to other AI agents running in parallel, each in their own context window, with their own model, permissions, and reasoning approach.

Run Codex on a refactor while Gemini reviews your diff. Spawn a cheap Claude haiku sub-agent for a quick analysis while you continue working. Cross-check a decision by asking three different models and comparing the answers. All from a single session, without polluting your context.

The skills handle the hard parts: permission confirmation, shell safety, background execution, structured output parsing, and result summarization. You just describe the task.

---

## Why

Large tasks don't belong in a single context window. Delegating to a sub-agent means:

- **Specialized capabilities** — agents excel at different things. Gemini has built-in web search and can crawl the web natively. Minimax M2.7 (via opencode) is highly capable and extremely cost-effective. Use the right tool for each job.
- **Different thinking strategies** — each agent reasons differently; running the same problem through multiple models surfaces blind spots and produces better decisions than any single agent would alone
- **Cost efficiency** — offload execution to cheaper models; reserve the orchestrator for decisions that need it. Also lets you balance load across different coding plans and API subscriptions.
- **Parallelism** — run multiple agents simultaneously on independent subtasks
- **Context isolation** — the sub-agent starts fresh, no pollution from the current session

## Skills

| Skill | Agent | Strengths |
|-------|-------|-----------|
| `codex` | [Codex CLI](https://github.com/openai/codex) | Strong sandbox model, `codex review` for git-aware code review |
| `gemini` | [Gemini CLI](https://github.com/google-gemini/gemini-cli) | Fast, built-in web search, `--worktree` for isolated writes, explicit approval modes |
| `opencode` | [opencode](https://opencode.ai) | Session continuity, `--file` attachment, multi-provider model support |
| `claude-code` | [Claude Code](https://claude.ai/code) | Structured JSON output via `--json-schema`, `--allowedTools` for fine-grained control, `--max-budget-usd` cap |

## How It Works

Each skill follows the same orchestration pattern:

1. **Preflight** — verify the agent is installed, warn on dirty git state for write tasks
2. **Confirm permissions** — tell the user what access level you're about to grant before running
3. **Craft a precise prompt** — absolute paths, tight scope, structured output format
4. **Run in background** — non-blocking execution, parse output path from task notification
5. **Summarize** — extract the structured section, triage git diff, report back
6. **Let the user decide** — never commit or push sub-agent changes without explicit instruction

## Installation

Copy the skill directories into your Claude skills folder:

```bash
mkdir -p ~/.claude/skills
cp -r codex gemini opencode claude-code ~/.claude/skills/
```

Or symlink for live updates:

```bash
mkdir -p ~/.claude/skills
for skill in codex gemini opencode claude-code; do
  ln -s "$(pwd)/$skill" ~/.claude/skills/$skill
done
```

## Usage

Invoke naturally in Claude Code:

```
use codex to refactor the auth module
ask gemini to review my uncommitted changes
run opencode on the test suite and fix failures
delegate the database migration to a claude sub-agent
```

Each skill handles permission confirmation, prompt construction, background execution, and result summarization automatically.

## Agent Comparison

| | Codex | Gemini | opencode | Claude Code |
|---|---|---|---|---|
| **Permissions** | `--sandbox`: `read-only` / `workspace-write` / `danger-full-access` | `--approval-mode` + `-s/--sandbox` | None (user confirmation only) | `--permission-mode`: `plan` / `acceptEdits` / `auto` |
| **Prompt input** | stdin (`printf \| codex exec -`) | `-p "$PROMPT"` (string arg) | Positional args (`"$PROMPT"`) | stdin (`printf \| claude -p`) |
| **Output capture** | `-o <file>` | stdout redirect | stdout redirect | stdout redirect |
| **Session resume** | `codex resume --last` | `gemini --resume latest` | `opencode run --continue` | `claude --continue` |
| **Worktree isolation** | — | `-w/--worktree` | — | `-w/--worktree` |
| **Structured output** | Prompt template | Prompt template | Prompt template | `--json-schema` (enforced) |
| **Cost control** | `model_reasoning_effort` config | — | `--model` | `--max-budget-usd`, `--effort`, `--model` |

## Strategy

See [STRATEGY.md](./STRATEGY.md) for the full design rationale: permission policy, token efficiency, shell safety, output handling, and concurrency patterns that apply across all agents.

## Security Risks

Orchestrating agents via CLI introduces attack surface that doesn't exist in single-agent workflows. Be aware of the following before deploying.

**Shell injection.** If a user's prompt is interpolated directly into a shell command, it can contain backticks, subshells, or redirections that execute on the host. Always pipe prompts via stdin (`printf '%s' "$PROMPT" | agent exec -`). Agents that only accept positional args (e.g., opencode) require careful quoting — never construct the command by concatenating raw user input.

**Unauthorized permission escalation.** The god-model decides the sub-agent's sandbox level on behalf of the user. If that decision is made silently or incorrectly, the sub-agent can get write or full-access permissions the user never intended to grant. Always confirm with the user before escalating beyond read-only.

**No technical enforcement of permission boundaries.** The claim "the sub-agent won't exceed the god-model's permissions" is a policy, not a guarantee. There is no enforcement layer between Claude's runtime constraints and a sub-agent's sandbox flags. A misconfigured skill can grant a sub-agent more access than the parent session has.

**Trust chain ambiguity.** The user trusts the god-model. The god-model trusts the sub-agent. But the user never explicitly approved the sub-agent. If the sub-agent is compromised, misbehaves, or is manipulated via prompt injection in a file it reads, the impact radius is the sub-agent's sandbox — not just the god-model's context.

**Prompt injection via file content.** Sub-agents read files from the filesystem. A malicious or compromised file could contain instructions designed to hijack the sub-agent's behavior — exfiltrating content, modifying unintended files, or escalating permissions. Scope sub-agent access to only the directories and files it needs.

**Runaway writes without a safety net.** `workspace-write` and equivalent modes let agents modify files directly. Without a clean git state before the run, there's no clear baseline for what changed. If the agent makes sweeping incorrect changes, recovery depends on git history. Always warn on dirty state; encourage commits before write-enabled runs.

**Concurrent agents on the same workspace.** Two agents running in parallel with write access to the same directory will race. There's no built-in locking. The result is undefined — files can end up in a partially-written, inconsistent state. Don't run parallel write-enabled agents on the same project without coordination.

**Credential exposure.** Sub-agents inherit the shell environment. Any secrets in env vars (API keys, tokens) are visible to the sub-agent and any tools it invokes. Scope `--allowedTools` tightly and avoid giving sub-agents access to credentials they don't need.

---

## Lessons from Building This

These skills were designed iteratively by running all four agents against the same review prompts simultaneously and consolidating findings. A few things that came up:

- **`reasoning_effort: high`** (Codex default) turns a 3-minute task into a 1-hour task. Always override to `medium` for orchestration use.
- **`head -50` truncates the wrong part** — agents emit reasoning before structured output. Use `sed -n '/## Done/,$p'` instead.
- **`mktemp` on macOS** requires X's at the very end — `mktemp /tmp/name_XXXXXX.md` does not substitute the X's.
- **`2>&1` corrupts JSON output** — separate stderr with `2>"${OUTPUT}.err"`.
- **Gemini and opencode finished in seconds** for the same task Codex ran for an hour. Speed varies wildly across agents.
