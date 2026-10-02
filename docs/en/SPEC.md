# Safe C++ Compiler Specification

> **English translation**
>
> The Japanese documents under `docs/jp/*.md` are authoritative.

## 1. Purpose

Safe C++ Compiler accepts ordinary C and C++ while detecting or defining dangerous operations and undefined behavior, then emits LLVM IR that avoids dangerous optimization assumptions.

The project favors **letting generally safe code compile normally and reporting only genuinely unsafe or suspicious operations**, rather than maintaining a huge blanket deny-list.

## 2. Language baselines

Current stable standards:

- C++: **ISO/IEC 14882:2024** (commonly C++23)
- C: **ISO/IEC 9899:2024** (commonly C23)

Unpublished draft standards such as C++26 are not the default language baseline.

C23 and C++23 may coexist in the same project.

## 3. Principles

1. Accept ordinary standard C/C++ when its meaning can be preserved safely.
2. Use `unsafety error` when safety cannot be guaranteed.
3. Use `unsafety warning` when lowering remains defined but the construct deserves caution.
4. Do not pass C/C++ UB through as LLVM optimization freedom.
5. Convert UB-prone operations to a defined result, runtime trap, or unsafety error.
6. Preserve ordering of traps and observable side effects.
7. Only LLVM IR that passes the **Safe LLVM Validator** is an official MVP1 output.
8. MVP2 performs only optimizations whose equivalence is proven.

## 4. Diagnostics

### unsafety error

Stops compilation because safe semantics cannot be guaranteed.

```text
unsafety error: use of uninitialized value
  --> src/main.c:18:9
```

### unsafety warning

Compilation may continue because defined LLVM lowering is possible, but the operation is generally risky or requires intent confirmation.

```text
unsafety warning: raw pointer crosses an external FFI boundary
  --> src/legacy.cpp:42:5
```

Warnings never permit UB to remain in emitted LLVM IR.

Stable internal rule keys exist for JSON ignore settings but need not appear in normal human-readable diagnostics.

## 5. Generally safe code

Ordinary standard constructs are accepted when the compiler can preserve their meaning, including functions, variables, ordinary control flow, structs/classes/enums, RAII, templates, constexpr, concepts, references, checked arithmetic, standard containers/strings, and ordinary C functions/structs/arrays.

C or C++ features are not rejected merely because of the language they originate from.

## 6. Representative unsafety errors

- uninitialized reads
- unpreventable null dereference
- out-of-bounds access whose safety cannot be defined
- use-after-lifetime
- invalid/double free or allocator mismatch
- unavoidable data race
- invalid alignment
- invalid function-pointer call or ABI mismatch
- inline assembly in MVP
- unverifiable intrinsics/extensions
- unsafe object-representation manipulation
- LLVM IR rejected by Safe LLVM Validator

## 7. Representative warnings

- raw pointers crossing external FFI boundaries
- narrowing conversions
- C-style casts whose exact defined meaning can still be preserved
- legacy `void*` APIs
- bounds-losing array decay immediately entering a checked wrapper
- format APIs with limited static validation
- ignored safety checks/files

## 8. UB handling

| Operation | Safe C/C++ behavior |
| --- | --- |
| signed overflow | runtime trap |
| unsigned overflow | standard wrap |
| division by zero | runtime trap |
| signed MIN / -1 | runtime trap |
| invalid shift | runtime trap |
| null dereference | static error or runtime trap |
| out of bounds | static error or runtime trap |
| invalid alignment | static error or runtime trap |
| uninitialized read | unsafety error |
| use-after-lifetime | unsafety error or runtime trap |
| unsupported UB source | unsafety error |

## 9. Mixed C/C++

- `.c`: C23
- `.cpp` / `.cc` / `.cxx`: C++23
- headers follow the including language context

Type, bounds, ownership, and lifetime information should be preserved across C/C++ boundaries where possible.

## 10. C APIs

C APIs are not banned by name alone.

If safety conditions can be proven, the call is accepted. If defined lowering is possible but safety confidence is limited, emit a warning. If safety cannot be guaranteed, emit an error.

Examples include checking memcpy size/overlap/object representation, validating static printf formats, validating scanf destinations, and proving destination bounds for strcpy-like calls.

## 11. FFI

Libraries built outside Safe C++ Compiler control are FFI boundaries.

FFI itself is allowed, but unknown pointer/buffer/lifetime/ownership contracts may produce warnings or errors. Wrappers or configuration can provide contracts.

## 12. JSON policy

Default file: `safe-cpp.json`.

It configures ignore rules/files, diagnostic severity behavior, target selection, and LLVM validation. See [CONFIG.md](CONFIG.md).

## 13. LLVM pipeline

```text
C23 / C++23
    |
    v
Clang frontend / AST
    |
    v
Safety Analyzer + Rewriter
    |
    v
LLVM IR
    |
    +--> LLVM structural verifier
    |
    +--> Safe LLVM Validator
    |
    v
Validated LLVM IR   <-- MVP1 output
```

Machine-code generation is not required for MVP1 completion.

## 14. Safe LLVM Validator

At minimum:

- LLVM module verification succeeds
- undef/poison are not used as safe values
- unproven nsw/nuw are rejected
- unproven inbounds GEP is rejected
- fast-math flags are rejected
- UB-based unreachable is rejected
- unproven llvm.assume is rejected
- unproven nonnull/dereferenceable/noalias/alignment claims are rejected
- required trap/check lowering follows canonical validated patterns
- runtime/helper declarations and ABI are checked

The validator verifies the compiler-generated safe LLVM subset; it is not a full proof system for arbitrary third-party LLVM IR.

## 15. MVP2

MVP2 retains source semantics and applies only transformations proving:

```text
SafeMeaning(before) == SafeMeaning(after)
```

## 16. Documents

- [SPEC.md](SPEC.md)
- [CONFIG.md](CONFIG.md)
- [RULES.md](RULES.md)
- [IR.md](IR.md)
- [MVP.md](MVP.md)

## 17. License

MIT License. The repository-root `LICENSE` is authoritative.
