---
description: Turn a free-form request into a loop-ready prompt (implement -> L0 -> validator iteration recipe) for the self-paced /loop. Optionally appends a /goal line anchored to deterministic evidence.
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
1. **Restate the definition of done in one line.** If `$ARGUMENTS` is empty, ask one question
   only: "what do you want to build?"
2. **Freeze 3-5 acceptance criteria** at a YES/NO-decidable granularity. If something can't be
   checked deterministically, ask 1-3 questions to pin it down (is there a test / what command
   runs it / which files / what's the pass line). Minimal — don't over-confirm.
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
  Acceptance criteria (frozen — never loosen later):
    1. <YES/NO-decidable criterion>
    2. <YES/NO-decidable criterion>
    3. <YES/NO-decidable criterion>
  Each iteration: implement -> L0(<concrete command>) -> validator review (pass only the
    criteria) -> fix only the flagged defects.
  Stop on PASS or after 3 iterations. Re-run all tests every iteration (no regressions).
  ```
- 🧪 L0 command: <the concrete command that actually produces that criterion>
- (optional) 🎯 /goal appendix: `/goal <L0-anchored condition, e.g. node --test shows 0 failures>`
- (big jobs only) a minimal feature-decomposition sketch + one loop-ready prompt per feature

Close with one line: "Hand this loop-ready prompt to the self-paced `/loop` to run it (or I can
run it inline). To prevent unattended early-exit, also use the `/goal` line above. `/loop` and
`/goal` can't be launched from my side, so paste and run them yourself."
