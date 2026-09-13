---
name: lean_playbook
description: Use whenever an agent does ANY Lean 4 work - proving, formalizing, verifying, debugging, or refactoring theorems, editing any .lean file, running lake/lakefile builds, working with Mathlib, or using the lean-lsp MCP tools. Loads the session playbook (loop model, hard gates, proof-ladder planning, failure modes, handoff discipline).
---

# Lean 4 Session Playbook

Reference for agents approaching Lean 4 work (proof development, debugging,
formalization, refactoring). Distilled from post-mortems of multi-session
agentic Lean projects. Assume a Lean LSP is available (ideally the `lean-lsp`
MCP server); everything below assumes it.

## Core principles

1. **The elaborator is the only oracle.** Never mentally simulate what a
   tactic, unifier, or `cases`/`rcases` pattern will do. If you have spent
   more than a short paragraph reasoning about "what this tactic would do",
   stop and compile a probe instead (scratch file or LSP goal query). Mental
   simulation of proofs is the single most expensive failure mode observed:
   it produces no compiling code, burns context, and its conclusions still
   require a compile to confirm.
2. **Statement before proof.** Before proving any nontrivial lemma, try to
   break it: construct a counterexample, or check edge/abstract cases. If the
   statement fights back, weaken or restructure the statement rather than
   pushing harder. A false lemma costs an entire debugging arc; a
   counterexample probe costs minutes.
3. **Transcribe, don't construct.** Before proving anything, find the
   strongest already-proved proof of a neighboring result (in this codebase,
   in an earlier commit, or in a library) and adapt it as a template. A new
   corollary over existing machinery beats a from-scratch construction.
4. **Design maturity predicts landing speed.** Do not start appending code
   until the statement structure, definitions, and proof skeleton have
   converged. If a proof design stalls, simplify the *statement* or the
   *definition* - change what makes the proof hard, don't out-muscle it.
5. **Compilers test, gates certify.** If it builds with zero sorries and only
   standard axioms, it is done. Nothing is "done" that hasn't passed the
   verification ladder (below).
6. **Proofs are audits.** Executable definitions (enumerators, checkers,
   decoders) are paired with soundness AND completeness proofs before being
   trusted downstream, and the proofs are expected to find real bugs in the
   definitions - that is their function. Smoke tests only confirm; proofs
   catch over-approximation, forged cases, and gap classes tests cannot
   reach. An audit that finds no bugs is itself a result - record it as one.

## Session workflow

- **Baseline first.** Before any edit: build, grep for `sorry` (word-boundary,
  excluding comments/docstrings), and confirm green. Never edit on an
  unverified assumption of green.
- **Size the unit before starting.** Read the status file and gotcha ledger,
  state the proof ladder (below), and decide the dispatch plan up front. If
  the unit needs more than one session or subagent, split it now - monolithic
  work units die mid-flight; small ones land.
- **Small increments.** Append at most ~50-100 lines of new Lean per compile
  cycle. Prove helpers first, then the theorem that composes them. Large
  "big-bang" blocks reliably cost an order of magnitude more debug cycles
  than the same code landed incrementally.
- **Fast inner loop, slow outer gate.**
  - Inner loop: single-file check (`lake env lean File.lean`) or LSP
    diagnostics after each edit.
  - Outer gate, at every milestone: fresh build + sorry census + `#print
    axioms` sweep + smoke checks. "Fresh" is load-bearing: when in doubt,
    delete the module's `.olean` AND `.trace` before building - trace-cache
    replay can report success without producing the artifact, and a green
    build without a fresh `.olean` is a false gate.
  - Include `#print axioms` / axiom scanning in the definition of done for
    proof artifacts, not as an optional extra. Banned: `sorryAx`. Allowed:
    the standard trio (`propext`, `Quot.sound`, `Classical.choice`) plus any
    project-approved extras, stated as an explicit baseline.
  - Write the gate as a single command (e.g. a `make gate` target) so every
    run ends identically and resume points are unambiguous.
  - Compile-time `#eval`/`#check` smoke probes belong in the gate, with the
    expected output written in an adjacent comment so drift is visible on
    re-run.
- **Stop at green.** At every green point: commit (WIP commits with design
  rationale in the message are fine and valuable), update the project's
  status/gotcha documents, and re-assess. Never continue past a stated
  stopping point - that is where runaway loops begin.
- **One task at a time; keep task state current.** Maintain a todo list and
  update it as work proceeds, not on request.

## Plan the proof ladder before tactics

Before writing tactics on a nontrivial theorem, fix the architecture:

- **Induction scheme**: which induction principle, and which motives. Mutual
  blocks usually require the FULL statement as the motive - proving the
  convenient sub-statement first is a known dead end, and retro-fitting
  motives later costs a rewrite.
- **Lemma ladder**: what proves what, in dependency order, each lemma stating
  exactly the conclusion its consumer needs.
- **Case table**: constructor -> expected witness shape, so a stuck case is
  recognizable as "missing lemma" vs "wrong statement".

If you cannot state the ladder, you are not ready to prove. Tactic-first
work on a large theorem degenerates into error-driven wandering.

## Debugging rules (LSP-first)

- Query goal state at the error site (`lean_goal` or equivalent) *before*
  attempting a fix. Most recurring compile errors are information-retrieval
  failures, not mathematical ones.
- Hover / `#check` the exact signature of any borrowed definition or lemma
  before destructuring its return type, passing its arguments, or matching
  its binder structure. Guessing nesting, arity, or premise order from error
  text wastes a compile cycle per guess.
