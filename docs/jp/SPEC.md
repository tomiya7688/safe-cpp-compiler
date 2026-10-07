# Safe C++ Compiler 仕様

> **正本 (Authoritative Specification)**
>
> 日本語版 `docs/jp/*.md` を正本とする。英語版は翻訳であり、差異がある場合は日本語版を優先する。

## 1. 目的

Safe C++ Compiler は、通常の C/C++ をできるだけそのまま使いながら、危険な操作と未定義動作を検出・定義し、危険な最適化前提を含まない LLVM IR を生成することを目的とする。

設計は **「危険な機能を大量に禁止する」より「一般的に安全なコードを普通に通し、危険な箇所だけ unsafety として扱う」** 方針を取る。

## 2. 基準言語規格

現時点の安定版規格を基準とする。

- C++: **ISO/IEC 14882:2024**（一般に C++23）
- C: **ISO/IEC 9899:2024**（一般に C23）

C++26 など未発行の draft standard は既定言語モードにしない。
将来新しい ISO International Standard が正式発行された場合、次の compiler language-version update で基準を更新する。

同一 project 内で C23 と C++23 を混在してよい。

## 3. 基本原則

1. 一般的に安全で、意味を定義できる標準 C/C++ は普通に受理する。
2. 危険性が高い、または安全な意味を維持できない操作は `unsafety error` とする。
3. 意味は定義できるが注意が必要な操作は `unsafety warning` とする。
4. C/C++ の UB を LLVM optimization の自由として流さない。
5. UB になり得る操作は defined result / runtime trap / unsafety error のいずれかへ変換する。
6. trap と observable side effect の順序を勝手に変更しない。
7. LLVM IR を出力する前後で検査し、**Safe LLVM Validator を通過した IR だけを MVP1 の正式出力**とする。
8. 証明できない最適化は MVP2 で行わない。

## 4. 診断モデル

### 4.1 unsafety error

安全な意味を保証できない、または明確に危険な操作。

compile を停止する。

例:

```text
unsafety error: use of uninitialized value
  --> src/main.c:18:9
```

### 4.2 unsafety warning

compile は可能で、生成される Safe LLVM IR の意味も定義できるが、一般的に危険・脆弱・意図確認が必要な操作。

例:

```text
unsafety warning: raw pointer crosses an external FFI boundary
  --> src/legacy.cpp:42:5
```

warning があるからといって UB を LLVM に残してよいわけではない。

### 4.3 rule key

JSON の `ignore.rules` 用に内部的な stable rule key を持つ。
通常の表示は `unsafety error` / `unsafety warning` を中心とし、rule key は verbose/JSON diagnostics で表示できる。

## 5. 一般的に安全として受理するもの

標準規格に適合し、Safe C++ Compiler が意味を保持できる限り、次のような通常コードは原則受理する。

- function / variable / namespace
- if / switch / for / while / do
- break / continue / return
- struct / class / enum
- constructor / destructor / RAII
- template / constexpr / concepts
- references
- standard arithmetic（危険条件は check）
- automatic/static storage
- standard containers / strings など、既知の通常 API
- exceptions など標準機能（実装が安全に lower できる場合）
- C の通常の function / struct / enum / array

「C++ の機能だから危険」「C の機能だから危険」と一括禁止しない。

## 6. unsafety error の代表例

次は安全性を保証できない限り error。

- 未初期化 read
- null dereference を回避できないコード
- bounds を定義できない out-of-bounds access
- use-after-lifetime
- double free / invalid free
- allocator mismatch
- data race を避けられない操作
- 不正 alignment
- invalid function pointer call
- 不正 ABI/calling convention
- inline assembly（MVPでは解析不能）
- compiler intrinsic / extension で意味を検証できないもの
- 未検証の object representation 書き換え
- Safe LLVM Validator が拒否する LLVM IR

## 7. warning の代表例

安全な lowering はできるが注意を促したいもの。

