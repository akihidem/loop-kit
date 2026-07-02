---
description: Turn a free-form request into a loop-ready prompt (freeze -> implement -> L0 -> checker iteration recipe) for the self-paced /loop. Optionally appends a /goal line anchored to deterministic evidence.
argument-hint: <what you want to build or do, in free form>
---

The user's rough request: $ARGUMENTS

Your job: convert the request above into a **loop-ready prompt** that the self-paced
`/loop` can run — i.e. a recipe that, on each iteration, actually builds the thing and then
verifies it. **This is the deliverable.** The `/goal` line (a stop condition) is an optional
appendix; if you add it, anchor it to **deterministic evidence that shows up in the
transcript** (e.g. `node --test` reporting 0 failures). `/goal`'s judgment is a soft
self-evaluation, so soft conditions get gamed (Goodhart).

Steps:
0. **Entry gate.** If the request is trivial (a one-line fix, a typo, an investigation, a
   question), state in one line that a loop isn't warranted (and why), then offer to handle it
   directly — the deliverable of this command is either a recipe or that explicit "no loop"
   verdict, never silent work. If there is no deterministic check (L0) and none can reasonably
   be built, say the loop can't bite (iteration graded only by soft model judgment is Goodhart
   theater) and propose either building the smallest L0 first or doing the work plainly with
   human judgment.
1. **Restate the definition of done in one line.** If `$ARGUMENTS` is empty, ask one question
   only: "what do you want to build?"
2. **Freeze 3-5 acceptance criteria** at a YES/NO-decidable granularity. If something can't be
   checked deterministically, ask 1-3 questions to pin it down (is there a test / what command
   runs it / which files / what's the pass line). Minimal — don't over-confirm.
   Freeze them in a **unique run dir**: instantiate a *concrete* path (timestamp + slug, e.g.
   `~/.claude/tmp/loop/20260701-1415-payments-fix/`), `mkdir -p` it, write `criteria.md` there,
   and record its `sha256sum`. The generated prompt must contain the concrete path, not an
   unexpanded variable — it has to run standalone when pasted into `/loop`. (Never a shared
   fixed path — concurrent sessions reading each other's criteria is a measured failure mode.)
3. **Identify L0 (the deterministic check)** — the test/lint/typecheck that runs on this
   artifact (e.g. `node --test` / `pytest` / `tsc --noEmit`). If none exists, propose one in a
   line (e.g. "extract the logic into a pure function and add `node --test`").
4. **If it's big (multiple features), decompose by feature**, and make **one loop-ready prompt
   per feature** (a loop only works on a YES/NO-decidable unit = a feature). If you add `/goal`,
   put exactly one at the top (don't wrap each feature).

Output (concise, just this):
- 🔁 **loop-ready prompt (the star — hand it straight to /loop)**:
  ```
  [Task]: <definition of done>
  Run dir: <concrete path, e.g. ~/.claude/tmp/loop/20260701-1415-payments-fix> — mkdir -p it first
  Freeze: write the criteria below to <run dir>/criteria.md and record its sha256 (immutable from
    here; a discovery outside the contract is promoted + re-frozen, or logged out-of-scope)
  Acceptance criteria (frozen — never loosen later):
    1. <YES/NO-decidable criterion>
    2. <YES/NO-decidable criterion>
    3. <YES/NO-decidable criterion>
  Artifact: code -> the git diff is the artifact; text/docs/design -> write and refresh the
    deliverable to <run dir>/artifact.md every iteration
  Each iteration: implement (single builder — fix only the flagged defects)
    -> L0(<concrete command>); on failure feed the raw failing output back verbatim (no summary)
    -> checker review (loop-verify's independent_verify if available, else the validator;
       pass only paths: <run dir>/criteria.md + the artifact above) -> fix only the flagged defects.
  Stop: on PASS / at 3 iterations / early if the same defect survives twice or the diff is
    empty (spinning). If unmet, list the unresolved defects honestly (no fake pass).
  Completion gate: the strongest independent checker available, on the full final artifact
    (the git diff for code; <run dir>/artifact.md for text/docs/design).
  Re-run all tests every iteration (no regressions).
  ```
  (Replace `<run dir>` with the concrete instantiated path — the prompt must run standalone,
  with no unexpanded variables.)
- 🧪 L0 command: <the concrete command that actually produces that criterion>
- (optional) 🎯 /goal appendix: `/goal <L0-anchored condition, e.g. node --test shows 0 failures>`
- (big jobs only) a minimal feature-decomposition sketch + one loop-ready prompt per feature

Close with one line: "Hand this loop-ready prompt to the self-paced `/loop` to run it (or I can
run it inline). To prevent unattended early-exit, also use the `/goal` line above. `/loop` and
`/goal` can't be launched from my side, so paste and run them yourself."
