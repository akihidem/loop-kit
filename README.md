# loop-kit

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

[English](README.md) · [日本語](README.ja.md)

> 📖 **New here?** There's an [illustrated, plain-language explainer (Japanese)](docs/explainer-ja/) — see the [preview image](docs/explainer-ja/preview.png).

A **self-verifying build loop** for Claude Code. Freeze YES/NO acceptance criteria
(sha256-pinned in a per-run dir), then on each iteration: **implement → L0 deterministic gate
(test/lint/typecheck) → adversarial checker (builder ≠ checker, cross-vendor when available) →
fix → repeat (bounded, honest stop)**. The point isn't multi-agent magic — it's **grounding the
loop in external verification** so quality stops depending on the model's own lenient
self-grade.

## Why

- Ask an LLM "is this done?" and it grades itself leniently — it will loosen the bar until it
  passes (Goodhart).
- The fix is to put a **deterministic gate the model can't talk its way past** inside the loop:
  `node --test`, `pytest`, `tsc --noEmit`, `ruff`. That's **L0 — the fort**.
- On top of L0, a **validator** subagent inspects adversarially (its job is finding defects, not
  approving), and **builder ≠ checker** so a single model's blind spots are less likely to pass.
- Criteria are **frozen up front** (no loosening to force a pass) — written to a unique per-run
  dir and **sha256-pinned**, so a tampered freeze is detectable — and iteration is **bounded**
  (~3, with an early stop when it's spinning) so it doesn't churn.
- Not everything deserves a loop: an **entry gate** skips trivial work, and if no deterministic
  check exists (and none can be built), the honest answer is "a loop can't bite here".

## How it works

```
  /loopify <your request>
        |
        |  entry gate: trivial work, or no buildable L0 -> don't loop
        v
  +----------------------------------------------------------------+
  | Freeze: 3-5 YES/NO criteria -> unique run dir, sha256-pinned   |
  +----------------------------------------------------------------+
        |
        v   each iteration (driven by the built-in /loop)
  +------------+   +----------------------+   +----------------------------+
  | implement  | > | L0 (deterministic):  | > | checker (adversarial):     |
  | (single    |   | test / lint / typeck |   | loop-verify if available,  |
  |  builder)  |   +----------+-----------+   | else validator (haiku)     |
  +------------+        fail > next iter,     +-------------+--------------+
                        raw output fed back                 |
                        verbatim (checker skipped)          v
                                    PASS both > stop & output
                                    defect    > fix only that, iterate
                                    spinning  > early stop (same defect 2x / empty diff)
                                    3 iters   > best + open defects (no fake pass)
```

Iteration is driven by Claude Code's **built-in `/loop`** (self-paced) and, optionally, **`/goal`**
(run-to-completion). loop-kit does **not** ship those — it ships the *recipe* they run.

## Independent (cross-vendor) checker — the stronger form

The bundled `validator` is **same-family** (Claude checking Claude), so a defect Claude
*systematically* makes slips past both writer and checker. The stronger form hands grading to a
**different model lineage**: [loop-verify](https://github.com/akihidem/loop-verify) (codex / GPT /
Gemini). Its verdict is the **same contract** as `validator`, so `loop-protocol` uses it as a
**drop-in** checker when it's available and falls back to the haiku validator otherwise.

- **Default (zero-config):** haiku `validator` — no external accounts, works the moment loop-kit is
  installed.
- **Opt-in upgrade:** run loop-verify (needs a codex / OpenAI / Gemini backend). Measured 4 vs 1
  independent defects over the same-family check — see loop-verify's edge bench.

This keeps loop-kit dependency-free by default while letting the loop reach cross-lineage
independence when you wire loop-verify in (run it as an MCP server so `independent_verify` is in the
session). A different lineage reduces shared blind spots; it doesn't make the check infallible.

## Install

```bash
# the akihidem marketplace is hosted in the claude-env-coach repo
claude plugin marketplace add akihidem/claude-env-coach
claude plugin install loop-kit@akihidem
```

## Usage

- **`/loopify <what you want>`** → get a loop-ready prompt (frozen criteria + L0 + checker
  recipe). It starts with an entry gate: trivial work or a target with no buildable L0 gets
  "don't loop" instead of a recipe. Hand the prompt to `/loop`, or let Claude run it inline.
- The **`loop-protocol`** skill auto-surfaces on YES/NO-decidable deliverable tasks and runs the
  freeze → implement → L0 → checker → judge cycle.
- The **checker** does the adversarial inspection against the frozen criteria, read from the
  per-run dir (`~/.claude/tmp/loop/<timestamp>-<slug>/criteria.md`, sha256-pinned):
  **loop-verify** (cross-vendor) if available, else the bundled **`validator`** agent (haiku).
  The completion gate uses the strongest independent checker available; the fast rung is for
  in-loop iterations.
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
| `loop-protocol` | skill | The freeze → implement → L0 → checker → judge procedure |
| `validator` | agent (haiku) | Adversarial inspection against frozen criteria (same-family default) |
| `goal-loop-template` | template | Anchor `/goal` to deterministic evidence |
| `loop-verify` | external, optional | Cross-vendor independent checker — drop-in for `validator` ([separate repo](https://github.com/akihidem/loop-verify)) |

## Honest limits

- **Not independent by default.** The bundled validator is the same Claude family, so a defect
  Claude *systematically* makes is missed by both. For cross-lineage independence, wire in
  **loop-verify** (cross-vendor) as the checker — see
  [above](#independent-cross-vendor-checker--the-stronger-form). Either way it reduces blind spots;
  it doesn't eliminate them.
- **L0 is the real guardrail.** For code, the deterministic test/lint/typecheck is the fort; the
  validator is a second pass. If there's no L0, the loop is much weaker — add one (extract logic
  into a pure function and test it).
- **For fact-checking, self-inspection is not the fort** — source grounding + human approval is.
- **Don't blanket every task.** A one-line fix, an investigation, or a question doesn't need a loop
  (token waste). Iteration plateaus around 3.
- **The loop guarantees the floor, not the ceiling.** L0 + checker can prove the frozen contract
  holds; taste, judgment and "is this the right thing to build" stay with a human (or your
  strongest model) at freeze time and at the final review.
- **Irreversible actions stay outside.** Merge / production deploy / publishing / sending
  externally are never part of the loop body — human gate.

## Related work

Designs that independently converge on the same shape (worth reading — loop-kit was built from
the author's own measurements, then cross-checked against these):

- [ralph-wiggum](https://github.com/anthropics/claude-code/blob/main/plugins/ralph-wiggum/README.md)
  (Anthropic's official plugin) — raw persistence: re-feed a prompt until done, one task per
  iteration. loop-kit adds the frozen contract and the external checker ladder.
- [proof-loop](https://github.com/LeoStehlik/proof-loop) — frozen acceptance criteria, builder ≠
  verifier in a fresh session, per-task evidence dir. The closest design; loop-kit adds the
  L0-first cost order and the cross-vendor rung.
- [aider](https://github.com/Aider-AI/aider)'s auto-test loop — feeds raw test output back and
  keeps the best attempt after the retry cap: the same rich-feedback and honest-stop instincts.

## License

MIT © akihidem
