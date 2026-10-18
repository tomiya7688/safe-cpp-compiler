# 危険ライブラリ / API カタログ仕様

> **正本 (Authoritative Specification)**
>
> この文書は Safe C++ Compiler の library/API safety catalog の日本語正本である。
> `docs/en/LIBRARIES.md` は翻訳版。

## 1. 目的

Safe C++ Compiler は、標準ライブラリを含む C/C++ ライブラリの API について、
既知の危険性を catalog として保持し、使用時に `unsafety warning` または `unsafety error` を出せる。

重要なのは **library/header 単位で一括禁止しない**こと。

例:

- `std::vector` 自体は通常利用可能
- `std::vector::operator[]` は bounds を証明できなければ check/trap 対象
- `std::vector::at()` は標準で checked access なので通常 warning 不要

同じ library 内でも API ごとに危険度が違うため、symbol/API 単位で判定する。

## 2. 危険度

catalog entry は次の severity を持てる。

### safe

特別な警告なし。

### checked

危険条件があるが compiler が静的証明または runtime check を挿入できる。

通常は warning を出さず、必要な check を入れる。

### warning

意味は定義できるが、誤用しやすい・ownership/lifetime/bounds を失いやすい。

```text
unsafety warning: std::unique_ptr::release() transfers ownership to a raw pointer
```

### error

安全条件を保証できず、そのまま lowering すると memory safety を維持できない。

```text
unsafety error: destination bounds for std::strcpy cannot be proven
```

## 3. 標準ライブラリ初期カタログ

### 3.1 unchecked index/access

次は index/state が証明できなければ `checked` とする。

- `std::vector::operator[]`
- `std::array::operator[]`
- `std::basic_string::operator[]`
- `std::basic_string_view::operator[]`
- `std::span::operator[]`
- `std::optional::operator*`
- `std::optional::operator->`
- `std::expected::operator*` / `operator->`（利用可能な規格/実装で）

compiler は bounds/engaged state を証明できなければ runtime trap/check へ lower してよい。

checked accessor が存在する場合は replacement hint を出せる。

例:

```text
note: consider vector::at() if exception-based checked access is acceptable
```

### 3.2 raw memory / byte operations

`<cstring>` / C library 由来:

- `std::memcpy`
- `std::memmove`
- `std::memset`
- `std::memcmp`
- `std::strlen`
- `std::strcpy`
- `std::strcat`
- `std::strncpy`
- `std::strncat`

判定:

- size / lifetime / object representation / overlap が証明できる → safe/checked
- defined lowering は可能だが情報不足 → warning
- bounds/lifetime を保証できない → error

特に `strcpy` / `strcat` 系は destination bounds を証明できない場合 error。

### 3.3 formatted I/O

`<cstdio>` / `<stdio.h>`:

- `printf` family
- `scanf` family
- `sprintf` / `vsprintf`
- `snprintf` / `vsnprintf`

format string が compile-time constant の場合、型整合を検査する。

`sprintf` の destination bounds を保証できない場合 error。

`snprintf` は capacity が検証できれば通常 warning 不要。

dynamic format で型・buffer safety を十分検証できない場合 warning/error。

### 3.4 ownership transfer

`<memory>`:

- `std::unique_ptr::release()` → warning
- raw pointer から `std::shared_ptr` を独立して複数生成する可能性 → warning/error
- custom deleter と allocation origin が整合しない → error
- `std::shared_ptr::get()` / `std::unique_ptr::get()` で得た raw pointer が owner より長く生存する可能性 → warning/error

単に smart pointer を使うこと自体は safe 扱い。

### 3.5 lifetime view

non-owning view:

- `std::string_view`
- `std::span`
- iterator / pointer range
- raw pointer returned by `.data()`

view の lifetime が owner を超える可能性がある場合 warning/error。

temporary object から長寿命 view を作るなど、dangling が静的に分かる場合 error。

