# Goal: migrate-syntax

**Status**: planned — mapping complete 2026-08-10; implementation pending
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-syntax/
**Primary surface**: `radix/crates/radix-syntax/src/{ast,span,trivia,visit,annotation_promotion,braced_annotation}.rs` → `rivus/src/syntax/{ast,span,trivia,visit}.fab`
**Depends on**: migrate-diagnostics
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Port the radix-syntax crate to the Rivus `src/syntax/` module tree: the
source-shaped AST (`Program`/`Stmt`/`Expr`/`TypeExpr`/`Annotation` + payload
classes), the `Spanned` provenance trait, trivia attachment, and the visitor
walkers. The port is pure parse-shape — no inference, no resolution, no
targets. The only shape change is the campaign L4 arena-ID data model: every
recursive `Box`/`Vec` child reference becomes an integer index into arena
lists owned by the parse result. Annotation *view* helpers
(annotation_promotion.rs + braced_annotation.rs) fold into `ast.fab` as
pure functions over the AST + interner; codegen-facing and CLI-only helpers
are dropped.

## Crate analysis

Architecture: `radix-syntax` is the parser's output contract — a faithful,
untyped record of the source grammar consumed by every later phase. It is a
foundation crate (`radix::syntax`), depends only on `radix-lexer` (for
`Span`, `Symbol`, `Token`) and `serde`, and owns no inference: `NodeId`
(u32) keys phase side tables, spans are diagnostic provenance, `TypeExpr`
is written type syntax (not a checked type), and annotations are preserved
as metadata whose meaning is decided by lowering. File roles:

- `ast.rs` (~1.6k lines): `Program`, `Stmt`/`StmtKind` (30 variants),
  `Expr`/`ExprKind` (27 variants), all declaration/statement/expression
  payload classes, `TypeExpr`/`TypeExprKind`/`TypeMode`/`FiguraExpr`/
  `FuncTypeExpr`, `Ident`, `Literal`, `Annotation`/`AnnotationKind` +
  braced forms, value enums (`BinOp`, `UnOp`, `AssignOp`, `TernaryStyle`,
  `RangeKind`, `CallablePosture`, `AstMutability`, …).
- `span.rs`: `Spanned` trait only — the concrete `Span` is **not** owned
  here (see Span ownership below).
- `trivia.rs`: `Trivia` union (`CommentLine`/`Newline`), payload is an
  interned `Symbol`.
- `visit.rs`: `Visitor` trait (default methods + `walk_*` continuations),
  `MAX_VISIT_DEPTH = 1024` fail-closed depth guard via a thread-local cell.
- `annotation_promotion.rs` + `braced_annotation.rs` (~800 lines): view
  classes (`NondumView`, `RadixLaneView`, `RadixTypusView`, `VerteView`,
  `BracedOptioView`, `BracedOperandusView`, `AnnotationView`, `BracedView`)
  and extraction functions for `@` annotation families — pure functions
  over AST + `Interner`.

**Span ownership (resolved):** the canonical `Span` is
`radix-diagnostics/src/span.rs` (half-open `u32` byte range;
`new`/`merge`/`len`/`is_empty`). `radix-lexer` re-exports it
(`radix-lexer/src/token.rs:27`), and `radix-syntax` uses `radix_lexer::Span`
everywhere without defining its own. Therefore in Rivus **one** `Span`
class lives in `src/diagnostics/span.fab` (migrate-diagnostics), and
`src/syntax/span.fab` mirrors radix-syntax/span.rs = the `Spanned` trait.
Syntax imports `import from "rivus:diagnostics/span"`. Nothing in
`src/syntax/` defines a second Span.

**Label:** the syntax crate is inference-free — labels (types, defs, flow,
rejects) attach later via `NodeId` keys. Do not let semantic concerns leak
in; the goal is the parser contract only.

## Module mapping (crate → rivus)

| Radix file | Rivus file | Faber representation (en) |
|---|---|---|
| `ast.rs` | `src/syntax/ast.fab` | classes + unions, arena-ID children, value enums; folds in annotation view helpers |
| `span.rs` | `src/syntax/span.fab` | `interface Spanned { fn span() -> Span }` for Program/Stmt/Expr/Ident/TypeExpr/Annotation |
| `trivia.rs` | `src/syntax/trivia.fab` | `union Trivia` + `class CommentLine` + `class Newline` (Symbol payload, Span) |
| `visit.rs` | `src/syntax/visit.fab` | `interface Visitor` + free `walk_*` functions + `MAX_VISIT_DEPTH` |
| `annotation_promotion.rs` | → `ast.fab` (folded) | view classes + extraction functions |
| `braced_annotation.rs` | → `ast.fab` (folded) | view classes + `braced_*` extraction functions |
| `span_test.rs`, visit tests | `proba` cases in `visit.fab`/`ast.fab` | depth-limit fail-closed tests |

`ast.fab` imports: `rivus:diagnostics/span` (Span), `rivus:lexer/interner`
(Symbol), `rivus:lexer/token` (Token — `AnnotationStmt.args`), `rivus:syntax/trivia` (Trivia).

## Port notes / frictions

