# Rivus — Vision & Campaign

**Status:** active — skeleton landed 2026-08-10; campaign open (P0–P3).

## Vision

Rivus is a **Faber interpreter written in Faber** — `Fab(Faber → run Faber)`.

It reads a Faber program or script and runs it internally: lex, parse, HIR,
lower to MIR, then step through the program in-process — producing the
program's observable behavior (stdout, exit code, host effects) without
codegen, without building, without a host toolchain. This is the purest form
of the language: **the language hosts the execution of the language.**

This is not an invented architecture. It is the design Radix itself chose for
its scripting path (`faber run`): source → `analyze_source` (typed frontend)
→ `mir::lower_analyzed_unit_with_context` (HIR→MIR) → the MIR stepper, which
"dispatches the same intrinsics directly without emit/instantiate"
(`radix-mir-stepper/src/lib.rs`). Rivus re-implements that proven pipeline in
Faber, in `locale = "en"`, mirroring the Radix module tree file-for-file.

## Why an interpreter

| Argument | Detail |
|---|---|
| **Purest form** | The language runs the language. No intermediate artifact; the proof is execution, not bytes. |
| **Radix's own design** | `faber run` / `run_source` already ship in Radix; the oracle is real and tested. |
| **Honest metric** | Behavior parity (stdout + exit code) is true and hard to fake. Byte-exact emit parity was arbitrary and *was* faked (span-copy). |
| **Self-host changes shape** | `rivus run src/main.fab` interprets the interpreter — no emit correctness required. The old #78 type-inference cliff (compile yourself) becomes "interpret yourself" (subset support). |
| **MIR-first keeps doors open** | Interpret now, compile later — same typed IR. The canonicalizer locked the project to one output forever. |
| **Real programmability** | Host kernels (files, json/toml, subprocess, args, random) make Rivus able to run real scripts — a far better product demo than a normalizer. |

## History — two failed attempts, and what we keep

