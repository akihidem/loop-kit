# loop-kit — CONTEXT

## North star
Package the self-verification loop so anyone can `claude plugin install loop-kit@akihidem` and get:
`/loopify` (recipe generator), `loop-protocol` (skill), `validator` (agent), `goal-loop-template`.
The transferable value = grounding the loop in external verification (L0 = fort, builder != checker),
NOT multi-agent structure.

## Topology
- This repo = the loop-kit plugin (standalone). Future GitHub: akihidem/loop-kit (public).
- Distributed via the existing `akihidem` marketplace (manifest lives in the claude-env-coach repo):
  add a loop-kit entry to `plugins[]` with a github source. env-coach itself stays untouched.
- Built-in `/loop` and `/goal` drive iteration — NOT shipped here, documented as dependencies.

## Frozen acceptance criteria (v0.1.0)
1. `claude plugin validate .` exits 0.
2. `claude plugin details loop-kit` lists 4 components (loopify / validator / loop-protocol / template).
3. Zero personal/private references in shipped content (no wiki-style memory links, no personal absolute paths, no private repo names).
4. README (EN+JA) has an Honest-limits section (same-family self-grading caveat + "L0 is the fort").
5. README/INSTALL state `/loop` and `/goal` are built-in dependencies, not shipped.

## Out of scope (v1)
- A design-specialized variant (e.g. a visual/layout L0 gate) — out of scope for v1.
- Re-pointing the akihidem marketplace to a dedicated marketplace repo.
- A framework-agnostic reference harness.

## Source (author's private originals, kept as-is)
~/.claude/loop-protocol.md, commands/loopify.md, agents/validator.md, goal-loop-template.md
