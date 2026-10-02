# Rule Catalog

> English translation. [docs/jp/RULES.md](../jp/RULES.md) is authoritative.

## 1. Rule ID requirements

Rule IDs use lowercase snake_case, are stable, are never reused for a different meaning, and always appear in diagnostics.

## 2. Policy rules

| Rule ID | Default | Meaning |
| --- | --- | --- |
| `no_goto` | error | forbid goto |
| `no_inline_asm` | error | forbid inline assembly |
| `no_reinterpret_cast` | error | forbid reinterpret_cast |
| `no_const_cast` | error | forbid const_cast |
| `no_c_style_cast` | error | forbid C-style casts |
| `no_pointer_integer_cast` | error | forbid arbitrary pointer/integer casts |
| `no_raw_pointer_arithmetic` | error | forbid raw pointer arithmetic |
| `no_raw_memory` | error | forbid raw allocation/free in user code |
| `no_manual_lifetime` | error | forbid placement new / explicit lifetime manipulation |
| `no_union_punning` | error | forbid union punning/inactive member access |
| `no_c_varargs` | error | forbid C varargs/va_list |
| `no_setjmp_longjmp` | error | forbid setjmp/longjmp |
| `no_exceptions` | error | forbid exceptions in MVP |
| `no_unchecked_concurrency` | error | forbid unverified concurrency |
| `no_unlisted_intrinsic` | error | forbid non-allowlisted builtins |
| `no_vendor_extension` | error | forbid non-allowlisted extensions |
| `no_unsafe_c_api` | error | forbid dangerous C APIs |
| `no_unchecked_format_io` | error | forbid unchecked format I/O |
| `no_unsafe_void_pointer` | error | forbid unsafe void* type/ownership loss |
| `no_unsafe_array_decay` | error | forbid bounds-losing array decay |
| `no_unchecked_raw_bytes` | error | forbid unchecked raw-byte operations |

## 3. Semantic rules

| Rule ID | Default | Meaning |
| --- | --- | --- |
| `signed_overflow` | trap | signed overflow |
| `division_by_zero` | trap | integer division by zero |
| `signed_div_overflow` | trap | MIN / -1 |
| `null_dereference` | trap | null dereference |
| `out_of_bounds` | trap | bounds violation |
| `invalid_shift` | trap | invalid shift |
| `invalid_alignment` | trap | alignment violation |
| `invalid_pointer_arithmetic` | trap/error | pointer-range violation |
| `uninitialized_read` | error | read of uninitialized value |
| `use_after_lifetime` | trap/error | access after lifetime ends |
| `data_race` | error | unproven data-race safety |
| `unsupported_unsafe_construct` | error | no safe semantics defined |

## 4. C APIs

Initial checked/banned symbols include strcpy, strcat, sprintf, vsprintf, unchecked scanf-family use, and unverified printf-family formats.

memcpy/memmove/memset may become allowed when size, overlap, object representation, and lifetime are proven safe.

## 5. Diagnostics

```text
<severity>[<rule-id>] <file>:<line>:<column>: <message>
```

Examples:

```text
error[no_raw_memory] src/a.c:10:12: direct malloc/free is outside the default profile
trap[out_of_bounds] src/a.cpp:18:14
```

Runtime metadata may be reduced in release builds, but the trap itself must remain.

## 6. Ignore semantics

ignore.rules may suppress policy diagnostics.

For semantic rules, existing safe semantics remain in effect. Ignore alone does not make an unimplemented semantic operation compilable.

ignore.files creates a legacy/trusted boundary.

## 7. Compatibility

New rules may be added. Existing rule meanings should not change without a policy-version compatibility change.
