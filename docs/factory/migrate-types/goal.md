# Goal: migrate-types

**Status**: planned — mapping complete 2026-08-10; implementation pending
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-types/
**Primary surface**: `radix/crates/radix-types/src/{types,def_id,index,numeric_width,instans_precision,lib}.rs` → `rivus/src/types/{types,def_id,numeric_width,instans_precision}.fab`
**Depends on**: migrate-diagnostics
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Port `radix-types`, the semantic type foundation crate (~2.6k lines: 1790 types.rs, 530 numeric_width.rs, 160 instans_precision.rs, 140 index.rs, 30 def_id.rs), to `rivus/src/types/`. Five source files fold into four Faber files (index.rs rides inside types.fab because `TypeTable` owns the index arena). Data model is arena-ID throughout: `TypeId`/`DefId`/`IndexId` are indices into `lista<T>` arenas; Rust hash maps become `tabula<K,V>` (ordered, deterministic). Identifiers port 1:1 (en surface). The stepper check (Q1) shows Radix's stepper decodes the **full** `Type` union plus `IndexExpr`/`IndexId`/`NumericWidth`/`InstansPrecision`, so all type variants survive as data; only tensor-lane analysis/sugar helpers and MIR-image serialization are dropped.

## Crate analysis

- **types.rs** (`radix-types/src/types.rs`) — the core. `TypeId(u32)` arena index; `Type` union (35 variants: 14 primitives, collections, tensor family, sized numeric/modular/instans, option/ref/record/tuple, struct/enum/interface/alias, func/param/applied, infer/infer-union/union, error); `TypeTable` holding `Vec<Type>` types arena, hash-cons `intern_map`, `Vec<IndexExpr>` indices arena, primitives index, three `RefCell` memos (equals/assignable/find_equal), `first_modular_word`, `variant_parent`. `intern` hash-conses and canonicalizes unions (member sort). `equals`/`assignable` are the semantic comparison engines with alias-peel + cycle guards; `union_member_sort_key` makes union dedup deterministic. Also: `Primitive` (14), Latin surface tables `PRIMITIVE_TYPE_SPECS` / `COLLECTION_TYPE_SPECS` / `TENSOR_FAMILY_TYPE_SPECS` (+ lookups; `objectum`/`quidlibet` alias to `Ignotum`), `CollectionKind`, `TensorFamilyKind`, `FuncSig`/`ParamType`/`TypeParamConstraint`/`SemanticParamMode`/`SemanticMutability`, `InferVar`. **Label inference** — `Infer`/`InferUnion`/`InferVar` and `IndexVar` (`fresh_index_infer_var`) are the inference-placeholder machinery; keep (typecheck uses it).
- **def_id.rs** (`radix-types/src/def_id.rs`) — `DefId(pub u32)` with `INVALID = u32::MAX` sentinel. Shared by type system and HIR (`radix-hir` re-exports `radix_types::DefId`). `USER_DEF_ID_BASE: u32 = 0x0000_1000` (lib.rs) partitions compiler-injected builtins from user defs — keep as a const.
- **index.rs** (`radix-types/src/index.rs`) — `IndexId`/`IndexVar` arena handles and `IndexExpr` (Literal/Param/Tuple/Infer/Unspecified); `literal_index_i64s`, `compute_cotangent_reduction_axes`, `index_equals` (folded into TypeTable in types.rs).
- **numeric_width.rs** (`radix-types/src/numeric_width.rs`) — `NumericWidth` (I8…U64, F16/F32/F64), `WidthError`, the v1 widening lattice (`widening_assignable`), `from_marker`, `effective_width`, `numeric_assignable`, `explicit_narrowing_conversio`, `unsigned_max`, standalone width sugar (`f32` → `fractus<f32>`), tensor/vector/matrix/sparsa/lista sugar markers, and `lower_sized_primitive`/`lower_modular_word`/`lower_standalone_width_type` (consume `radix_syntax::TypeExpr`).
- **instans_precision.rs** (`radix-types/src/instans_precision.rs`) — `InstansPrecision` (Millis/Micros/Nanos), `InstansPrecisionRank` (+Seconds), `InstansPrecisionError`, `effective_instans_precision`, `instans_assignable`, `lower_sized_instans`.
- **Dependencies**: `radix-lexer` (Symbol/Interner — used by `Record` fields and `FuncSig`), `radix-syntax` (TypeExpr — only in the `lower_*` helpers), `rustc_hash` (FxHashMap/FxHashSet → `tabula<K,V>`/`copia<T>`), `serde` (drops with snapshot), `RefCell` (memo interior mutability — not needed in Faber where the table is owned mutably).
- **Stepper evidence (Q1)**: `radix-mir-stepper` non-test source imports exactly `effective_width, IndexExpr, IndexId, InstansPrecision, NumericWidth, Primitive, Type, TypeId, TypeTable` (lib.rs:31, runtime.rs:50-52, value.rs:32, conversio.rs:16). It decodes `Type::Sparsa/Vector/Matrix` for aggregate materialization (runtime.rs:540,585,611), `Type::Tensor` (runtime.rs:1639-1640, conversio/aggregate.rs:134-135,249), `Type::SizedInstans` → runtime `InstansPraecisio` (conversio.rs:172-174), and reads `IndexExpr`/`IndexId` at runtime (`literal_sparse_shape`, runtime.rs:3599-3619). Its TypeTable surface is `get` + `get_index` only. **The stepper demonstrably needs every radix-types type → nothing on the L2 drop list is actually dropped.**

