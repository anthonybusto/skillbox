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
"Scale Engineer", "Security Reviewer", "DX Champion"). Keep the fixed
3-role default even for niche domains — dynamic, per-topic-generated roles
are opt-in only (see Round 0, step 0). Relabeling is cheap and encouraged;
inventing a variable-count role set by default is not, and has no evidence
of improving output over a well-chosen fixed set.

For genuinely high-stakes or irreversible decisions, default to running the
full protocol rather than a quick single pass — treat the extra cost as
insurance, even without proof it improves the outcome.

### Griller Agent

A 4th agent with **no position of its own**. Its job:
- Read all three positions after each position phase.
- Generate 2–3 hard, specific questions per agent targeting weak reasoning,
  unstated assumptions, contradictions, and gaps.
- Apply the grill-me philosophy: walk down each branch of the decision tree,
  resolve dependencies between decisions one by one.
- Never accept vague answers. If a position says "we could do X", the griller
  asks "what's the migration path, what breaks, what's the rollback plan?"
- **Spot-check at least one checkable evidence claim per agent per round**
  (see Tool-Grounded Verification below) by actually looking it up, not by
  judging whether the explanation sounds confident. If a cited file, doc, or
  URL doesn't say what the agent claims, that's a hard fail on that point —
  call it out explicitly as a fabricated/incorrect citation, not as "weak
  reasoning."

### Tool-Grounded Verification

The core limitation of this skill is that every "agent" is the same model
reasoning from the same training-data prior — restating an unverified belief
in a different persona's voice doesn't make it true. The only lever that
actually grounds a claim outside the model's own distribution is checking it
against something real: the actual codebase, actual docs, actual current web
information, or an actual executed test/command. Use it whenever the topic
makes this possible:

- When launching position or griller subagents, they run as `generalPurpose`
  Task agents and already have tool access (`Read`, `Grep`, `Glob`, `Shell`,
  `WebSearch`, `WebFetch`, etc.) — explicitly instruct them to **use tools to
  verify checkable claims before asserting them**, not just reason from
  memory. E.g., "if you claim this function has a specific behavior, `Read`
  or `Grep` the actual file and quote what it says" or "if you claim a
  library's API works a certain way, `WebSearch`/`WebFetch` its current docs
  rather than relying on training data, which may be stale or wrong."
- Only checkable claims need this. Subjective/strategic claims (tradeoffs,
  preferences, predictions about the future) have no ground truth to check
  — mark those "no evidence, this is a judgment call" per the Position
  Format rather than forcing a tool lookup that can't apply.
- If the topic itself has no external ground truth available (pure opinion,
  no codebase/docs/web relevant, e.g. "which of these two names sounds
  better") skip tool-grounding entirely and say so — don't force it where
  nothing is actually checkable, that just wastes calls without adding
  truth-value.
- The griller's spot-check is the actual enforcement mechanism: an agent
  citing "checkable evidence" that the griller then fails to verify (or
  actively disproves) is the single most valuable signal this protocol can
  produce, because it's the one thing that isn't just rhetoric evaluating
  rhetoric.

## Protocol

### Round 0 — Seed

0. **(Optional) Dynamic role check**: only if the topic is genuinely
   cross-domain or the user explicitly asks for custom roles, propose 2-5
   alternative role labels tailored to the topic and briefly state why
   before launching them. Otherwise skip this step entirely — fixed
   Advocate/Skeptic/Pragmatist (relabeled if useful) is the default, not
   because it's proven better, but because it's cheaper and equally
   unvalidated either way.
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

**Phase 1 — Positions (blind-first-draft order)**: Launch three subagents in
parallel, each receiving:
- The original topic.
- Its own role (same throughout).
- **The griller's questions directed at them from the previous grill phase.**
- All three positions from the previous round.
- Instruction, in this order: "FIRST, before reading the other two positions
  below, directly answer the griller's questions with specifics — no
  hand-waving. SECOND, now read the other two positions, identify agreements
  and disagreements, and refine your position. Explicitly flag anything you
  changed your mind on." Answering the griller first (even though the other
  positions are included in the same prompt) keeps the agent's own reasoning
  from being contaminated by the others' framing before it commits to an
  answer. This doesn't create real independence (same model, same weights)
  — it only reduces sequential-contamination/anchoring, which is a narrower
  and more honest claim.

