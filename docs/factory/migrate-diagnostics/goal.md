# Goal: migrate-diagnostics

**Status**: planned — mapping complete 2026-08-10; implementation pending
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-diagnostics/
**Primary surface**: `radix/crates/radix-diagnostics/src/{lib.rs, diagnostic.rs, span.rs, spec.rs}` → `rivus/src/diagnostics/{diagnostic.fab, span.fab}`
**Depends on**: none (foundation)
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Port `radix-diagnostics` — the Shape 1B foundation for diagnostic transport and source spans — into the en-surface module `rivus/src/diagnostics/`. It defines the canonical `Span` (byte-range source-location currency shared by every later crate), the `Diagnostic` builder (severity + phase + typed error kind + named args + span), the phase error-kind taxonomies, and `DiagnosticSpec`. The crate has no dependencies on lexer/parser/semantic/driver, so this is the first migratable unit; `migrate-lexer` and `migrate-syntax` both depend on its `Span`.

## Crate analysis

Source read: `radix/crates/radix-diagnostics/src/` (4 files, ~725 lines; no tests beyond lib.rs `smoke_tests`). Dependencies: serde derives only, `std::fmt` (Display), `std::path::Path` + `std::io::Error` (only inside `Diagnostic::io_error`).

- **lib.rs** (~28 ln) — module wiring, re-export list, invariants. Ports as no file: re-exports become direct imports at consumer sites; the invariants become the module doc comment.
- **span.rs** (~60 ln) — `Span { start: u32, end: u32 }` half-open byte range; `new` (debug-asserts `start <= end`), `merge`, `len`, `is_empty`, `Default` (= 0..0). **This is the concrete Span definition in radix.**
- **diagnostic.rs** (~610 ln) — the core model: `DiagnosticArg` (named fact: `name: &'static str`, `value: String`); `Severity` (Error / Warning / Info); `DiagnosticPhase` (Io / Lex / Parse / Resolve / Lower / Typecheck / Analysis / Mir / Codegen / Tool, with Display→lowercase name); `LexErrorKind` (8 unit variants); `ParseErrorKind` (~30 unit variants); `WarningKind` (15 unit variants); `SemanticErrorKind` (~40 variants, ends `Warning(WarningKind)`); `PhaseErrorKind` (Lex / Parse / Semantic wrapping); `Diagnostic` struct (10 fields) + ~20 builder/accessor methods + `io_error`, `codegen_error`; free fns `diagnostic_identity`, `parse_arg_value`, `get_line_at_offset`, `nearest_char_boundary`.
- **spec.rs** (~55 ln) — `DiagnosticSpec { code, help }` + `locale_fallback_spec` / `locale_suggestion_spec` / `locale_unknown_spec` (LOCALE001–003).

Label inference: identifiers port 1:1 (hard rule L3) — `Diagnostic` → `Diagnostic`, `Span` → `Span`, etc. Rust types → en data model: `u32` → `int`, `String`/`&str` → `string`, `Vec<T>` → `list<T>`, `Option<T>` → nullable (`T ∪ null` or `optional` field), `&'static str` code → `string`. No char/ord, no tuples (data model); record types are `class`, sum types are `union` (triga pattern: `union` + `case` in `match`, free constructor fns).

## Module mapping (crate → rivus)

| Radix file | Rivus file | Content → Faber (en) |
|---|---|---|
| `src/span.rs` | `src/diagnostics/span.fab` | `class Span { int start; int end }`; free constructor `span(int start, int end) → Span`; methods `merge(Span other) → Span`, `len() → int`, `is_empty() → bool`; zero-span constructor for the `Default` sites |
| `src/diagnostic.rs` | `src/diagnostics/diagnostic.fab` | `class DiagnosticArg { string name; string value }`; `union Severity`; `union DiagnosticPhase` (with `name() → string` porting Display); `union LexErrorKind`; `union ParseErrorKind`; `union WarningKind`; `union SemanticErrorKind` (incl. nested `Warning(WarningKind)` payload variant); `union PhaseErrorKind` (wrapping variants); `class Diagnostic` (10 fields, nullable/optional where `Option<T>`) + builder/accessor methods; `is_permissive_check_downgrade()`; free fns `diagnostic_identity`, `parse_arg_value`, `get_line_at_offset`, `nearest_char_boundary` |
| `src/spec.rs` | folded into `diagnostic.fab` | `class DiagnosticSpec { string code; string help optional }`; `locale_fallback_spec()`, `locale_suggestion_spec()`, `locale_unknown_spec()` |
| `src/lib.rs` | — (no file) | re-exports become `import from "rivus:diagnostics/span" private span` / `import from "rivus:diagnostics/diagnostic" private diagnostic` at consumer sites; invariants → header comment |

STRUCTURE.md lists exactly two files for `src/diagnostics/`, so `spec.rs` folds into `diagnostic.fab` (smallest correct; a `spec.fab` split is a rename-only deviation, not required for mirror fidelity).