## Module mapping (crate → rivus)

| Radix file | Rivus file | Faber representation (en identifiers) |
|---|---|---|
| `radix-types/src/types.rs` | `src/types/types.fab` | `class TypeId` (arena handle); `union Type` (all 35 variants, payloads as arena IDs / named payload classes); `class TypeTable` with `lista<Type>` + `lista<IndexExpr>` arenas, `tabula` intern/primitive indexes, memo fields as `tabula`; `union Primitive` (14); spec tables as `class` + `lista<…>`; `class FuncSig`/`ParamType`/`TypeParamConstraint`/`InferVar`; `union SemanticMutability`/`SemanticParamMode`; `fn type_equal`/`fn assignable` engines; 14-primitive seeding in `new()` |
| `radix-types/src/def_id.rs` | `src/types/def_id.fab` | `class DefId` (u32 arena handle) + INVALID sentinel; `const USER_DEF_ID_BASE` |
| `radix-types/src/index.rs` | `src/types/types.fab` (folded) | `class IndexId`/`IndexVar`; `union IndexExpr`; `fn index_equals` — lives beside `TypeTable`, which owns the `indices` arena |
| `radix-types/src/numeric_width.rs` | `src/types/numeric_width.fab` | `enum NumericWidth` (11); `enum WidthError`; `fn from_marker`/`widening_assignable`/`effective_width`/`numeric_assignable`/`explicit_narrowing_conversio`/`unsigned_max`; `fn standalone_width_type` |
| `radix-types/src/instans_precision.rs` | `src/types/instans_precision.fab` | `enum InstansPrecision`; `enum InstansPrecisionRank`; `enum InstansPrecisionError`; `fn effective_instans_precision`/`instans_assignable` |
| `radix-types/src/lib.rs` | (folded) | re-export surface + `USER_DEF_ID_BASE` live in def_id.fab / types.fab |

