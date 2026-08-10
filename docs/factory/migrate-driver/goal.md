# Goal: migrate-driver

**Status**: planned — mapping complete 2026-08-10; implementation pending
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-driver/
**Primary surface**: `radix/crates/radix/src/driver/{session.rs, source.rs, frontmatter.rs, file_frontmatter.rs, mod.rs}` → `rivus/src/driver/`; run orchestration `radix/crates/radix/src/mir/stepper/mod.rs` + `mir/mod.rs` (stepper re-export) → `rivus/src/mir/stepper/mod.fab` + `src/mir/mod.fab`; CLI note in `src/main.fab`
**Depends on**: all migrate-* goals (orchestration) — needs lexer, parser, hir, semantic, mir, mir-stepper, diagnostics
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Port the thin driver surface of the main radix crate to Rivus: session/config state, source identity + line indexing, `+++` frontmatter peeling, and the `analyze_source` front-half orchestration (parse → semantic → `AnalyzedUnit`). Include the driver-facing run glue `mir/stepper/mod.rs` (`run_source` / `run_analyzed` / `RunSourceError`) and the `mir/mod.rs` re-export barrel so `rivus run <file.fab>` executes analyze → lower → step. Drop everything codegen-bound: `compile*`, target policy, forma, shader/graphics, AIR/GPU lanes, and the `cli` analysis the stepper path never uses.

## Crate analysis

