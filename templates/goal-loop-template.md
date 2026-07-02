# goal-loop template — anchor `/goal` to deterministic evidence

A prompt template for handing a big job to an AI. The point is one thing:
**tie `/goal`'s stop condition to "deterministic evidence appearing in the transcript".**
`/goal`'s judgment is a soft evaluation by a small model looking only at the output text
(official: code.claude.com/docs/en/goal), so a self-evaluable condition passes too easily
(Goodhart). Binding it to the result of running test/compile/lint makes it actually bite.

> When to use which: **`/goal <condition>`** = run autonomously until the condition is met (good
> for "build it to completion"). **`/loop`** = repeat at an interval / self-paced (polling / periodic).

---

## 1. Template (copy and fill in)

```
# North-star goal (immutable, the finished picture)
<1-3 lines. What state counts as done? Can be pinned in .claude/CONTEXT.md.>

# Frozen acceptance criteria (YES/NO, human-approved before starting, immutable after)
- [ ] Criterion 1: <observable fact>
- [ ] Criterion 2: ...
(3-5 of them. For each, note which command's output proves it true. Write them to a unique
run dir — `~/.claude/tmp/loop/<timestamp>-<slug>/criteria.md` — and record its `sha256sum`:
the hash makes the freeze tamper-evident, and a per-run dir keeps concurrent sessions from
reading each other's criteria.)

# Verification anchor (deterministic, NOT the AI's self-grade)
- L0: `<test/compile/lint command>` shows <the concrete passing output>
- (for non-code / NL: one deterministic verify — schema validation / verbatim source match / etc.)

# Roles (builder != checker)
- Implement: <model A / me>
- Inspect: <model B / a different lineage. e.g. an external CLI, or the validator subagent>
- Human: only the initial criteria freeze and the final demo inspection

# Run
/goal <the L0 anchor's passing output appears in the transcript, and git diff -- <protected paths> is 0 lines>
```

---

## 2. Good vs bad conditions (cheat sheet)

| ❌ soft (a small model waves it through = Goodhart) | ✅ tied to deterministic evidence (actually bites) |
|---|---|
| `/goal finish the app nicely` | `/goal python3 -m unittest acceptance_test prints "Ran 9 tests ... OK"` |
| `/goal fix the bug` | `/goal the failing test_X shows pass, and all existing tests are still OK` |
| `/goal make the code clean` | `/goal ruff/eslint show 0 errors and existing tests are green in the output` |
| `/goal summarize it` | `/goal each source claim is verbatim-grounded in the summary, schema/verify shows score=1.0` |

Principle: **write only conditions you can prove with the AI's own output** (the official
warning). Don't write a condition whose evidence never appears in the transcript.

---

## 3. Worked example (generic — CLI TODO)

```
# North-star goal
A CLI TODO (Python stdlib only). add/list/done/rm work, state persists, error cases don't crash.

# Frozen acceptance criteria (S1-S9 in acceptance_test.py, hash-frozen)
S1 add -> shows in list / S2 done marks [x], --open hides it / S3 persists / S4 rm /
S5 error cases exit non-zero + stderr, no traceback / S6 no false success on write failure /
S7,S8 no traceback on a corrupt store / S9 reject a bool id

# Anchor (deterministic)
L0: `python3 -m unittest acceptance_test` shows "Ran 9 tests ... OK"

# Roles
Implement = model A, Inspect = a different-lineage CLI (adversarial review), Human = final demo

# Run
/goal python3 -m unittest acceptance_test shows 9 tests OK, and nothing but todo.py changed
```

Result pattern: the independent inspector finds robustness bugs outside the contract -> fix ->
promote them into S6-S9 and re-freeze. The discovery becomes an asset.

---

## 4. Traps (measured)

- **The built-in evaluator is weak**: `/goal`'s judgment is a same-family small model on the
  transcript only. For high stakes don't leave it to `/goal` — stack an independent inspector
  (a different lineage) + a human final demo.
- **`/goal` is top-level only, no nesting**: the stop condition = the aggregated L0 evidence of all
  features (e.g. "all `node --test` 0 fail"). Wrapping each feature in its own `/goal` is overkill —
  it stacks a weak soft judge on a strong L0+validator and only widens the Goodhart surface. `/goal`
  and `/loop` are flat session-level controls and can't nest. **If you need hierarchy, use subagents
  / a workflow** (a parent spawning children).
- **Completion boundary = the frozen criteria**: outside the contract, adversarial inspection
  surfaces endless holes. Promote a useful finding into a criterion and re-freeze to bank it.
- **Iteration is bounded**: it plateaus around 3 iterations. And stop *early* when it's
  spinning — the same defect surviving two consecutive iterations, or an empty diff, means more
  iterations won't help. Report what's unresolved instead of faking a pass.
- **Irreversible actions stay outside the loop**: merge / production deploy / publishing /
  sending externally are never part of the loop body — keep them behind a human gate.
- **Don't blanket every task**: don't apply this to a one-line fix / investigation / question (token waste).
- **When calling an external CLI inspector in the background, close stdin** (`</dev/null`) — an open
  stdin hangs forever.

Related: `skills/loop-protocol/SKILL.md`
