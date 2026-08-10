# Goal: migrate-hir

**Status**: planned — mapping complete 2026-08-10; implementation pending
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-hir/
**Primary surface**: `crates/radix-hir/src/{nodes.rs, visit.rs, lib.rs}` + `crates/radix/src/hir/lower/{mod,decl,expr,stmt,types,pattern,literal,annotation}.rs` → `rivus/src/hir/{nodes.fab, visit.fab}` + `rivus/src/hir/lower/{mod,decl,expr,stmt,types,pattern,literal}.fab`
**Depends on**: migrate-syntax, migrate-types, migrate-semantic
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Port Radix's HIR layer — the `radix-hir` foundation crate (node types + visitors) and the main-crate AST→HIR lowering (`radix::hir::lower`) — into Rivus as `src/hir/` in locale=en Faber. HIR is the bridge between the resolved AST and the typed MIR: the stepper's input is `mir::lower_analyzed_unit_with_context(AnalyzedUnit)`, so HIR itself and its lowering are the contract this unit must reproduce. The MIR *consumers* of HIR (emit/codegen crates, `hir-fhir` artifact serialization) are out of scope; Rivus needs HIR only to feed its own MIR lowering (migrate-mir) and stepper. All Radix identifiers port 1:1 (en surface, per AGENTS.md hard rule 7).

## Crate analysis

**`radix-hir`** is the foundation crate (re-exported by package `radix` as `radix::hir` for path stability). `lib.rs` is a barrel. `nodes.rs` (~1550 lines) owns all HIR type definitions; `visit.rs` (~1300 lines) owns the read-only `HirVisitor` + `walk_*` fns and a mutable mirror `HirVisitorMut` (explicit `match` walks, no macro). Deps: `radix-lexer` (Span, Symbol, Token), `radix-syntax` (CallablePosture, GenericParamKind, Visibility, JsonValue), `radix-types` (DefId, TypeId), `rustc-hash`, `serde`/`postcard` (serialization — dropped).

Node hierarchy: `HirProgram { items: Vec<HirItem>, entry: Option<HirBlock>, entry_annotations, entry_args_name, entry_is_async }`; `HirItem { id: HirId, def_id: DefId, kind: HirItemKind, span }`; `HirItemKind` = Function(Box<HirFunction>)/Struct/Enum/Interface/TypeAlias/Constant/Import. Expressions and statements are thin envelopes: `HirExpression { id: HirId, kind: HirExpressionKind, ty: Option<TypeId>, span }` (~45 kinds: Path, Literal, Vacua, Binary, Unary, Call, MethodCall, Field, Index, OptionalChain, NonNull, Block, Si, Discerne, Loop, Dum, Itera, Intervallum, Assign, ConversioAssign, Array, Struct, Tuple, Scribe, Scriptum, ReadLine, Praefixum, Adfirma, Panic, Throw, Handled, Clausura, Cede/Reddet/Tacebit/Yield, Verte, Conversio, Ad, Ref, Deref, TypeCheck, Error), `HirStatement { id, kind, span }` (9 kinds: Local, Expr, Redde, Rumpe, Perge, Tacet, IncDec, Custodi). `HirBlock { statements, expr, span, breakable: Option<HirBreakableKind>, end: Option<HirBlockEnd> }`. `HirCape` (catch binding + body), `HirCasuArm`, `HirPattern` (Wildcard/Binding/Alias/Variant/Literal). Leaf taxonomies: `HirLiteral` (Int u64, Float, String, Ascii, Octeti, JsonValor, Regex, Bool, Nil), `HirBinOp` (25), `HirUnOp`, `HirScribeKind`, `HirIteraMode`, `HirRangeKind`, `HirRefKind`, `HirIncDecOp`. Identity: `HirId(u32)`; `DefId` lives in radix-types (lowering allocates synthetic ones starting at 1_000_000); a separate `HirSourceAnchorId(u32)` namespace feeds the presentation sidecar.

Library provenance types (`LibraryProvider`, `LibraryIdentity`, `LibraryItemKind`, `LibraryBinding`, `LibraryItem`, `LibraryRegistry` — FxHashMap-keyed side tables) are declared in radix-hir but *populated* by collect/resolve; Rivus carries the types in nodes.fab, construction belongs to migrate-semantic.