Imports: `import from "rivus:types/def_id"` etc. `Symbol` (Record fields, FuncSig `type_params`) imports from `rivus:lexer/interner` (assumes migrate-lexer's interner landed; see frictions).

## Port notes / frictions

1. **Q1 narrowing — keep the union whole.** The L2 drop list (Tensor/Vector/Matrix/Sparsa/SizedInstans) all fail the "stepper needs them" check above, so every `Type` variant stays as data. What drops is the *helper surface*: `compute_cotangent_reduction_axes`, `literal_index_i64s`, `tensor_media_width`, `is_tensor_numeric_element`/`is_tensor_element_type`, `tensor_method_requires_numeric_element`, `tensor_element_conversio_allowed`, `is_integer_index_vector`/`is_integer_index_scalar`, `TENSOR_ELEMENT_NUMERIC_MSG`/`TENSOR_INDEX_VECTOR_MSG`, and the sugar-marker fns (`tensor_/vector_/matrix_/sparsa_/lista_sugar_width_marker`, `is_numeric_width_type_sugar`) — parser/typecheck conveniences for a lane Rivus rejects. `Atomic`/`Intervallum` are **not** on the drop list and are cheap data; keep, but flag: the stepper never matches them, so if reject-parity work wants a leaner union they are the first cut (re-add on demand).
2. **`lower_sized_primitive`/`lower_modular_word`/`lower_standalone_width_type`/`lower_sized_instans` defer** to the semantic unit — they consume parser `TypeExpr` and a diagnostics `report` callback. numeric_width.fab / instans_precision.fab keep only the pure lattice and marker parsing.
3. **Hash maps → ordered maps.** `intern_map` (Type→TypeId) and memos become `tabula<K,V>`. Determinism win: `Record` fields (FxHashMap today) become an ordered `tabula<Symbol, TypeId>`. Union members stay sorted on intern — `union_member_sort_key` ports as-is.
4. **Hash-cons risk.** `intern_map` needs structural equality/hash on the `Type` union. If en-surface unions can't be tabula keys, fall back to linear-scan `find_equal` (semantically sufficient; the arena is append-only). Keep the positive-only memo semantics.
5. **Memo / RefCell.** `equals`/`assignable` are `&self` in Rust because of interior mutability. In Faber the frontend owns the table, so either drop the memos initially or make them plain `tabula` fields — correctness is independent of them.
6. **Structural recursion.** `type_equal_with_seen`/`assignable_inner`/`nullable_inner` are recursive with a seen-set cycle guard. Confirm Faber direct recursion for bounded-depth type walks; if restricted, rewrite as explicit worklists (same semantics).
7. **`TypeTableSnapshot`/`from_snapshot` + serde derives drop** — that is MIR-image serialization for codegen/CLI; the stepper receives a live table and never snapshots (grep confirms zero stepper use).
8. **`variant_parent` field defers** — populated by typecheck for exhaustiveness/return-path; not a types-crate concern at this unit.

## Done-when

1. `src/types/def_id.fab` exists: `class DefId`, `INVALID` sentinel, `const USER_DEF_ID_BASE` — en surface, `import from "rivus:types/def_id"` pattern.
2. `src/types/types.fab` exists: `class TypeId`, `union Type` with all Radix variants, `class TypeTable` (lista arenas for types + index exprs, tabula intern/primitive indexes, 14-primitive seeding, `intern`/`get`/`get_index`/`equals`/`assignable`/`find_equal`/`union_member_sort_key`/`nullable_inner` + collection/sized constructors), plus `union Primitive`, spec tables, `FuncSig`/`ParamType`/`TypeParamConstraint`/`SemanticMutability`/`SemanticParamMode`/`InferVar`.
3. `src/types/numeric_width.fab` and `src/types/instans_precision.fab` exist with the full pure surfaces (lattice, marker parsing, effective-width/precision, assignability).
4. **No `FxHashMap`/`HashMap`/`BTreeMap` equivalent in the port** — `tabula<K,V>` and `lista<T>` only; iteration order deterministic.
5. Smoke test: build a `TypeTable`, verify the 14 primitives resolve, `numerus<i32>` → `numerus<i64>` widening assignable, `numerus` → `fractus` assignable, `numerus<u32>` → `numerus<i32>` not assignable, `instans<ns>` → `instans<ms>` reject, `instans<ms>` → `instans<ns>` assignable.
6. The module compiles under the en package (`faber check` on touched files); one closeout `./scripta/test --stage 1-3` from radix/ green, then stop.

## Out of scope

- Type lowering from parser `TypeExpr` (`lower_*` helpers) — the semantic/typecheck unit owns it (depends on migrate-syntax/parser + migrate-diagnostics reporting).
- `TypeTableSnapshot` / MIR-image serialization / serde derives.
- Tensor-lane analysis helpers, cotangent-reduction, width-sugar markers.
- `variant_parent` and exhaustiveness hooks.
- Stepper runtime values and the tensor *method* lane (`radix-mir-stepper` value/runtime dispatch) — separate mir-stepper units.
- Lattice property tests and full parity fixtures — auditor-owned at named boundaries.
