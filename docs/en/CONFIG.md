# JSON Policy Specification

> English translation. [docs/jp/CONFIG.md](../jp/CONFIG.md) is authoritative.

## 1. Configuration file

Default file: `safe-cpp.json` in the project root. `--config <path>` overrides it.

MVP1 uses one effective configuration per project and does not require automatic configuration merging.

## 2. Top level

```json
{
  "language": "SafeCpp",
  "version": 1,
  "safety": {},
  "ignore": {},
  "forbid": {},
  "semantics": {},
  "optimizer": {},
  "target": {}
}
```

Unknown keys and unsupported versions are configuration errors by default.

## 3. safety

```json
{
  "safety": {
    "feature_policy": "default_deny",
    "trusted_code": "runtime_only",
    "unsafe_escape_hatch": false
  }
}
```

MVP1 may support only these safety-preserving values.

## 4. ignore

```json
{
  "ignore": {
    "rules": ["no_goto", "no_raw_memory"],
    "files": ["third_party/**", "generated/**"]
  }
}
```

### 4.1 ignore.rules

Uses stable rule IDs and disables the corresponding policy diagnostic.

It does not disable parser/type errors, constructs the backend cannot generate, compiler internal errors, or mandatory target constraints.

If safe semantics are already defined for a semantic rule, ignoring the diagnostic does not remove those semantics.

### 4.2 ignore.files

Globs are relative to the project root and use `/` in configuration syntax.

Matched files are legacy/trusted boundaries and are outside the full safety guarantee.

## 5. forbid

```json
{
  "forbid": {
    "statements": ["goto"],
    "statement_groups": ["jump"],
    "features": ["inline_assembly"]
  }
}
```

Initial statement names: goto, break, continue, return.

The `jump` group contains at least break, continue, return, and goto.

Representative feature identifiers:

- inline_assembly
- raw_memory_ownership
- raw_pointer_arithmetic
- reinterpret_cast
- const_cast
- c_style_cast
- placement_new
- manual_lifetime
- c_varargs
- setjmp_longjmp
- exceptions
- unchecked_concurrency
- compiler_intrinsics
- vendor_extensions
- unsafe_c_apis
- unchecked_format_io
- unsafe_void_pointer
- unsafe_array_decay
- unchecked_raw_byte_operations

## 6. semantics

```json
{
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
  }
}
```

Base actions are `trap`, `compile_error`, and `wrap` where explicitly defined.

Unsupported requested semantics must fail rather than silently weakening safety.

## 7. optimizer

```json
{
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

MVP safety profiles need not accept settings that weaken these guarantees.

## 8. target

```json
{
  "target": {
    "triple": "x86_64-unknown-linux-gnu"
  }
}
```

Target-dependent properties include integer/pointer width, alignment, ABI, endianness, and calling convention.

## 9. Precedence

1. required parser/type errors
2. hard semantic invariants
3. ignore.files
4. ignore.rules
5. explicit forbid
6. default-deny
7. semantics action
8. optimizer policy

Ignore is a policy exception, not a magic escape hatch for unsupported code.

## 10. Complete example

The canonical complete example is maintained in the authoritative Japanese [CONFIG.md](../jp/CONFIG.md); the key structure and semantics above are normative for the English translation.
