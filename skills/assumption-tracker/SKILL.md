---
name: assumption-tracker
description: >-
  Surface, classify, and validate hidden assumptions in plans, specs, or
  codebases. Use when the user says "assumptions", "what are we assuming",
  "validate assumptions", "hidden assumptions", "check my assumptions",
  "what am I missing", or when reviewing a plan or architecture that may
  rest on unverified beliefs.
---

# Assumption Tracker

Systematically extract implicit assumptions from a plan, spec, conversation,
or codebase. Classify each assumption, assess confidence, and propose
validation methods. Maintain a living assumptions register that gets updated
as assumptions are confirmed or invalidated.

Most project failures don't come from risks you identified and accepted —
they come from assumptions you didn't know you were making.

## Protocol

### Step 1 — Identify the Source

Determine what to analyze for assumptions:
- A plan or spec the user just described
- A conversation or decision that was just made
- An existing codebase or architecture
- A feature proposal or PRD

Read the source material thoroughly before proceeding.

### Step 2 — Extract Assumptions

Scan the source for implicit beliefs. Assumptions hide in:

| Signal | Example |
|--------|---------|
| **"Obviously" / "clearly" / "just"** | "We'll just use the existing auth" — assumes auth supports the new use case |
| **Unstated scale** | "Store user preferences" — assumes hundreds of users, not millions |
| **Happy path only** | "User uploads a file and we process it" — assumes valid file, reasonable size, supported format |
| **Dependency stability** | "We'll call the Stripe API" — assumes Stripe's API won't change, rate limits are sufficient |
| **Team knowledge** | "The team will maintain this" — assumes someone understands it 6 months from now |
| **Infrastructure** | "Deploy to production" — assumes CI/CD exists, infra can handle it |
| **User behavior** | "Users will fill out all fields" — assumes compliance with intended flow |
| **Timing** | "This will take 2 weeks" — assumes no blockers, context switches, or scope changes |

For each assumption found, record:
1. **The assumption itself** — stated clearly.
2. **Where it came from** — quote the source or reference the code.
3. **Why it matters** — what breaks if this assumption is wrong?

### Step 3 — Classify and Assess

Present assumptions in a structured register:

```
## Assumptions Register

| # | Assumption | Category | Confidence | Impact if Wrong | Source |
|---|-----------|----------|-----------|----------------|--------|
| 1 | [Clear statement] | Technical | High/Med/Low | [What breaks] | [Where you found it] |
| 2 | ... | Business | ... | ... | ... |
```

#### Categories

- **Technical** — how the system works (APIs, databases, performance)
- **Business** — market, users, revenue, requirements
- **User Behavior** — how people will actually use the system
- **Infrastructure** — deployment, scaling, environments
- **Team** — skills, availability, knowledge retention
- **Timeline** — estimates, deadlines, sequencing
- **External** — third-party services, regulations, market conditions

#### Confidence Levels

- **High** — verified by evidence (code exists, docs confirm, tested)
- **Medium** — reasonable belief but not verified
- **Low** — gut feeling or inherited from someone else's claim

### Step 4 — Validation Plan

For each Medium or Low confidence assumption, propose how to validate it:

```
## Validation Plan

### Assumption #1: [Statement]
- **Current confidence:** Medium
- **Validation method:** [Read the code / Run a test / Ask the user / Check docs / Benchmark]
- **Effort:** [Quick check / Half-day investigation / Needs a spike]
- **Recommendation:** [Validate now before building / Accept the risk / Design to handle both cases]
```

If an assumption can be validated by reading code or running a command, do
it immediately and report the result.

### Step 5 — Present and Prioritize

Show the user the full register and validation plan. Ask them to:

1. **Confirm or correct** each assumption.
2. **Flag any missing assumptions** — the user often knows things the
   codebase doesn't reveal.
3. **Prioritize validation** — which assumptions should be verified before
   building starts?

Use AskQuestion for assumptions where the user needs to make a judgment call.

### Step 6 — Update the Register

As the conversation continues or implementation proceeds, update the register:

- Assumptions that are validated → mark as **Confirmed** with evidence.
- Assumptions that are invalidated → mark as **Invalid** and flag the
  impact on the plan.
- New assumptions that surface → add to the register.

If maintaining a file, write the register to `ASSUMPTIONS.md` in the project
root or alongside the relevant plan document.

## Rules

- **Extract, don't invent.** Only flag assumptions that are actually present
  in the source material. Don't manufacture hypothetical risks.
- **Be specific.** "We're assuming the database can handle the load" is
  vague. "We're assuming PostgreSQL on a db.t3.medium can handle 500
  concurrent connections with an average query time under 50ms" is an
  assumption you can actually validate.
- **Validate what you can.** If the codebase or docs can confirm or deny an
  assumption, check immediately instead of leaving it as "Medium confidence."
- **Highlight the dangerous ones.** Low confidence + high impact = the
  assumptions that kill projects. Make these unmissable.
- **Keep the register alive.** An assumptions register that's written once
  and forgotten is just documentation theater. Prompt the user to revisit
  it at key milestones.
