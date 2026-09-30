# Safe C++ Compiler Design Specification

> **English translation**
>
> The authoritative specification for Safe C++ Compiler is `docs/jp/SPEC.md`.
> This file is an English translation. If there is any discrepancy, the Japanese specification takes precedence.

## 1. Purpose

Safe C++ Compiler aims to separate C++ programs as much as possible from compiler-specific dangerous optimizations
and to prevent the same input from unexpectedly changing meaning due to compiler or optimization-level differences.

In particular, Undefined Behavior (UB) must not be treated as optimization freedom.
Every operation supported by the Safe C++ semantics must have an explicit meaning.

The core design principles are:

1. **Do not perform an optimization unless it can be proven safe.**
2. **Do not use undefined behavior as an optimization assumption.**
3. **Treat runtime errors (traps) as part of observable behavior and preserve them.**
4. **Do not require programmers to know compiler-specific optimization rules to write correct code.**
5. **Remove safety checks only when their redundancy has been proven.**
6. **Use default-deny for dangerous language features and low-level operations; explicitly allow only features with defined safety semantics.**

## 2. No Undefined Behavior in Safe C++

Every supported construct and operation in Safe C++ is classified into one of these categories:

1. **Defined result** — the result is explicitly defined by the language.
2. **Defined trap** — the operation is defined to terminate with an explicit runtime error.
3. **Compile error** — compilation is rejected because a safe meaning is unavailable or policy forbids the operation.

There is no fourth state where "anything may happen."

If the implementation does not yet provide safe semantics for a category that would be UB in ordinary C++,
it must be rejected as a compile error instead of silently falling back to traditional C++ UB.

### 2.1 Representative behavior

| Operation | Default Safe C++ policy |
| --- | --- |
| signed integer overflow | trap |
| unsigned integer overflow | wrap |
| division by zero | trap |
| `INT_MIN / -1` | trap |
| null dereference | trap |
| array out of bounds | trap |
| invalid shift | trap |
| invalid pointer arithmetic | trap or compile error |
| uninitialized read | compile error |
| use-after-lifetime | trap or compile error |
| data race | compile error, except supported synchronization proven safe |
| invalid alignment | trap |
| unsupported unsafe construct | compile error |

## 3. JSON Policy

Project-specific permissions, prohibitions, and safety semantics can be configured with JSON.
The JSON file is not merely a set of optimization flags; it defines the allowed Safe C++ subset.

Example:

```json
{
  "language": "SafeCpp",
  "version": 1,
  "safety": {
    "feature_policy": "default_deny",
    "trusted_code": "runtime_only",
    "unsafe_escape_hatch": false
  },
  "ignore": {
    "rules": [],
    "files": []
  },
  "forbid": {
    "statements": ["goto"],
    "statement_groups": [],
    "features": [
      "inline_assembly",
      "raw_memory_ownership",
      "raw_pointer_arithmetic",
      "reinterpret_cast",
      "const_cast",
      "c_style_cast",
      "placement_new",
      "manual_lifetime",
      "c_varargs",
      "setjmp_longjmp",
      "exceptions",
      "unchecked_concurrency",
      "compiler_intrinsics",
      "vendor_extensions",
      "unsafe_c_apis",
      "unchecked_format_io",
      "unsafe_void_pointer",
      "unsafe_array_decay",
      "unchecked_raw_byte_operations"
    ]
  },
  "semantics": {
    "signed_overflow": "trap",
    "unsigned_overflow": "wrap",
    "division_by_zero": "trap",
    "null_dereference": "trap",
    "out_of_bounds": "trap",
    "invalid_shift": "trap",
    "invalid_pointer_arithmetic": "trap",
    "uninitialized_read": "compile_error",
    "use_after_lifetime": "trap",
    "data_race": "compile_error"
  },
  "optimizer": {
    "require_semantic_proof": true,
    "preserve_traps": true,
    "allow_ub_assumptions": false,
    "allow_speculative_load": false,
    "allow_fast_math": false,
    "signed_overflow_assumption": false,
    "strict_aliasing_assumption": false
  }
}
```

### 3.1 Forbidden statements

Individual statements can be listed in `forbid.statements`.

Example:

```json
{
  "forbid": {
    "statements": ["goto", "continue"]
  }
}
```

Statement groups can also be forbidden:

```json
{
  "forbid": {
    "statement_groups": ["jump"]
  }
}
```

The `jump` group includes at least:

- `break`
- `continue`
- `return`
- `goto`

