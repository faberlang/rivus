# Rivus — Agent Instructions

Rivus is a **Faber interpreter written in Faber** (`Fab(Faber → run Faber)`).
It reads a Faber program and runs it in-process: lex → parse → HIR → lower to
MIR → step (the MIR stepper, Radix's `faber run` design). No codegen.

Source of truth for the language: `radix/EBNF.md` (grammar), `radix/` front-end
crates (structure to mirror), `radix-mir-stepper` (interpreter to mirror),
`radix/stdlib/locale/en/pack.toml` (keyword + intrinsic-method surface).

## Identity

- **Written in:** Faber, `locale = "en"` — declared in `faber.toml`
  (`[reader] locale = "en"`). No per-file frontmatter needed. Keywords:
  `fn`, `const`, `let`, `var`, `class`, `union`, `enum`, `type`, `match`,
  `if`/`else`/`elif`, `while`, `for`, `return`, `import from "pkg:module"`.
- **Intrinsic method spellings:** the en reader-locale pack maps intrinsic
  names to English surfaces (`continet → contains`, `longitudo → length`,
  `sectio → slice`, `appende → append`, …), so en-locale source writes
  `.contains(c)` — e.g. `ASCII_ALNUM_CHARS.contains(c)` in
  `is_ascii_alphanumeric`. Canonical Latin spellings (`.continet(c)`) still
  resolve in any locale — lookup is surface-first and the canonical name is
  the registry identity — and both spellings lower to the same intrinsic.
- **Module imports:** `§` template paths, source-root-relative —
  `import from "§src/lexer/token" private token` (faber.toml
  `[paths.templates] src = "src"`; coreutils pattern). Provider-qualified
  imports (`rivus:lexer/token`) require `build.kind = "lib"` and are rejected
  for this bin package; bare paths are an unknown scheme (see migrate-
  diagnostics pilot notes, 2026-08-10). **Never inline to work around an
  import error; fix the import.** Inlining is what killed the prior attempt.
- **Entry:** `src/main.fab` (bin package, target rust).
- **String templates:** interpolate with `§` holes filled positionally by the
  trailing parenthesized args — `"ref-§-§"(name, i)`, `"cannot read '§': §"(path, err)`
  (corpus `discerne.fab` multi-hole; tela single-hole). Prefer templates over
  `+` concat chains for readability.
- **textus indexing:** `s[i]` returns the **`ascii` char type** (non-
  allocating), NOT a 1-char string — compare `s[i] ≡ '\n'`, never
  `s[i] ≡ "\n"` (exact-type equality fails). Range slices use the `‥`
  operator: `s[start‥end]` returns a `textus` substring.

## Code style (reference-quality)

This codebase is the reference example of what Faber looks like — future LLM
implementations pattern-match on it. Write it to be attractive, not just
correct. These rules match the upcoming faber `forma` auto-format behavior:

- **No brace cuddling.** A block-joiner (`else`, `elif`, `catch`, …) starts
  on its own line — never `} else {`. The closing brace and the joiner are
  separate lines. This applies to everything that attaches to a prior block,
  not just `if`/`else`.
- **Full-line comments only.** An inline `#` after code is a lex error
  (`LexErrorKind::InlineComment`), so rhythm comes from blank lines plus
  whole-line comments — never trailing notes.
- **Section rhythm.** Separate logical sections with a blank line and a brief
  comment saying what the section does and why (including defaults). A
  multi-scan function should make its structure visible at a glance.
- **Name magic numbers.** A bare numeric comparison gets a named constant
  with an explanation; the name is the documentation.
- **Hoist repeated receiver calls.** `source.longitudo()` is read once into
  `const int len` and reused — one read, single source of truth, no repeated
  calls that could disagree if the receiver ever becomes mutable.

## The pipeline (Radix mirror)

```
source → driver (session/frontmatter) → lexer → parser → hir → hir/lower
       → semantic (collect/resolve/typecheck/analysis) → mir/lower → mir-stepper
```

MIR is **typed**: the typecheck pass runs before lowering (NumericWidth,
TypeTable). Do not skip typecheck to "make it faster."

## Metric (the only gates)

- **Behavior parity:** `rivus run foo.fab` stdout + exit code ≡ Radix's
  `run_source` (stepper, StdioHost) on the same input.
- **Reject agreement:** accept/reject must match Radix's stepper
  `StepperDispatchStatus` (Implement / HostDelegate / IntentionalReject) and
  `CapabilityGap` classification. A mismatch is a defect.
- **No byte targets.** No emit oracle, no `--target`, no formatting parity.
  If a gate says "exit-0 not required," it is not a gate — remove it.

## Hard rules (bought with two failed attempts)

1. **No inlining.** Mirror Radix's module split. Fix imports; never collapse
   files into `main.fab`.
2. **No fake stages.** A stage that "passes through" its input is a stub, not
   a stage. Every stage transforms its input.
3. **No doctored gates.** Never amend a done-when to bless unshipped work.
4. **No source-text re-reading after lex.** Semantics + interned symbols only.
5. **No codegen / no MIR codegen backends.** The stepper is the only consumer.
6. **No GPU/tensor lane.** Match Radix's reject behavior; do not implement.
7. **en surface everywhere.** Radix identifiers port 1:1 — do not Latinize.
   Intrinsic method calls included: en source writes `.contains(c)`,
   `.length()`, `.slice()`, `.append()` (the pack's intrinsic surface), not
   `.continet(c)` — though canonical spellings remain valid.

## Validation discipline (workspace rules apply)

- Narrow in-loop checks only: `faber check` on touched files, a single
  touched module, or `./scripta/test --check` from `radix/` when the change
  crosses the toolchain. Never hold the shared Cargo lock with full builds.
- One closeout run after the last product edit, then stop. Full runs are
  auditor-owned.
- Tests strongly preferred for new behavior; red-green; smallest proof.

## Git discipline (workspace law)

- Path-limit every commit: `git add -- <exact paths>` then
  `git commit -- <same paths>`. Never bare `git commit`.
- Staged-path check before committing (`git diff --cached --name-only`):
  only intended paths; unstage foreign ones.

## Archive

The prior attempt is preserved at `rivus.to-be-deleted/` (forensics + the
planning docs). It is non-authoritative for current syntax and structure —
do not copy code from it. Lessons are extracted into `docs/CAMPAIGN.md`.

## Structure map

See `docs/STRUCTURE.md` for the module tree, Radix mirrors, sizes, and status.
