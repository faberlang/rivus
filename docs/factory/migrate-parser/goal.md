# Goal: migrate-parser

**Status**: planned — mapping complete 2026-08-10; implementation pending
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-parser/
**Primary surface**: `radix/crates/radix-parser/src/{lib,decl,expr,stmt,types,pattern,claim,error}.rs` → `rivus/src/parser/{core,decl,expr,stmt,types,pattern,error}.fab`
**Depends on**: migrate-lexer, migrate-syntax, migrate-diagnostics
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Port `radix-parser` (recursive-descent front end, ~7.2k production lines) to Faber en, producing the AST consumed by collection/resolution/lowering in Rivus. It is a hand-written predictive parser: keyword registry dispatch for statement introducers, an explicit precedence-climbing call chain for expressions, coarsely-synchronizing error recovery at statement/block boundaries, and a post-parse completeness check that every `#` comment token is claimed by a trivia carrier. Lexer errors are terminal (converted to parse diagnostics); parser errors are collected and recovery resumes at the next boundary. The port is a straight en-surface transliteration plus an arena-ID AST construction change; the AST node types themselves (`Stmt`, `Expr`, `TypeExpr`, `Pattern`, `Trivia`) are owned by migrate-syntax and only constructed here.

## Crate analysis

- **Entry** (`lib.rs:126-260`): `parse(LexResult) -> ParseResult`; `parse_with_options`; lexer errors short-circuit to `ParseResult { program: None, errors }` via `lex_errors_to_parse_errors` (kind `ParseErrorKind::LexError`, upstream `PhaseErrorKind::Lex` preserved). `ParseResult { program: Option<Program>, errors: Vec<ParseError>, interner }` with `success()`.
- **Parser core** (`lib.rs:97-215`): `Parser { tokens, pos, non_trivia_indices, non_trivia_cursor, errors, next_node_id, interner, options, reader_keyword_latin_keys, reader_type_latin_keys }`. The non-trivia index (`partition_point`) makes `peek_at` O(log N + offset); `sync_non_trivia_cursor` is re-synced after `restore`. `checkpoint()/restore()` roll back `(pos, next_node_id)` for the compact-closure speculation — the only speculative construct, per `expr.rs:20-25`.
- **Token navigation** (`lib.rs:300-665`): `peek`/`peek_at`/`peek_raw`, `advance`/`advance_raw`, `eat`/`eat_keyword`/`eat_keyword_ident`, `check`/`check_keyword`/`check_keyword_ident`, `expect`/`expect_keyword_ident`, `parse_leading_trivia`/`parse_block_end_trivia`, `is_at_end`. Locale-aware helpers `current_keyword_ident`, `check_keyword_in_owner`, `peek_raw_keyword_is`, `eat_identifier_text`, `identifier_text_or_keyword_latin_key` resolve tokenless keywords through `lookup_keyword_spec`; **slot_helpers.rs does not exist** — those dual-mode helpers live in `lib.rs` and are pinned by `slot_helpers_test.rs`. In en, ident text is already the latin key, so the reader maps are empty and the registry helpers reduce to identity lookups.
- **Recovery** (`lib.rs:670-755`): `is_recovery_boundary` is deliberately coupled to `parse_stmt`'s keyword-led start set plus `{`/`}`/EOF/`@`; `synchronize(stop_at_rbrace)` skips to the next boundary, keeping `}` for the enclosing block. `parse_program` and `parse_block` (`decl.rs:2010-2048`) push errors and synchronize in their loops.
- **Statement dispatch** (`decl.rs:44-145`): `parse_stmt` dispatches on the keyword identity — declarations (`discretio/fixum/varia/functio/genus/implendum/ordo/sit/typus/abstractus/importa`), statements (`proba/probandum/custodi/itera/iace/mori/cede/adfirma/scribe/incipit|incipiet/ex`), control flow (`discerne/dum/elige/fac/si/perge/redde/rumpe`), resource/await (`cura/tacet/ad/figendum|variandum/reddet/tacebit`), annotations (`@`), else expr-stmt. `decl.rs` additionally owns var/await/sit bindings and `BindingPattern` (Ident/Wildcard/Array/Object with `ceteri` rest), func decls (type params, param list with `de/in/ex` mode + `sponte` + `ut` alias + `vel` default, modifiers, posture `fiet/fiunt/fient`, `→ T ⇥ E`, bodyless option), class/abstract/interface, type alias, enum (checked i64 payloads), union variants, imports (sugar + record forms, name inference), `probandum`/`proba`/test modifiers, annotations (`@cli/imperium/optio/operandus/radix typus domain/radix lane/braced`), generic params (`typus`/`magnitudo`), and retired-form diagnostics (`prae typus`, `fixus`).
- **Statements** (`stmt.rs`): `si`/`sin`/`secus` chains with `IfBody` (Block | Ergo) and centralized `try_parse_cape_stmt`, `dum`, `itera` (ex/de/ab), `elige` (value dispatch, `casu`/`ceterum`), `discerne` (pattern dispatch + `omnia`), `custodi`, `fac` (+ `dum` do-while, shared with closure bodies via `allow_while`), `redde/rumpe/perge/cede/tacet`, `iace`/`mori` with same-line `si` guard desugar, `adfirma`, `scribe/vide/mone/nota`, `incipit/incipiet`, `cura` (arena/page), `ex` destructure, expr-stmt + `⊕`/`⊖` incdec.
- **Expressions** (`expr.rs`): precedence chain `parse_expression` → assignment (`←`/`↤` + `⇥` recovery on `↤` only; `parse_assignment_tail` shared with typed initializers) → ternary (`?:` and `sic/secus`) → or → and → equality (`≡/≈/≢/≠/est/non est`, `est`+type-name → `TypeCheck`) → comparison (`≺/≻/≼/≽`, `intra`/`inter`) → bitwise or/xor/and → shift → range (`‥`/`…`/`ante`/`usque`, `per` step) → additive → multiplicative → coalesce (`vel`, tigher than `+`/`*`, interval-constructor RHS) → unary (`-`/`¬`/`non`, `finge`) → postfix loop (call with type args, member, index, `?.`/`?[`/`?(` chains, `!.` etc, `verte`, `↦` conversio) → primary. Primary handles literals, tokenless keyword claims (`iuncta`, `ego`, `praefixum`, `clausura`, `ad`, `scriptum`/`lege`/`lineam`, `verum/falsum/nihil`, `vacua`), typed constructors (`Type { = fields }`, qualified paths with rollback), JSON literals (bare `{}` — dedicated parser, not `parse_object_fields`), `iuncta<A,B> [..]`, closures (`clausura` and compact `∴` form with speculative rollback), `finge` variants, retired-prefix-predicate rejections (line-boundary aware).
- **Types** (`types.rs`): `TypeExpr { nullable, mode, kind, span }`, `TypeExprKind` (Named/Qualified/Infer/Union/UnionHole/Func/Atomic/PromissumFailable/Tensor/Vector/Matrix/Sparsa), `TypeMode` (De/In), `FiguraExpr` (Infer/Literal/Var/Tuple). Union flattening preserves member-level mode/nullable. Width sugar (`lista<t4>` etc.) delegates to `radix_types::numeric_width` marker fns — **extra dependency on migrate-types**. Postfix `[]` is rejected (`postfix_array_syntax`).
- **Patterns** (`pattern.rs`): deliberately narrow — Wildcard/Literal/Ident/Path, `PatternBind` (Alias `ut` | Bindings `fixum/varia` + ExField list), separators `,`/`et`, stopped by `{`/`ergo`.
- **Errors** (`error.rs`): `ParseError { kind, args: Vec<DiagnosticArg>, span, error_kind: Option<PhaseErrorKind> }`; `ParseErrorKind` is **re-exported from radix-diagnostics**, so the enum lands in migrate-diagnostics and `error.fab` only carries the struct + `with_arg`.
- **Comments** (`claim.rs`): post-parse diff of every `LineComment` token start vs spans claimed by trivia carriers across the whole `Program` (`claim::check`), emitting `ParseErrorKind::InvalidCommentPlacement` (`PARSE060.invalid_comment_placement`) per unclaimed comment. In Rivus this walks the arena `Program` — pairs with the migrate-syntax visitor.

