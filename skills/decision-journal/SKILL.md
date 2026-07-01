---
name: decision-journal
description: >-
  Create and maintain Architecture Decision Records (ADRs). Use when the user
  says "decision journal", "log this decision", "ADR", "architecture decision",
  "record this decision", "why did we choose", "document this choice", or when
  a significant technical decision has just been made and should be captured.
---

# Decision Journal

Capture architectural and technical decisions as lightweight ADRs in a
`decisions/` directory. Each decision gets a numbered markdown file that
records the context, options, rationale, and consequences.

## When to Use

Trigger this skill when:
- A meaningful technical choice has been made (framework, pattern, data model,
  API design, dependency, infrastructure).
- The user explicitly asks to record a decision.
- A debate or discussion has concluded and the outcome should be preserved.
- Someone asks "why did we do it this way?" about an existing decision.

Do NOT create ADRs for trivial choices (variable names, formatting preferences,
import ordering).

## Protocol

### Creating a New Decision

#### Step 1 — Identify the Decision

If the user says something like "log this decision" without specifying what,
ask: "What decision are we recording?" Wait for a clear answer.

If you just helped the user make a decision (e.g., via rubber-duck or
agent-convergence), summarize it and ask: "Want me to log this as a decision
record?"

#### Step 2 — Gather Context

Walk through these questions one at a time. If you can answer any by reading
the codebase, do that instead of asking the user.

1. **What is the decision?** — One sentence.
2. **What's the context?** — What problem or situation prompted this decision?
3. **What options did we consider?** — List each option with a brief pro/con.
4. **What did we choose and why?** — The decision and the reasoning.
5. **What are the consequences?** — What becomes easier? What becomes harder?
   What are we giving up?

#### Step 3 — Check for Existing Decisions

Look for a `decisions/` directory in the project root. If it exists, read the
existing ADRs to:
- Determine the next number in the sequence.
- Check if this decision supersedes or relates to an existing one.

If no `decisions/` directory exists, create it.

#### Step 4 — Write the ADR

Create a file at `decisions/NNNN-slug.md` using this format:

```markdown
# NNNN. [Decision Title]

**Date:** YYYY-MM-DD
**Status:** Accepted | Superseded by [NNNN] | Deprecated

## Context

[What situation or problem prompted this decision? Include relevant technical
constraints, business requirements, and team context.]

## Options Considered

### Option A: [Name]
- **Pros:** [...]
- **Cons:** [...]

### Option B: [Name]
- **Pros:** [...]
- **Cons:** [...]

[Add more options as needed]

## Decision

[What we chose and why. Be specific about the reasoning — this is the most
important section. Future readers need to understand not just *what* but *why*.]

## Consequences

### Positive
- [What becomes easier or better]

### Negative
- [What becomes harder or what we're giving up]

### Neutral
- [Trade-offs that are neither clearly good nor bad]
```

#### Step 5 — Confirm and Save

Show the draft to the user. Ask if anything needs adjusting. Write the file
only after confirmation.

### Reviewing Existing Decisions

When the user asks "why did we choose X" or "review our decisions":

1. Read all files in `decisions/`.
2. Find the relevant ADR(s).
3. Summarize the decision and its reasoning.
4. If the codebase has evolved since the decision was made, note whether
   the context or constraints have changed. Flag decisions that might need
   revisiting.

### Revisiting a Decision

When an ADR should be reconsidered:

1. Read the original ADR.
2. Identify what has changed since it was written.
3. Walk through the decision process again with updated context.
4. If the decision changes, create a new ADR that supersedes the old one.
   Update the old ADR's status to "Superseded by [NNNN]".

## Rules

- **Number sequentially.** `0001`, `0002`, etc. Pad to 4 digits.
- **Slug the filename.** Use lowercase-hyphenated titles:
  `0003-use-postgres-over-mongodb.md`.
- **Don't edit old ADRs** (except status). Decisions are a log, not a living
  doc. If a decision changes, write a new one that supersedes it.
- **Keep it brief.** An ADR that takes 20 minutes to read won't get read.
  Aim for 1 page.
- **Capture the "why" not the "what".** The codebase shows *what* was built.
  The ADR should explain *why* it was built that way.
