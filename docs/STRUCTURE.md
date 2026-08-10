# Rivus — Structure Map

The module tree mirrors Radix's front-end + MIR + stepper crates file-for-file.
Each directory below is a planned module group; files land as campaign units
are implemented. **Status: `planned` until a unit creates the file.** The
line counts are Radix production sizes (excluding `*_test.rs`) — the port
target, not the current state.

## Naming rule (crate mirror)

Directory names mirror Radix **crate** names with the `radix-` prefix stripped
(`radix-mir-stepper` → `src/mir-stepper/`). Internal modules of the main
`radix` crate (not separate crates) mirror their path under
`crates/radix/src/` verbatim (`crates/radix/src/hir/lower/*.rs` →
`src/hir/lower/*.fab`). The import surface uses the directory name with the
`rivus:` package prefix and the file name appended:
`src/mir-stepper/value.fab` → `import from "rivus:mir-stepper/value"`.

| Radix crate | Rivus directory |
|---|---|
| `radix-lexer` | `src/lexer/` |
| `radix-syntax` | `src/syntax/` |
| `radix-parser` | `src/parser/` |
| `radix-hir` | `src/hir/` |
| `radix-types` | `src/types/` |
| `radix-mir` | `src/mir/` |
| `radix-mir-stepper` | `src/mir-stepper/` |
| `radix-diagnostics` | `src/diagnostics/` |
| `radix-runtime-contract` | not a dir yet — ABI constants (e.g. stepper runtime error codes) fold into the crate that uses them; a `src/runtime-contract/` appears only if a real boundary emerges |
| `radix-mir-llvm` / `-wasm` / `-wgsl` / `-metal` / `-fmir` / `-sexp` / `-python` / `-coverage` | dropped (no codegen); would be `src/mir-<backend>/` if ever needed |

Note: `diagnostics` stays `src/diagnostics/` — there is no
`radix-mir-diagnostics` crate; `radix-diagnostics` serves the whole
front-end. The `mir-` prefix marks only the MIR-family crates.

## Radix mirror reference

| Radix location | Role |
|---|---|
| `crates/radix-lexer` | token scanner, keywords, cursor, `Interner`/`Symbol` |
| `crates/radix-syntax` | AST node types, `Span`, trivia, visitor |
| `crates/radix-parser` | recursive-descent parser (decl/expr/stmt/types/pattern) |
| `crates/radix-hir` | HIR node types |
| `crates/radix/src/hir/lower` | AST → HIR normalization |
| `crates/radix-types` | `Type`/`TypeId`/`TypeTable`, `DefId`, width/precision |
| `crates/radix/src/semantic/passes` | collect, resolve, typecheck, analysis passes |
| `crates/radix-mir` | typed MIR core (nodes, control, ty, names, capability, …) |
| `crates/radix/src/mir/lower.rs` | HIR → MIR lowering (2.5k) |
| `crates/radix-mir-stepper` | in-process interpreter: Value, runtime, Host, kernels |
| `crates/radix-diagnostics` | diagnostics |
| `crates/radix/src/driver` | session, source, frontmatter, analyze orchestration |

## Module tree

```
src/
  main.fab            CLI entry: rivus run <file.fab>            [bin entry]
  driver/             driver subset (session, source, frontmatter)
  lexer/              radix-lexer mirror (~3.3k)
    token.fab         TokenKind enum
    keywords.fab      keyword table (en surface)
    scan.fab          scanner
    cursor.fab        UTF-8/textus cursor
    interner.fab      Symbol ↔ string interner
  syntax/             radix-syntax mirror (~3.7k)
    ast.fab           AST node types (arena-ID)
    span.fab          byte spans
    trivia.fab        comment/whitespace attachment
    visit.fab         visitor
  parser/             radix-parser mirror (~7.3k)
    decl.fab
    expr.fab
    stmt.fab
    types.fab
    pattern.fab
    error.fab         parse errors + recovery
  hir/                radix-hir mirror (~2.9k)
    nodes.fab         HIR node types
    visit.fab
    lower/            hir/lower mirror (~5.1k)
      decl.fab
      expr.fab
      stmt.fab
      types.fab
      pattern.fab
      literal.fab
  types/              radix-types mirror (~2.7k)
    types.fab         Type / TypeId / TypeTable
    def_id.fab
    numeric_width.fab
    instans_precision.fab
  semantic/           passes mirror (needed subset ~10–16k)
    collect.fab
    resolve.fab
    typecheck.fab     the ladder — biggest single pass
    definite_assignment.fab
    exhaustive.fab
    return_path.fab
    lint.fab
    visibility.fab
    import_path.fab
  mir/                radix-mir core mirror (~5.3k needed subset)
    nodes.fab         MirInstruction etc.
    control.fab       MirBlock / MirTerminatorKind
    ty.fab
    names.fab
    capability.fab    MirRuntimeAbiFamily → dispatch status
    layout.fab
    collection_ops.fab
    generic.fab
    lower.fab         HIR → MIR lowering (mirror of mir/lower.rs, 2.5k)
  mir-stepper/        radix-mir-stepper mirror (~10k)
    value.fab         Value enum + arithmetic/compare/conversion ops
    runtime.fab       MirRuntimeCall evaluation (the big match)
    host.fab          Host trait + StdioHost-equivalent
    capability.fab    StepperDispatchStatus (Implement/HostDelegate/IntentionalReject)
    textus_literal.fab
    kernel/           host kernel bindings (~1.3k)
      solum.fab       file IO
      processus.fab   subprocess
      toml.fab
      json.fab
      aleator.fab     random
      args.fab        CLI args
    conversio/        conversion semantics
      scalar.fab
      aggregate.fab
  diagnostics/        radix-diagnostics mirror (~0.8k)
    diagnostic.fab
    span.fab

scripts/
  run-parity.fab      planned: rivus run vs radix run_source — stdout + exit diff
```

## Dropped (deliberately, per CAMPAIGN.md scope locks)

| Radix surface | Why dropped |
|---|---|
| All MIR codegen backends (wasm/wgsl/metal/llvm/fmir/sexp/python/coverage) | No codegen. The stepper is the only MIR consumer. |
| `radix-hir-faber` + `radix-hir-{rust,ts,go,swift,lean,fhir}` | No emit oracle, no other targets. |
| GPU/tensor lane (tensor_*, gradient, device_program, air, broadcast…) | L2 — match Radix reject behavior instead. |
| `borrow.rs`, `air_purity.rs`, `kernel_namespace` | Gated off for the Faber surface / GPU lane. |
| forma, full driver (cli/target_policy/shader_contract/…) | Thin driver only. |

## Status legend

- `planned` — directory exists in this map; file not yet created.
- `scaffold` — file exists, minimal.
- `implementing` / `done` — set by campaign units as they land.

All modules are `planned` at skeleton time.
