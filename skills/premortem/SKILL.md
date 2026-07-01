---
name: premortem
description: >-
  Structured premortem analysis for software projects. Use when the user says
  "premortem", "what could go wrong", "risk assessment", "before we build",
  "what are the risks", "failure modes", or when evaluating a plan, spec, or
  architecture before committing to implementation.
---

# Premortem

Assume the project has already failed. Work backward to figure out why.

A premortem inverts the usual planning question. Instead of "how do we
succeed?", it asks "we failed — what happened?" This reframe makes it
psychologically easier to surface risks that optimism bias would otherwise hide.

## Protocol

### Step 1 — Gather Context

Understand what's being built. If the user has a plan, spec, or architecture
doc, read it. If they describe it verbally, restate the scope back to them
in 2-3 sentences and confirm.

Identify:
- What is being built
- Who it's for
- What tech stack / dependencies are involved
- What the timeline looks like (if mentioned)

### Step 2 — Set the Scene

Tell the user:

> "It's [timeline] from now. This project has failed. Not a small hiccup —
> it missed its goals entirely. Let's figure out what went wrong."

### Step 3 — Systematic Failure Analysis

Work through each failure category below. For each category, generate 2-3
specific, concrete failure scenarios relevant to **this** project. Do not
produce generic risks — every failure should reference actual components,
dependencies, or decisions from the project.

#### Categories

| Category | What to look for |
|----------|-----------------|
| **Scope** | Feature creep, underestimated complexity, missing requirements discovered late |
| **Technical** | Wrong abstraction, scaling bottleneck, data model that doesn't support future needs |
| **Dependencies** | Third-party API changes, library deprecation, vendor lock-in |
| **Integration** | Services that don't talk to each other correctly, state sync issues, auth flow gaps |
| **Performance** | N+1 queries, unbounded lists, missing pagination, cold start times |
| **Security** | Auth bypass, data exposure, injection vectors, secrets in code |
| **Data** | Migration failures, data loss scenarios, inconsistent state, missing backups |
| **User-facing** | Confusing UX, error messages that don't help, broken flows on edge cases |
| **Operational** | No monitoring, no rollback plan, manual deployment steps, alert fatigue |
| **Team/Process** | Knowledge silos, unclear ownership, blocked on reviews, context switching |

Skip categories that genuinely don't apply. Don't force-fit risks.

### Step 4 — Risk Register

Present all identified risks as a ranked register:

```
## Risk Register

| # | Risk | Category | Likelihood | Impact | Mitigation |
|---|------|----------|-----------|--------|------------|
| 1 | [Specific failure scenario] | [Category] | High/Med/Low | High/Med/Low | [What to do about it] |
| 2 | ... | ... | ... | ... | ... |
```

Sort by `Likelihood x Impact` (High/High at the top).

### Step 5 — Top 3 Deep Dive

For the top 3 risks, go deeper:

For each:
1. **Scenario**: Describe exactly how this failure plays out, step by step.
2. **Early warning signs**: What signals would tell us this is happening?
3. **Prevention**: What can we do now to prevent it?
4. **Mitigation**: If it happens anyway, what's the recovery plan?
5. **Cost of prevention vs. cost of failure**: Is preventing this worth the
   effort?

Present each one and ask the user: "Does this risk feel real? Anything to
add or adjust?"

### Step 6 — Action Items

Distill the analysis into concrete action items:

```
## Premortem Action Items

### Do Before Building
- [ ] [Action that prevents a top risk]
- [ ] ...

### Build Into the Architecture
- [ ] [Design decision that mitigates a risk]
- [ ] ...

### Monitor After Launch
- [ ] [Signal to watch for]
- [ ] ...
```

## Rules

- **Be specific, not generic.** "Security could be an issue" is useless.
  "The OAuth callback doesn't validate the `state` parameter, enabling CSRF
  on the login flow" is a premortem.
- **Reference the actual project.** Every risk should name a component, file,
  API, or decision from the project being analyzed.
- **Don't catastrophize everything.** A premortem that flags 30 critical risks
  is as useless as one that flags zero. Rank honestly.
- **Explore the codebase.** If the project has existing code, read it. The
  best premortems are grounded in what actually exists, not hypotheticals.
- **Ask, don't lecture.** After presenting risks, ask the user which ones
  feel real and which seem unlikely. Their domain knowledge matters more than
  your speculation.
