# Unsafety Rules

> 日本語版が正本。

## 1. 診断レベル

Safe C++ Compiler の安全診断は原則として次の2種類。

### unsafety error

安全な意味を保証できないため compile を停止する。

### unsafety warning

Safe LLVM IR への defined lowering は可能だが、一般的に危険または意図確認が必要。compile は継続できる。

通常表示は category を中心にする。

```text
unsafety error: uninitialized read
unsafety warning: raw pointer crosses FFI boundary
```

## 2. Rule key

JSON ignore 用に stable key を持つが、標準 human diagnostics で key 表示は必須ではない。

代表 key:

| key | default | 内容 |
| --- | --- | --- |
| `uninitialized_read` | error | 未初期化 read |
| `null_dereference` | error/trap | null access |
| `out_of_bounds` | error/trap | bounds violation |
| `use_after_lifetime` | error/trap | lifetime 終了後 access |
| `invalid_free` | error | invalid/double free |
| `data_race` | error | safety を保証できない race |
| `inline_asm` | error | MVP で解析できない asm |
| `unchecked_intrinsic` | error | 未検証 intrinsic |
| `ffi_raw_pointer` | warning | raw pointer FFI |
| `narrowing_conversion` | warning | narrowing |
| `legacy_void_pointer` | warning/error | void* legacy API |
| `unchecked_format` | warning/error | format verification 不十分 |
| `ignored_safety_check` | warning | ignore 適用 |

## 3. runtime trap

runtime check が可能な UB は error ではなく trap に変換できる。

- signed_overflow
- division_by_zero
- signed_div_overflow
- invalid_shift
- null_dereference
- out_of_bounds
- invalid_alignment

trap は defined behavior。

## 4. C API

API 名だけで禁止しない。引数・buffer length・format・object representation を解析して severity を決める。

安全条件を証明できる → 通す。

defined lowering は可能だが注意が必要 → unsafety warning。

安全条件を保証できない → unsafety error。

## 5. ignore

`ignore.rules` は指定 rule の policy diagnostic を除外できる。
`ignore.files` は指定 file を safety checking の legacy boundary とする。

ignore 自体について `unsafety warning` を出せる。

ただし Safe LLVM Validator は ignore されない。危険な LLVM IR を ignore で通してはならない。