## Module mapping (crate → rivus)

| radix-parser file | rivus file | Faber representation (en) |
|---|---|---|
| `lib.rs` — Parser core, token nav, recovery, entry, keyword-slot helpers | `parser/core.fab` (new) | `class Parser` (arena build state + nav/recovery methods); `class ParseResult`; `class ParseOptions`; `fn parse(...)` free fns. `Checkpoint` class replaces the `(pos, NodeId)` tuple; `Resultus`-style outcome class replaces `Result<_, ParseError>`. |
| `error.rs` | `parser/error.fab` | `class ParseError { kind, args: lista<DiagnosticArg>, span, error_kind }`; `enum ParseErrorKind` imported from `rivus:diagnostics`. |
| `decl.rs` — dispatch + declarations + blocks | `parser/decl.fab` | Free grammar fns taking `&mut Parser`, returning arena ids / outcome class. `parse_stmt` dispatcher and `parse_block`. |
| `stmt.rs` | `parser/stmt.fab` | Same shape; `parse_si_stmt`, loops, `fac`, transfer, scribe, `incipit`, `cape` tail, expr-stmt/incdec. |
| `expr.rs` | `parser/expr.fab` | Precedence chain fns; each constructs arena `Expr` via an append helper (mirrors `binary_expr`/`postfix_expr`, but pushes into the arena instead of boxing). |
| `types.rs` | `parser/types.fab` | `TypeExpr` construction; union flattening; width-sugar markers imported from `rivus:types/numeric_width` (migrate-types). |
| `pattern.rs` | `parser/pattern.fab` | `parse_patterns`, `parse_pattern`, `parse_pattern_bind`, literal patterns. |
| `claim.rs` | `parser/core.fab` (post-parse check) | `fn check_claimed_comments(parser, program)` walking arena `Program` + `Program.end_trivia`; needs the migrate-syntax visitor or explicit arena walk. |
| `mod_test.rs`, `*_test.rs`, `test_support.rs` | dropped | Behavior pinned by the parity harness, not ported radix unit tests. |