**Attempt 1 (Dec 2025 – Feb 2026):** a prior Rivus peaked at six codegen
targets in four days, collapsed to TypeScript-only, never reached self-hosting
(failed at pattern-binding type inference, old issue #78), and was replaced by
the Rust-written Radix. **Failure mode: scope creep.**

**Attempt 2 (this repo, Jul 2026):** a real lexer + parser were built; then
commit `4f53722` (u5) *inlined* the pipeline into a 1,055-line `main.fab`
monolith instead of fixing a one-line import syntax error. Emit became a
span-reprinter (`fons.sectio` — copied source bytes, emitted nothing useful on
real input). The parity gate was then doctored to "exit-0 not required," and a
reboot assessment documenting all of it was committed and **never executed**.
**Failure mode: inlining collapse + gate-doctoring under the pressure of a
deferred 3–5k-line port.**

Both failures were **process failures, not architecture or language failures.**
The planning docs from attempt 2 were excellent — the implementation betrayed
them, and the docs were amended to ratify the betrayal.

**What we keep:** the old tree lives at `rivus.to-be-deleted/` (git history,
forensics, and the planning docs — the Q1 corpus analysis, the unit specs, the
gate inventory). It is non-authoritative for current syntax; the lessons are
extracted into this document and `AGENTS.md`.

## Design

```
Faber source
   │
   ▼
driver (session, frontmatter)
   │
   ▼
lexer ─▶ parser ─▶ hir ─▶ hir/lower ─▶ semantic (collect/resolve/typecheck/…)
                                            │
                                            ▼
                                     mir/lower (typed MIR)
                                            │
                                            ▼
                                       mir-stepper (Value + runtime + Host kernels)
                                            │
                                            ▼
                                     stdout / exit code / host effects
```

- **en surface:** `faber.toml` `[reader] locale = "en"` — keywords are
  `fn`/`const`/`let`/`class`/`union`/`match`/…; Radix identifiers port 1:1.
- **Arena-ID:** all AST/HIR/MIR nodes are indices into arena vectors; no
  recursive struct fields (sidesteps the direct-recursion limitation).
- **Source-text firewall:** lex consumes source once; every later stage works
  on semantic structures + interned symbols. Nothing after lex re-reads bytes.
- **Kernels:** the interpreter's `Host` (StdioHost-equivalent) defaults to
  reject; file/subprocess/json/toml/random/args are opt-in (see SECURITY.md).
- **No MIR codegen backends** (wasm/wgsl/metal/llvm/fmir/sexp/…), **no
  GPU/tensor lane** (match Radix's reject behavior instead), **no borrow
  analysis** (Radix gates it off for the Faber surface).

Module tree with Radix mirrors and sizes: [docs/STRUCTURE.md](STRUCTURE.md).

## Metric

1. **Behavior parity:** for each corpus file, `rivus run foo.fab` stdout +
   exit code ≡ Radix's `run_source` (stepper, StdioHost) output + exit code.
2. **Reject agreement:** accept/reject matches Radix's stepper
   `StepperDispatchStatus` (`Implement` / `HostDelegate` / `IntentionalReject`)
   and `CapabilityGap::UnsupportedMirShape` classification. Reject agreement
   is a correctness floor (100%), not a coverage slider.
3. **Determinism:** same input → same output, run to run (ordered `tabula`).

No byte-level gates exist. Any gate phrased as "wired, exit-0 not required"
is void by definition.

## Phases

| Phase | Goal | Done-when |
|---|---|---|
| **P0 — Skeleton → spine** | The `en`-surface package builds green; lexer subset + interner; `rivus run` parses CLI args and reports "not implemented" honestly. | `faber build .` exit 0 on the en package; lexer smoke on a trivial file; no fake stages. |
| **P1 — Vertical slice** | A trivial program runs end-to-end: lex → parse → HIR → minimal typed MIR → mini-stepper for `const`/`print`/arithmetic/strings. | `rivus run salve-munde` prints what Radix's stepper prints; reject behavior matches on the first unsupported construct. |
| **P2 — Full runtime (the product milestone)** | Typecheck ladder, full MIR lowering, stepper runtime + host kernels → corpus run parity. | Parity on a growing % of the runnable corpus; 100% reject agreement on the expected-reject set. |
| **P3 — Self-interpret (aspirational)** | `rivus run src/main.fab` — the interpreter interprets itself. | Property: runs its own source; never a release gate. |

The typecheck ladder (from the prior campaign plan, consensus C3) applies:
rung 1 explicit-annotated monomorphic → rung 2 numeric widening → rung 3
nullable/union/optional → rung 4 generic substitution → rung 5 pattern-binding
inference (**optional**, the old #78 cliff — deferred to P3 at most).

## Hard rules (bought with two failures)

1. **No inlining.** Fix imports (`rivus:`-prefixed, subdirs allowed); never
   collapse modules into `main.fab`.
2. **No fake stages.** Pass-through is a stub, not a stage.
3. **No doctored gates.** Done-whens are real or the unit is not done.
4. **Behavior is the product, not bytes.** No emit oracle, no `--target`,
   no formatting parity, no span-copy.
5. **Reject agreement is the honesty floor.** Matched against the Radix
   stepper from P1 onward.
6. **Typecheck is required before MIR.** MIR is typed; don't skip semantics
   to make a slice faster.
7. **Small units, green after every unit.** One closeout run per unit, then
   stop (workspace Cargo discipline).

## Scope locks

| Lock | Meaning |
|---|---|
| **L1** | One product: the interpreter (`rivus run`). No codegen, ever, in this repo. |
| **L2** | No MIR codegen backends and no GPU/tensor lane; match Radix reject behavior instead. |
| **L3** | en surface for Rivus source; Radix identifiers port 1:1. Input programs are whatever the corpus is (Latin default); Rivus's *own* source is en. |
| **L4** | Arena-ID data model throughout. |
| **L5** | The stepper's supported subset is defined by the Radix classifier, not by what we "feel like" implementing. |

## Archive

- Old tree: `rivus.to-be-deleted/` (preserved; not authoritative).
- Prior planning docs: readable there for the Q1 corpus analysis, unit
  specs, and gate inventory — treat as reference, not current state.
