# Goal: migrate-semantic

**Status**: planned — mapping complete 2026-08-10; implementation pending
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-semantic/
**Primary surface**: `crates/radix/src/semantic/passes/{collect,resolve,definite_assignment,exhaustive,return_path,lint,visibility_annotation,import_path}.rs` + `passes/typecheck/` → `rivus/src/semantic/{collect,resolve,typecheck,definite_assignment,exhaustive,return_path,lint,visibility,import_path}.fab`
**Depends on**: migrate-hir, migrate-types, migrate-syntax
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Port Radix's typed-frontend passes (`collect` → `resolve` → `typecheck` → definite-assignment/exhaustive/return-path/lint/visibility/import-path) to Faber en-surface, mirroring the radix file paths 1:1. These passes are what make MIR **typed**: they must run before `mir/lower.fab` (hard rule 6). The port is the reject-agreement floor — every reject the stepper emits on invalid programs originates here. Drop the GPU/tensor helpers, borrow, air-purity, kernel-namespace, and companion passes per scope lock L2.

## Crate analysis

**Pipeline order** (from `crates/radix/src/semantic/mod.rs::analyze_with_cli` and `passes/mod.rs`; the driver orchestrates — rivus keeps this order in `src/driver/`):

1. `passes/collect.rs` (170) — AST top-level decl scan; one resolver symbol per decl; duplicate names hard errors; enum/union variants registered with parent links; imports bind user-visible name.
2. `passes/import_path.rs` (210) — import specifier form validation (runs early, unconditional): `..`, absolute paths, unknown schemes/`§name` templates, multiple `§` holes.
3. `passes/resolve.rs` (2219) — AST name resolution: lexical availability, scope stack for blocks/loops/matches, `redde`/`rumpe`/`perge` placement, type-alias lowering into `TypeTable` via a fixed-point loop with cycle detection. Only pre-HIR type lowering. Warnings only: the `curata` allocator-name lint.
4. HIR lowering (owned by migrate-hir) — runs only when collect/resolve produced no hard errors.
5. `passes/typecheck/` (9143 prod, excl. tensor/test files) — HIR inference + checking (see ladder below).
6. `passes/definite_assignment.rs` (498) → `passes/exhaustive.rs` (386) → `passes/return_path.rs` (350) → `passes/lint.rs` (438) → `passes/visibility_annotation.rs` (104). Exhaustive and return-path share helpers (`collect_enum_variants`, `discerne_definitely_covered`). Lint and visibility run under `PassConfig.lint`; exhaustive under `PassConfig.exhaustiveness`; the stepper config (`for_target(MirStepper)`) enables all three and disables `borrow_analysis`.

**Core types** (radix → en Faber): `Resolver`/`Scope`/`ScopeId`/`ScopeKind`/`Symbol`/`SymbolKind` (`semantic/scope.rs`, 1980 — shared symbol universe) → classes in `resolve.fab`; `Type`/`TypeId`/`TypeTable`/`FuncSig`/`ParamType`/`InferVar` (`semantic/types.rs`, owned by migrate-types) → `types.fab`; `InitState` (`semantic/init_state.rs`, 49) → union in `definite_assignment.fab`; `SemanticError` (`semantic/error.rs`, 75) → class in `semantic/error.fab` (kind taxonomy is re-exported from radix-diagnostics; rivus keeps it local to avoid a cycle); `TypeChecker` (`typecheck/mod.rs`, 551) → class in `typecheck.fab`; `PassConfig` → class in the driver orchestration. `SemanticResult`/`AnalyzedApplication` orchestration (`semantic/mod.rs`) is driver-owned, not a pass.

**Dependencies**: passes consume `Program`/`StmtKind`/`TypeExpr` (migrate-syntax), `HirProgram`/`DefId`/`HirExpression` + `hir::visit` (migrate-hir), `TypeTable`/`NumericWidth`/`InstansPrecision` (migrate-types — canonical source is `radix-types`, the `semantic/numeric_width.rs`/`instans_precision.rs` files are re-export facades), `Interner`/`Symbol`/`Span` (migrate-syntax/lexer). `typecheck/intrinsics.rs` (636) needs the shared intrinsic registry `crates/radix/src/intrinsics/registry.rs` — see port notes.

