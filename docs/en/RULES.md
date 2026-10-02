# Unsafety Rules

> English translation. The Japanese version is authoritative.

## 1. Diagnostic levels

### unsafety error

Stops compilation because safe semantics cannot be guaranteed.

### unsafety warning

Defined LLVM lowering remains possible, but the operation is risky or deserves intent confirmation.

Normal diagnostics emphasize these categories:

```text
unsafety error: uninitialized read
unsafety warning: raw pointer crosses FFI boundary
```

## 2. Rule keys

Stable internal keys exist for JSON ignore configuration, but human diagnostics do not need to display them by default.

Representative keys:

| key | default | meaning |
| --- | --- | --- |
| `uninitialized_read` | error | uninitialized read |
| `null_dereference` | error/trap | null access |
| `out_of_bounds` | error/trap | bounds violation |
| `use_after_lifetime` | error/trap | access after lifetime |
| `invalid_free` | error | invalid/double free |
| `data_race` | error | unprovably safe race |
| `inline_asm` | error | unanalyzable assembly in MVP |
| `unchecked_intrinsic` | error | unverified intrinsic |
| `ffi_raw_pointer` | warning | raw pointer across FFI |
| `narrowing_conversion` | warning | narrowing conversion |
| `legacy_void_pointer` | warning/error | legacy void* API |
| `unchecked_format` | warning/error | limited format validation |
| `ignored_safety_check` | warning | ignore applied |

## 3. Runtime traps

UB that can be checked dynamically may become a defined runtime trap: signed overflow, division by zero, signed division overflow, invalid shift, null dereference, out of bounds, and invalid alignment.

## 4. C APIs

Do not ban APIs by name alone. Prove their safety conditions where possible, warn when defined lowering is possible but confidence is limited, and error when safety cannot be guaranteed.

## 5. Ignore

ignore.rules and ignore.files affect source-level safety diagnostics.

Safe LLVM Validator is never bypassed by ignore settings.