- Use multi-tactic attempts (`multi_attempt`) to choose between candidate
  tactics in one round trip instead of one compile per guess.
- Use `lean_run_code` or a throwaway scratch file for tactic-behavior probes
  ("what does the goal look like after this rewrite?") - never the main file.
- Read the actual error. When a tactic fails with an equation it could not
  solve, read that equation carefully before trying workaround tactics
  (typed rewrites, generalize, classical choice). Workaround ladders stack
  complexity; the root cause is usually visible in the unsolved equation.
- **Two-strike rule.** The same command or tactic twice with no new
  information means stop and re-anchor. If `simp` (or any automation) fights
  back twice, do not iterate a third time blindly - change the approach
  (minimal `simp only` set, explicit peeling lemmas, different induction)
  or consult the gotcha ledger, where the pattern is probably already
  numbered.
- After any interruption, resume, or model/context switch: re-verify file and
  git state before acting. Never patch based on remembered state.

## Statement and definition design

- Prefer making side-conditions *outputs* of a definition or lemma rather
  than properties to be re-derived at each use site.
- Prefer factored, strengthened intermediate lemmas (one-step versions with
  exactly the conclusion the induction needs) over monolithic theorems.
- Avoid definitions that force dependent-elimination pain: if a type is
  indexed by a term containing a `match`, casing order matters - eliminate
  the index-determining hypothesis first, while terms are still variables.
- In projects without Mathlib, assume nothing about the stdlib. Probe for
  lemma/tactic existence (`local_search`, a scratch file) before use;
  derive `Inhabited` instances early for enums used in inductive proofs.
- Check definitions compile to usable recursors early: recursors into `Type`,
  mutual blocks, and nested inductives all have restrictions that are cheap
  to discover with a smoke file and expensive to discover mid-proof.

## Authoring discipline

- Edit Lean files with file-editing tools, never shell heredocs or scripted
  text splices. Scripted surgery on unicode-heavy Lean reliably corrupts
  tokens and indentation. If a scripted transform is unavoidable, immediately
  re-read the transformed region and compile.
- Re-read any drafted block before compiling it - a zero-cost pass catches
  placeholders, mismatched delimiters, and unprovable steps that each
  otherwise cost a compile cycle.
- Never write `sorry` silently. If a proof cannot land, leave the statement
  with an explicit `sorry` plus a comment describing the blocker, or revert
  to the last green state and commit the blocker description instead.

## Failure modes to actively avoid

| Failure mode | Countermeasure |
|---|---|
| Long pure-thinking proof simulation | Compile a probe; query goal state; timebox reasoning with no tool calls |
| Proving an unverified statement | Counterexample probe or template check first |
| Big-bang code appends | <=100 lines per compile cycle; helpers first |
| Blind patching from error text | Goal-state query at the error site before each fix |
| Stdlib/Mathlib API assumptions | Existence probe before every unfamiliar name |
| Silent verification gates | Scripts must fail loudly: strict exit-code handling, word-boundary sorry scan, no `\|\|` fallbacks that mask path errors |
| Stale state after interruption/compaction | Re-check git/file state before the first action after any resume |
| Continuing past green | Commit + document at every green point |
| Tactic rabbit hole (same failure repeating) | Two-strike rule: change approach or consult the gotcha ledger |
| Stale-olean false green | Delete `.olean` + `.trace` before any gate you intend to trust |
| Trusting an agent's completion report | Verify side effects (files/commits/build) on disk before recording the outcome |
| Silent tooling failure (nothing in any log) | Treat missing log entries as the finding; fix visibility (logs-to-file) before debugging content |
| Over-claimed results | Track exactly what is proved vs argued; keep a proves / does-not-prove list; verify claims against the artifact by grep/recompile, not memory |

## Handoff and continuity

- Project state lives in files, not chat history. Maintain, per project:
  - a status file: phase, loop/iteration number, current blocker, next step;
  - a **gotcha ledger**: numbered, dated entries for every tool/tactic/
    language pitfall encountered, written so a fresh session can recognize
    the failure on first sight. Ledgers compound: they predict failures and
    turn repeat errors into instant recognitions. Recurring patterns
    graduate into a small project-local snippet/lemma library instead of
    being re-derived each time.
  - commits whose messages carry design rationale, written for the next
    reader, not the diff.
- When resuming (new session, new model, post-compaction): reconstruct state
  from files and `git log` first, chat history second. Verify the described
  green state actually builds before trusting it.
- Dispatching subagents and long runs:
  - one small work package per dispatch; monolithic units exceed output
    budgets and die mid-emit;
  - externalize progress from minute one - a scratch log updated per chunk,
    not held for a final report, so a dead run is resumable rather than lost;
  - resume a dead subagent by its session id instead of re-dispatching from
    scratch; its state lives in the log, the scratch files, and the platform
    session store;
  - verify completion claims against side effects on disk (files, commits,
    build artifacts) before recording the outcome - false-positive AND
    false-negative completion reports both occur; the artifact trail is the
    truth.
- When blocked: stop, record the blocker and current green state, and hand
  off cleanly rather than thrashing.

## Session close checklist

1. Fresh build green (`.olean`/`.trace` cleared if in doubt); `sorry` census
   matches the project's declared baseline (0, or every remaining sorry
   documented with its blocker).
2. `#print axioms` profile recorded for flagship theorems (sweep where
   feasible); any delta from the expected profile explained.
3. Status file and gotcha ledger updated at the current green point.
4. All work committed; nothing relies on uncommitted state.
5. Any claims made (in docs or to the user) verified against the artifact.
