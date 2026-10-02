# Safe C++ Compiler Specification

> **English translation**
>
> The Japanese documents under `docs/jp/*.md` are authoritative. The English documents under `docs/en/*.md` are translations. If they differ, the Japanese version takes precedence.

## 1. Purpose

Safe C++ Compiler accepts C and C++ and aims to prevent hard-to-predict semantic changes caused by undefined behavior and compiler-specific dangerous optimization.

Core principles:

1. **Do not optimize unless the transformation is proven safe.**
2. **Do not use undefined behavior as optimization freedom.**
3. **Operations that would otherwise be undefined must become a defined result, defined trap, or compile error.**
4. **A trap is observable behavior and must not be removed or moved arbitrarily.**
5. **Dangerous C/C++ features are default-deny.**
6. **Safe C/C++ meaning must not change merely because the backend or optimization level changes.**
7. **MVP1 is the fast safety-oriented path; MVP2 is the correctness-first source-aware optimizer.**

## 2. Non-goals

Initially, the project does not require:

- complete implementation of every C/C++ feature
- optimization output identical to GCC/Clang
- accepting every existing C/C++ program unchanged
- a mathematical guarantee that the compiler implementation itself can never contain a bug
- maximum machine-code performance in MVP1

A feature whose safe meaning has not yet been defined may be rejected as a compile error.

## 3. Terminology

### 3.1 Safe code

Code checked by the Policy Checker and Safe C/C++ semantics and covered by the full safety guarantee.

### 3.2 Ignored code

Code partly or fully excluded using `ignore.files` or `ignore.rules`. The excluded scope is outside the full safety guarantee.

### 3.3 Trusted boundary

A boundary containing low-level implementation code such as the compiler runtime, verified standard-library components, and external FFI wrappers. Normal user code has no general-purpose `unsafe` escape hatch.

### 3.4 Defined trap

A specified runtime failure that detects a dangerous condition and terminates execution with a defined trap reason. A trap is not undefined behavior.

### 3.5 Observable behavior

Behavior that Safe C/C++ semantics must preserve, including I/O, explicitly modeled volatile/atomic operations, external calls, side effects, traps, and their ordering.

## 4. Input languages and projects

A single project may contain both C and C++.

- `.c` — C frontend
- `.cpp`, `.cc`, `.cxx` — C++ frontend
- `.h` — language context of the including translation unit or explicit configuration
- other extensions — language selected by build/configuration

```text
C source --------> C frontend -----+
                                   |
                                   +--> Policy Checker
                                   |        |
C++ source ----> C++ frontend -----+        v
                                      Safety Analyzer
                                           |
                                           v
                                   Safe semantics / Safe IR
```

C code compiled inside the same project by Safe C++ Compiler is ordinary checked project code, not external code.

## 5. Safe C/C++ semantics

Every supported operation is classified as one of:

1. **Defined result**
2. **Defined trap**
3. **Compile error**

Safe code does not retain a fourth state where "anything may happen."

### 5.1 Main defaults

| Operation | Default |
| --- | --- |
| signed integer overflow | trap |
| unsigned integer overflow | wrap |
| division by zero | trap |
| signed MIN / -1 | trap |
| null dereference | trap |
| array/span out of bounds | trap |
| invalid shift | trap |
| invalid alignment | trap |
| invalid pointer arithmetic | trap or compile error |
| uninitialized read | compile error |
| use-after-lifetime | trap or compile error |
| data race | compile error |
| unsupported unsafe operation | compile error |

See [RULES.md](RULES.md).

## 6. Default-deny

Only features with defined safe semantics and implementation support are allowed.

Representative default-deny features:

- inline assembly
- `reinterpret_cast` / `const_cast` / C-style casts
- arbitrary pointer ↔ integer casts
- raw pointer arithmetic
- raw `new/delete`
- direct `malloc/calloc/realloc/free`
- placement new and manual lifetime manipulation
- union type punning / inactive member access
- C varargs / `va_list`
- `setjmp/longjmp`
- `goto`
- exceptions in the standard MVP profile
- unverified shared mutable concurrency
- non-allowlisted intrinsics, builtins, and vendor extensions
- dangerous C memory/string APIs
- unchecked format I/O
- `void*` use that loses type or ownership
- array-to-pointer decay that loses bounds
- arbitrary raw-byte reinterpretation

## 7. Memory, ownership, and lifetime

By default, Safe code does not represent ownership with raw pointers and manual deallocation. Containers, ownership types, checked references, and spans/views are preferred.

Raw pointer dereference requires proof or runtime checks for nullness, lifetime, alignment, type validity, and bounds as applicable.

A bounds check may be removed only after safety is proven.

Use-after-lifetime must not be passed to the backend as ordinary UB. It is a compile error when statically known, or may become a runtime trap when tracked dynamically.

## 8. Integers, floating point, and conversions

Signed overflow traps by default. Unsigned overflow wraps modulo the type width.

Invalid shifts trap or fail compilation.

