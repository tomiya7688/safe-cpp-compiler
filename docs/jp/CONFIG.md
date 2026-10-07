# JSON Policy 仕様

> 日本語版が正本。
>
> このファイルは安全性 policy 専用。build/project 構成は [BUILD.md](BUILD.md) と `safe-build.json` に分離する。

## 1. 基本

既定 file は project root の `safe-cpp.json`。

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

## 2. language

MVP1 baseline:

- C: `c23` / ISO/IEC 9899:2024
- C++: `c++23` / ISO/IEC 14882:2024

## 3. ignore

`ignore.rules` は stable rule key。
`ignore.files` は project root 相対 glob。

ignore は source-level safety diagnostics 用。
LLVM validator の拒否を無効化する用途には使わない。

## 4. diagnostics

`warnings_as_errors`: unsafety warning を unsafety error として扱う。

`show_rule_keys`: human diagnostics に rule key を表示。

通常:

```text
unsafety warning: raw pointer crosses FFI boundary
```

詳細表示:

```text
unsafety warning[ffi_raw_pointer]: raw pointer crosses FFI boundary
```

## 5. severity override

将来:

```json
{
  "diagnostics": {
    "severity": {
      "narrowing_conversion": "error",
      "ffi_raw_pointer": "warning"
    }
  }
}
```

ただし semantic safety invariant や LLVM Validator error を warning へ下げてはならない。

## 6. llvm_validation

MVP1 では `enabled: true` が必須。

最低限、LLVM structural verification と Safe LLVM Validator を実行する。

## 7. target

target triple、ABI などを指定可能。
言語規格の基準とは分離する。