**Phase 2 — Grill**: Launch one griller subagent receiving:
- The original topic.
- All three updated positions.
- The griller's previous questions and how each agent responded.
- Instruction: "Evaluate how well each agent answered your previous questions.
  Flag evasive or weak answers. Generate new questions targeting remaining
  gaps, new contradictions, or claims that still lack evidence."
- **In-round re-ask (optional, capped at 1 per agent per round)**: if one
  agent's answer is a clear non-answer (hedging, no concrete claim, ignores
  the question), the griller may immediately re-ask that agent once within
  the same round instead of waiting a full cycle. The re-asked answer
  *replaces* that agent's contribution for this round — it does not add an
  extra full round on top. This only improves the odds of catching a missed
  point; it does not guarantee it, and must never be used more than once per
  agent per round or it reintroduces unbounded cost.

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
- You've completed 2 rounds (default ceiling). Only continue past round 2 if
  the griller's assessment explicitly names *specific, substantive* open
  questions (not just "could go deeper") — write that justification down
  before starting round 3.
- You've hit 5 cycles (hard ceiling, should rarely be reached).

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

2. Present the consensus document to the user. Synthesis is **exactly one
   subagent call** producing one fixed-length document — do not chain
   additional synthesis/summarization passes, which would silently add cost
   in the step meant to be the cheap wrap-up.

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

### Follow-Up Audit (outcome tracking)

The protocol has no way to know whether "the debate converged" ever
correlated with "the debate was right" unless someone checks later. Close
that loop with a persistent log plus a periodic automated audit:

1. **Log every debate.** After presenting the consensus (and once the user
   has finished the post-consensus grill), append one line to
   `~/.cursor/skills/agent-convergence/debate-log.jsonl`:
   ```json
   {"id": "<uuid>", "date": "<ISO date>", "topic": "<short topic summary>",
    "decisions": [{"text": "<decision>", "verify_hook": "<from Verification Hooks>", "status": "pending"}],
    "model_assignment": "<from Guidance cross-model note, or 'same-model'>"}
   ```
   Create the file/directory if it doesn't exist. This step is one file
   append — no extra subagent call, effectively free.

2. **Audit pending entries on request or on a loop.** When the user asks to
   check on past debates, or via a scheduled `/loop` (e.g.
   `/loop 1d audit the agent-convergence debate log`), for every entry with
   a `status: "pending"` decision:
   - Run its `verify_hook` for real (`Grep`/`Read`/`Glob`/`Shell` against
     the current repo, or `WebFetch`/`WebSearch` if the hook is external)
     against **current** state, not memory.
   - Update `status` to `"held"` (still true), `"reversed"` (deliberately
     changed), `"contradicted"` (current state disagrees, no clear record
     of a deliberate change), or `"unverifiable"` (hook no longer
     applicable, e.g. file moved/deleted) — always with the actual evidence
     found (file:line, command output), not a restated impression.
   - Leave decisions with no checkable proxy out of the audit; they were
     already flagged as unauditable at synthesis time.
   - Only surface a notification to the user for `"reversed"` or
     `"contradicted"` outcomes — stay quiet on `"held"` so the loop isn't
     noisy.
3. **`/loop` caveat**: a loop only runs while its session/terminal stays
   open — it is not a persistent cron across restarts. For always-on
   auditing independent of an open session, this needs an OS-level
   scheduled task (cron/launchd) invoking a headless CLI call instead;
   treat that as a separate, heavier setup to build only once the
   session-scoped version has proven useful.
4. **Honesty caveat**: the audit is still the same kind of model doing the
   checking — but it's now checking against actual current file/doc
   contents rather than recalled opinion, which is a real (if partial)
   improvement. It does not replace the user's own judgment on whether a
   held decision actually turned out well in practice, only on whether it's
   still being followed.

