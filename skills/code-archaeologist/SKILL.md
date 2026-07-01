---
name: code-archaeologist
description: >-
  Reconstruct why code exists using git history, blame, and surrounding context.
  Use when the user says "why does this code exist", "archaeology", "code
  history", "who wrote this and why", "understand this legacy code", "can I
  safely change this", "what's the story behind this", or when investigating
  unfamiliar or seemingly-unnecessary code before modifying it.
---

# Code Archaeologist

Go beyond "explain this code" by reconstructing the *story* of why code
exists — what problem it solved, what constraints shaped it, and whether those
constraints still hold. Produce a narrative report that answers the question
every developer asks about legacy code: "Can I safely change this?"

## Protocol

### Step 1 — Identify the Target

The user will point to code they want to understand. This could be:
- A file or function ("why does `processQueue.ts` exist?")
- A pattern ("why do we wrap every API call in this retry decorator?")
- A mysterious block ("what is this 200-line regex doing?")

Read the code first. Form initial hypotheses about its purpose before
investigating history.

### Step 2 — Git Archaeology

Run these investigations in parallel where possible:

#### Blame Analysis
```bash
git blame <file>
```
Identify who wrote the code and when. Look for:
- Was it written by one person or many?
- Was it written all at once or accumulated over time?
- How recently was it last modified?

#### Commit History
```bash
git log --follow -p -- <file>
```
Read the commit messages and diffs. Reconstruct the timeline:
- What did the original version look like?
- What changed and when?
- Do commit messages explain *why* changes were made?

#### Related Commits
Search for commits that mention the file, function, or related keywords:
```bash
git log --all --oneline --grep="<keyword>"
```

#### PR / Merge Context
Look for merge commits that brought this code in:
```bash
git log --merges --ancestry-path <first-commit>..HEAD -- <file>
```
If the repo uses GitHub, the merge commit message often contains a PR number.

### Step 3 — Context Investigation

Beyond git history, examine:

- **Tests**: Are there tests for this code? What do they test? Test names
  often explain intent better than the code itself.
- **Comments and TODOs**: Read any inline comments, especially ones marked
  `HACK`, `WORKAROUND`, `TODO`, `FIXME`, or `NOTE`.
- **Callers**: Who calls this code? How many callers depend on it? Use grep
  to find all references.
- **Config / Feature Flags**: Is this code behind a flag? Is it
  conditionally enabled?
- **Related Files**: Were other files changed in the same commits? They
  provide context about the broader change.

### Step 4 — Constraint Analysis

For each constraint or design choice you've identified, evaluate:

| Constraint | Still Valid? | Evidence |
|-----------|-------------|----------|
| [e.g., "API rate limit of 100/min"] | Yes / No / Unknown | [How you determined this] |
| [e.g., "MySQL 5.6 didn't support JSON columns"] | No — upgraded to 8.0 | [Evidence from config/deps] |

This is the most valuable part of the report. Code that exists for a reason
that no longer applies is safe to remove. Code that exists for a reason that
still holds is dangerous to touch.

### Step 5 — Archaeology Report

Present findings as a narrative report:

```
## Archaeology Report: [Code Target]

### Summary
[1-2 sentence answer to "why does this code exist?"]

### Timeline
- [Date]: [What happened and why — from commits/PRs]
- [Date]: [Subsequent changes]
- ...

### Original Problem
[What problem this code was written to solve. Cite commit messages, PR
descriptions, or comments as evidence.]

### Constraints That Shaped It
| Constraint | Still Valid? | Evidence |
|-----------|-------------|----------|
| ... | ... | ... |

### Dependencies
- [N] files/functions call this code
- [List the critical callers]

### Safety Assessment
[Can this code be safely modified? What would break? What tests cover it?]

### Recommendations
- [ ] [Concrete recommendation — keep, refactor, remove, or investigate further]
```

### Step 6 — Answer the Real Question

After presenting the report, ask the user: "What were you hoping to do with
this code?" Their answer determines the next step:

- **Remove it** → Confirm no active callers, check test coverage, propose
  a removal plan.
- **Modify it** → Identify the constraints that must be preserved and which
  can be relaxed.
- **Understand it** → Report is sufficient. Ask if any section needs more
  depth.

## Rules

- **Git is your primary source.** Commit messages, blame, and diffs tell the
  story. Read them before speculating.
- **Don't just explain what the code does.** Any agent can do that. Your job
  is to explain *why it exists* and *whether the reasons still hold*.
- **Cite your evidence.** Every claim should reference a commit hash, PR
  number, comment, or file. No hand-waving.
- **Flag uncertainty.** If git history doesn't explain something, say so.
  "I couldn't find evidence for why this retry logic uses exactly 3 attempts"
  is more useful than making something up.
- **Check if constraints still hold.** This is the key insight. Code written
  for MySQL 5.6 limitations, a now-deprecated API, or a requirement that was
  later removed — these are opportunities.
