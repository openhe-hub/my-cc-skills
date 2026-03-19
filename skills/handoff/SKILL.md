---
name: handoff
description: Generate a detailed handoff prompt so the next Claude Code conversation can quickly pick up where this one left off. Use when the conversation is getting long, context is filling up, or the current work phase is complete.
argument-hint: "[focus area or notes]"
allowed-tools: Bash, Read, Glob, Grep
---

# Handoff Prompt Generator

You help the user create a **handoff prompt** — a self-contained message that a fresh Claude Code session can use to immediately continue the current work without starting from scratch.

## When the user invokes this skill

Gather context from the current conversation and the project state, then produce a handoff prompt the user can paste into a new session.

If the user provides **"$ARGUMENTS"**, treat it as additional notes or focus areas to emphasize in the handoff.

## Information Gathering

Collect the following from your conversation history and the project:

1. **What was being worked on** — the high-level goal or task
2. **What has been done** — completed steps, files changed, decisions made
3. **What remains** — unfinished steps, known issues, next actions
4. **Key decisions & context** — architectural choices, trade-offs, constraints that aren't obvious from the code alone
5. **Relevant files** — the specific files that the next agent should read first
6. **Current state** — does the code compile/pass tests? Are there uncommitted changes? Is there a branch?

Use tools to verify the current state:

```
git status          # uncommitted changes?
git log --oneline -5  # recent commits in this session?
git branch --show-current  # current branch?
```

## Output Format

Generate the handoff prompt inside a fenced code block so the user can easily copy it. Use this structure:

~~~
```
## Context

[1-3 sentences: what project this is, what the overall goal is]

## What Was Done

- [Completed step 1]
- [Completed step 2]
- [...]

## Current State

- Branch: `<branch-name>`
- Uncommitted changes: [yes/no, brief description]
- Tests: [passing/failing/not run]
- Build: [compiles/broken/N/A]

## Key Decisions & Context

- [Decision or constraint 1 — and WHY]
- [Decision or constraint 2 — and WHY]
- [...]

## What Remains (Next Steps)

1. [Next step 1 — be specific]
2. [Next step 2]
3. [...]

## Key Files to Read First

- `path/to/file1` — [why it's important]
- `path/to/file2` — [why it's important]
- [...]

## Gotchas & Warnings

- [Anything the next agent should watch out for]
```
~~~

## Guidelines

- **Be specific, not vague.** "Fix the auth bug" is bad; "The JWT validation in `src/auth/verify.ts:42` rejects valid tokens when the `iss` claim contains a trailing slash — a fix was started but not completed" is good.
- **Include file paths with line numbers** where relevant so the next agent can jump straight to the right location.
- **Capture the WHY behind decisions.** The next agent can read the code to see WHAT was done, but cannot reconstruct why approach A was chosen over approach B unless you spell it out.
- **Don't dump entire file contents** into the handoff. Reference paths — the next agent has the same tools to read files.
- **Include any failed approaches** so the next agent doesn't repeat them.
- **Keep it concise but complete.** The goal is a prompt that is under ~800 words but contains enough detail that the next agent needs zero follow-up questions to get started.
- **If there are uncommitted changes**, mention them prominently so the next agent doesn't accidentally discard them.