- raw pointer の外部 FFI 受け渡し
- narrowing conversion
- C-style cast のうち安全に意味を確定できるもの
- `void*` を使う legacy API
- bounds 情報を失うが直後に安全 wrapper へ入る array decay
- format string API で static validation が限定的なケース
- ignore.rules / ignore.files による検査除外

warning を error に上げる strict mode は将来提供できる。

## 8. UB の扱い

代表既定:

| 操作 | Safe C/C++ |
| --- | --- |
| signed overflow | runtime trap |
| unsigned overflow | standard wrap |
| division by zero | runtime trap |
| signed MIN / -1 | runtime trap |
| invalid shift | runtime trap |
| null dereference | static error または runtime trap |
| out of bounds | static error または runtime trap |
| invalid alignment | static error または runtime trap |
| uninitialized read | unsafety error |
| use-after-lifetime | unsafety error または runtime trap |
| unsupported UB source | unsafety error |

## 9. C / C++ 混在

- `.c`: C23
- `.cpp` / `.cc` / `.cxx`: C++23
- header: include 元の言語文脈

同じ project で両方を解析し、可能な限り型、bounds、ownership、lifetime 情報を C/C++ 境界でも保持する。

## 10. C API

C API は名前だけで全面禁止しない。

compiler が安全条件を検証できる場合は通す。
検証できない危険 API は unsafety error、部分的にしか検証できないが defined lowering が可能なら warning。

例:

- `memcpy`: size / overlap / object representation を確認
- `printf` family: format が static なら型整合を検証
- `scanf` family: destination size を確認できなければ warning/error
- `strcpy` 等: destination bounds を証明できなければ error

## 11. FFI

Safe C++ Compiler 管理外の library は FFI boundary とする。

FFI 自体は許可するが、pointer/buffer/lifetime/ownership が不明な境界は warning または error。
可能なら wrapper annotation / config で契約を与える。

## 12. JSON policy

既定: `safe-cpp.json`。

- `ignore.rules`
- `ignore.files`
- warning/error severity override
- target
- LLVM validation policy

詳細は [CONFIG.md](CONFIG.md)。

## 13. LLVM IR pipeline

MVP1 の正式 pipeline:

```text
C23 / C++23
    |
    v
Clang frontend / AST
    |
    v
Safety Analyzer + Rewriter
    |
    v
LLVM IR
    |
    +--> LLVM structural verifier
    |
    +--> Safe LLVM Validator
    |
    v
Validated LLVM IR   <-- MVP1 output
```

MVP1 の完成条件は machine code 生成ではなく、**検証済み LLVM IR を生成できること**。

## 14. Safe LLVM Validator

Validator は arbitrary LLVM IR の完全な形式証明器ではなく、Safe C++ Compiler が生成する LLVM subset の独立検査層。

最低限:

- LLVM module verifier に合格
- `undef` / `poison` を安全値として利用していない
- 未証明 `nsw` / `nuw` を拒否
- 未証明 `inbounds` を拒否
- fast-math flags を拒否
- UB を根拠にした `unreachable` を拒否
- 未証明 `llvm.assume` を拒否
- 未証明 nonnull / dereferenceable / noalias / alignment 属性を拒否
- trap/check が必要な lowering が規定 pattern を満たすことを検査
- Safe C++ Compiler runtime/helper の宣言と ABI を検査

MVP1 は proof metadata に頼りすぎず、危険な属性を基本的に生成しない方針から始める。

## 15. MVP2

MVP2 は validated LLVM IR を作るだけでなく、元 C/C++ AST / semantic information を保持した optimizer を含む。

最適化は

```text
SafeMeaning(before) == SafeMeaning(after)
```

を証明できる場合だけ行う。

## 16. 文書

- [SPEC.md](SPEC.md)
- [CONFIG.md](CONFIG.md)
- [BUILD.md](BUILD.md)
- [RULES.md](RULES.md)
- [IR.md](IR.md)
- [MVP.md](MVP.md)

## 17. ライセンス

MIT License。repository root の `LICENSE` を正とする。
