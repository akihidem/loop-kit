---
name: loop-protocol
description: |
  The self-verification loop. Take a deliverable task whose pass/fail is YES/NO-decidable
  (code, docs, design) and raise its quality through bounded iteration: freeze criteria ->
  implement -> L0 (deterministic: test/lint/typecheck) -> checker inspection -> fix.
  builder != checker, frozen criteria (sha256-pinned in a per-run dir), 3-iteration cap,
  honest stop. L0 is the fort; any model checker is a second pass.
  TRIGGER: "/loopify", "build it with a loop", "build it at higher quality",
  "self-verifying loop", and any time a deliverable task with YES/NO-decidable pass/fail is
  handed in.
---

# loop-protocol — the self-verification loop

<!--
Premises and limits (do not delete):
- A loop only improves a deliverable while it is grounded in verification that lives OUTSIDE
  the builder. In order of independence: L0 deterministic checks (machine) > a different model
  lineage (loop-verify: codex/GPT/Gemini) > the same-family `validator` (haiku). A same-family
  checker shares blind spots with the builder — measured 4 vs 1 independent defects in favor of
  cross-vendor. Model inspection never eliminates error; it narrows it.
- The only guardrail that never depends on a model is the L0 deterministic check
  (test/lint/typecheck). For code that is the fort. Any checker is a second pass on top of L0.
- For fact-checking, do NOT make model self-inspection the fort of correctness. The fort is
  source grounding (web search) plus human approval.
- Iteration is driven by the built-in self-paced loop. This file defines the contents of each iteration.
- Independent convergence with prior art: Anthropic's official ralph-wiggum plugin (persistent
  loop, one task per iteration), proof-loop (frozen acceptance criteria + fresh verifier +
  per-task evidence dir), aider (feeds raw test output back; adopts the best attempt after the
  retry cap). What this protocol adds on top: the frozen contract x a checker ladder x an
  honest stop.
-->

Take the task the user handed in, rewrite it into a loop-ready prompt that embeds this
procedure, and run it with the self-paced loop.

## 0. Entry gate (decide whether to loop at all)
- **Floor test**: is there a YES/NO deterministic judgment (an L0) that grounds correctness —
  or can a minimal one be built? If there is none and none can be built, **don't loop**:
  iteration graded only by soft model judgment is Goodhart theater. Do the work plainly and
  leave judgment to a human. If one can be built, build the smallest L0 first (extract logic
  into a pure function and test it / schema-validate / verbatim-grep).
- **Don't loop trivial work**: a one-line fix, a typo, an investigation, or a question is
  handled directly (token waste). When in doubt, confirm in one line.

## 1. Freeze (make the contract)
1. Classify the task type in one line and declare the approach you'll take.
2. **Define 3-5 acceptance criteria** at a YES/NO-decidable granularity and **freeze them in a
   unique run dir**: `RUN=~/.claude/tmp/loop/$(date +%Y%m%d-%H%M%S)-<slug>; mkdir -p "$RUN"`,
   write `$RUN/criteria.md`, and record `sha256sum $RUN/criteria.md` in the transcript (tamper
   detection). The run dir is disposable and doubles as the evidence dir (inspection logs,
   check scripts live there too). **The artifact convention**: for code, the git diff is the
   artifact; for text/docs/design deliverables, write and refresh the deliverable to
   `$RUN/artifact.md` every iteration so the checker has a stable path to read. **Never use a
   shared fixed path** — concurrent sessions reading each other's criteria is a measured
   failure mode.
3. With a human present, present criteria + main design decisions + risks before launching.
   Unattended, the frozen file + hash stand in for approval.
4. **The contract changes only explicitly**: a discovery outside the contract is either
   (a) promoted into the criteria and re-frozen (new hash — the finding becomes an asset), or
   (b) recorded as out of scope. Never silently widen, and never loosen to force a pass.

