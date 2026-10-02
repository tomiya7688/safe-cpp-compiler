# Rule Catalog

> 日本語版が正本。rule ID は diagnostics と `ignore.rules` の安定識別子。

## 1. Rule ID

- lowercase snake_case
- 意味を変えない限り rename しない
- 別の意味に再利用しない
- 削除後も ID を再利用しない
- diagnostic に必ず表示

## 2. Policy rules

| Rule ID | 既定 | 概要 |
| --- | --- | --- |
| `no_goto` | error | goto 禁止 |
| `no_inline_asm` | error | inline asm 禁止 |
| `no_reinterpret_cast` | error | reinterpret_cast 禁止 |
| `no_const_cast` | error | const_cast 禁止 |
| `no_c_style_cast` | error | C-style cast 禁止 |
| `no_pointer_integer_cast` | error | 任意 pointer/integer cast 禁止 |
| `no_raw_pointer_arithmetic` | error | raw pointer arithmetic 禁止 |
| `no_raw_memory` | error | user code の raw allocation/free 禁止 |
| `no_manual_lifetime` | error | placement new/explicit destructor 等 |
| `no_union_punning` | error | union punning/inactive member |
| `no_c_varargs` | error | C varargs/va_list |
| `no_setjmp_longjmp` | error | setjmp/longjmp |
| `no_exceptions` | error | MVP exception 禁止 |
| `no_unchecked_concurrency` | error | 未検証並行処理 |
| `no_unlisted_intrinsic` | error | allowlist 外 builtin |
| `no_vendor_extension` | error | allowlist 外 extension |
| `no_unsafe_c_api` | error | 危険 C API |
| `no_unchecked_format_io` | error | 未検証 format I/O |
| `no_unsafe_void_pointer` | error | 型/ownership を失う void* |
| `no_unsafe_array_decay` | error | bounds を失う array decay |
| `no_unchecked_raw_bytes` | error | 未検証 raw byte 操作 |

## 3. Semantic rules

| Rule ID | 既定 | 概要 |
| --- | --- | --- |
| `signed_overflow` | trap | signed overflow |
| `division_by_zero` | trap | 0 除算 |
| `signed_div_overflow` | trap | MIN / -1 |
| `null_dereference` | trap | null dereference |
| `out_of_bounds` | trap | bounds violation |
| `invalid_shift` | trap | invalid shift |
| `invalid_alignment` | trap | alignment violation |
| `invalid_pointer_arithmetic` | trap/error | pointer range violation |
| `uninitialized_read` | error | 未初期化 read |
| `use_after_lifetime` | trap/error | lifetime 終了後 access |
| `data_race` | error | 安全性未証明 data race |
| `unsupported_unsafe_construct` | error | safe semantics 未定義 |

## 4. C API

初期 banned/checked symbol set は実装と test で version 管理する。

少なくとも strcpy, strcat, sprintf, vsprintf、未検証 scanf family、未検証 format の printf family を対象候補とする。

memcpy/memmove/memset は名前だけで永久禁止せず、size/overlap/object representation/lifetime が証明できる場合は将来許可可能。

## 5. Diagnostics

基本形式:

```text
<severity>[<rule-id>] <file>:<line>:<column>: <message>
```

例:

```text
error[no_raw_memory] src/a.c:10:12: direct malloc/free is outside the default profile
trap[out_of_bounds] src/a.cpp:18:14
```

release mode で runtime metadata を縮小しても trap 自体を削除してはならない。

## 6. ignore

ignore.rules は policy diagnostic を抑制できる。

semantic rule は、定義済み safe semantics があるなら semantics を維持する。未実装 semantics を ignore だけで通さない。

ignore.files は file 全体を legacy/trusted boundary とする。

## 7. 追加・互換性

新 rule は追加可能。default-deny により未知危険 feature は unsupported_unsafe_construct で拒否できる。

既存 rule の意味変更は compatibility に影響するため policy version 変更を伴う。
