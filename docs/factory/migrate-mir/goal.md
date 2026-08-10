# Goal: migrate-mir

**Status**: planned — mapping complete 2026-08-10; implementation pending
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-mir/
**Primary surface**: radix `crates/radix-mir/src/{nodes,control,ty,local_ty,names,capability,layout,placement,collection_ops,generic,figura,kernel_module,dump}.rs` + `crates/radix-mir/src/validate/` + `crates/radix/src/mir/{lower.rs,lower/,mod.rs}` → rivus `src/mir/{nodes,control,ty,names,capability,layout,collection_ops,generic,lower}.fab`
**Depends on**: migrate-hir, migrate-types, migrate-semantic
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Port the scalar/control core of the `radix-mir` leaf crate and the main-crate HIR→MIR lowering into `rivus/src/mir/` in Faber (`locale = "en"`, arena-ID data model, `tabula` ordered maps, tuples available as `tuple<T1,T2>[a, b]`). MIR is the typed IR the Rivus stepper executes: `MirProgram`/`MirFunction`/`MirBlock`/`MirTerminator`, the value/operand/place/constant model, type helpers, names, capability vocabulary, layout table, generic binding, validation, and the `lower_analyzed_unit_with_context` pipeline (fail-closed `MirError`). The tensor/GPU/device surface is dropped per L2.

**Label inference:** status `planned` (STRUCTURE.md legend; pre-implementation). This goal is the mapping contract for the `src/mir/` units; body work waits on migrate-types + migrate-semantic (MIR is typed — `MirType` wraps `TypeId`).

## Crate analysis