**Inference label**: line counts measured 2026-08-10 (production only, excl. `*_test.rs`); rivus placement per the STRUCTURE naming rule; tensor/GPU classification inferred from scope lock L2. `live_tuus_cursor_escape.rs` (308) is unmapped: cursor confinement is part of the `faber:*` cursor lane — dropped by omission, add later only if reject-agreement demands it.

**What to drop** (~2.6k lines): `passes/borrow.rs` (683), `passes/air_purity.rs` (434), `passes/kernel_namespace.rs` (108), `passes/companion_predeclare.rs` (129), `passes/companion_signature.rs` (137), `passes/live_tuus_cursor_escape.rs` (308), and semantic-root tensor helpers `matrix_shape.rs` (53), `tensor_bracket.rs` (48), `broadcast_reduction.rs` (255), `index_infer.rs` (193), `lista_tensor_conversio.rs` (213) plus `typecheck/{tensor_index.rs,tensor_type_directed.rs,index_infer.rs}` (628). Dropping tensor types makes tensor programs reject at type resolution — the intended L2 reject behavior.

## Module mapping (crate → rivus)

| radix file (prod lines) | rivus file | Faber representation |
|---|---|---|
| `passes/collect.rs` (170) | `semantic/collect.fab` | `fn collect(program: Program, resolver: &mut Resolver, types: &mut TypeTable) -> Result<vacuum, lista<SemanticError>>`; symbol-defining helper per decl kind; variant-parent registration |
| `passes/resolve.rs` (2219) + `semantic/scope.rs` (1980) | `semantic/resolve.fab` | classes `Resolver`/`Scope`/`Symbol`; enums `ScopeKind`/`SymbolKind`; `fn resolve(...)`; alias fixed-point + `TypeAliasCycle` error; scope enter/leave walker |
| `passes/import_path.rs` (210) | `semantic/import_path.fab` | `fn validate_import_paths(...)`; path-form checks on `textus` (no char ops) |
| `passes/visibility_annotation.rs` (104) | `semantic/visibility.fab` | `fn check(program, interner)`; protecta rejection + decorative-visibility warnings |
| `passes/typecheck/mod.rs` (551) | `semantic/typecheck.fab` | `class TypeChecker` (fields: resolver, interner, types, errors, scopes, binding_shadow, functions, consts, structs, interfaces, variant tables, current_return/error, substitutions, union_members, error_type); entry `typecheck_with_config` |
| `typecheck/collect.rs` (303) | `typecheck.fab` | contract collection: `function_signature`, `param_types`, `callable_boundary_return`, struct/interface/enum tables |
| `typecheck/expr.rs` (334) | `typecheck.fab` | `check_expr_with_expected` — central dispatcher, expected-type propagation |
| `typecheck/stmt.rs` (253) | `typecheck.fab` | local decls, block scopes, `redde` return typing |
| `typecheck/item.rs` (147) | `typecheck.fab` | item-body checking (params scoped to body; rigid type params) |
| `typecheck/access.rs` (517) | `typecheck.fab` | path/member/index/lvalue; optional-chain strip/re-wrap; non-null chain |
| `typecheck/ops.rs` (837) | `typecheck.fab` | operator/assignment rules; nihil/coalescing; numeric-widening hooks |
| `typecheck/call.rs` (786) | `typecheck.fab` | call/ctor/method typing; arity; failable-call handler enforcement |
| `typecheck/aggregate.rs` (390) | `typecheck.fab` | struct literals, closures, arrays, spread; empty-array infer |
| `typecheck/control.rs` (169) | `typecheck.fab` | `fac`/`cape` ErrorSink, `si` branch joins, `discerne` arm typing |
| `typecheck/pattern.rs` (99) | `typecheck.fab` | pattern-vs-scrutinee checking; alias `name @ Variant(..)` |
| `typecheck/lookup.rs` (632) | `typecheck.fab` | table access helpers, error constructors (`error_issue`), category predicates |
| `typecheck/infer.rs` (604) | `typecheck.fab` | `unify`/`assignable` flow, `occurs_in` + memo, `InferUnion` hole accumulation |
| `typecheck/finalize.rs` (581) | `typecheck.fab` | canonicalize aliases/substitutions, write stable `TypeId`s back to HIR; missing-annotation holes |
| `typecheck/convert.rs` (1491) | `typecheck.fab` | `conversio`/`verte`/deref shape checks (strip `lista_tensor_conversio` call sites) |
| `typecheck/generic.rs` (272) | `typecheck.fab` | call-site instantiation: fresh `Type::Infer` per param symbol; explicit type args |
| `typecheck/intrinsics.rs` (636) | `typecheck.fab` | receiver intrinsic checks via the shared registry |
| `typecheck/intervallum.rs` (158) | `typecheck.fab` | `intervallum<T>` construction + `intra` consumption |
| `typecheck/scalar_text.rs` (169) | `typecheck.fab` | single-scalar `textus`/`ascii` proofs for ordering + interval bounds |
| `typecheck/json_genus.rs` (208) | `typecheck.fab` | `@ json` genus contract validation |
| `passes/definite_assignment.rs` (498) + `init_state.rs` (49) | `semantic/definite_assignment.fab` | `union InitState` (`Unassigned`/`Assigned`/`MaybeAssigned`, `join_paths`); `class DefiniteAssignmentChecker`; use-before-init + write-once (`fixum`) |
| `passes/exhaustive.rs` (386) | `semantic/exhaustive.fab` | `collect_enum_variants`, `discerne_definitely_covered`, `ExhaustiveChecker` |
| `passes/return_path.rs` (350) | `semantic/return_path.fab` | `Terminates`/`FallsThrough` outcome lattice; reuses exhaustive helpers |
| `passes/lint.rs` (438) | `semantic/lint.fab` | `LintContext`: unused decl/import/binding, shadowing (hard error), unnecessary cast/varia, unreachable, explicit ignotum |