**Lowering** (`radix/src/hir/lower/`, ~5.1k production lines) is a stateful coordinator `Lowerer` (`mod.rs`) holding `&mut Resolver`, `&mut TypeTable`, `&mut Interner`, per-run id counters (`next_id`, `next_def_id`, `next_anchor_id`), recoverable `errors: Vec<LowerError>`, `local_scopes` + `type_param_scopes` stacks, `current_ego_struct`, kernel-glob import maps, optional `CliProgram`/`LocalePack`/annotation-contract metadata. Entry fns `lower` / `lower_with_cli` / `lower_with_cli_and_locale_pack` / `lower_with_cli_locale_pack_and_contracts` return `(LoweredProgram, Vec<LowerError>)`. `lower_program` partitions top-level items vs the implicit/explicit `incipit` entry block and calls `crate::builtins::inject_hir_items`. Sibling modules: `decl.rs` (varia/functio/gens/ordo/discretio/implendum/typus/importa items + proba test items → zero-arg functions with `HirTestMetadata`; shader-local injection), `expr.rs` (all ExprKind forms incl. forma capture, scriptum, finge, iuncta, closure, conversion, ego), `stmt.rs` (statement forms, destructure/ex fan-out via `lower_stmt_expanded`, fac→breakable, elige→single-subject Discerne, cape clauses), `types.rs` (`lower_type`: TypeExpr → interned TypeId; union `T ∪ nihil`→Option normalization; primitive/collection spellings; numeric widths; instans precision; iuncta; intervallum; scrinium; tensor/vector/matrix/sparsa/atomic error arms; figura index exprs), `pattern.rs` (variant-vs-binding discrimination through the resolver), `literal.rs` (pure `Literal`→`HirLiteral` map; forma rejected), `annotation.rs` (`@` families → normalized `HirAnnotation`; `@radix` typus-domain constraints; `nondum` availability).

Drop list: serde/postcard (serialization); `HirVisitorMut` until a mutating pass exists; `HirTestModifier`/test harness behavior (node shape only); shader local injection; CLI contract records (default `lista<textus>` args only); presentation sidecar (below).

## Module mapping (crate → rivus)

| radix file | rivus file | Faber representation |
|---|---|---|
| `radix-hir/src/lib.rs` | (no file — barrel) | re-export surface folds into nodes.fab/visit.fab |
| `radix-hir/src/nodes.rs` | `src/hir/nodes.fab` | every struct → `class`; every payload enum → `union` (discretio) or `enum` for unit-only; children as arena indices |
| `radix-hir/src/visit.rs` | `src/hir/visit.fab` | `visit_program/item/function/expr/stmt/block/pattern/…` as standalone fns over arena indices + a visitor `class` with the def hook |
| `radix/src/hir/mod.rs` (`LoweredProgram` envelope) | `src/hir/lower/mod.fab` | `LoweredProgram { hir: HirProgram, presentation }` — presentation dropped; envelope returns `HirProgram` + errors |
| `radix/src/hir/lower/mod.rs` | `src/hir/lower/mod.fab` | `Lowerer` class (id counters, scope stacks, error list) + `lower` entry fns |
| `…/lower/decl.rs` | `src/hir/lower/decl.fab` | item lowering + proba items; **annotation.rs folds here** (STRUCTURE.md lists no annotation.fab) |
| `…/lower/expr.rs` | `src/hir/lower/expr.fab` | ExprKind → HirExpressionKind |
| `…/lower/stmt.rs` | `src/hir/lower/stmt.fab` | StmtKind → HirStatementKind + expansion fan-out |
| `…/lower/types.rs` | `src/hir/lower/types.fab` | TypeExpr → TypeId via migrate-types `TypeTable` |
| `…/lower/pattern.rs` | `src/hir/lower/pattern.fab` | Pattern → HirPattern with scope bindings |
| `…/lower/literal.rs` | `src/hir/lower/literal.fab` | pure Literal→HirLiteral map |
| `radix/src/hir/{nodes,visit}.rs` | (no files — re-export barrels of radix-hir) | covered by nodes.fab/visit.fab |

