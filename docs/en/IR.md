# Safe IR / LLVM Lowering Specification

> English translation. [docs/jp/IR.md](../jp/IR.md) is authoritative.

## 1. Purpose

Safe IR fixes frontend semantics in a form that does not give the backend arbitrary UB-based freedom.

## 2. Invariants

1. No language-level UB remains in Safe IR.
2. Uninitialized values are not propagated as arbitrary values.
3. Poison-like semantics are not part of the model.
4. Traps are explicit control effects.
5. Side-effect/trap ordering is preserved.
6. Bounds/null/lifetime/alignment information is retained as needed.
7. Source locations can be retained.
8. Provenance needed for optimizer proofs can be retained.

## 3. Conceptual operations

```text
%r = checked_sadd i32 %a, %b
%r = checked_sdiv i32 %a, %b
%p = checked_index %base, %index, %length
%v = checked_load %p
checked_store %p, %v
check_nonnull %p
check_alive %object
check_align %p, 4
trap out_of_bounds
```

Concrete syntax may change; semantics are normative.

## 4. Arithmetic

Signed add/sub/mul trap on overflow unless safety is proven.

Unsigned arithmetic wraps modulo its width; it must not be replaced with LLVM `nuw` semantics.

Division checks zero divisor and signed MIN/-1.

## 5. Memory

Memory access may require checks for nullness, bounds, lifetime, alignment, read/write permission, and type/representation compatibility.

Check elimination is allowed only when failure is impossible under Safe semantics.

## 6. Trap

A trap is a defined noreturn control effect.

Representative reasons include signed_overflow, division_by_zero, signed_div_overflow, null_dereference, out_of_bounds, invalid_shift, invalid_alignment, and use_after_lifetime.

Trap ordering relative to observable side effects must be preserved.

## 7. LLVM lowering

Overflow may use `llvm.*with.overflow` intrinsics or equivalent checked sequences.

Division instructions are emitted only after required safety checks.

Null/bounds checks lower to branches and traps unless proven unnecessary.

## 8. Information not sent to LLVM without proof

- nsw
- nuw
- inbounds
- nonnull
- dereferenceable
- strong alignment
- noalias/provenance claims
- llvm.assume
- fast-math
- UB-based unreachable

MVP1 may conservatively omit most of these entirely.

## 9. unreachable

Allowed after structurally noreturn operations such as a trap. Forbidden merely because ordinary C/C++ would call a path UB.

## 10. Reordering and DCE

Reordering must preserve traps, external calls, volatile/atomic semantics, and observable side effects.

DCE requires proof of unused value, no side effect, no possible trap, and no lifetime/control effect.

## 11. Check elimination

Bounds/null/lifetime checks are removed only when failure is proven impossible on the relevant execution path.

## 12. Source-aware metadata

MVP2 may retain original AST/source ranges, C/C++ types, object identity, lifetime, bounds, ownership, nullability, alignment, control regions, and originating rules/policy.

## 13. MVP1

Minimal implementation:

```text
Clang AST
  -> Policy Checker
  -> Safety Rewriter
  -> conservative LLVM IR
```

Safe IR need not be serialized/materialized as a standalone structure if equivalent semantics are preserved.

## 14. MVP2

The source semantic graph and Safe IR become central to optimization.

Only transformations proving

```text
SafeMeaning(before) == SafeMeaning(after)
```

may be applied.