## Big deliverables: decompose (apps etc.)
A large target whose pass/fail can't be cut YES/NO ("the app is done") can't be the goal of a
single loop (a 3-iteration cap can't decide it, so it spins). In that case:
1. **Fix the goal (immutable, the finished picture) as the north star** — put it in prose at the
   top of the prompt or in `.claude/CONTEXT.md`. (`/goal <condition>` exists and runs until the
   condition is met, but its judgment is a soft evaluation by a small model looking only at the
   transcript — left alone it's Goodhart. If you use it, tie the stop condition to deterministic
   evidence, e.g. "`node --test` prints 0 fail". `/loop` is the separate interval/self-pacing tool.)
2. **Break the finished picture into features / milestones.**
3. **Run one loop per feature** — a loop only works on a unit you can cut YES/NO (= a feature).
   The goal is the direction; each loop is a one-feature quality gate.

**If you use /goal, exactly one at the top** (stop condition = the aggregated L0 evidence of all
features). Wrapping each feature in its own /goal stacks a weak soft judge on a strong
L0+checker and only widens the Goodhart surface. `/goal` and `/loop` are flat, session-level
controls and can't nest; if you need hierarchy, build it with subagents / a workflow.

## 2. Inside each iteration (implement -> inspect -> judge)
The self-paced loop repeats the body. Each iteration does the following.

### Implement
- **Single builder.** Build per the acceptance criteria / fix only the defects flagged last
  iteration. (Splitting the builder into fixed roles measured *worse* than a single builder
  with real test feedback — author's SWE-bench probe: role-split 0.5 < solo 0.667 < single
  builder + test feedback 1.0.)
- Keep loop state in files and git (fresh-context principle): what carries forward between
  iterations is the contract (criteria.md) and the latest failure output — not prior
  iterations' conversational residue. Read whatever repo context the implementation itself
  needs, as always.

### Inspect — the checker ladder (cheapest first; trust the most independent)
1. **L0 deterministic check** (every iteration, in cost order: test -> lint -> typecheck).
   On failure, don't call any model checker — **feed the raw failing output back verbatim** as
   the next iteration's input (don't summarize or paraphrase it; rich feedback is what stays
   Goodhart-safe).
2. **Hash check**: `sha256sum $RUN/criteria.md` must match the frozen value. A mismatch means
   the contract was tampered with -> stop immediately and report.
3. **In-loop checker** (the fast rung is fine here): if loop-verify's `independent_verify` tool
   is in the session, use it — it grades with a **different model lineage** (codex / GPT /
   Gemini) and its verdict is the same contract as the `validator` (drop-in replacement).
   Otherwise fall back to the bundled `validator` subagent (haiku, same-family, zero external
   accounts). **Pass only paths** — `$RUN/criteria.md` plus the artifact (the git diff for
   code, `$RUN/artifact.md` for text) — don't paste the deliverable or criteria into the body.
4. **Completion gate** (the strongest independent rung available): before declaring the task
   done, run the most independent checker you have — loop-verify / a different-vendor review if
   available, else the validator — against the full final artifact (the git diff for code;
   `$RUN/artifact.md` for text/docs/design). In-loop inspections may use the fast rung; the
   completion gate should not.

### Judge and loop control
- L0 and the checker both pass -> stop the loop (don't schedule the next wakeup). Go to output.
- A defect -> fix only the flagged defects and keep iterating.
- **Early stop**: the same defect surviving two consecutive iterations, or an empty diff (no
  progress), means the loop is spinning -> stop before the cap and report what's unresolved.
- **3-iteration cap.** If unmet, present the best version + an explicit list of unresolved
  defects and stop (**don't fake a pass**).

## 3. The membrane (lines the loop must not cross)
Irreversible or outward-facing actions (merge / production deploy / sending externally /
publishing) stay OUT of the loop body, behind a human gate. (Removing that membrane measured
31% wrong-ship in the author's gate experiment.)

## 4. Casting (who does what)
- **Freeze quality is the one thing to protect**: freezing the criteria and the final
  inspection belong to your strongest model (or a human).
- Implementation may be delegated to a cheaper model — delegate only the *frozen execution*,
  never split the judgment.
- The checker is not the builder (prefer a different lineage).

## Output phase
Output only a passing deliverable. Append an [inspection log] (iteration count / L0 result /
fixes / criteria hash) in 3 lines or fewer.

## Where this sits
This is the verification *inner* loop. Driving iteration = `/loop` and `/goal` (L0-anchored);
persistence = your worklog / `.claude/CONTEXT.md`; handoff = the frozen run dir. Don't make one
mechanism carry the whole circumference.
