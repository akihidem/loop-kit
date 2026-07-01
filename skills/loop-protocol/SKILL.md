---
name: loop-protocol
description: |
  The self-verification loop. Take a deliverable task whose pass/fail is YES/NO-decidable
  (code, docs, design) and raise its quality through bounded iteration: implement -> L0
  (deterministic: test/lint/typecheck) -> validator inspection -> fix. builder != checker,
  frozen criteria, 3-iteration cap. L0 is the fort; validator is the second pass.
  TRIGGER: "/loopify", "build it with a loop", "build it at higher quality",
  "self-verifying loop", and any time a deliverable task with YES/NO-decidable pass/fail is
  handed in.
---

# loop-protocol — the self-verification loop

<!--
Premises and limits (do not delete):
- The checker is either the same-family `validator` (default, zero external accounts) or, when
  loop-verify is available, a different model lineage (codex/GPT/Gemini). With the same-family
  validator this is NOT fully independent verification: a defect Claude systematically makes is
  missed by both generation and inspection. loop-verify (cross-vendor) narrows that gap — a
  different lineage reduces shared blind spots, but model inspection never eliminates error.
- The only guardrail that never depends on a model is the L0 deterministic check
  (test/lint/typecheck). For code that is the fort. Any checker (validator or loop-verify) is a
  second pass on top of L0.
- For fact-checking, do NOT make Claude self-inspection the fort of correctness. The fort is
  source grounding (web search) plus human approval.
- Iteration is driven by the built-in self-paced loop. This file defines the contents of each iteration.
-->

Take the task the user handed in, rewrite it into a loop-ready prompt that embeds this
procedure, and run it with the self-paced loop.

## Big deliverables: decompose (apps etc.)
A large target whose pass/fail can't be cut YES/NO ("the app is done") can't be the goal of a
single loop (a 3-iteration cap can't decide it, so it spins). In that case:
1. **Fix the goal (immutable, the finished picture) as the north star** — put it in prose at the
   top of the prompt or in `.claude/CONTEXT.md`. (`/goal <condition>` exists and runs until the
   condition is met, but its judgment is a soft evaluation by a small model looking only at the
   transcript — left alone it's Goodhart. If you use it, tie the stop condition to deterministic
   evidence, e.g. "`node --test` prints 0 fail" = the L0 output below. `/loop` is the separate
   interval/self-pacing tool.)
2. **Break the finished picture into features / milestones.**
3. **Run one loop per feature** — turn each feature into the loop-ready prompt below and process
   them in order.

= "one goal + N loops (one per feature)". A loop only works on a unit you can cut YES/NO (= a
feature). The goal is the direction; each loop is a one-feature quality gate.

**If you use /goal, exactly one at the top** (stop condition = the aggregated L0 evidence of all
features, e.g. "all `node --test` show 0 fail"). Wrapping each feature in its own /goal is
overkill — it stacks a weak soft judge on top of a strong L0+validator and only widens the
Goodhart surface. `/goal` and `/loop` are flat, session-level controls and can't nest; if you
need hierarchy, build it with subagents / a workflow (a parent spawning children).

## Proposal phase (present before launching the loop)
1. Classify the task type in one line and declare the approach you'll take.
2. **Define 3-5 acceptance criteria** at a YES/NO-decidable granularity. These criteria are
   frozen from here on (no loosening them later to force a pass).
3. Present the deliverable's structure, the main design decisions, and the risks as bullets.

Present these three to the user before launching the loop.

## Inside each iteration (implement -> inspect -> judge)
The self-paced loop repeats the body. Each iteration does the following.

### Implement
Build the deliverable per the acceptance criteria / fix only the defects flagged last iteration.

### Inspect
1. **L0 deterministic check (cost 0, runs first)**: for code, run test / lint / typecheck via
   Bash. If it fails, go straight to the next iteration without calling the validator (save tokens).
2. After L0 passes, freeze the inspection inputs:
   - acceptance criteria -> `~/.claude/tmp/criteria.md` (the proposal text verbatim)
   - text deliverable -> `~/.claude/tmp/artifact.md` (code deliverables: the changes under git as-is)
3. **Pick the checker (prefer cross-vendor).** The job is adversarial inspection against the frozen
   criteria; **builder != checker** is what makes it worth running.
   - **If loop-verify is available** (its MCP tool `independent_verify` is in the session): use it
     as the checker. It grades with a
     **different model lineage** (codex / GPT / Gemini), so it catches defects a same-family
     self-check waves through — measured 4 vs 1 independent defects vs the haiku validator. Its
     verdict is the **same contract** as the `validator` below (PASS/FAIL, per-criterion OK/NG,
     defects-outside, fix_instructions), so it's a **drop-in replacement**. Call it as
     `independent_verify(criteria, artifact)` — pass the frozen criteria text (from
     `~/.claude/tmp/criteria.md`) and the artifact (the `git diff` or `~/.claude/tmp/artifact.md`)
     as the arguments.
   - **Otherwise fall back to the `validator` subagent** (haiku, same-family). loop-kit ships this
     so the loop runs with **zero external accounts**. Here **pass paths only** (don't paste the
     deliverable/criteria into the body): "Read `~/.claude/tmp/criteria.md`, read the target
     (`git diff` or `~/.claude/tmp/artifact.md`) yourself, and inspect it."

### Judge and loop control
- The relevant L0 and the chosen checker (loop-verify or the validator) both pass -> stop the loop
  (don't schedule the next wakeup). Go to the output phase.
- A defect in either -> fix only the flagged defects and keep iterating.
- 3-iteration cap. If unmet, present the best version + an explicit list of unresolved defects and
  stop (don't fake a pass).

## Output phase
Output only a passing deliverable. Append an [inspection log] (iteration count / L0 result /
fixes) in 3 lines or fewer.