Imports inside the module tree use `import from "rivus:hir/nodes" private nodes` (STRUCTURE.md naming rule).

## Port notes / frictions

- **Arena-ID (L4).** `Box<HirExpression>`/`Box<HirPattern>` children become numeric indices into per-program `lista<HirExpression>`/`lista<HirPattern>` arenas (the Lowerer owns them); `HirPattern::Alias(DefId, Symbol, pattern)` keeps its child as an index too. `HirCasuArm.body` is an index into the expression arena. This is what makes the otherwise-direct recursion legal in Faber.
- **Tuples are available** (`tuple<T1,T2>[a, b]`, EBNF `iunctaExpr`): `(Symbol, HirExpression)` pairs in object entries may port as tuples, or as named pair `class` entries (e.g. `{name: Symbol, value: numerus}`) where field names aid readability (verify tuple codegen in the en build on first use). `HirObjectField { key, value }` already a named class.
- **Type/def identities are indices, not values.** `TypeId`/`DefId`/`Symbol` are interner/arena ordinals; `Option<TypeId>` slots on exprs, locals, returns stay empty until typecheck — port as nullable `TypeId` fields (en `?`/Option), never fill them in lowering.
- **Dependencies on earlier units.** `Span`/`Symbol`/`Token` from rivus `lexer/` + `syntax/span.fab` (migrate-syntax); `CallablePosture`/`GenericParamKind`/`Visibility`/`JsonValue` from `syntax/ast.fab`; `TypeId`/`DefId`/`TypeTable`/`Primitive`/`NumericWidth`/`IndexExpr` from `types/` (migrate-types). The Lowerer needs the resolver surface (`lookup`, `lookup_type`, `get_symbol`, `variant_child`, `define`) — migrate-semantic must expose it before this unit lands.
- **en-surface identifiers port 1:1.** `HirExpressionKind`, `HirBinOp::ApproxNotEq`, `lower_varia`, `next_def_id` — mixed-case Rust spellings are valid en identifiers; do not Latinize. Keywords come from the en pack (`fn`/`const`/`class`/`union`/`enum`/`match`/`import`…).
- **Presentation sidecar: drop initially.** `HirPresentation`, `HirTrivia*`, `HirOwnerRef`, `HirBlockEnd`, `HirProgramEnd`, `HirSourceAnchorId`, and `HirBlock::end` exist to carry comment/trivia attachment for diagnostics and emit. The interpreter neither renders trivia diagnostics nor emits; drop the namespace and the `end` field. `HirBreakableKind::Fac` stays (semantic — `fac`/`rumpe`).
- **Builtins coupling.** `lower_program` calls `crate::builtins::inject_hir_items`; `expr.rs::lower_forma_capture` builds the builtin `forma` genus; `types.rs` resolves `scrinium` frame types; `mod.rs` handles `importa ex "faber:*"` via the kernel manifest. Decision: port a minimal builtins surface (forma genus, scrinium, kernel-glob map) in this unit, or stub with honest reject behavior (L5) and defer to migrate-semantic/driver. Either way the behavior must match radix's accept/reject — the metric, not the mechanism.
- **GPU/tensor lane (L2).** Keep the *reject* arms of `lower_tensor/vector/matrix/sparsa/atomic` and the `tensor_incomplete`-family diagnostics so reject agreement holds; do not port `inject_vertex_shader_locals`/`inject_fragment_shader_locals` or the vertex/fragment `is_*` wiring beyond carrying the annotation flags.
- **CLI packaging out.** `cli_program`/`CliProgram`/`CliType`, `command_args_type`, record-shaped `incipit_args_type`, and the `optio`/`operandus`/`imperium` contract families serve the CLI product surface. Port only the default `incipit argumenta args` binding (`lista<textus>`); normalize the annotation families into records but skip contract resolution (migrate-semantic owns `ResolvedContractApplication`).
- **Lowerer state.** The Rust `&mut` borrows of resolver/types/interner map naturally to en `in` params (mutable reference parameters) on the `Lowerer` class. `LowerError` → a named class carrying issue facts + span; diagnostics rendering belongs to migrate-diagnostics.