## Position Format

Instruct each subagent to structure its output as:

```
## Position: [Role Name]

### Core Stance
[1-2 sentence summary of overall position]

### Key Points
1. [Claim] — [Reasoning] — [Evidence: checkable pointer if available
   (file:line, doc/URL, quoted spec text) — else state "no evidence, this
   is a judgment call"] — [Confidence: high/medium/low, relative to your
   own other points, not a calibrated probability]
2. ...

### Open Questions
- [Anything you flagged as unresolved or needing more info, not yet a
  claim either way]

### Answers to Griller (rounds 1+)
- Q: [Question] → A: [Direct, specific answer]

### Agreements with Others
- [What you agree with and from whom]

### Disagreements
- [What you challenge and why]

### Changed Minds (rounds 1+)
- [What you previously argued but now concede, and why]
```

Confidence labels are a *relative* signal only (did this claim get weaker or
stronger versus this same agent's other claims or its own prior round) —
never treat them as calibrated probabilities, and never let them alone
justify a final decision weight. Use them only as a secondary trigger for
whether the griller should re-ask (see In-round re-ask above); the primary
trigger is always a substantive change in the claim's text itself.

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

### Verification Spot-Checks
- [Claim checked] → [Tool used] → [Confirmed / Contradicted / Could not verify]
- ... (at least one per agent, where a checkable claim exists)

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

### Verification Hooks
For each Agreed Decision, name a concrete, mechanically-checkable proxy that
a future audit (see Follow-Up Audit below) can check against the actual
repo/plan state — not a re-judgment of whether the decision was "right,"
just whether it's still being followed. If a decision has no checkable
proxy (pure preference/subjective), say so explicitly rather than forcing
one.
1. [Decision] → [Verify: e.g. "`grep -r 'ROW LEVEL SECURITY' migrations/`
   still returns matches" / "`SKILL.md` still contains X" / "no checkable
   proxy — subjective preference"]
2. ...

### Dissents Worth Preserving
Only include a dissenting position here if it passes **all three** checks —
otherwise fold it into "Unresolved Points" as a minor detail or drop it:
1. It survived the griller's strongest cross-examination on that specific
   point (not just unchallenged persistence).
2. It cites at least one checkable evidence pointer, not just restated
   preference.
3. The dissenting agent stated an explicit falsification condition ("I'd
   change my mind if X were shown") that isn't something already presented
   and rejected in this debate.

This exists so "the process converged" and "the process ran out of things
to disagree about" aren't treated as the same thing — a checklist-passing
dissent is preserved as a live, useful disagreement even after synthesis.

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

You have tool access (Read, Grep, Glob, Shell, WebSearch, WebFetch, etc.).
For any claim you make that is checkable against reality — code behavior,
docs, current facts, test/command output — use a tool to verify it before
asserting it, and cite what you found. For claims that are genuinely
subjective or predictive with no ground truth to check, mark them as
judgment calls instead of forcing a lookup.

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

You have tool access. Before writing your questions, pick at least one
checkable evidence claim per agent (a cited file, doc, URL, or asserted
behavior) and actually verify it with a tool. If it checks out, move on. If
it doesn't check out or you can't verify it, that's your highest-priority
question for that agent — name the discrepancy directly rather than asking
a generic follow-up.

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
   No hand-waving. If a question challenges a checkable claim, re-verify it
   with a tool rather than re-asserting it from memory.
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
For each agreed decision, also produce a Verification Hook: a concrete way
a future audit could mechanically check the repo/plan/docs to see if the
decision is still being followed (a grep pattern, a file's expected
content, a doc's expected claim). If a decision has no checkable proxy,
say so explicitly instead of forcing one.
Incorporate the griller's remaining concerns as caveats.
Highlight non-obvious insights that only emerged through the debate.
```

## Guidance

- **What this protocol actually is**: all four "agents" are the same
  underlying model with different prompts, not genuinely independent
  reasoners. "Convergence" here means the model agreed with itself across
  several re-samplings under adversarial framing — it is not evidence of
  correctness, and the skill should never be described to a user as
  producing independently-verified conclusions. Its real, more modest value
  is *forced articulation*: making claims explicit, evidenced, and
  cross-examined, which makes the resulting artifact easier for a human to
  audit and locally repair than an unstructured single-pass answer. Frame
  the process to users as structured elicitation / a disagreement-surfacing
  tool, not as adversarial proof.
- **Cost accounting**: a baseline run is 3 agents x 2 rounds = 9 subagent
  calls (6 position + 2 grill) + 1 synthesis = 10 calls, growing to ~21 at
  the old 5-round ceiling. Prefer a **default cap of 2 rounds**, not 5;
  require going past round 2 only when the griller's final assessment for
  that round explicitly states genuine, substantive disagreement remains
  (not just "could dig deeper") — write that justification out before
  starting a 3rd round. Reserve the full 5-round ceiling as a hard stop, not
  a default target.
- **The "free" boundary**: reorganizing or reformatting output that would be
  generated anyway (blind-first-draft ordering, structured claim/evidence
  fields, dissent-gating) costs nothing extra and should be adopted
  unconditionally. Anything that adds a **new subagent call** beyond the
  4-per-round baseline (e.g., a per-round summarizer pass, a dynamic
  role-selection pass, cross-model role assignment) is a real cost with a
  real tradeoff — justify it explicitly, don't call it free.
- **Dynamic roles / agent count**: opt-in only (see Round 0, step 0). Keep
  default agent count at 3, cap any deliberate expansion at 5. Going from 3
  to 5 agents roughly doubles per-agent context (each now reads more peer
  positions) on top of the linear increase in agent count — treat it as a
  ~2-2.5x cost multiplier per round, not a small increment.
- **Cross-model diversity (optional, not default)**: the single most
  credible fix for the "same model, no real independence" problem is
  assigning genuinely different model families to different roles, since a
  single model's persona prompts still sample from one underlying belief
  distribution. This is real cost, not free — expect roughly another
  30-60% on top of the multi-agent baseline from lost prompt-caching and
  cross-provider latency/rate-limit handling. Treat it as something to pilot
  once on a real topic and compare against a same-model run, not something
  to default to. Only pilot when the user explicitly asks for it.

  Mechanism depends on environment — probe for what's actually available
  before committing to a plan, fall back gracefully, and always disclose in
  the output which mechanism was actually used per role. Detect the
  environment by checking what the `Task` tool's `model` parameter actually
  offers (this is visible in the current session's own tool
  descriptions/system context — no external check needed):
  - If the available model list includes families beyond one lab's tiers
    (e.g. it lists both Claude and GPT models, as Cursor's does — currently
    Claude, GPT, and Cursor's own Composer), that's a **Cursor-like
    environment**: use mechanism 1 below for those families directly.
    Google/Gemini-lineage is typically *not* in that native list even in
    Cursor — for a role that specifically needs that lineage, still fall
    through to mechanism 2 (`agy`) for that one role, and use mechanism 1
    for the others. Mixing 1 and 2 within the same debate is fine as long
    as each role's mechanism stays fixed for that debate's duration.
  - If the available model list is Claude-tiers-only (e.g. Claude Code),
    that's a **Claude-Code-like environment**: mechanism 1 can only produce
    same-lab-tier diversity (weak, see tier 3 below) — prefer mechanism 2
    (external CLIs) for any role that needs real cross-lab diversity.
  1. **Native multi-provider Task tool**: if the `Task` tool exposes
     multiple model families, assign one family per role directly via its
     `model` parameter. Simplest, no extra plumbing, no auth/process
     overhead — prefer this whenever the needed family is natively
     available.
  2. **External CLI agents** (any environment with `Shell` access) —
     **verified working**. Required for any role needing real cross-lab
     diversity in Claude-Code-like environments (mechanism 1 can't provide
     it there); needed only for the Gemini/Google-lineage slot in
     Cursor-like environments (mechanism 1 already covers Claude/GPT there):
     - Probe with `command -v codex` and `command -v agy`. Do **not** probe
       for `gemini` — the legacy Gemini CLI's consumer tier was deprecated
       in June 2026 (`IneligibleTierError`, unfixable client-side); treat it
       as permanently unavailable and go straight to `agy` (Antigravity
       CLI, its replacement) instead.
     - **Codex**: requires `CODEX_API_KEY` to be set — check with
       `[ -n "$CODEX_API_KEY" ]` before using it. Do **not** rely on the
       interactive/session-based `codex login` (ChatGPT-subscription OAuth)
       for this skill: that session token is shared with the Codex desktop
       app and its auto-connecting plugins/MCP servers (Figma, GitHub,
       etc.), and concurrent refreshes from those other processes will
       invalidate/revoke the token mid-run — confirmed in practice, not
       theoretical. `CODEX_API_KEY` (from platform.openai.com, a separate
       pay-per-token credential) avoids that shared-session entirely and is
       the reliable path. Invocation:
       ```bash
       CODEX_API_KEY=$CODEX_API_KEY codex exec --sandbox read-only \
         --skip-git-repo-check "<full role prompt including Position Format>" \
         < /dev/null
       ```
       `--skip-git-repo-check` is required unless run inside a trusted git
       repo, or it refuses to start. `< /dev/null` is required or it hangs
       waiting on stdin in non-interactive contexts. Expect harmless
       stderr noise (an auto-attempted Figma MCP connection that fails with
       `AuthRequiredError`) — ignore it, it doesn't affect the result.
     - **Antigravity (`agy`)**: needs one-time interactive `agy` login
       (device-code OAuth flow — the user pastes a code into their own
       terminal, never share that code). It may not be on `PATH` by default
       after install (`~/.local/bin/agy`) — add that dir to `PATH` if
       `command -v agy` fails but the binary exists there. Invocation:
       ```bash
       agy --print "<full role prompt including Position Format>" --sandbox < /dev/null
       ```
     - Force a read-only/sandboxed mode for both — a debate position agent
       should verify, not edit. Capture stdout as that role's position text
       and feed it into the transcript exactly like any other position.
     - Third-party CLI names/flags/auth churn quickly (this was confirmed
       directly — Gemini CLI died mid-project). Re-probe each run rather
       than assuming last run's detection still holds, and don't assume
       "installed" means "authenticated" — actually test with a throwaway
       prompt before committing a role to it.
  3. **Cross-tier, same-lab fallback** (e.g. Claude Code, no other provider
     CLI available/authenticated): assign different model tiers (e.g. Opus
     vs. Sonnet) per role. Weakest option — same training lineage — but
     still not zero diversity. Only use if 1 and 2 are unavailable.
  4. **No mechanism available**: fall back to same-model-different-prompt
     and say so explicitly in the output ("no cross-model mechanism was
     available in this environment; roles share one model").
  **Role-to-model assignment must not default to a fixed order** (e.g.
  "always give Codex to Advocate because it was detected first") — that
  silently swaps one bias (same-model correlation) for another (role
  outcomes confounded with which provider happens to always play that
  role). Randomize which available mechanism plays which role at the start
  of each debate, and keep that specific assignment fixed only for the
  duration of that one debate's rounds (changing it mid-debate would
  reintroduce "is this a new argument or a model swap?" ambiguity). Label
  the assignment actually used in the final consensus doc (e.g. "Model
  assignment: Advocate=agy/gemini-3, Skeptic=codex/gpt-5.5,
  Pragmatist=same-model-prompted") so it's visible and comparable across
  runs — and so a pattern like "the Advocate role is always optimistic"
  can be checked against whether it's actually "Codex is always assigned
  to Advocate" instead.
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
- **High-stakes/irreversible decisions**: default to running the full
  protocol (not a shortcut) as a deliberate cost/insurance tradeoff, even
  though there is no proof it improves the outcome versus a single careful
  pass — treat this as an explicit, named judgment call, not a hidden
  assumption.
