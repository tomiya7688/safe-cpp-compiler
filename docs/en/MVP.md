# MVP1 / MVP2 Specification

> English translation. The Japanese version is authoritative.

## 1. MVP1

Goal:

> **Parse the latest stable C/C++ baselines, classify dangerous operations as errors/warnings/traps, and emit validated LLVM IR.**

Baselines:

- C23 / ISO/IEC 9899:2024
- C++23 / ISO/IEC 14882:2024

Pipeline:

```text
C23/C++23
 -> frontend/AST
 -> Safety Analyzer/Rewriter
 -> LLVM IR
 -> LLVM Verifier
 -> Safe LLVM Validator
 -> Validated LLVM IR
```

### Definition of Done

1. mixed C23/C++23 project support
2. ordinary safe code is not over-rejected
3. unsafety error / unsafety warning diagnostics
4. major UB classes become traps/errors
5. ignore.rules / ignore.files
6. LLVM IR generation
7. LLVM Verifier passes
8. Safe LLVM Validator passes
9. validator detects dangerous flags/attributes
10. validated LLVM IR can be written as .ll or .bc

Machine-code generation is not mandatory for MVP1; backend smoke tests are allowed.

## 2. MVP2

MVP2 adds a source-aware optimizer and machine-code generation while retaining all MVP1 safety checks.

Only transformations proving

```text
SafeMeaning(before) == SafeMeaning(after)
```

are allowed.

## 3. Difference

| Item | MVP1 | MVP2 |
| --- | --- | --- |
| Language baseline | C23/C++23 | same starting baseline |
| Diagnostics | unsafety error/warning | same |
| Output | validated LLVM IR | machine code |
| Optimizer | minimal/conservative | source-aware |
| LLVM validator | required | required/equivalent validation |
| Compile speed | priority | correctness priority |

## 4. Implementation issue

MVP1 is tracked in GitHub Issue #1.
