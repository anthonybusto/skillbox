---
name: agent-convergence
description: >-
  Run a multi-agent debate loop where agents with distinct perspectives argue
  back and forth about a plan, spec, concept, or design until convergence emerges.
  A dedicated griller agent cross-examines all positions each round to force
  deeper reasoning. Use when the user says "debate", "convergence", "argue this
  out", "stress-test this idea", "devil's advocate", or wants multiple
  viewpoints resolved into a single agreed-upon output.
---

# Agent Convergence

Orchestrate a structured debate between three position-holding subagents and
one cross-examining griller agent. Each cycle has two phases: agents stake
positions, then the griller tears them apart. Repeat until consensus emerges,
then synthesize a final output and grill the user on the result.

## Roles

### Position Agents

Assign three perspectives (adapt labels to the domain):

| Role | Stance | Purpose |
|------|--------|---------|
| **Advocate** | Pushes the idea forward | Finds strengths, opportunities, builds on the proposal |
| **Skeptic** | Challenges assumptions | Finds flaws, risks, edge cases, missing requirements |
| **Pragmatist** | Grounds in reality | Evaluates feasibility, cost, trade-offs, incremental paths |

For domain-specific topics, rename roles to fit (e.g., for architecture:
"Scale Engineer", "Security Reviewer", "DX Champion").

### Griller Agent

A 4th agent with **no position of its own**. Its job:
- Read all three positions after each position phase.
- Generate 2–3 hard, specific questions per agent targeting weak reasoning,
  unstated assumptions, contradictions, and gaps.
- Apply the grill-me philosophy: walk down each branch of the decision tree,
  resolve dependencies between decisions one by one.
- Never accept vague answers. If a position says "we could do X", the griller
  asks "what's the migration path, what breaks, what's the rollback plan?"

## Protocol

### Round 0 — Seed

1. Gather the user's topic. If vague, ask one clarifying question before
   starting.
2. **Position phase**: Launch **three `generalPurpose` subagents in parallel**
   via the Task tool. Each gets:
   - The full user topic/plan/spec.
   - Its assigned role and stance.
   - Instruction to produce a structured position (see Position Format).
   - Instruction to return its position as the final response.
3. Set `run_in_background: false` so you block until all three return.
4. **Grill phase**: Launch **one `generalPurpose` subagent** (the griller)
   with:
   - The original topic.
   - All three positions from the position phase.
   - Instruction to generate 2–3 pointed questions per agent (see Grill
     Format).
   - Block until it returns.

### Rounds 1–N — Debate Cycles

Each round is a two-phase cycle:

**Phase 1 — Positions**: Launch three subagents in parallel, each receiving:
- The original topic.
- Its own role (same throughout).
- All three positions from the previous round.
- **The griller's questions directed at them from the previous grill phase.**
- Instruction: "First, directly answer the griller's questions with specifics.
  Then read the other two positions, identify agreements and disagreements,
  and refine your position. Explicitly flag anything you changed your mind on."

**Phase 2 — Grill**: Launch one griller subagent receiving:
- The original topic.
- All three updated positions.
- The griller's previous questions and how each agent responded.
- Instruction: "Evaluate how well each agent answered your previous questions.
  Flag evasive or weak answers. Generate new questions targeting remaining
  gaps, new contradictions, or claims that still lack evidence."

After each cycle, evaluate convergence by checking:
- Are all three agents agreeing on core decisions?
- Are remaining disagreements narrowing to minor details?
- Is the griller running out of meaningful questions to ask?
- Has any agent stopped changing its position between rounds?

**Stop when any of these are true:**
- All three agents agree on all major points and the griller has no critical
  questions remaining (consensus reached).
- Two consecutive rounds produced no meaningful position changes (stalled).
- Disagreements have narrowed to subjective preferences with no objective
  resolution (agree-to-disagree territory).
- You've hit 5 cycles (hard ceiling to prevent runaway costs).

### Final Synthesis

When the loop ends:

1. Launch **one `generalPurpose` subagent** with:
   - The original topic.
   - All positions from the final round.
   - The griller's final assessment.
   - Instruction: "Synthesize these three positions into a single consensus
     document. For each decision point, state the agreed position. If any
     point remains contested, present both sides and flag it as unresolved.
     Incorporate the griller's final assessment — if the griller flagged
     remaining weaknesses, note them. Use the Consensus Format."

2. Present the consensus document to the user.

### Post-Consensus: Grill the User

After presenting the consensus, apply the grill-me pattern to the user:

1. Walk through each major decision in the consensus document one at a time.
2. For each decision, ask the user a pointed question:
   - "The agents agreed on X because of Y. Does that match your constraints?"
   - "This assumes Z — is that true in your case?"
   - "The agents couldn't resolve A vs B. Which way do you lean and why?"
3. Ask questions **one at a time**. Use AskQuestion when the decision is a
   clear choice between options.
4. If the user challenges or overrides a decision, inject their input and run
   **one more debate cycle** (position + grill) with the user's constraint
   added to all agent prompts, then re-synthesize.
5. If the user agrees with everything, you're done.

## Position Format

Instruct each subagent to structure its output as:

