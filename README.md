# Rivus

> Rivus is a **Faber interpreter written in Faber** — `Fab(Faber → run Faber)`.
> It reads a Faber program or script and runs it in-process: lex, parse, HIR,
> lower to MIR, step — no codegen, no build, no host toolchain.

[![License: MIT](https://img.shields.io/github/license/faberlang/rivus)](LICENSE)

**Status:** skeleton, pre-implementation. This repo was restarted 2026-08-10
after the prior attempt was parked at `rivus.to-be-deleted/` (see
[docs/CAMPAIGN.md](docs/CAMPAIGN.md) for the full history and the reboot
rationale).

## What Rivus is

Rivus exists for one reason: the milestone of writing a compiler-runtime in
Faber that can intake a Faber program and **run it internally**. The stepper —
an in-process evaluator over the MIR — is the interpreter backend. It is the
design Radix itself chose for its scripting path (`faber run`): source →
analyze (typed frontend) → lower to MIR → step, dispatching host kernels
directly without emit/instantiate.

```
Faber source ─▶ Lex → Parse → HIR → Lower (MIR) → Step ─▶ stdout / exit code
```

- **Written in:** Faber, `locale = "en"` (English keyword surface — `fn`,
  `const`, `let`, `class`, `union`, `match`, ...), so the source reads like
  the Rust Radix it mirrors. Identifiers, module names, and type names port
  1:1 from Radix.
- **Hosted on Radix:** Rivus is compiled to a native binary by Radix. It uses
  the subset of Faber that Radix compiles today.
- **Metric:** behavior parity — `rivus run foo.fab` produces the same stdout
  and exit code as Radix's own `run_source` path on the same input, and
  rejects exactly what the Radix stepper rejects (reject agreement).

## What Rivus does not do

- No codegen. No emit oracle, no byte-level output targets, no `emit
  --target` port. Behavior is the product, not bytes.
- No MIR codegen backends (wasm/wgsl/metal/llvm/...). The MIR core + stepper
  are kept; the codegen consumers of MIR are not.
- No GPU/tensor lane initially (match Radix's reject behavior instead).
- No borrow/lifetime analysis (Radix gates it off for the Faber surface).
- No reader-locale packs beyond what the input programs use; Rivus itself is
  written in `en`.

## Repository layout

```text
rivus/
  faber.toml         package manifest (target rust, kind bin, reader en)
  src/               compiler/interpreter source (Faber, en surface)
    main.fab         CLI entry: rivus run <file.fab>
  docs/
    CAMPAIGN.md      vision + campaign (history, design, phases, hard rules)
    STRUCTURE.md     module tree map — Radix mirror, sizes, status
  scripts/           parity harness (planned: rivus run vs radix run)
```

The module tree mirrors Radix's front-end + MIR + stepper crates file-for-file
(see [docs/STRUCTURE.md](docs/STRUCTURE.md)).

## Getting started (once the skeleton builds)

```bash
cd rivus && faber build .
./target/debug/rivus run path/to/program.fab
```

The first campaign gate (P0) is proving the `en`-surface package builds green.

## License

MIT — see [LICENSE](LICENSE).