1. **Arena-ID (L4).** Every recursive reference becomes an index: `Box<T>` /
   `Option<Box<T>>` → `NodeId`, `Vec<Stmt>`/`Vec<Expr>`/`Vec<TypeExpr>` →
   `lista<NodeId>`. Only Stmt/Expr carry an explicit `id: NodeId` (as in
   radix); TypeExpr/JsonValue indices are implicit by arena position. The
   arenas live on the parse result: `Program` gains `stmt_arena`/
   `expr_arena`/`type_arena` (`lista<T>` each, plus `json_arena` for the
   recursive `JsonValue` JSON-literal tree); `Program.statements` and
   `BlockStmt.statements` become `lista<NodeId>`. Non-recursive value
   structs (Ident, TypeParam, Param, Argument, Annotation, ExField,
   AnnotationField, FingeFieldInit, ClausuraParam, ObjectField, trivia)
   stay plain classes, inline or in `lista<V>`.
2. **u64 magnitudes.** `Literal::Int(u64)` carries a non-negative magnitude
   (negation is `UnaryExpr::Neg`). Faber has no `u64`/`char`/`ord`; port as
   the integer type with the by-construction invariant that magnitudes are
   non-negative — the parser never emits a negative `Int`.
3. **serde derives dropped.** `JsonValue`/`JsonMember`/`CallablePosture`/
   `Visibility`/`GenericParamKind` lose `Serialize`/`Deserialize`; they
   become native Faber classes/unions.
4. **Visitor depth state.** `thread_local! VISIT_DEPTH` has no en-surface
   equivalent. Port the depth as a `depth: numerus` field carried by a
   small state holder threaded through `walk_*` (or by the Visitor class
   itself if the en surface allows stateful polymorphic classes). Keep
   `MAX_VISIT_DEPTH = 1024`; `on_depth_exceeded` default must **fail
   closed** — raise an interpreter diagnostic rather than panic.
5. **Dropped.** `verte_rust_methodus` (Rust codegen boundary, L1); the
   `*_strict` views and `braced_optio_is_flag` are consumed only by
   `radix/src/cli.rs` CLI-surface collection — defer to the driver/args
   kernel goal (rivus driver is thin); `clippy` allows and `#[cfg(test)]`
   scaffolding are not ported.
6. **Open decision.** STRUCTURE.md plans 4 files under `syntax/`; the two
   annotation-view modules (~800 lines) fold into `ast.fab` to honor that
   tree. If `ast.fab` outgrows ~900 lines, split
   `syntax/annotation_view.fab` and update STRUCTURE.md in the same unit.

## Done-when

1. `src/syntax/ast.fab` passes `faber check` on the en package and defines:
   `Program` as the arena carrier, `Stmt`/`StmtKind` (all 30 variants),
   `Expr`/`ExprKind` (all 27 variants), `TypeExpr`/`TypeExprKind`/
   `TypeMode`/`FiguraExpr`/`FuncTypeExpr`, `Ident`, `Literal`, `Annotation`/
   `AnnotationKind` + braced forms, value enums, and NodeId child
   references per port note 1. No `Box`-style recursive fields remain.
2. `src/syntax/span.fab` defines `Spanned` (span() for Program/Stmt/Expr/
   Ident/TypeExpr/Annotation) importing Span from
   `rivus:diagnostics/span`; grep across `src/syntax/` finds **one** Span
   class definition, in `diagnostics/span.fab`.
3. `src/syntax/trivia.fab` defines `Trivia` + `CommentLine` + `Newline`
   with interned `Symbol` payloads.
4. `src/syntax/visit.fab` defines `Visitor` and `walk_program`/`walk_stmt`/
   `walk_expr`/`walk_type_expr`/`walk_block`/`walk_if_body`/
   `walk_secus_clause`/`walk_binding_pattern`/`walk_pattern`/
   `walk_pattern_bind`/`walk_figura_expr`, with a match arm for **every**
   StmtKind/ExprKind/TypeExprKind variant (a new variant fails to compile),
   and `MAX_VISIT_DEPTH` fail-closed depth limiting.
5. Annotation view helpers from annotation_promotion.rs + braced_annotation.rs
   land per port note 6; `verte_rust_methodus` and the `*_strict`/CLI-only
   views are absent or deferred with a comment.
6. Depth-limit behavior is ported as `proba` cases: walk at `MAX_VISIT_DEPTH`
   succeeds, `MAX_VISIT_DEPTH + 1` fails closed without recursion blowup.
7. Closeout: one `./scripta/test --check` run (stages 1–3) green; package
   stays in the planned module tree — no inlining, no new top-level
   monolith.

## Out of scope

- Parser (radix-parser) and its error recovery — separate goal.
- Semantics: resolution, typecheck ladder, analysis passes — later goals.
- Diagnostics catalogs/rendering (migrate-diagnostics owns Span + Diagnostic).
- CLI/package annotation surfaces (strict views, `braced_optio_is_flag`).
- Codegen-boundary helpers (`verte_rust_methodus`) and all backend-specific
  annotation interpretation.
- `radix-hir`, `radix-types`, MIR, stepper, and lexer/parser interplay
  beyond the declared imports.
