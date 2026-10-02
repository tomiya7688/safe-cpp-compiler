# JSON Policy 仕様

> 日本語版が正本。全体方針は [SPEC.md](SPEC.md)。

## 1. 設定ファイル

既定名は `safe-cpp.json`。project root から読み込む。`--config <path>` で上書き可能。

MVP1 は **1 project = 1 effective config** を基本とし、自動 merge は行わない。

## 2. トップレベル

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

未知 key は既定 configuration error。未対応 version も compile 前に error。

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

MVP1 はこの安全側の値のみサポートしてよい。

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

stable rule ID を指定し、その policy diagnostic を無効化する。

ただし parse/type error、backend 生成不能、compiler internal error、target 必須条件は無効化しない。

semantic rule を ignore しても、compiler が定義済み safe semantics を持つ場合はその semantics を維持する。未実装 semantics を ignore だけで通してはならない。

### 4.2 ignore.files

project root 基準 glob。config 上の path separator は `/`。

一致 file は legacy/trusted boundary とし完全な safety guarantee 対象外。

header が一致する場合、その header 由来 AST node の policy enforcement を外せるが、Safe code との ABI/type boundary は可能な限り検査する。

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

`statements` 初期候補: goto, break, continue, return。

`jump` group は少なくとも break, continue, return, goto。

feature identifier 例:

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

default-deny では未指定でも危険 feature は禁止される。

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

基本 action:

- trap
- compile_error
- wrap（定義可能な整数演算のみ）

指定 action を安全に実装できない場合、黙って弱めず configuration/compile error。

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

MVP safety profile では意味を弱める値への変更を許可しなくてよい。

## 8. target

```json
{
  "target": {
    "triple": "x86_64-unknown-linux-gnu"
  }
}
```

target に属する事項: integer/pointer width、alignment、ABI、endianness、calling convention。

## 9. 優先順位

1. parser/type system 必須エラー
2. hard semantic invariant
3. ignore.files
4. ignore.rules
5. explicit forbid
6. default-deny
7. semantics action
8. optimizer policy

ignore は policy exception であり、未実装命令を生成する escape hatch ではない。

## 10. 完全例

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
    "files": ["third_party/**"]
  },
  "forbid": {
    "statements": ["goto"],
    "statement_groups": [],
    "features": ["inline_assembly"]
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
  },
  "target": {
    "triple": "x86_64-unknown-linux-gnu"
  }
}
```
