# MVP1 / MVP2 Specification

> English translation. [docs/jp/MVP.md](../jp/MVP.md) is authoritative.

## 1. MVP1

> **A fast path that checks C/C++ policy/safety and emits conservative LLVM IR without dangerous unproven assumptions.**

```text
C/C++ source
   -> Clang frontend / AST
   -> Policy Checker
   -> Safety Analyzer / Rewriter
   -> Conservative LLVM IR
   -> LLVM backend
   -> Machine code
```

### Required functionality

- mixed C/C++ project support
- safe-cpp.json
- stable rule IDs
- ignore.rules / ignore.files
- AST-based checker
- resolved C API checks
- default-deny
- initial UB trap/error semantics
- conservative LLVM IR
- LLVM backend integration
- optimization-level differential tests

### Initial semantic set

- signed overflow -> trap
- unsigned overflow -> wrap
- division by zero -> trap
- signed MIN/-1 -> trap
- null dereference -> trap
- bounds violation -> trap
- invalid shift -> trap
- uninitialized read -> compile error where detectable
- unsupported unsafe construct -> compile error

Detected unsupported lifetime/data-race cases must not silently fall back to ordinary UB.

### Forbidden LLVM assumptions without proof

nsw, nuw, inbounds, fast-math, UB-based unreachable, unproven llvm.assume, nonnull, dereferenceable, noalias, and alignment claims.

### Definition of Done

1. stable-rule-ID diagnostics
2. rule-level ignore.rules
3. file-level ignore.files
4. supported UB becomes trap/error
5. conservative LLVM IR output
6. machine-code generation
7. same Safe observable behavior across tested O0/O1/O2/O3-style levels
8. dangerous LLVM flag/attribute tests
9. C/C++ cross-call integration tests

### May be deferred

- custom parser
- custom backend
- advanced lifetime/alias proof
- vectorizer
- exceptions
- full concurrency model
- full standard-library safety model
- Safe IR serialization

## 2. MVP2

> **A source-aware optimizer that knows the original C/C++ meaning and emits machine code while refusing optimizations whose safety cannot be proven.**

```text
C/C++ source
   +-----------------------------+
   |                             |
   v                             v
AST / Semantic Graph          Safe IR
   |                             |
   +-------------+---------------+
                 |
                 v
       Source-aware Optimizer
                 |
                 v
           Low-level IR
                 |
                 v
            Machine code
```

LLVM or a custom backend may be used for final code generation, but semantic control of optimization remains in Safe C++ Compiler.

### Optimizer rule

```text
SafeMeaning(before) == SafeMeaning(after)
```

If equivalence cannot be proven, the optimization is not performed.

Initial candidates: constant folding, copy propagation, branch elimination, safe DCE, bounds/null-check elimination, inlining, and simple loop optimization.

## 3. Difference

| Item | MVP1 | MVP2 |
| --- | --- | --- |
| Primary goal | fast compilation | correctness-first |
| Frontend | may use Clang | Clang or custom |
| Optimizer | conservative LLVM usage | source-aware custom optimizer |
| Proof | minimal | central |
| Safe IR | logical semantics may suffice | central representation |
| Machine code | LLVM backend | LLVM or custom |
| Unproven optimization | do not pass dangerous assumptions | do not perform |
| Compile speed | target faster | may be slower |

"MVP1 is faster" mainly refers to compilation speed.

## 4. Tests

MVP1: positive, forbidden syntax, C API, UB trap, ignore, mixed C/C++, LLVM IR patterns, optimization-level differential tests.

MVP2: all MVP1 tests plus optimizer equivalence, proof-failure skip, metadata preservation, randomized/differential tests, and regression corpus.

## 5. Issue workflow

Once a feature is specified sufficiently for implementation, create a GitHub Issue with goal, normative spec link, scope, non-scope, acceptance criteria, tests, and unresolved questions.