```
## Position: [Role Name]

### Core Stance
[1-2 sentence summary of overall position]

### Key Points
1. [Point] — [Reasoning]
2. [Point] — [Reasoning]
...

### Answers to Griller (rounds 1+)
- Q: [Question] → A: [Direct, specific answer]

### Agreements with Others
- [What you agree with and from whom]

### Disagreements
- [What you challenge and why]

### Changed Minds (rounds 1+)
- [What you previously argued but now concede, and why]
```

## Grill Format

The griller structures its output as:

```
## Cross-Examination: Round [N]

### Questions for Advocate
1. [Specific question targeting a weakness, assumption, or gap]
2. [Follow-up or new angle]
3. [Optional third question if warranted]

### Questions for Skeptic
1. ...

### Questions for Pragmatist
1. ...

### Previous Answer Assessment (rounds 1+)
- Advocate: [Answered well / Evaded question about X / Still lacks evidence for Y]
- Skeptic: [...]
- Pragmatist: [...]

### Overall Gaps
- [Blind spots all three agents share]
- [Questions none of them are asking]
```

## Consensus Format

```
## Consensus: [Topic Title]

### Agreed Decisions
1. [Decision] — [Rationale summarizing why all three converged here]
2. ...

### Unresolved Points
- [Point]: [Side A reasoning] vs [Side B reasoning]

### Griller's Final Assessment
- [Remaining weaknesses or untested assumptions in the consensus]

### Key Insights from Debate
- [Non-obvious insight that emerged from the back-and-forth]

### Recommended Next Steps
1. [Concrete action item]
2. ...
```

## Prompt Templates

### Round 0 position prompt (per agent)

```
You are the [ROLE] in a structured debate about the following topic:

---
[USER TOPIC]
---

Your stance: [STANCE DESCRIPTION]

Produce your initial position using this format:
[Position Format]

Be specific and concrete. Cite requirements, constraints, or evidence where
possible. Return your position as your final response.
```

### Griller prompt (round 0)

```
You are a cross-examiner in a structured debate. You hold no position of your
own. Your job is to find weaknesses, unstated assumptions, contradictions, and
gaps in the three positions below.

Topic:
---
[USER TOPIC]
---

Positions:
---
[ALL THREE POSITIONS]
---

For each agent, generate 2-3 hard, specific questions. Don't ask generic
questions — target the weakest parts of each argument. If an agent made a
claim without evidence, demand evidence. If an agent ignored a scenario,
name it. If two agents contradict each other, force both to address it.

Use the Grill Format. Return your cross-examination as your final response.
```

### Round N position prompt (per agent)

```
You are the [ROLE] in cycle [N] of a structured debate.

Original topic:
---
[USER TOPIC]
---

Previous round positions:
---
[ALL THREE POSITIONS FROM PREVIOUS ROUND]
---

Griller's questions for you:
---
[GRILLER QUESTIONS DIRECTED AT THIS AGENT]
---

Your task:
1. FIRST: Directly answer each of the griller's questions with specifics.
   No hand-waving.
2. Read the other two positions carefully.
3. Identify agreements and disagreements.
4. Refine your position — update, concede, or strengthen points.
5. Explicitly flag anything you changed your mind on.

Return your updated position using the Position Format.
```

### Griller prompt (round N)

```
You are the cross-examiner in cycle [N] of a structured debate.

Topic:
---
[USER TOPIC]
---

Updated positions (including their answers to your previous questions):
---
[ALL THREE UPDATED POSITIONS]
---

Your previous questions:
---
[YOUR PREVIOUS GRILL OUTPUT]
---

Your task:
1. Assess how well each agent answered your previous questions. Flag evasions.
2. Identify new weaknesses, contradictions, or gaps in the updated positions.
3. Generate 2-3 new questions per agent. Don't repeat resolved questions.
4. Note any blind spots all three agents share.

Use the Grill Format. Return your cross-examination as your final response.
```

### Synthesis prompt

```
Three agents debated the following topic across [N] cycles, with a
cross-examiner grilling them each round.

Topic:
---
[USER TOPIC]
---

Final round positions:
---
[ALL THREE FINAL POSITIONS]
---

Griller's final assessment:
---
[FINAL GRILL OUTPUT]
---

Synthesize into a single consensus document using the Consensus Format.
For agreed points, state the decision and the reasoning that won.
For unresolved points, present both sides fairly.
Incorporate the griller's remaining concerns as caveats.
Highlight non-obvious insights that only emerged through the debate.
```

## Guidance

- **Cost per cycle**: 4 subagent calls (3 position + 1 griller). Warn the
  user if the topic is broad enough to run many cycles.
- **Domain adaptation**: Rename roles to fit the domain. For API design:
  "Consumer Advocate", "Platform Engineer", "Security Reviewer". For product
  specs: "User Champion", "Engineering Lead", "Business Analyst".
- **Scope control**: If the topic has multiple independent sub-decisions, split
  them into separate consensus loops rather than one mega-debate.
- **User override**: If the user challenges the consensus during the
  post-consensus grill, run one more cycle with the user's constraint injected,
  then re-synthesize.
- **Griller quality**: The griller is the engine of depth. Its questions should
  be the kind that make the agents uncomfortable — specific, evidence-demanding,
  scenario-based. Generic questions like "have you considered scalability?"
  are failures; "what happens to your proposed cache when you have 10M users
  with 50KB session objects — that's 500GB, where does it live?" is the bar.
