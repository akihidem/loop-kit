# loop-kit

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

[English](README.md) · [日本語](README.ja.md)

> 📖 **New here?** There's an [illustrated, plain-language explainer (Japanese)](docs/explainer-ja/) — see the [preview image](docs/explainer-ja/preview.png).

A **self-verifying build loop** for Claude Code. Freeze YES/NO acceptance criteria, then on each
iteration: **implement → L0 deterministic gate (test/lint/typecheck) → adversarial validator
(builder ≠ checker) → fix → repeat (bounded)**. The point isn't multi-agent magic — it's
**grounding the loop in external verification** so quality stops depending on the model's own
lenient self-grade.

## Why

- Ask an LLM "is this done?" and it grades itself leniently — it will loosen the bar until it
  passes (Goodhart).
- The fix is to put a **deterministic gate the model can't talk its way past** inside the loop:
  `node --test`, `pytest`, `tsc --noEmit`, `ruff`. That's **L0 — the fort**.
- On top of L0, a **validator** subagent inspects adversarially (its job is finding defects, not
  approving), and **builder ≠ checker** so a single model's blind spots are less likely to pass.
- Criteria are **frozen up front** (no loosening to force a pass), and iteration is **bounded**
  (~3) so it doesn't spin.

## How it works

```
  /loopify <your request>
        |
        v
  +---------------------------------------------------------+
  | Proposal: classify task, freeze 3-5 YES/NO criteria     |
  +---------------------------------------------------------+
        |
        v   each iteration (driven by the built-in /loop)
  +-----------+   +----------------------+   +---------------------------+
  | implement | > | L0 (deterministic):  | > | validator (adversarial):  |
  |           |   | test / lint / typeck |   | haiku, finds defects      |
  +-----------+   +----------+-----------+   +-------------+-------------+
                       fail > next iter (skip validator)  |
                                                          v
                                         PASS both > stop & output
                                         defect    > fix only that, iterate
                                         3 iters   > best + open defects (no fake pass)
```

Iteration is driven by Claude Code's **built-in `/loop`** (self-paced) and, optionally, **`/goal`**
(run-to-completion). loop-kit does **not** ship those — it ships the *recipe* they run.

## Install

```bash
# the akihidem marketplace is hosted in the claude-env-coach repo
claude plugin marketplace add akihidem/claude-env-coach
claude plugin install loop-kit@akihidem
```

## Usage

- **`/loopify <what you want>`** → get a loop-ready prompt (frozen criteria + L0 + validator
  recipe). Hand it to `/loop`, or let Claude run it inline.
- The **`loop-protocol`** skill auto-surfaces on YES/NO-decidable deliverable tasks and runs the
  implement → L0 → validator → judge cycle.
- The **`validator`** agent (haiku) does the adversarial inspection against the frozen criteria
  (read from `~/.claude/tmp/criteria.md`).
- **`templates/goal-loop-template.md`** — for "build it to completion" jobs, anchor `/goal` to
  deterministic evidence (not a soft self-grade).

### Optional: auto-propose on every deliverable task
Paste the snippet in [INSTALL.md](INSTALL.md) into your `~/.claude/CLAUDE.md` to make Claude
proactively propose a loop whenever you hand in a YES/NO-decidable deliverable. The `loop-protocol`
skill already triggers without it; the snippet just makes the proposal automatic.

## Components

| Component | Type | What it does |
|---|---|---|
| `loopify` | command | Convert a free-form request into a loop-ready prompt |
| `loop-protocol` | skill | The implement → L0 → validator → judge procedure |
| `validator` | agent (haiku) | Adversarial inspection against frozen criteria |
| `goal-loop-template` | template | Anchor `/goal` to deterministic evidence |

## Honest limits

- **Not independent verification.** Generation and inspection are the same Claude family, so a
  defect Claude *systematically* makes is missed by both. The validator reduces blind spots; it
  doesn't eliminate them.
- **L0 is the real guardrail.** For code, the deterministic test/lint/typecheck is the fort; the
  validator is a second pass. If there's no L0, the loop is much weaker — add one (extract logic
  into a pure function and test it).
- **For fact-checking, self-inspection is not the fort** — source grounding + human approval is.
- **Don't blanket every task.** A one-line fix, an investigation, or a question doesn't need a loop
  (token waste). Iteration plateaus around 3.

## License

MIT © akihidem
