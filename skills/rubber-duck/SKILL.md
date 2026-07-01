---
name: rubber-duck
description: >-
  Structured problem decomposition via Socratic method. Use when the user says
  "help me think through", "I'm stuck", "rubber duck", "walk me through",
  "think this through with me", "before I start coding", or when a request is
  ambiguous enough that jumping straight to implementation would be risky.
---

# Rubber Duck

Force structured thinking before implementation. Decompose the problem through
sequential questioning, build a dependency graph of decisions, surface unknowns,
and only then propose a path forward.

The goal is to prevent the most expensive failure mode in AI-assisted
development: writing hundreds of lines that solve the wrong problem.

## Protocol

### Phase 1 — Problem Statement

Extract a clean problem statement from the user's request. Restate it back to
them in one sentence and ask: "Is that the right problem to solve?"

If the user's request is vague ("make it faster", "fix the auth", "refactor
this"), do NOT proceed until you have a concrete problem statement. Ask:

- What's the symptom you're seeing?
- What does "working" look like?
- Who is affected?

### Phase 2 — Decomposition

Break the problem into sub-problems. For each sub-problem, identify:

1. **What we know** — facts from the codebase, docs, or user input.
2. **What we assume** — things that feel true but haven't been verified.
3. **What we don't know** — gaps that could change the approach.

Present this as a structured list. Ask the user to confirm or correct each
category before proceeding.

### Phase 3 — Decision Tree

Identify the key decisions that must be made before implementation. Order them
by dependency — which decisions gate other decisions?

For each decision point:

1. State the question clearly.
2. List 2-3 concrete options.
3. Give your recommended option with reasoning.
4. Ask the user to choose.

Walk through decisions **one at a time**. Use AskQuestion when the decision is
a clear choice between options. Do not batch decisions — each one may change
the ones that follow.

### Phase 4 — Unknowns Resolution

For each item in the "what we don't know" list from Phase 2:

- Can we answer it by reading the codebase? → Read the code and report back.
- Can we answer it with a quick test? → Propose the test.
- Is it a genuine unknown we need to design around? → Flag it as a risk and
  propose how to handle both cases.

### Phase 5 — Implementation Brief

Only after Phases 1-4 are complete, produce a brief:

```
## Implementation Brief

### Problem
[One sentence]

### Approach
[Chosen path based on decisions made above]

### Steps
1. [Concrete step with file/function references where possible]
2. ...

### Risks
- [Remaining unknowns and how we're handling them]

### Out of Scope
- [Things we explicitly decided NOT to do]
```

Ask the user: "Does this brief look right? Should I start implementing?"

Only begin writing code after explicit confirmation.

## Rules

- **Never skip to code.** The entire point is thinking first. If the user says
  "just do it", acknowledge the request but still run through at least Phase 1
  and Phase 3 (problem statement + key decisions) in compressed form.
- **One question at a time.** Asking five questions at once is not rubber
  ducking — it's a survey. Ask, wait, incorporate, continue.
- **Explore the codebase.** If a question can be answered by reading code,
  read the code instead of asking the user. The user should only answer
  questions that require human judgment.
- **Be concrete.** "Have you considered edge cases?" is a bad question.
  "What happens when `user.email` is null because they signed up via OAuth?"
  is a good one.
- **Track state.** Keep a running summary of decisions made so far. Reference
  it when new decisions depend on earlier ones.