**Span ownership resolution** (ripples into migrate-lexer + migrate-syntax):
- In radix the concrete `Span` is defined in **radix-diagnostics** (`span.rs`); `radix-lexer` re-exports it (`radix-lexer/src/token.rs:27` `pub use radix_diagnostics::Span;`, re-exported from `lib.rs:53`); `radix-syntax/src/span.rs` is **not** a Span type — it is the `Spanned` trait (`fn span(&self) -> Span`) consuming `radix_lexer::Span`.
- **Decision:** canonical `Span` for rivus lives in `src/diagnostics/span.fab`. The lexer goal imports it from `rivus:diagnostics/span` (and may re-export through `lexer/token.fab` to mirror radix-lexer's `pub use`). `src/syntax/span.fab` (per STRUCTURE.md) carries only the provenance contract — the `Spanned` trait port (en `interface` or per-class `span` fields; decided in migrate-syntax, which depends on this goal). No second Span type anywhere.

## Port notes / frictions

1. **en surface everywhere (L3).** Radix identifiers port 1:1: `Severity`, `DiagnosticPhase`, `DiagnosticArg`, `Span`, variant names, method names (`with_span`, `merge`, …). Do not Latinize. Module-qualified access (`span.Span`, `diagnostic.Diagnostic`) is the triga-proven pattern.
2. **Drop serde.** `Serialize`/`Deserialize` derives serve no rivus boundary (no cross-process diagnostics serialization). Drop both derives and `spec.rs`'s `#![allow(dead_code)]`.
3. **Drop `codegen_error`.** No codegen (scope lock L1); the `Codegen` *phase variant* stays in the union (enum completeness), the constructor goes.
4. **`io_error` port.** Rust takes `&Path` + `std::io::Error`; rivus reads files through host kernels, so port as `io_error(string path, string error_message) → Diagnostic` and let the driver/kernel goal supply the message. Exact io text is stderr — not parity-critical.
5. **Builder chain.** Rust builders take `mut self` and chain (`#[must_use]`). If en methods can't take self by value, port chains as sequential mutating calls (`let d ← diagnostic.error("…")`; `d.with_span(...)`; …) — behavior identical, no fake stages.
6. **Byte offsets and textus.** `nearest_char_boundary` / `get_line_at_offset` are byte-offset helpers over source text. No `char`/`ord` in the data model; port boundary-snapping with textus runtime ops (or a shared byte-boundary helper used by the lexer cursor). Offsets stay `int`.
7. **`diagnostic_identity` quirk.** The `WARN014 file_interface_export_skipped` special case references a dropped surface (no file-interface exports in rivus). Decision: fold into the general `code.issue` path; flag if an exporter ever appears.
8. **`debug_assert!(start <= end)`** — port as a documented precondition on `span(...)` (runtime assert only if en offers one).

## Done-when

1. `src/diagnostics/span.fab` defines `class Span { int start; int end }` with constructor, `merge`, `len`, `is_empty`, zero-span default — en identifiers.
2. `src/diagnostics/diagnostic.fab` defines `DiagnosticArg`, `Severity`, `DiagnosticPhase`, `LexErrorKind`, `ParseErrorKind`, `WarningKind`, `SemanticErrorKind`, `PhaseErrorKind`, `Diagnostic` (all fields + builder/accessor methods), `diagnostic_identity`, `parse_arg_value`, `get_line_at_offset`, `nearest_char_boundary`, `DiagnosticSpec`, and the three locale spec constructors.
3. Imports resolve: `import from "rivus:diagnostics/span"` and `import from "rivus:diagnostics/diagnostic"`; `faber check` on the two touched files passes green (en locale). No imports out of the module to lexer/parser/semantic/driver — dependency direction is consumer → diagnostics only.
4. Proba mirrors the radix smoke tests: `span_merge_covers_both` (merge covers both ranges) and a `Diagnostic` builder chain attaching phase/code/lex kind/arg/span with `is_error()` true.
5. No inlining (module split exactly as above), no stubs (every public surface carries real behavior).
6. Dependency handoff documented: migrate-lexer and migrate-syntax consume `rivus:diagnostics/span` as the canonical Span (no duplicate definition).

## Out of scope

- `radix/crates/radix/src/diagnostics/` — catalogs (`catalog.rs`, `catalog_syntax.rs`, `catalog_semantic.rs`), conversions (`convert.rs`), renderers (`render.rs`). Phase catalogs keyed by error enums belong to the driver/semantic goals once those enums land.
- `Spanned` trait (`radix-syntax/src/span.rs`) → migrate-syntax (depends on this goal's canonical Span).
- radix-lexer's `pub use radix_diagnostics::Span` re-export → migrate-lexer.
- `StepperError` and `MirRuntimeAbiFamily::RuntimeDiagnostic` (radix-mir-stepper) → migrate-mir-stepper.
- Reader-locale spec consumers (driver frontmatter) → driver goal.
- `codegen_error`, serde, `DiagnosticPhase::Codegen` behavior, file-interface export identity quirk — dropped per L1/L3 and notes above.