**Typecheck ladder as phases** (the 5-rung consensus-C3 ladder; each rung is a horizontal slice across the files above — finish a rung's `faber check` closeout before the next):

- **Phase 1 — rung 1, explicit monomorphic**: entry + collect contracts + expr/stmt/item/access/ops/call(non-generic)/aggregate/control/pattern(simple) + lookup + infer(unify on concrete types) + finalize. Fully-annotated programs typecheck; no inference holes.
- **Phase 2 — rung 2, numeric widening**: scalar `convert.rs`, ops numeric rules, `NumericWidth`/`InstansPrecision` integration, widening in `unify`, `scalar_text.fab`.
- **Phase 3 — rung 3, nullable/union/optional**: `InferUnion` accumulation + solve, `nihil`/coalescing, optional chains, `fac`/`cape` error sink + failable-call handler rules, `json_genus`.
- **Phase 4 — rung 4, generic substitution**: `generic.fab`, type-param collection, call-site instantiation, rigid param bodies.
- **Phase 5 — rung 5, pattern-binding inference (optional)**: `pattern.fab` binding inference — the old #78 cliff; deferred to P3 at most, never a gate.

Analysis passes (`definite_assignment`/`exhaustive`/`return_path`/`lint`/`visibility`/`import_path`) are HIR/AST-phase and independent of the rungs — they can land as parallel units after rung 1 fixes the typed-HIR shape.

## Port notes / frictions

1. **en surface**: identifiers port 1:1 (`Resolver`→`Resolver`, `TypeChecker`→`TypeChecker`, `SymbolKind`→`enum SymbolKind`, `InitState`→`union InitState`). Imports: `import from "rivus:semantic/resolve"` etc.
2. **Arena-ID**: `TypeTable`/`Resolver` are `tabula<id, T>` + `lista<T>` arenas; `TypeId`/`DefId`/`ScopeId` are index `type` aliases (migrate-types owns the table; passes only thread `&mut` references).
3. **tabula for hash maps**: `FxHashMap`→`tabula<K, V>`, `FxHashSet`→`tabula<K, vacuum>` (ordered — determinism is the metric). Shadow-map (`binding_shadow`) stays `tabula<DefId, BindingInfo>`.
4. **No `RefCell`**: radix wraps `substitutions`/`canonical_cache`/`occurs_cache`/`union_members` for `&self` queries; Faber methods freely take `&mut self` — restructure, don't port the cells.
5. **No char/ord**: `resolve.rs`'s `unicode_normalization` combining-mark logic is textus-level; the en surface is ASCII-safe — port as a `textus` helper or drop if the corpus never exercises it (verify against corpus first).
6. **No tuples**: multi-return helpers (e.g. `callable_boundary_return`'s `(ret, err)`) become small result classes.
7. **Intrinsics registry**: `typecheck/intrinsics.rs` needs radix's shared `src/intrinsics/registry.rs` (also used by MIR). Not in STRUCTURE.md's tree — **open decision**: land `src/intrinsics/registry.fab` (mirror rule) or fold into `mir/collection_ops.fab`.
8. **`semantic/error.fab`**: `SemanticError` class + `SemanticErrorKind`/`WarningKind` enums (radix re-exports from radix-diagnostics; rivus keeps them local here to avoid a cross-module cycle). `format_faber_type` (radix `semantic/format_type.rs`, 176) folds into `typecheck.fab` for diagnostics.
9. **No pass-through**: collect and import_path genuinely transform/validate; the driver calls all nine in radix order and stops before MIR when collect/resolve/lowering produced hard errors (warnings non-fatal, `is_error` boundary).

## Done-when

1. `semantic/collect.fab` and `semantic/import_path.fab` + `semantic/resolve.fab` ported: duplicate-definition, unresolved-name, alias-cycle, and import-form diagnostics match radix issues on the reject corpus.
2. `typecheck.fab` rung 1 green: fully-annotated programs check and finalize; unannotated holes diagnose as missing annotations.
3. Rungs 2–4 green per ladder order: numeric widening, nullable/union/optional, and generic substitution each close with their corpus subset — behavior parity (stdout + exit) with radix `run_source`, and reject agreement on the expected-reject set.
4. `definite_assignment.fab`, `exhaustive.fab`, `return_path.fab`, `lint.fab`, `visibility.fab` landed and wired in driver order; use-before-init, non-exhaustive match, missing-return, and lint/shadowing rejects match radix.
5. No tensor/borrow/companion imports remain in `src/semantic/` (grep-clean).
6. One closeout: `faber check` on touched modules (workspace Cargo discipline; no `cargo` runs). No fake stages — every pass transforms its input.
7. Rung 5 (pattern-binding inference) is explicitly deferred, not silent: the goal closes without it.

## Out of scope

- `passes/borrow.rs`, `air_purity.rs`, `kernel_namespace.rs`, `companion_predeclare.rs`, `companion_signature.rs`, `live_tuus_cursor_escape.rs` — dropped (L2 / Faber-surface gating).
- Tensor/GPU lane: `matrix_shape`, `tensor_bracket`, `broadcast_reduction`, `index_infer`, `lista_tensor_conversio`, `typecheck/{tensor_index,tensor_type_directed,index_infer}` — dropped; tensor programs reject.
- `semantic/mod.rs` orchestration, `AnalyzedApplication`/`AnalyzedPackage`/`artifact_plan`, `annotation_contract`, `radix_lane`, `function_facts`, `figura`, file-interface/import resolution (driver-owned), `builtins::register` (migrate-hir) — driver/other goals.
- HIR lowering (`hir/lower/`), MIR lowering, and the stepper — migrate-hir / migrate-mir goals.
- Running `cargo`, `faber build`, or test suites (auditor-owned); this goal's closeout is `faber check` only.