## Port notes / frictions

- **No `impl Parser` blocks across files.** radix extends `Parser` with `impl` blocks per file; Faber has no extension impls. Decision: `core.fab` owns the `Parser` class; every grammar file defines free `fn parse_* (state: &mut Parser) -> ...` and imports `rivus:parser/core`. This keeps the module split (hard rule 1) and adds one file to STRUCTURE.md.
- **Arena-ID construction.** radix builds boxed trees (`Expr { id, kind, span }` with `Box` children). Rivus nodes are indices into `lista<T>` arenas (L4); `next_id()` becomes "append node, return index". Parse fns return ids or nullable id slots; recursion stays via ids, not boxes. `Stmt`/`Expr`/`TypeExpr`/`Pattern` type shapes are migrate-syntax's job; the parser's `binary_expr`/`postfix_expr`/`with_id` helpers become arena-append helpers.
- **Tuples are available** (`tuple<T1,T2>[a, b]`, EBNF `iunctaExpr`): `checkpoint() -> (pos, NodeId)` may port as a tuple, or as `class Checkpoint { pos: numerus, next_node_id: numerus }` where the name aids readability (verify tuple codegen in the en build on first use). `Result<T, ParseError>` return/error flow: parser already accumulates `errors` on `self`; port as an en outcome genus (`Resultus`-style) in `error.fab` carrying value-or-nihil plus having pushed any `ParseError` onto `parser.errors`, preserving the `?`→recovery structure.
- **No char/ord.** `is_capitalized_type_name` and `valid_inferred_import_name` (`expr.rs:1160`, `decl.rs:1295`) use `textus.sectio`/`continet` on interned spellings instead of `char::is_uppercase`/`is_alphabetic`.
- **En-surface simplification.** `reader_keyword_latin_keys`/`reader_type_latin_keys` are empty in en (ident text is the latin key). Drop the two `tabula` fields and the `canonical_type_ident`/`eat_identifier_text` locale branches; keep `current_keyword_ident`/`check_keyword_ident`/`check_keyword_in_owner` against the en keyword table (they are the dispatch and recovery-boundary backbone). `keyword_allowed_as_ident`/`lookup_keyword_spec`/`keyword_spelling_for_token` come from migrate-lexer.
- **GPU-lane shapes are not dropped at parse level.** `tensor/vector/matrix/sparsa` type sugar, `@ radix lane`, and `@ radix typus domain` are parsed and rejected later (reject-agreement floor, L2/L5), so `types.fab` keeps the width-sugar grammar. It needs the `numeric_width` marker helpers — either from `rivus:types/numeric_width` (migrate-types) or a parser-local port; decision left open until migrate-types lands.
- **JSON literal parser** (`expr.rs:1890-2043`) is a separate grammar from object fields — port as-is; wire `true/false/null` idents stay distinct from `verum/falsum/nihil`.
- **Recovery coupling:** `is_recovery_boundary` must stay aligned with the ported `parse_stmt` keyword set; the comment-claim check must keep claiming end-trivia on `Program` and every `BlockStmt` or it regresses `PARSE060`.
- **`peek_at` linear-vs-index equivalence test** (`lib.rs:895-944`) is test-only; the index/`partition_point` optimization itself is portable with `tabula`/binary search on `non_trivia_indices: lista<numerus>`.