Prohibitions are checked on the parsed AST, not with simple string matching.
This makes the policy apply to the resulting syntax even when macros are involved.

### 3.2 ignore

`ignore` explicitly selects checks or files that are excluded from Safe C++ policy enforcement.

```json
{
  "ignore": {
    "rules": [
      "no_goto",
      "no_raw_memory"
    ],
    "files": [
      "third_party/**",
      "generated/**"
    ]
  }
}
```

- `ignore.rules` — disables checks for the specified rule IDs.
- `ignore.files` — excludes files matching the specified glob patterns from the Safe C++ policy checker.

Rule IDs are stable identifiers published by the compiler.
Diagnostics must always include the rule ID so that the same identifier can be used directly in JSON.

Example:

```text
error[no_goto]: goto statement is forbidden by Safe C++ policy
```

Files matched by `ignore.files` are outside the full Safe C++ safety guarantee and are treated as a trusted/legacy boundary.
This allows third-party libraries, generated code, and code being migrated gradually to coexist with Safe C++.

`ignore` is always an explicit exception and must never be applied implicitly.

### 3.3 Default-deny for dangerous features

The standard Safe C++ safety profile is **default-deny**.
A feature is not usable merely because it exists in C++; only features for which Safe C++ explicitly defines semantics and safety conditions are allowed.

Unknown syntax, unsupported implementation extensions, and low-level operations whose semantics have not been made safe are compile errors.
The standard MVP1/MVP2 safety profiles do not provide a general user-code `unsafe` escape hatch that disables safety rules.
Low-level operations required by the implementation are restricted to a **trusted code boundary** containing the compiler, runtime, and verified standard-library components.

The default forbidden set includes at least:

| Category | Default behavior |
| --- | --- |
| inline assembly | compile error |
| `reinterpret_cast` | compile error |
| `const_cast` | compile error |
| C-style cast | compile error; safe conversions use explicit safe casts |
| arbitrary pointer ↔ integer conversion | compile error |
| casts between unrelated pointer types | compile error |
| raw pointer arithmetic | compile error; use checked pointers/indexes with provable bounds |
| unverified raw pointer dereference | compile error or lower to checked access |
| raw `new` / `delete` in user code | compile error |
| direct `malloc` / `calloc` / `realloc` / `free` | compile error |
| placement `new` | compile error |
| explicit destructor calls and manual lifetime manipulation | compile error |
| low-level lifetime operations such as `std::launder` | compile error |
| union-based type punning | compile error |
| access to an inactive union member | compile error |
| C varargs (`...`, `va_list`) | compile error |
| `setjmp` / `longjmp` | compile error |
| `goto` | compile error |
| exceptions (`throw` / `try` / `catch`) | compile error in the standard MVP safety profile |
| unverified threads/shared mutable state | compile error |
| concurrency where data races cannot be excluded | compile error |
| compiler intrinsics / builtins | compile error unless allowlisted |
| vendor-specific attributes / pragmas / extensions | compile error unless allowlisted |
| unchecked memory/string APIs | compile error or checked wrappers only |
| invalid function-pointer casts | compile error |
| calling-convention casts that violate the ABI | compile error |
| code depending on undefined or unspecified results | compile error unless semantics are explicitly defined |

"Forbidden" does not mean a feature can never be supported.
A feature may be added to the allowlist in the future after Safe C++ defines safe semantics, required runtime checks, and optimizer preservation rules for it.

### 3.4 Raw-memory principle

By default, Safe C++ user code does not represent memory ownership with raw pointers and manual deallocation.

```cpp
int* p = new int[100];
delete[] p;
```

Code of this form is rejected by the standard safety profile.
Users should instead use containers, ownership types, and checked references/views whose ownership and lifetime can be tracked.

The compiler/runtime may internally allocate memory when required, but such implementation code belongs to the trusted code boundary and is separated from ordinary user code.
This is intended to remove forgotten `free` / `delete`, double-free, use-after-free, and allocator mismatch from normal user code by construction.

### 3.5 Library APIs are also subject to safety policy

Dangerous operations can be hidden behind APIs rather than syntax.
Therefore, the Policy Checker validates resolved function and method calls in addition to AST syntax.

Unverified C memory/string APIs, dangerous system APIs, compiler builtins, and similar interfaces are rejected by default.
Only checked wrappers or APIs explicitly allowlisted with Safe C++ semantics are accepted.

### 3.6 Mixed C / C++ projects

Safe C++ Compiler **accepts both C and C++ within the same project**.
C and C++ files living in the same repository, build, or library are treated as a normal use case.

