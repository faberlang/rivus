# Rivus — Security

Rivus is an **interpreter for untrusted Faber source**. Its security profile
is the interpreter's, not a compiler's: running a program means executing its
semantics in-process.

## Threat model

| Input | Trust | Consequence |
|---|---|---|
| `rivus run <file.fab>` (user-chosen) | Trusted | None beyond user intent |
| Program the user runs, fed untrusted data | Semi-trusted | The program is trusted; its *inputs* are not. Resource limits apply. |
| `rivus run` in any automated/agent context | Untrusted by default | Default-reject host kernels + resource budgets required |

## Posture (mirrors Radix's stepper)

1. **Host kernels default-reject.** The interpreter's `Host` implementations
   follow Radix's stepper contract: provider/kernel calls (file IO, subprocess,
   environment, network) are **rejected unless the host explicitly enables
   them**. A `StdioHost`-equivalent enables only stdout/stdin/exit; everything
   else is opt-in. This is a structural gate, not a policy flag — the
   `StepperDispatchStatus` classifier (Implement / HostDelegate /
   IntentionalReject) is ported and matched against Radix.
2. **No ambient capability.** Imported kernel surfaces (`solum`, `processus`,
   `toml`, `json`, `aleator`, `args`) resolve only through the active `Host`.
   No capability is ambient.
3. **Resource budgets.** Spec fields (thresholds enforced from P2):
   `max_source_bytes`, `max_tokens`, `max_ast_nodes`, `max_parse_depth`,
   `max_interner_entries`. Enforcement lives in the driver before/inside the
   pipeline — never as a post-hoc kill.
4. **No network by default.** No kernel performs network IO unless a host
   explicitly provides one.
5. **Determinism by default.** `tabula`-backed maps keep iteration ordered;
   the stepper is deterministic given identical inputs — no hidden nondeterminism
   for the parity oracle.

## Design constraints

- **Source-text firewall:** lex consumes source once; every later stage works
  on semantic structures + interned symbols. Nothing after lex re-reads
  source bytes, so no stage can be tricked by malformed bytes it "reparses."
- **Reject agreement is a correctness floor:** if Radix's stepper rejects a
  construct and Rivus accepts it (or vice versa), that is a defect, not a
  coverage gap.
- Untrusted input is the default assumption for any future server/agent
  surface; budgets + reject-by-default are baked in from the start, not added
  later.
