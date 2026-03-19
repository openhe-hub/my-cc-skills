# handoff

Claude Code Skill for generating handoff prompts when switching to a new conversation.

## When to Use

- Your conversation is getting long and context is filling up
- You've finished a phase of work and want to start fresh
- You want to hand off to a colleague (or a future you) with full context

## Usage

Copy `SKILL.md` to your project's `.claude/skills/` directory:

```bash
cp SKILL.md /path/to/your/project/.claude/skills/handoff.md
```

Then trigger it in Claude Code:

```
/handoff
/handoff focus on the remaining API endpoints
/handoff the auth refactor is blocked on the upstream PR
```

## What It Does

1. Reviews the current conversation history and project state
2. Checks `git status`, recent commits, and current branch
3. Generates a structured, copy-pasteable handoff prompt containing:
   - What was being worked on
   - What was completed
   - Current project state (branch, uncommitted changes, test status)
   - Key decisions and their rationale
   - Remaining next steps
   - Important files to read first
   - Gotchas and warnings

## Example Output

The skill produces a fenced code block you can paste directly into a new Claude Code session:

```
## Context
Working on adding OAuth2 support to the REST API in the `feature/oauth2` branch.

## What Was Done
- Added OAuth2 middleware in src/middleware/oauth2.ts
- Created token refresh logic in src/auth/refresh.ts
- Updated all protected routes to use new middleware

## Current State
- Branch: `feature/oauth2`
- Uncommitted changes: yes — src/auth/refresh.ts has WIP error handling
- Tests: 3 failing in auth.test.ts (expected, not yet updated for new flow)

## Key Decisions & Context
- Chose PKCE flow over implicit grant for security (see RFC 7636)
- Token storage uses httpOnly cookies, NOT localStorage — team decision from security review

## What Remains
1. Fix the 3 failing tests in src/auth/auth.test.ts
2. Add token revocation endpoint (DELETE /api/auth/token)
3. Update API docs in docs/api.md

## Key Files to Read First
- src/middleware/oauth2.ts — core middleware, start here
- src/auth/refresh.ts — WIP, needs error handling completed

## Gotchas
- Do NOT use localStorage for tokens — see Key Decisions above
- The refresh endpoint returns 401, not 403, on expired tokens (matches existing API convention)
```