## Done-when

1. `src/hir/nodes.fab` declares the arena-ID HIR model — `HirProgram`, `HirItem(+Kind)`, `HirFunction`, `HirStruct`, `HirEnum`, `HirInterface`, `HirTypeAlias`, `HirConst`, `HirImport`, `HirExpression(+Kind)`, `HirStatement(+Kind)`, `HirBlock`, `HirLocal`, `HirParam(+Mode)`, `HirMethod`, `HirReceiver`, `HirField`, `HirVariant(+Field)`, `HirCape`, `HirCasuArm`, `HirPattern`, `HirLiteral`, `HirBinOp`, `HirUnOp`, `HirId`, `HirBreakableKind`, `HirCustodi(+Clause)`, `HirIncDec(+Op)`, `HirScribeKind`, `HirArrayElement`, `HirObjectField/Key`, `HirOptionalChainKind`, `HirNonNullKind`, `HirIteraMode`, `HirConversioTarget`, `HirRangeKind`, `HirRefKind`, `HirTypeParam(+Constraint)`, `HirTestMetadata(+Modifier)`, `HirAnnotation(+Field+Value)`, `HirAvailability` — Radix names, children as arena indices, no `HirBlock::end`/presentation types.
2. `src/hir/visit.fab` walks items/statements/expressions/blocks/patterns in the same pre-order as radix `visit.rs` (definition hook included), over the arena representation.
3. `src/hir/lower/mod.fab` provides `Lowerer` + `lower(...)`: partitions items vs `incipit`/implicit entry, allocates fresh `HirId`/synthetic `DefId`, collects `LowerError`-shaped diagnostics, returns `HirProgram`. Builds green via `faber check`/the P0 closeout (one run, then stop).
4. `src/hir/lower/types.fab` interns types through the migrate-types `TypeTable` (primitives, collections, union/nihil normalization, named/qualified/applied, widths, instans, iuncta, intervallum) with the tensor-family reject arms intact.
5. `src/hir/lower/decl.fab` (annotation normalization folded in) lowers var/fn/class/union/enum/interface/type/import items and proba test items.
6. `src/hir/lower/expr.fab`, `literal.fab`, `stmt.fab`, `pattern.fab` lower every interpreter-relevant ExprKind/StmtKind/Pattern form (incl. destructure expansion, custodi, fac, elige/discerne, cape, forma capture or honest reject) to the matching HIR nodes.
7. Reject agreement at the HIR boundary: for the P1 sample set (`salve-munde` + unit corpus), Rivus lowering accepts exactly the statements radix lowering accepts and reports the same `LowerError` issues for the same shape errors.

## Out of scope

- `radix/src/hir/artifact.rs`, `package.rs`, `serialize.rs` (hir-fhir analyzed-unit envelopes + serialization; serde/postcard dropped).
- HIR consumers: `radix-hir-faber` and the `radix-hir-{rust,ts,go,swift,lean,fhir}` emit surfaces — no codegen (L1/L2). HIR's only Rivus consumer is `src/mir/lower.fab` (migrate-mir).
- Presentation/trivia sidecar (`HirPresentation`, `HirTrivia*`, `HirOwnerRef`, `HirBlockEnd`, `HirProgramEnd`, `HirSourceAnchorId`) and `HirBlock::end`.
- Annotation-contract resolution (`ResolvedContractApplication`, `ContractPayloadValue`) — migrate-semantic.
- CLI packaging (`CliProgram`/`CliType`, record args, optio/operandus/imperium contracts); `LibraryRegistry` construction; kernel-manifest glob imports — driver/semantic boundary.
- GPU/tensor lane implementation (shader local injection, tensor/vector/matrix/sparsa/atomic construction beyond reject arms).
- Test harness behavior for proba suites (node shape only).
- radix test files (`mod_test.rs`, `presentation_test.rs`, `generic_call_test.rs`, `instans_type_test.rs`) — port intent as smallest-proof rivus tests, never verbatim.