The frontend is selected from file extensions and build configuration by default:

- `.c` — C frontend
- `.cpp`, `.cc`, `.cxx` — C++ frontend
- `.h` — follows the including translation unit's language context or explicit configuration
- other extensions — language must be specified by JSON/build configuration

The C and C++ frontends may have different syntax and type rules, but should share the Policy Checker, Safety Analyzer, and Safe IR as much as practical.

```text
C source --------> C frontend -----+
                                   |
                                   +--> Policy Checker
                                   |        |
C++ source ----> C++ frontend -----+        v
                                      Safety Analyzer
                                           |
                                           v
                                         Safe IR
```

The same default-deny safety policy applies to both C and C++ code.
Safety rules and UB handling must not be weakened merely because a translation unit is written in C.

### 3.7 Dangerous operations inherited from C

C++ has extensive compatibility with C and can use many C-derived constructs and library APIs.
Safe C++ does not exempt them merely because they originate in C; the same safety policy applies to C-derived and C++-specific features.

C translation units are subject to the same default-deny policy.
Importing C-compatible code must not automatically weaken the safety guarantee.

At minimum, the following are considered dangerous by default:

| C-derived feature or API | Default behavior |
| --- | --- |
| unchecked string APIs such as `strcpy`, `strcat`, and `sprintf` | compile error; use checked wrappers |
| unchecked `scanf`-family input | compile error, or only a wrapper that validates formats and destination sizes |
| dynamic or unverified `printf`-family format strings | compile error, or only a type-checked formatting API |
| arbitrary raw-byte operations with `memcpy`, `memmove`, or `memset` | compile error unless size, overlap, and object-representation rules are validated |
| loss of type or ownership through `void*` | compile error; use typed checked handles/views |
| C-array decay to a raw pointer that loses bounds | compile error at API boundaries by default; use a checked span/view that retains length |
| raw C-string (`char*`) access with unknown length | compile error, or checked string/view with provable length |
| C-style iterators implemented with pointer arithmetic | compile error; use checked iterators/indexes |
| `malloc` family / `free` | compile error in user code |
| C varargs / `va_list` | compile error |
| `setjmp` / `longjmp` | compile error |
| union type punning | compile error |
| C-style casts | compile error |
| partial use of uninitialized structs or arrays | compile error |
| raw-byte reinterpretation that ignores the object's type | compile error |
| compiler-specific C extensions | compile error unless allowlisted |

Including a C header does not automatically make its APIs safe.
Even when a function uses C linkage (`extern "C"`), the Safe C++ boundary validates arguments, lengths, ownership, nullability, lifetimes, and return values.

C code compiled inside the same Safe C++ Compiler project is treated as ordinary checked project code rather than as an external library.
When both sides of a C/C++ boundary are under Safe C++ Compiler control, the compiler should preserve and validate type, size, ownership, nullability, and lifetime information as far as practical.

External C libraries built outside Safe C++ Compiler control are treated as an **FFI/trusted boundary**.
For those libraries, Safe C++ should expose verified wrappers and avoid exposing the raw C ABI directly to ordinary user code by default.

C-derived checks may be excluded through `ignore.rules` or `ignore.files`, but the excluded scope is outside the full Safe C++ safety guarantee.

## 4. Safe IR

Before lowering C++ semantics to LLVM IR or machine code, Safe C++ uses a Safe IR that itself does not contain undefined behavior.

Conceptual example:

```text
%1 = checked_sadd i32 %a, %b
%2 = checked_index %array, %index
%3 = checked_load %2
%4 = checked_div_i32 %1, %3
```

Each checked operation explicitly represents both the successful value and the failure trap.

Safe IR should not carelessly introduce concepts such as `undef`, `poison`, or UB-based `unreachable`
that grant downstream optimizers freedom to choose arbitrary results.

## 5. Optimization Principles

Optimization follows this rule:

```text
Want to transform code
        |
        v
Can equivalence under Safe C++ semantics be proven?
        |-- YES --> transformation is allowed
        `-- NO  --> preserve the original form
