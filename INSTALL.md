# Optional: make the loop proposal automatic

loop-kit works as soon as it's installed — `/loopify` is available, the `loop-protocol` skill
auto-surfaces on YES/NO-decidable deliverable tasks, and the `validator` agent is callable.

If you also want Claude to **proactively propose a loop every time you hand in a deliverable task**
(without you typing `/loopify`), paste this into your `~/.claude/CLAUDE.md`:

```markdown
### Self-verification loop

When handed a deliverable task whose pass/fail is YES/NO-decidable (code, docs, design), before
starting:

1. Propose a "loop-ready prompt" rewrite of the task (3-5 frozen acceptance criteria + an L0
   deterministic check + validator inspection). Present it first.
2. Then run that prompt with the built-in self-paced `/loop`. Each iteration: implement -> L0
   check -> checker inspection (loop-verify if its `independent_verify` tool is available, else the
   validator) -> fix only the flagged defects. Stop on the checker's PASS or at a 3-iteration cap.

Do NOT apply this to trivial work — a one-line/typo fix, an investigation, or a question — (token
waste). When in doubt, confirm in one line before applying.
```

## Notes
- `/loop` and `/goal` are **Claude Code built-ins**, not shipped by loop-kit. loop-kit provides the
  recipe; the built-ins drive iteration.
- The loop writes its frozen criteria to `~/.claude/tmp/criteria.md` and (for text deliverables)
  `~/.claude/tmp/artifact.md`; the `validator` agent reads them back from there.
- `/loop` and `/goal` can't be launched by Claude — paste and run them yourself.
