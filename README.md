# lean_playbook — a skill for agentic Lean 4 work

A user-level [agent skill](https://code.claude.com/docs/en/skills)
that installs a battle-tested session playbook for **any** Lean 4 work:
proving, formalizing, verifying, debugging, and refactoring in lake-based
projects (standalone or Mathlib).

## What it encodes

The playbook is distilled from post-mortems of multi-session agentic Lean
projects — what repeatedly worked, what repeatedly failed, and the mechanisms
behind both. Every rule carries its tell and countermeasure.

Highlights:

- **Two-loop model** — a seconds-scale inner loop (single-file
  `lake env lean` checks / LSP diagnostics) and a milestone outer gate, never
  blurred.
- **Hard gates** at every unit boundary: fresh build (delete `.olean` +
  `.trace` first — trace-cache replay can false-green), axiom sweep
  (`sorryAx` banned, standard trio allowed), sorry census against a declared
  baseline, and smoke `#eval`s with expected output written adjacent.
- **Probe-first / statement-before-proof** — counterexample probes and
  scratch files before touching load-bearing modules; the elaborator is the
  only oracle, mental proof simulation is the most expensive failure mode.
- **Ladder-first planning** — induction scheme and motive discipline
  (mutual blocks need the FULL statement as motive), lemma ladder, case
  table, before any tactics.
- **Two-strike rule** — same tactic twice with no new information means
  re-anchor, not iterate.
- **Proofs are audits** — executable definitions get soundness AND
  completeness proofs, which are expected to find real bugs.
- **Log-first dispatching** — one small work package per subagent dispatch,
  progress externalized from minute one, dead runs resumed by session id,
  completion claims verified against artifacts on disk.
- **Gotcha ledger** — numbered, append-only toolchain landmines per project,
  with recurring patterns graduating into a local lemma library.

## Install

```sh
# Claude Code (user level)
git clone git@github.com:virtualfunction/lean4_playbook_skill.git \
  ~/.claude/skills/lean_playbook
```

Or copy `SKILL.md` into the skills directory of any agent platform that
supports the SKILL.md format.

The skill triggers automatically whenever an agent has to do Lean work —
editing `.lean` files, proving/formalizing, running `lake` builds, working
with Mathlib, or using the [`lean-lsp-mcp`](https://github.com/oOo0oOo/lean-lsp-mcp)
tools.

### opencode

For [opencode](https://opencode.ai), copy the same folder to
`~/.config/opencode/skills/lean_playbook/` — the format is compatible.

## Pairing

The skill assumes a Lean LSP is available (ideally the `lean-lsp` MCP
server) and its debugging rules reference the `lean_*` MCP tools. Install
the server alongside:

```sh
claude mcp add lean-lsp -s user -- uvx lean-lsp-mcp
```

## Files

- `SKILL.md` — the skill itself (frontmatter trigger + full playbook body).
- `README.md` — this file.

## License

MIT