### 3.6 manual lifetime / object model

- placement new
- `std::launder`
- allocator の raw storage API
- `std::construct_at` / `std::destroy_at`
- uninitialized memory algorithms

これらは通常コードより高度な lifetime 管理を必要とする。

compiler が lifetime transition を追跡できる → checked/warning。

追跡できない → error。

### 3.7 raw allocation

`<cstdlib>` / C APIs:

- `malloc`
- `calloc`
- `realloc`
- `free`

および direct `new/delete`。

利用自体を常に禁止するのではなく、ownership/lifetime を analyzer が追跡する。

追跡可能 → checked。

ownership leak/double free/use-after-free の可能性を排除できない → warning/error。

### 3.8 concurrency

- `std::thread`
- `std::jthread`
- mutex/condition variable
- atomics
- shared mutable state

thread library 自体を危険 library として禁止しない。

data race の可能性を analyzer が排除できない場合 warning/error。
明確な data race は error。

### 3.9 callable / ABI hazards

- function pointer cast を経由した call
- C callback API
- variadic function
- incompatible calling convention

library call の解決先 signature を保証できない場合 error。

## 4. 標準ライブラリ以外

third-party library も catalog に追加できる。

例:

```json
{
  "library_catalogs": [
    "safe-library-rules.json"
  ]
}
```

project local catalog は標準 catalog を追加・強化できる。

標準 safety rule を project catalog で弱めることは既定では禁止する。
弱める場合は既存の `ignore.rules` と同様に明示的な safety exception として扱う。

## 5. catalog entry の概念形式

実装内部または外部 catalog は概念的に次の情報を持つ。

```json
{
  "symbol": "std::unique_ptr::release",
  "category": "ownership_transfer",
  "severity": "warning",
  "message": "ownership is transferred to a raw pointer",
  "conditions": [
    "returned_pointer_must_be_owned_or_freed"
  ]
}
```

条件付き rule も許可する。

```json
{
  "symbol": "std::memcpy",
  "category": "raw_memory",
  "severity": "checked",
  "conditions": [
    "destination_size >= count",
    "source_size >= count",
    "object_representation_compatible"
  ]
}
```

## 6. symbol resolution

単純な文字列検索ではなく、frontend の resolved declaration / canonical symbol を使用する。

以下を区別する。

- namespace
- class / member
- overload
- template specialization
- C linkage symbol
- macro wrapper 経由の call

wrapper function の中に危険 API がある場合、その wrapper 自体へ summary を伝播できるようにする。

## 7. library summary

関数ごとに safety summary を生成・利用できる。

例:

```text
reads: buffer[0..n)
writes: out[0..n)
requires: out_size >= n
ownership: none
lifetime: no_escape
may_trap: false
```

MVPでは built-in catalog だけでもよい。
将来は project function から summary を自動生成し、別 translation unit でも利用する。

## 8. warning の抑制

`safe-cpp.json`:

```json
{
  "ignore": {
    "rules": [
      "library.unique_ptr_release"
    ]
  }
}
```

または library catalog/category 単位の severity override を将来追加できる。

warning を ignore しても Safe LLVM Validator は無効化されない。

## 9. update policy

標準ライブラリ catalog は C23/C++23 の基準規格と compiler version に紐づけて管理する。

新しい standard/API 対応時は catalog も更新する。

third-party catalog は library version を条件にできる設計が望ましい。

## 10. 初期実装範囲

MVP ではまず以下を実装する。

1. resolved symbol ベースの library checker
2. built-in standard/C library catalog
3. safe / checked / warning / error の4分類
4. `<cstring>` / `<cstdio>` 系
5. unchecked container/string/span access
6. smart pointer ownership escape
7. string_view/span lifetime escape
8. raw allocation/free
9. diagnostics
10. project local catalog の読み込み

ライブラリ全体の高度な interprocedural proof は後回しでよい。
