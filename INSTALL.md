# Optional: make the loop proposal automatic

loop-kit works as soon as it's installed — `/loopify` is available, the `loop-protocol` skill
auto-surfaces on YES/NO-decidable deliverable tasks, and the `validator` agent is callable.

If you also want Claude to **proactively propose a loop every time you hand in a deliverable task**
(without you typing `/loopify`), paste this into your `~/.claude/CLAUDE.md`:

```markdown
### Self-verification loop

When handed a deliverable task whose pass/fail is YES/NO-decidable (code, docs, design), before
starting:

0. Entry gate: skip trivial work (a one-line/typo fix, an investigation, a question — token
   waste). If no deterministic check (L0) exists and none can be built, say a loop can't bite
   and work plainly instead. When in doubt, confirm in one line.
1. Propose a "loop-ready prompt" rewrite of the task (3-5 acceptance criteria frozen into a
   unique run dir under ~/.claude/tmp/loop/ and sha256-pinned + an L0 deterministic check +
   checker inspection). Present it first.
2. Then run that prompt with the built-in self-paced `/loop`. Each iteration: implement -> L0
   check (on failure, feed the raw failing output back verbatim) -> checker inspection
   (loop-verify if its `independent_verify` tool is available, else the validator) -> fix only
   the flagged defects. Stop on the checker's PASS, at a 3-iteration cap, or early if the same
   defect survives two iterations (report what's unresolved — no fake pass). Before declaring
   done, run the strongest independent checker available on the full final artifact (the git
   diff for code; the run dir's artifact.md for text).
```

## Notes
- `/loop` and `/goal` are **Claude Code built-ins**, not shipped by loop-kit. loop-kit provides the
  recipe; the built-ins drive iteration.
- The loop freezes its criteria into a **unique per-run dir** —
  `~/.claude/tmp/loop/<timestamp>-<slug>/criteria.md` (plus, for text deliverables, an artifact
  file there) — and records the criteria file's `sha256sum`. The checker reads them back from
  the paths it is given, and only from those (a shared fixed path let concurrent sessions read
  each other's criteria — a measured failure mode, fixed in 0.2.0). Run dirs are disposable.
- `/loop` and `/goal` can't be launched by Claude — paste and run them yourself.
