# Changelog

## 0.2.0 — 2026-07-02

The protocol, refined against a year of the author's measurements and a prior-art sweep
(Anthropic's ralph-wiggum, proof-loop, aider, Reflexion). No new components; every shipped
piece got sharper.

### Fixed
- **Per-run criteria dir (was: a shared fixed path).** Criteria now freeze into a unique
  `~/.claude/tmp/loop/<timestamp>-<slug>/` instead of `~/.claude/tmp/criteria.md`. The fixed
  path let concurrent sessions read *each other's* criteria (a measured failure). The run dir
  doubles as an evidence dir (inspection logs, check scripts), and the checker is told to read
  only the paths it is given — if the file is missing it fails loudly instead of hunting around.
- **Validator no longer has a defect quota.** The "read with the premise that you must find at
  least one defect" instruction is gone: a forced-defect stance measurably kills precision.
  The validator still hunts adversarially, but the verdict must follow the evidence — a clean
  PASS is a legitimate output, and inventing or inflating defects to look diligent is
  explicitly forbidden.

### Added
- **Entry gate** (in `/loopify` and the skill): trivial work isn't looped, and a target with no
  buildable deterministic check gets an honest "a loop can't bite here" instead of Goodhart
  theater.
- **sha256-pinned freeze**: the criteria file's hash is recorded at freeze time and re-checked
  each iteration; a mismatch means the contract was tampered with and stops the loop. The
  validator echoes the hash of the criteria file it read.
- **Artifact convention**: for code the git diff is the artifact; for text/docs/design the
  deliverable is written and refreshed to `<run dir>/artifact.md` every iteration, so the
  checker always has a stable path to read.
- **Standalone prompts**: `/loopify` instantiates a *concrete* run-dir path in the generated
  prompt (no unexpanded variables) — the loop-ready prompt must run as-is when pasted into
  `/loop`.
- **Checker ladder made explicit**: L0 deterministic checks first (cost order), the fast rung
  (validator / loop-verify) in-loop, and the *strongest independent* rung as the completion
  gate on the full final artifact (the git diff for code; the run dir's artifact.md for text).
- **Rich feedback rule**: on L0 failure the raw failing output is fed back verbatim — never a
  summary (rich feedback is what stays Goodhart-safe in the author's measurements).
- **Single-builder rule**: splitting the builder into fixed roles measured worse than one
  builder with real test feedback (0.5 vs 1.0 in the author's SWE-bench probe); the loop keeps
  one builder and puts the diversity in the checker.
- **Early stop**: the same defect surviving two consecutive iterations, or an empty diff, stops
  the loop before the cap — with an honest unresolved list.
- **Membrane**: irreversible / outward-facing actions (merge, production deploy, publishing,
  external sends) are explicitly outside the loop body, behind a human gate.
- **Casting note**: freeze quality is the thing to protect — criteria freezing and final
  inspection belong to your strongest model (or a human); implementation may be delegated, the
  judgment never split.
- **Related-work section** in the README (ralph-wiggum / proof-loop / aider) with what loop-kit
  adds over each.

## 0.1.0 — 2026-06

Initial public release: `/loopify` command, `loop-protocol` skill, `validator` agent (haiku),
`goal-loop-template`, cross-vendor checker wiring for loop-verify.