- **`driver/session.rs` (~280 ln):** `Config` (caller compilation policy) + `Session` (per-compilation carrier) + `WarnPolicy`. Most `Config` fields are codegen-bound (`target`, `emit_comments`, `output_mode`, `module_name`, `no_fuse`, `import_path_templates`, `stdlib_path`) — dropped. `WarnPolicy`/`apply_warn_policy` is applied only at the codegen boundary (`mod.rs:66-76`, `145-158`); the run path never applies it — dropped. Keep: `Config` trimmed to analysis-relevant state + `Session::new`. `discover_dev_stdlib` (pack discovery) dropped — Latin/en lexer default only.
- **`driver/source.rs` (~230 ln):** `SourceFile` (path, name, peeled `content`, `raw_content`, `frontmatter`, `body_byte_offset`, private `line_starts`), `peel_raw_source` → `PeeledSource`, `SourceLoadError` (Frontmatter | Toml), `source_load_diagnostic` (PARSE052). Keep whole — diagnostics need byte→line/col (binary search over `line_starts`, one-based body-relative). `'a` borrows in `PeeledSource` do not port; make `body` owned (source-text firewall: bytes consumed once at lex).
- **`driver/frontmatter.rs` (~110 ln):** `split_frontmatter`, `SplitFrontmatter` (frontmatter_text, body, body_byte_offset), `FrontmatterError` (ByteOrderMark, OpeningDelimiterNotFirstLine, Unterminated). Keep whole, verbatim semantics.
- **`driver/file_frontmatter.rs` (~160 ln):** `FileFrontmatter` — a TOML-table wrapper with accessors (`group`, `sectio`, `locale`/`locale_result`, `build_target`, `build_kind`, `package_*`, `paths_*`, `dependentia_via`, `probanda_*`) and `parse_file_frontmatter` (empty/whitespace → empty table). Keep; folds into `frontmatter.fab`.
- **`driver/mod.rs` (~2900 ln):** keep the front half only: `analyze_source` + the `*_with_cli_program*` / `*_with_namespace_exports*` / `*_with_import_contract*` ladder (thinning to what Rivus needs), `parse_frontend_source` (peel → resolve lex pack → lex → parse → `ParsedSourceBody`), `analyze_semantic_program` (`PassConfig` + `semantic::analyze_with_cli`), `analyzed_unit_from_semantic` (→ `AnalyzedUnit`), `AnalyzedUnit` (+`application_view`), `import_contract_source_path`, `has_diagnostic_errors`. **Drop:** `compile*` family, `generate_output` + all `generate_<backend>_output`, `compile_cli_program`/`cli_codegen_diagnostic`, `SpannedCodegenError` + all `*_to_spanned`, `prepare_mir_lowering_interner` (emitter-only; the lower pipeline interning moves to `mir/lower.fab`), `prepare_air_backward_bundle`/`apply_air_backward_bundle`/`AirBackwardBundle`/`AirBackwardSnapshot` (AIR reverse-AD lane), `collect_graphics_source_facts` + `GraphicsSourceFacts`/`GraphicsVertexEntry`/`GraphicsFragmentEntry`/`GraphicsVertexBodyShape` (shader contract), `collect_lane_policy_diagnostics` + `collect_radix_lane_target_errors` (GPU lane policy; L2 reject behavior lives in the stepper classifier, not the driver), `collect_target_policy_diagnostics` (stepper target has no syntax-level policy — `mod.rs:2477` falls through empty), `analyze_cli_program_for_source` + `retarget_cli_program_symbols` (drops the `crate::cli` dependency; the stepper's `run_entry` picks the entry via `MirNames::explicit_entry_function`), `cuda_kernel_descriptor_json`, `partition_record_json`, `resolve_frontmatter_locale_pack`/`resolve_peeled_lex_pack` (reduce to Latin default; keep `FileFrontmatter::locale()` for inspection).
- **`mir/stepper/mod.rs` (~90 ln):** the run orchestration — `RunSourceError` (Frontend | Mir | Stepper), `run_source` (analyze → `run_analyzed`; lowering failures absorbed into `StepperError`), `run_analyzed` (`lower_analyzed_unit_with_context` → `run_entry` on `.validated`). Re-exports from leaf `radix-mir-stepper` become imports from `rivus:mir-stepper/*`.
- **`mir/mod.rs` (barrel, lines 164-167):** re-exports the stepper run surface at crate level; Rivus equivalent is an import barrel in `src/mir/mod.fab` re-exporting `rivus:mir/stepper/mod`.

## Module mapping (crate → rivus)

| radix file | rivus file | Faber representation / en identifiers |
|---|---|---|
| `driver/session.rs` | `src/driver/session.fab` | `class Session`, `class Config` (+builder-style methods retained as `fn with_*` where kept); no `WarnPolicy` |
| `driver/source.rs` | `src/driver/source.fab` | `class SourceFile` (`fn new`, `fn inline`, `fn from_raw`, `fn offset_to_line_col`, `fn line_content`), `class PeeledSource` (owned body), `union SourceLoadError` (Frontmatter \| Toml), `fn peel_raw_source`, `fn source_load_diagnostic` (PARSE052) |
| `driver/frontmatter.rs` | `src/driver/frontmatter.fab` | `class SplitFrontmatter`, `union FrontmatterError`, `fn split_frontmatter` |
| `driver/file_frontmatter.rs` | `src/driver/frontmatter.fab` (folded) | `class FileFrontmatter` (+accessors incl. `fn locale`), `fn parse_file_frontmatter` |
| `driver/mod.rs` (front half) | `src/driver/mod.fab` | `fn analyze_source`, `class AnalyzedUnit` (interner, types, resolver, hir, presentation, libraries, function_facts, resolved_uses, qualified_identities, diagnostics; drop cli_program, gpu_builtins, graphics_source, radix_lanes, annotation_contracts per subset), internal `fn parse_frontend_source`, `fn analyze_semantic_program`, `fn analyzed_unit_from_semantic` |
| `mir/stepper/mod.rs` | `src/mir/stepper/mod.fab` | `union RunSourceError` (Frontend \| Mir \| Stepper), `fn run_source`, `fn run_analyzed`; `import from "rivus:mir-stepper/..."` for Host/StdioHost/Value/StepperError/run_entry/classify/status |
| `mir/mod.rs` (stepper re-export) | `src/mir/mod.fab` | barrel re-exporting the stepper run surface + `lower_analyzed_unit_with_context` from `mir/lower.fab` |
| `src/main.fab` (note) | `src/main.fab` | `main { }` reads args via kernel `args`, parses `run <file.fab>`, reads file via kernel `solum`, builds `Session`/`Config`, calls `run_source(name, source, host)`; prints stdout + exit code |

Import spellings: `import from "rivus:driver/session" private session`, `rivus:driver/source`, `rivus:driver/frontmatter`, `rivus:driver/mod` (orchestration), `rivus:mir/stepper/mod`. Entry-point selection is MIR-side (`MirNames::explicit_entry_function`) — no driver CLI analysis.

## Port notes / frictions

1. **TOML frontmatter:** radix uses the `toml` crate. Rivus has no external crates — route through the host TOML kernel (`rivus:mir-stepper/kernel/toml`, HostDelegate) or a minimal TOML-subset parser; a full internal parser is out of scope. `parse_file_frontmatter`'s empty-text → empty-table behavior must hold.
2. **Borrows/lifetimes:** `PeeledSource<'a>`/`SplitFrontmatter<'a>` use borrowed slices. Fab (en) has no lifetimes; port as owned `String` body + `body_byte_offset`. Arena-ID model already avoids recursive structs; no new arenas needed here.
3. **Line index:** `partition_point` binary search ports as an iterative scan or the en stdlib search; `line_starts[0] == 0` and `\r\n` handling are behavior fixtures.
4. **`u32` spans/offsets:** `offset_to_line_col`, `line_content`, and 4 GB caps port directly; keep the `u32` types for parity.
5. **Generic `<H: Host>`:** en Fab has no traits; the `Host` in `run_source`/`run_analyzed` becomes a class with method-table dispatch (mirroring `radix-mir-stepper`'s Host surface, `StdioHost` default).
6. **Error surface:** `RunSourceError` absorbs lowering into `StepperError` (`stepper/mod.rs:44-57`) — keep the same absorption so `run_source` has exactly two failure arms.
7. **`locale` field:** `FileFrontmatter::locale()` ports as an accessor; the driver never resolves packs (no stdlib discovery). Declared-but-unknown locales degrade to Latin silently — flag if reject-agreement demands otherwise.
8. **CLI parsing:** `rivus run <file.fab>` — positional args only; no flags (no `--target`, no `--format`). Wrong arity → usage text + nonzero exit.

## Done-when

1. `src/driver/{session,source,frontmatter,mod}.fab` land with en identifiers and `rivus:` imports; `faber check` on the touched modules passes.
2. `SourceFile` peeling matches radix on fixtures: no frontmatter (offset 0), valid `+++` block (offset after closing delimiter), BOM → error, delimiter not on line 1 → error, unterminated → error, empty body valid.
3. `offset_to_line_col` / `line_content` match radix on multi-line + `\r\n` fixtures (one-based, body-relative, LF-trimmed).
4. `frontmatter.fab` exposes `SplitFrontmatter`, `FrontmatterError`, `FileFrontmatter` (incl. `locale()`), `split_frontmatter`, `parse_file_frontmatter` with empty-text behavior.
5. `src/mir/stepper/mod.fab` implements `run_source`/`run_analyzed`/`RunSourceError` over the mirrored lower + stepper; `src/mir/mod.fab` re-exports them.
6. `src/main.fab`: `rivus run <file.fab>` reads the file, builds `Session`, calls `run_source`, prints stdout + exit code; `rivus run` with no/extra args prints usage + nonzero exit.
7. **End-to-end (P1 slice):** `rivus run` on a trivial program (`const`/`print`/arithmetic) produces stdout + exit code identical to radix `run_source` (stepper, StdioHost).
8. Reject agreement: frontmatter failures surface the PARSE052-equivalent diagnostic; first unsupported construct yields `StepperDispatchStatus` rejection matching radix — no pass-through stubs.

## Out of scope

`driver/cli.rs` surface (CLI-program analysis/codegen), `target_policy`, forma, `shader_contract`, templating, `artifact_plan`, conversio-related driver modules, `amd_target`, all MIR codegen backends (`generate_*_output`), AIR/GPU lane (backward bundle, graphics facts, lane policy, cuda/partition JSON), `WarnPolicy` boundary promotion, locale-pack loading / stdlib discovery, `compile*` emit entrypoints. Any radix driver surface not named in the mapping table above is dropped by default.