- **nodes.rs** (1112) — the node model. `MirProgram { functions }`; `MirFunction` (id, source `DefId`, name, params/locals/temps, blocks, return/error ty, is_async/generator, span); `MirBlock { statements, terminator }`; `MirStatementKind` (Assign/Call/RuntimeCall/Construct); `MirTerminatorKind` (Return/ReturnError/TryCall/Goto/Branch/Switch/Unreachable) — terminators own CFG edges; `MirValue`/`MirValueKind` (Operand/Closure/Unary/Binary/Option); `MirOperand` (Place/Temp/Value/Constant); `MirPlace` (base Local|Temp + projections Field/VariantField/ClosureCapture/Index/VectorLane/MatrixCell); `MirConstant` (Int/UInt/Float/String/Ascii/Bool/Nil/Unit/Octeti/Regex/Function); `MirUnOp`/`MirBinOp`; `MirCallee`; `MirRuntimeCall`/`MirIntrinsic`; `MirCollectionOp`; `MirConversion`; `MirAggregate`/`MirAggregateKind`; `MirOptionOp`/`MirOptionChainLink`; closure + monomorphization structs; arena-id wrappers `Mir{Function,Block,Local,Temp,Value,Layout,ClosureEnvironment}Id(u32)`; `MirExternalFunctionIdentity`. **Label inference:** intrinsic `GpuBuiltin`/`Gradient` and all `Tensor*`/`Sparse*` variants are tensor lane → drop; `TypeCheck`, `Sermo*`, `Cede`, `CursorStream` stay (stepper dispatch ledger).
- **control.rs** (1445) — structured-CFG shape recognition for **device kernel lowering**. Only target-neutral scalar predicates survive: `structured_scalar_loop(_break/_continue)`, `structured_scalar_switch`, `structured_return_branch`, `structured_value_join_branch`, `structured_scalar_return_goto` (+ `Structured*` carriers). All `indexed_write_*`/`nested_*` predicates are GPU lane → drop. The stepper interprets blocks directly; it does not consume control.rs.
- **ty.rs** (204) + **local_ty.rs** (40) — `MirTypeError` (message + issue), helpers `option_payload_ty`/`option_chain_base_ty`/`constant_ty`/`index_projection_result_ty`/`place_base_ty`; `MirTypeLookup` + params-then-locals lookup.
- **names.rs** (299) — `MirNames` (id spellings, function names, `entry_function`/`explicit_entry_function`, monomorphized disambiguation), `sanitize_name`, literal identities. Stepper uses `MirNames::new` + `explicit_entry_function` (`radix-mir-stepper/src/capability.rs:119`). Drop `llvm_*`/wasm slot helpers + `external_function_import_name` (codegen).
- **capability.rs** (125) — `CapabilityGap` (5 variants) + `Lowerability` (Capable/Rejected); stepper's classifier imports both (`radix-mir-stepper/src/capability.rs:32`). Also port `MirRuntimeAbiFamily` (closed 15-variant enum, `radix-mir/src/abi/types.rs:69`) — `stepper_dispatch_status` needs it.
- **layout.rs** (737) — `MirScalarLayout`, `MirLayoutKind` (Scalar/Aggregate/Atomic/OpaqueHandle/Void/Never + device kinds), `MirAggregateFamily`/`MirAggregateCarrier`, `MirLayoutTable` (per-TypeId + per-id rows), `layout_kind_for_type` walk. Keep scalar + aggregate + opaque-handle; drop DeviceView/Vector/Matrix, PackedNumeric, wgsl/metal/llvm spellings. `MirLayoutTable::from_types` is consumed by lowering and validation.
- **placement.rs** (195) — `PlacementHost` device-execution contract + `PlacementError`. Device lane: port as error vocabulary only, never dispatched (L2 reject).
- **collection_ops.rs** (76) — `statement_collection_op` destructure; the whole-function op scans depend on `read_analysis::place_is_read` (device planning) → drop scans.
- **generic.rs** (968) — `MirGenericBindings` (`tabula<Symbol, MirType>`), `bind_generic_type_witness` (alias-peel cycle guard, union backtracking with insert-only bindings + undo), `instantiate_existing_generic_type` (content-addressed existing-type index), `type_contains_param`. Full port (validation uses it).
- **figura.rs** (42) — `literal_index_i64s`/`format_figura` (IndexExpr resolution); only tensor/vector/matrix shapes need it → fold into layout.fab or drop with the lane.
- **kernel_module.rs** (48) — `KernelModule` (Solum/Processus/Aleator/Json/Consolum/Toml) in `MirProviderKind::Kernel`; stepper host dispatch uses it (`stepper/src/kernel/mod.rs`) → fold into nodes.fab.
- **validate/** (~5.2k: mod 57, validator 249, context 622, error 64, stmt 401, value 570, intrinsic 1915, aggregate 555, type 726) — `MirValidationContext` (types + `MirLayoutTable` + functions/struct_fields/enum_variants/variant_positions/closure_environments/scalar_value_domains maps), `MirEnumRepresentation` (7 variants incl. PointerNiche/IntegerPayload sentinels), `MirScalarValueDomain`, `ValidatedMir` token, `validate_program`, `FunctionShape`, signatures, `MirValidationError`. Stepper consumes `MirValidationContext`/`ValidatedMir` throughout — full port.
- **dump.rs** (732) — `dump_program`; **not consumed by the stepper** → defer (optional fixture/test support).
- **visit.rs** — not in scope; the `MirVisitor` trait is dropped; traversal becomes plain recursive fns.
- **radix/src/mir/lower.rs** (2466) + **lower/** (13 files, ~7.2k prod) — lowering boundary. Entry points `lower_analyzed_unit[_with_context|_allowing_cli_*]` → `LoweredMirUnit { program, interner, monomorphizations, closure_environments, validated }`; `MirError` (unsupported/missing_type/validation + issue slugs). `MirLowerer` (whole-program order), `ItemLoweringPass`, `FunctionBuilder` (bindings, params/locals/temps, open blocks, breakables/handlers, value numbering), entry synthesis (`lower_entry`), `LoweringContextMaps`, CLI-record handling, type-zero, short-circuit CFG, `mir_un_op`/`mir_bin_op`. Submodules: aggregate 553, async_surface 47, callable 420, collection_higher_order 345, context 772, control 1579, expr 569, item 71, monomorphize 840, place 272, runtime 2584, stmt 103. Drop device-role/shader-stage seams, S8.2 CLI adapter, companions, tensor-bracket ensure. **mod.rs** (276) is re-export glue → becomes the `import from "rivus:mir/<file>"` surface.

## Module mapping (crate → rivus)

| Radix file | Rivus file | Faber representation (en identifiers 1:1) |
|---|---|---|
| nodes.rs, kernel_module.rs | `src/mir/nodes.fab` | node structs + closure/monomorphization → `class`; statement/terminator/value/operand/place/constant/callee/intrinsic/collection-op/aggregate-kind/option-op/unop/binop/param-mode → `union`; id wrappers → `class` holding `numerus<u32>`; `KernelModule` → `union` |
| control.rs | `src/mir/control.fab` | scalar predicates → `fn`; `Structured*` carriers → `class` |
| ty.rs + local_ty.rs | `src/mir/ty.fab` | `MirTypeError` → `class`; 5 helpers → `fn`; `MirTypeLookup` → two `fn` args over function context |
| names.rs | `src/mir/names.fab` | `MirNames` → `class` (function-name `tabula<MirFunctionId,textus>` cache); entry detection, `sanitize_name`, literal identities → `fn` |
| capability.rs + abi/types.rs enum | `src/mir/capability.fab` | `CapabilityGap`/`Lowerability`/`MirRuntimeAbiFamily` → `union` |
| layout.rs, figura.rs | `src/mir/layout.fab` | scalar/aggregate/opaque layout enums → `union`; `MirLayoutTable` → `class` with two `tabula`; `layout_kind_for_type`, `literal_index_i64s` → `fn` |
| placement.rs | folds into `layout.fab` | `PlacementError` → `union`; `PlacementHost` documented, not dispatchable |
| collection_ops.rs | `src/mir/collection_ops.fab` | `statement_collection_op` → `fn`; whole-function scans dropped |
| generic.rs | `src/mir/generic.fab` | `MirGenericBindings` = `tabula<Symbol,MirType>`; bind/instantiate/contains-param → `fn` |
| validate/context.rs + error.rs | folds into `ty.fab` | `MirValidationContext` → `class`; `MirEnumRepresentation`/`MirScalarValueDomain` → `union`; `MirFunctionSignature`/`MirParamSignature`/`MirValidationError` → `class`; enum-representation selection → `fn` |
| validate/validator.rs + stmt/value/intrinsic/aggregate/type.rs | folds into `lower.fab` | `ValidatedMir` → `class` token; `validate_program` + 5 check passes → `fn` (accumulate error list); `FunctionShape` → `class` |
| radix/src/mir/lower.rs + lower/*.rs | `src/mir/lower.fab` (+ `src/mir/lower/*.fab` verbatim mirror, recommended split) | `MirError` → `class`; `MirLowerer`/`FunctionBuilder`/`LoweringContextMaps` → `class`; entry points, `LoweredMirUnit`, monomorphize, `mir_un_op`/`mir_bin_op` → `fn` |
| radix/src/mir/mod.rs | (glue) | re-export surface becomes import lines; no file |

## Port notes / frictions

1. **Typed MIR needs the type table.** `MirType { semantic: TypeId, layout: Option<MirLayoutId> }`; `constant_ty`, `option_payload_ty`, `index_projection_result_ty`, `layout_kind_for_type`, `bind_generic_type_witness` all walk `TypeTable` (`get`/`primitive`/`find_sized_numeric`/`assignable`/`equals`/`index_equals`/`nullable_inner`/`type_count`/`get_index`) — requires migrate-types. Lowering additionally consumes `AnalyzedUnit` + typed `HirProgram` (migrate-semantic) and `DefId`/`Symbol`/`Span`/`Interner` (migrate-hir / lexer mirror).
2. **Data model (L4):** `Vec<T>` → `lista<T>`; maps/sets → `tabula<K,V>` (ordered — the determinism metric); `Option<T>` → nullable union / `MirConstant::Nil`; tuples (e.g. `(place, ty)` returns) may port as `tuple<T1,T2>[a, b]` or named `class` carriers where a name aids clarity; no `char`/`Ord` — `Symbol` sorts by `u32` id; no direct recursion — arena IDs already the design.
3. **en surface (L3):** identifiers port 1:1 (`MirProgram`, `MirTerminatorKind::TryCall`, `KernelModule::Solum`). Rust enums → `union`, structs → `class`, traits → plain `fn` params (no trait system).
4. **Validation home.** `validate/` is stepper-critical. Fold context/error/enum-representation into `ty.fab`; validator + check passes into `lower.fab` (lowering returns the `ValidatedMir` token). If the stepper unit needs the checks decoupled, lift into a `validate.fab` via STRUCTURE.md update — default is the fold.
5. **`MirRuntimeAbiFamily`** (`abi/types.rs:69`) is outside this goal's read scope but the dispatch ledger needs it; port the 15-variant enum into `capability.fab`. Full `abi/` (broadcast/contract/launch/reflection) is out of scope.
6. **Visitor trait disappears** (visit.rs dropped); FunctionShape/duplicate-finder/dump use plain recursive functions.
7. **`MirCollectionOp` scalar subset:** keep Append…RegexMatches; drop every `Tensor*`/`Sparse*` variant (they only feed L2-rejected lanes).
8. **Lowering submodule split.** `lower.rs` + `lower/*.rs` ≈ 9.6k lines; the STRUCTURE.md naming rule mirrors verbatim. Recommended: `lower.fab` (orchestration + entry points) + `src/mir/lower/{item,expr,stmt,place,aggregate,callable,runtime,control,context,monomorphize,collection_higher_order,async_surface}.fab`. The `rivus:mir/...` import surface is the contract.
9. **Drop inside lower:** `MirDeviceRole`/shader-stage seams, S8.2 CLI adapter (`cli_adapter_active`, `runtime_adapter_record_fields`), `companions`, `ensure_tensor_bracket_index_array_types`. Keep `ensure_map_iteration_array_types`, type-zero, short-circuit CFG, CLI record defaults (`rivus run` needs argv).
10. **MirError slugs are behavior:** keep `unsupported_mir_lowering` / `missing_type_information_for_mir_lowering` / `invalid_mir` for reject agreement.

## Done-when

1. `src/mir/{nodes,control,ty,names,capability,layout,collection_ops,generic,lower}.fab` exist and build under the en package; imports use `import from "rivus:mir/<file>"`; no inlined monolith.
2. nodes.fab ports the full node model with en identifiers and drops every `Tensor*`/`Sparse*` aggregate kind, collection op, projection, and intrinsic family; `KernelModule` folds in.
3. ty.fab ports `MirTypeError`, the 5 helpers, `MirTypeLookup`, `MirValidationContext`, `MirEnumRepresentation`/`MirScalarValueDomain`, signatures, error types; enum representation selection matches `validate/context.rs`.
4. capability.fab ports `CapabilityGap`/`Lowerability` + `MirRuntimeAbiFamily`; names.fab ports `MirNames` + entry detection; layout.fab ports the scalar/aggregate/opaque `MirLayoutTable`; generic.fab ports bind/instantiate/contains-param with union backtracking.
5. lower.fab lowers a trivial analyzed unit (top-level `const` + `print` + arithmetic) to a **validated** `MirProgram` whose shape matches Radix's `lower_analyzed_unit_with_context` output for the same input (dump-equivalent comparison); unsupported HIR surfaces produce `MirError` with the Radix issue slugs (reject agreement).
6. `validate_program` accumulates all errors (never repairs); the `ValidatedMir` token cannot be constructed for an invalid program.
7. One closeout run of the touched-module check after the last edit, then stop (workspace Cargo discipline; stage-4+ runs are auditor-owned).

## Out of scope

- All MIR codegen backends (wasm/wgsl/metal/llvm/fmir/sexp/coverage) and their naming/layout spellings (llvm/wasm slots, `wgsl_ty`/`metal_ty`/`llvm_ty`, `external_function_import_name`).
- Tensor/GPU/device lane (L2): `tensor_*.rs`, `gradient.rs`, `device_*`, `device_program`, `kernel_decomposition`, `shader_contract`, `host_partition`, `ai_workbench`, `static_shape_fold`, `tensor_bracket`, `tensor_operation_*`, `tensor_runtime_abi`, `tensor_type`, `tensor_placement`, `rank2_reduction`, `lista_tensor_conversio`, `abi/`, `read_analysis`, `boundary_abi`, ddcp1 fixtures.
- control.rs indexed-write / nested-loop predicates; `dump.rs` (deferred, optional test support); `visit.rs`; serde derives; borrow analysis (Radix gates it off for the Faber surface).
- The stepper itself (`src/mir-stepper/` — separate migrate-mir-stepper goal), `src/driver/` orchestration, full `abi/` runtime-boundary ledger beyond `MirRuntimeAbiFamily`.
