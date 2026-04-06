---
name: gemini-search
description: Use Gemini CLI to search the web and research topics. Gemini handles multi-hop searching internally. Use when the user asks to search for something, look something up, or research a topic online. Triggers on: "search for", "look up", "research", "what's the latest on", "find info about".
context: fork
agent: general-purpose
allowed-tools: Bash
---

# Gemini Web Search

Gemini has native Google Search — use it for any task needing current or external information.

For the execution pattern (how to invoke Gemini CLI non-interactively, handle output, errors, permissions), read:
`~/.claude/skills/gemini/SKILL.md`

Use `--approval-mode plan` (read-only).

## Prompt template

Build the `PROMPT` variable with this structure:

```
You are a research agent with access to Google Search.

Research question: {QUERY}

Instructions:
1. Break this into the key sub-questions that need answering.
2. Search for each. If results reveal important new angles, search those too.
3. Synthesize into a structured answer. Be concise — lead with findings, not process.

Output format:
## Key Findings
- bullet points

## Details
only what's needed to understand the findings

## Sources
- source name + direct URL (required — do not omit URLs even if you have to construct them)

## Uncertain / Not Found
anything you couldn't verify
```

For time-sensitive queries, add "as of today" to ground the temporal context.

## Follow-up

If there are clear gaps in the findings, run one targeted follow-up:

```bash
gemini -p "Previous findings: ${PREV_RESULTS}

Follow-up question: {FOLLOWUP}"
```

Cap at 2 total Gemini calls. If still incomplete, return what you have and flag the gaps.

## Output

Summarize Key Findings in 3-5 bullets. Always include source URLs.
