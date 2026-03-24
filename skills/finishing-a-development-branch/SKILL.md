---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Security review → Performance review → Present options → Execute choice → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Security Review

Dispatch a security reviewer subagent to scan the changed code for vulnerabilities.

**What to provide the subagent:**
- Output of `git diff <base-branch>...HEAD` (all changed code)
- File list: `git diff --name-only <base-branch>...HEAD`

**Subagent prompt:**
```
You are a security reviewer. Review the following code changes for security vulnerabilities.

## Changed Files
[list of changed files]

## Diff
[full git diff output]

## What to Check

**Critical (block merge):**
- Hardcoded secrets, API keys, passwords, tokens
- SQL/NoSQL injection vulnerabilities
- Command injection (unsanitized input passed to shell/exec)
- Path traversal (user input used in file paths without sanitization)
- XSS: unsanitized user input rendered as HTML (innerHTML, document.write)
- Insecure deserialization
- Authentication/authorization bypasses
- Sensitive data exposed in logs, error messages, or API responses

**Important (should fix):**
- Missing input validation at system boundaries (user input, external APIs)
- CSRF vulnerabilities on state-changing endpoints
- Insecure use of cryptography (MD5/SHA1 for passwords, weak random)
- Overly permissive CORS configuration
- Insecure direct object references (IDs without ownership check)
- Dependency vulnerabilities (newly added packages)

**Context-appropriate (note if applicable):**
- Missing rate limiting on sensitive endpoints
- Missing security headers (CSP, HSTS, X-Frame-Options)
- Overly broad error messages revealing internals

Read the actual changed code. Do not speculate about code you haven't seen.

Report:
- ✅ No security issues found
- ⚠️ Issues found:
  - **Critical:** [list with file:line — must fix before merge]
  - **Important:** [list with file:line — should fix]
  - **Notes:** [context-appropriate observations]
```

**If critical issues found:** Fix them before proceeding. Re-run review after fixes.

**If only important/notes:** Show findings to user, ask whether to fix before merge or track as follow-up issues.

**If no issues:** Continue to Step 3.

### Step 3: Performance Review

Dispatch a performance reviewer subagent to check the changed code for performance issues.

**Subagent prompt:**
```
You are a performance reviewer. Review the following code changes for performance issues.

## Changed Files
[list of changed files]

## Diff
[full git diff output]

## What to Check

**Critical (block merge):**
- N+1 query patterns (queries inside loops without batching)
- Unbounded loops or recursion with no termination guarantee
- Memory leaks (event listeners, timers, or large objects never released)
- Blocking the main thread with synchronous heavy computation
- Loading all data without pagination when dataset can be large

**Important (should fix):**
- Missing database indexes on frequently queried/filtered columns
- Unnecessary re-renders or recomputations on every state change
- Large assets (images, bundles) without optimization or lazy loading
- Repeated identical computations that could be memoized/cached
- Synchronous file I/O or blocking calls in async contexts

**Context-appropriate (note if applicable):**
- Missing debounce/throttle on high-frequency event handlers (scroll, resize, input)
- Eager loading of data/modules that could be deferred
- Expensive operations on the hot path (inside render, per-request, in tight loops)
- Large inline data that could be fetched lazily

Read the actual changed code. Only report issues visible in the diff.

Report:
- ✅ No performance issues found
- ⚠️ Issues found:
  - **Critical:** [list with file:line — must fix before merge]
  - **Important:** [list with file:line — should fix]
  - **Notes:** [context-appropriate observations]
```

**If critical issues found:** Fix them before proceeding. Re-run review after fixes.

**If only important/notes:** Show findings to user, ask whether to fix before merge or track as follow-up issues.

**If no issues:** Continue to Step 4.

### Step 4: Determine Base Branch

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

### Step 5: Present Options

Present exactly these 4 options:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Don't add explanation** - keep options concise.

### Step 6: Execute Choice

#### Option 1: Merge Locally

```bash
# Switch to base branch
git checkout <base-branch>

# Pull latest
git pull

# Merge feature branch
git merge <feature-branch>

# Verify tests on merged result
<test command>

# If tests pass
git branch -d <feature-branch>
```

Then: Cleanup worktree (Step 7)

#### Option 2: Push and Create PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

Then: Cleanup worktree (Step 7)

#### Option 3: Keep As-Is

Report: "Keeping branch <name>. Worktree preserved at <path>."

**Don't cleanup worktree.**

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

If confirmed:
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

Then: Cleanup worktree (Step 7)

### Step 7: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.

## Quick Reference

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | ✓ | - | - | ✓ |
| 2. Create PR | - | ✓ | ✓ | - |
| 3. Keep as-is | - | - | ✓ | - |
| 4. Discard | - | - | - | ✓ (force) |

## Common Mistakes

**Skipping test verification**
- **Problem:** Merge broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Skipping security or performance review**
- **Problem:** Vulnerabilities or regressions ship silently
- **Fix:** Always run both reviews after tests pass, before presenting options

**Open-ended questions**
- **Problem:** "What should I do next?" → ambiguous
- **Fix:** Present exactly 4 structured options

**Automatic worktree cleanup**
- **Problem:** Remove worktree when might need it (Option 2, 3)
- **Fix:** Only cleanup for Options 1 and 4

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

## Red Flags

**Never:**
- Proceed with failing tests
- Skip security or performance review
- Merge with known critical security or performance issues
- Delete work without confirmation
- Force-push without explicit request

**Always:**
- Verify tests before offering options
- Run security review (Step 2) and performance review (Step 3)
- Present exactly 4 options
- Get typed confirmation for Option 4
- Clean up worktree for Options 1 & 4 only

## Integration

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-git-worktrees** - Cleans up worktree created by that skill