Casts that lose essential information, break the object model, or lose ownership/lifetime information are denied by default.

For floating point, MVP keeps ordinary target/IEEE semantics and disables fast-math, reassociation, and unproven assumptions about NaN/Inf by default.

## 9. Control flow

Ordinary `if`, loops, calls, and returns are allowed.

`goto` is denied by default. `break`, `continue`, and `return` can be forbidden individually or via the `jump` group.

`throw/try/catch` are denied in the standard MVP profile.

## 10. Dangerous operations inherited from C

C is subject to the same safety policy.

Representative cases include:

- `strcpy`, `strcat`, `sprintf`
- unchecked `scanf`
- dynamic or unverified `printf` format strings
- arbitrary `memcpy/memmove/memset`
- `void*` that loses type information
- unknown-length `char*`
- pointer-arithmetic iterators
- raw allocation/free
- varargs
- setjmp/longjmp
- union type punning
- partially uninitialized aggregates

## 11. External libraries and FFI

C/C++ libraries built outside Safe C++ Compiler control are trusted/FFI boundaries.

Verified wrappers should be used instead of directly exposing raw ABI calls to Safe code. Wrappers should define type, nullability, buffer lengths, ownership, lifetime, thread-safety, and error semantics where possible.

`extern "C"` alone does not make an interface safe.

## 12. Preprocessor and macros

MVP may use the existing frontend preprocessor.

Policy checks are based primarily on the post-preprocessing AST and resolved calls, not simple source-text search. Diagnostics should include macro expansion/spelling locations when practical.

Non-allowlisted pragmas, attributes, builtins, and vendor extensions are denied.

## 13. Standard library and APIs

An API is not automatically considered safe merely because it belongs to a standard library. A safety model or checked wrapper may be required.

## 14. JSON policy

The default configuration file is `safe-cpp.json` at the project root. `--config <path>` overrides it.

The normative configuration specification is [CONFIG.md](CONFIG.md).

- `ignore.rules` — rule-level exclusion
- `ignore.files` — file glob exclusion
- `forbid` — extra prohibitions
- `semantics` — trap/wrap/compile_error choices
- `optimizer` — optimization safety constraints
- `target` — ABI/architecture

Ignored scope is outside the full Safe C/C++ guarantee.

## 15. Diagnostics

Stable rule IDs are mandatory.

```text
error[no_goto] src/main.cpp:18:5: goto statement is forbidden
note: ignored rules/files are outside the full safety guarantee
```

Diagnostics should include severity, rule ID, file, line/column, message, and notes/fix hints where useful.

## 16. Safe IR

Safe IR is an intermediate semantic layer that prevents the backend from gaining UB-based freedom.

Invariants:

- no undef/poison-like semantics
- no UB-based unreachable
- explicit traps
- checked arithmetic and memory access
- source/type/lifetime/bounds data retained as needed

See [IR.md](IR.md).

MVP1 may lower directly from AST to conservative LLVM IR without fully materializing Safe IR, provided equivalent invariants are preserved.

## 17. LLVM lowering

Without proof, the compiler must not emit or attach:

- `nsw` / `nuw`
- `getelementptr inbounds`
- UB-based `unreachable`
- unproven `llvm.assume`
- fast-math flags
- unproven nonnull/dereferenceable/alignment/noalias/provenance attributes

Traps must lower in a form whose semantics survive optimization.

## 18. Optimizer invariant

An optimization is allowed only after proving:

```text
SafeMeaning(before) == SafeMeaning(after)
```

Proofs based on "this path is UB in ordinary C/C++" are forbidden.

Dead-code elimination requires proof that the value is unused, there are no observable side effects, and no trap is possible.

Speculative loads, reordering, and check elimination must preserve trap and side-effect ordering.

## 19. Portability

For the same source, `safe-cpp.json`, compiler version, and target profile, the goal is the same Safe observable behavior.

ABI, pointer width, endianness, and similar properties belong to an explicit target profile.

## 20. MVP

See [MVP.md](MVP.md).

**MVP1:** C/C++ → policy/safety checks → conservative LLVM IR → LLVM backend. Compilation speed first.

**MVP2:** source-aware optimizer preserving original program meaning and refusing unproven transformations. Correctness first.

## 21. Conformance

Minimum testing includes:

- same defined observable behavior across optimization levels
- traps are not removed
- forbidden operations report correct rule IDs
- ignore.rules/files affect only the requested scope
- mixed C/C++ projects
- no unproven dangerous LLVM flags/attributes
- external FFI boundaries are explicit

## 22. Documents

- [SPEC.md](SPEC.md) — overall specification
- [CONFIG.md](CONFIG.md) — JSON policy
- [RULES.md](RULES.md) — rule catalog
- [IR.md](IR.md) — Safe IR / LLVM lowering
- [MVP.md](MVP.md) — MVP1/MVP2

## 23. License

MIT License. The repository-root `LICENSE` is authoritative.
