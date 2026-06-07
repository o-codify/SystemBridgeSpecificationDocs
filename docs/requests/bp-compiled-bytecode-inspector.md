---
id: request-compiled-blueprint-bytecode-kismet-read-only-inspector
title: "Request: compiled Blueprint bytecode (Kismet) read-only inspector"
status: request
version: 26.607.2208
tags: [ unreal, blueprint, kismet, diagnostic, request ]
---

# Request: compiled Blueprint bytecode inspector

## Status (2026-Q2)

**Shipped** in Companion v1.12.3 (commit 70d94b5):

- ✅ `bp_function_bytecode(bp_path, function_name)` — line-by-line disassembly.

Coverage: ~80 most common EX_* opcodes by name. Unknown opcodes surface as `EX_0xNN (raw)` — caller still sees offsets. Operand decode (the full "EX_CallMath → AnimSequenceLength" comment column) is NOT yet implemented; opcode sequence is usually enough to spot pruning / branch logic problems.

`bp_function_bytecode_summary` (the second proposed tool) deferred — class-grouped histogram across functions can be built in Python on top of the per-function output if needed.

---

Original request below.

# Request (original)

**Origin.** UAssetAPI ships **100 Kismet bytecode opcode classes**
(`EX_LocalVariable`, `EX_CallMath`, `EX_StructConst`, `EX_DynamicCast`,
`EX_VirtualFunction`, `EX_Vector3fConst`, `EX_RotationConst`,
`EX_TransformConst`, `EX_StringConst`, `EX_Let`, `EX_Jump`,
`EX_JumpIfNot`, …). The full UE 5.7 opcode set. SystemBridge today
works exclusively at the graph level (`UEdGraphNode`); we have zero
introspection of the compiled bytecode the BP actually executes.

This is a niche but unique capability — Kismet bytecode reading is the
ONLY place where "the graph compiled to this thing" is observable, and
that's the only diagnostic when a graph that "looks right" still
behaves wrong at runtime.

## What

A read-only disassembler for one BP function's compiled bytecode.

## Tools

| Tool | Returns |
|---|---|
| `bp_function_bytecode(bp_path, function_name)` | Linear list of `{offset, opcode, operands_text}` entries. |
| `bp_function_bytecode_summary(bp_path, function_name)` | Count of each opcode kind; function references; local variables. |

`function_name`: as it appears in the BP's My Blueprint panel.

Output format for `bp_function_bytecode`:

```jsonc
{
  "function": "OnReload",
  "params": ["WeaponMontage:AnimMontage"],
  "locals": ["i:Int32"],
  "bytecode": [
    { "offset": 0,  "opcode": "EX_Let",          "comment": "Local0 := Param0" },
    { "offset": 5,  "opcode": "EX_LocalVariable","comment": "Local0" },
    { "offset": 12, "opcode": "EX_CallMath",     "comment": "AnimSequenceLength" },
    { "offset": 24, "opcode": "EX_FinalFunction","comment": "PlayMontage" },
    …
  ]
}
```

## Why

Concrete blockers this would unblock:

1. **BP compiles to wrong bytecode** — graph looks correct, but the
   actual behavior differs. Currently invisible.
2. **Migration validation** — after an engine version bump, does the
   bytecode for `BeginPlay` look the same? Without bytecode access,
   you can only test runtime behavior.
3. **Self-context bug from blueprint-authoring docs** — the existing
   gotcha ("Variable node uses an invalid target. May depend on a
   pruned node") would have a deterministic diagnostic: read the
   compiled bytecode for the offending function and look for
   `EX_LocalOutVariable` with no `EX_Self` push.
4. **Pruned-node detection** — UE prunes nodes not on the exec chain
   silently. Bytecode shows what was actually compiled in.

## Implementation route

- Companion C++. `UFunction::Script` is a `TArray<uint8>` of the
  compiled bytecode; the opcode enum is `EExprToken` (defined in
  `Script.h`). A walker that reads the opcode + its operand layout
  (driven by switch over `EExprToken`) produces the disassembly.
- Module deps already linked (`CoreUObject`).
- No mutation surface in v1. Could come later for hot-patching, but
  that's a different request.

## Acceptance

- For a known-good BP function with a simple `If` + `PlayMontage`, the
  output lists the expected EX_* sequence (`EX_LocalVariable` →
  `EX_JumpIfNot` → `EX_FinalFunction(PlayMontage)`).
- For the self-context bug example, the output shows `EX_LocalOutVariable`
  without a preceding `EX_Self` — diagnostically actionable.
- Offsets match what the BP debugger shows when stepping through.

## Non-goals (defer)

- Mutation / patching of bytecode.
- Symbol-level decompilation back to graph nodes.
- Coverage analysis (which bytecode ran during a PIE session).