```

### 5.1 Potentially allowed optimizations

When safety is proven, the following may be allowed:

- constant folding
- copy propagation
- branch elimination
- dead code elimination
- bounds-check elimination
- null-check elimination
- inlining
- simple loop optimization

For Dead Code Elimination, "the result is unused" is not enough.
The compiler must also prove that the removed operation has no side effects and cannot trap.

### 5.2 Optimizations and assumptions forbidden by default

- assuming signed overflow never happens
- branch removal based on UB
- dangerous transformations based on strict aliasing
- speculative loads without a safety proof
- load/store reordering that changes trap or side-effect order
- floating-point reassociation
- fast-math
- unproven transformations based on pointer provenance
- converting code to `unreachable` because of UB

## 6. MVP1 — Fast / LLVM-oriented

The purpose of MVP1 is:

> **Convert relatively safe C++ into relatively safe LLVM input.**

Conceptual pipeline:

```text
C++ source
   |
   v
Parser / AST
   |
   v
Policy Checker + Safety Checks
   |
   v
Safe IR
   |
   v
Conservative LLVM IR
   |
   v
LLVM backend
   |
   v
Machine code
```

MVP1 prioritizes a faster compilation pipeline than MVP2.
It does not include the advanced proof-oriented optimizer planned for MVP2 and uses the LLVM backend where practical.

However, Safe C++ must not pass unproven facts to LLVM as assumptions.
For example, an addition must not receive LLVM's `nsw` marker unless the compiler has proven that signed overflow cannot occur.
The same rule applies to strong assumptions such as `inbounds`, `nuw`, and similar metadata or flags.

MVP1 is not intended to be "perfect."
Its goal is to greatly reduce optimization-related hazards compared with ordinary C++ while retaining fast compilation.

## 7. MVP2 — Correctness-first / Source-aware optimizer

The purpose of MVP2 is:

> **Compile to machine code using an optimizer that still knows the original C++ source and semantics, without allowing transformations that break safety.**

Conceptual pipeline:

```text
C++ source
   |
   +--------------------+
   |                    |
   v                    v
AST / Semantic Graph   Safe IR
   |                    |
   +---------+----------+
             |
             v
   Source-aware Safe Optimizer
             |
             v
        Low-level IR
             |
             v
   Instruction Selection
             |
             v
        Machine code
```

In MVP2, "bugs are not allowed" means at minimum that **semantic changes caused by optimization are forbidden by specification**.
If correctness of a transformation cannot be proven, that optimization is not performed.

This is distinct from a mathematical guarantee that the compiler implementation itself can never contain an implementation bug.
The central guarantee of MVP2 is that the optimization rules do not permit destruction of program meaning.

### 7.1 Preserve knowledge of the original C++

The MVP2 optimizer should retain or reference as much of the following information as possible:

- original AST
- types
- variable and object lifetimes
- array sizes
- ownership relationships
- nullability
- alignment
- original expressions
- control structures
- source locations
- semantics defined by policy

This allows the compiler to optimize the safety checks themselves through proof.

Example:

```cpp
for (int i = 0; i < 100; ++i) {
    sum += a[i];
}
```

If `a` is known to have length 100 and the loop condition proves that `0 <= i < 100`,
the bounds check for each iteration may be removed.

The reason is not that the access is "probably safe."
The check may be removed because the compiler has **proven that no trap can occur under Safe C++ semantics**.

## 8. Difference Between MVP1 and MVP2

| Item | MVP1 | MVP2 |
| --- | --- | --- |
| Primary goal | fast compilation with improved safety | correctness first |
| Output path | conservative LLVM IR | own optimization path to machine code |
| Optimizer | conservatively uses LLVM | custom optimizer aware of original C++ |
| Safety proofs | minimal | central feature |
| Unproven optimization | do not communicate dangerous assumptions to LLVM | do not perform it |
| Traps | preserve them | preserve them; remove only when proven impossible |
| Compilation speed | target faster than MVP2 | may be slower due to analysis and proofs |

"MVP1 is faster" primarily refers to the compilation pipeline.
It does not guarantee that programs produced by MVP1 always execute faster than programs produced by MVP2.

## 9. Portability Rule

For the same Safe C++ source, the same JSON policy, and the same explicit target profile,
conforming Safe C++ Compiler implementations should provide the same observable behavior.

Programmers should not need to silently account for backend- or optimizer-specific behavior.

Target-dependent properties such as integer widths, ABI, and endianness that affect semantics
must be represented as part of an explicit target profile rather than as hidden compiler differences.

## 10. Short Project Definition

**MVP1**

> A fast, conservative translator that avoids turning dangerous C++ into dangerous LLVM IR.

**MVP2**

> A compiler that preserves knowledge of the original C++ meaning, proves safety while optimizing, and produces machine code without permitting meaning-breaking optimizations.

## 11. License

This project is released under the MIT License.
The authoritative license text is the repository-root `LICENSE` file.
