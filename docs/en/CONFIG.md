# JSON Policy Specification

> English translation. The Japanese version is authoritative.
>
> This file is for safety policy only. Build/project configuration lives in [BUILD.md](BUILD.md) and `safe-build.json`.

## 1. Base configuration

Default file: project-root `safe-cpp.json`.

```json
{
  "version": 1,
  "language": {
    "c": "c23",
    "cpp": "c++23"
  },
  "ignore": {
    "rules": [],
    "files": []
  },
  "library_catalogs": [],
  "diagnostics": {
    "warnings_as_errors": false,
    "show_rule_keys": false
  },
  "llvm_validation": {
    "enabled": true,
    "reject_unsafe_flags": true
  },
  "target": {}
}
```

## 2. Language

MVP1 baselines:

- C23 / ISO/IEC 9899:2024
- C++23 / ISO/IEC 14882:2024

## 3. Ignore

ignore.rules uses stable rule keys. ignore.files uses project-root-relative globs.

Ignore applies only to source-level safety diagnostics and cannot bypass LLVM validation.

## 4. library_catalogs

Additional project-local library/API safety catalogs may be loaded by project-root-relative path.

Built-in standard/C library catalogs remain active; project catalogs add or strengthen rules.

## 5. Diagnostics

`warnings_as_errors` promotes unsafety warnings to unsafety errors.

`show_rule_keys` includes internal rule keys in human-readable diagnostics.

Future per-rule severity overrides may be supported, but hard semantic invariants and LLVM validation errors cannot be downgraded.

## 6. LLVM validation

For MVP1, LLVM validation is mandatory and includes the LLVM verifier plus Safe LLVM Validator.

## 7. Target

Target triple and ABI settings are configurable independently from language-standard selection.