## Done-when

1. `rivus/src/parser/` contains `core.fab`, `decl.fab`, `expr.fab`, `stmt.fab`, `types.fab`, `pattern.fab`, `error.fab`; all imports use `rivus:parser/<file>`; `STRUCTURE.md` updated with `core.fab`.
2. `faber build .` is green on the en package (P0 gate); `rivus run` on a parsed-only harness accepts the same inputs radix `parse_with_options` accepts and rejects with the same `ParseErrorKind` on a 50-file fixture mix (valid corpus + deliberately broken samples per statement class).
3. Recovery: each malformed fixture produces ≥1 `ParseError` and parsing resumes at the next statement boundary (fixture asserts following-statement span survives).
4. Comment completeness: every `#` in a comment-heavy fixture is claimed or reported as `InvalidCommentPlacement` (PARSE060 identity), matching radix's `claim::check`.
5. Precedence pin: 10 parenthesization fixtures (`a ← b ← c`, `a ↤ b ⇥ c`, `vel` vs `+`, postfix chains, ternary associativity) produce the same `ExprKind` nesting as radix.
6. Closeout: exactly one `./scripta/test --check` equivalent run (or the faber-side narrow ladder) after the last edit, then stop.

## Out of scope

- AST node type definitions (`Stmt`, `Expr`, `TypeExpr`, `Pattern`, `Trivia`, `Program`) — migrate-syntax.
- `ParseErrorKind`, `PhaseErrorKind`, `DiagnosticArg` — migrate-diagnostics.
- Token/`TokenKind`/`Symbol`/`Interner`/keyword registry — migrate-lexer.
- `numeric_width` width-sugar markers — migrate-types (or explicit parser-local decision).
- Porting `mod_test.rs`/`*_test.rs`/`test_support.rs` (~7k lines of Rust tests) — behavior is pinned by the parity harness instead.
- Any locale/reader-pack surface beyond en; emitted formatting; MIR/HIR downstream phases.
