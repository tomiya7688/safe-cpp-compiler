# Safe C++ Compiler 仕様

> **正本 (Authoritative Specification)**
>
> 日本語版 `docs/jp/*.md` を正本とする。`docs/en/*.md` は翻訳であり、差異がある場合は日本語版を優先する。

## 1. 目的

Safe C++ Compiler は C と C++ を入力として受け取り、未定義動作やコンパイラ固有の危険な最適化によって、ソースから予測しにくい意味変更が発生することをできるだけ防ぐコンパイラである。

中心原則:

1. **証明できない最適化は行わない。**
2. **未定義動作を最適化の自由として利用しない。**
3. **未定義動作になり得る操作には defined result / defined trap / compile error のいずれかを与える。**
4. **trap も observable behavior として扱い、勝手に削除・移動しない。**
5. **危険な C/C++ 機能は default-deny とする。**
6. **backend や最適化レベルが変わっても Safe C/C++ の意味を変えない。**
7. **MVP1 は高速な安全化変換、MVP2 は正しさ優先の source-aware optimizer とする。**

## 2. 非目標

初期段階では次を目標にしない。

- C/C++ 規格の全機能を完全実装すること
- GCC/Clang と完全に同じ最適化結果を出すこと
- 既存コードをすべて無変更で受理すること
- コンパイラ実装そのものにバグが絶対存在しないことを数学的に保証すること
- MVP1 で最高性能の機械語を生成すること

安全な意味をまだ定義できない機能は compile error としてよい。

## 3. 用語

### 3.1 Safe code

Policy Checker と Safe C/C++ 意味論の対象となり、完全な安全保証範囲に入るコード。

### 3.2 Ignored code

`ignore.files` または `ignore.rules` により検査の一部または全部を外したコード。除外範囲について完全な安全保証は行わない。

### 3.3 Trusted boundary

compiler runtime、検証済み標準ライブラリ、外部 FFI wrapper など、低レベル操作を閉じ込める境界。通常のユーザーコードに一般的な `unsafe` escape hatch は設けない。

### 3.4 Defined trap

危険条件を実行時に検出し、規定された trap reason を伴って停止する動作。trap は未定義動作ではない。

### 3.5 Observable behavior

I/O、明示的な volatile/atomic 操作、外部呼び出し、副作用、trap とその順序など、Safe C/C++ の意味として保存すべき動作。

## 4. 入力言語とプロジェクト

同一プロジェクト内の C と C++ を両方受理する。

- `.c` — C frontend
- `.cpp`, `.cc`, `.cxx` — C++ frontend
- `.h` — include 元の言語文脈または明示設定
- その他 — build/config で言語指定

```text
C source --------> C frontend -----+
                                   |
                                   +--> Policy Checker
                                   |        |
C++ source ----> C++ frontend -----+        v
                                      Safety Analyzer
                                           |
                                           v
                                   Safe semantics / Safe IR
```

同一プロジェクト内で Safe C++ Compiler がコンパイルする C コードは外部コード扱いしない。

## 5. Safe C/C++ の意味論

対応するすべての操作を次のいずれかへ分類する。

1. **Defined result** — 結果を明示的に定義する。
2. **Defined trap** — 実行時エラーとして定義する。
3. **Compile error** — 安全な意味を定義できない、または policy で禁止する。

Safe code 内に「何が起きてもよい」という第四の状態を残さない。

### 5.1 主要既定値

| 操作 | 既定 |
| --- | --- |
| signed integer overflow | trap |
| unsigned integer overflow | wrap |
| division by zero | trap |
| signed MIN / -1 | trap |
| null dereference | trap |
| array/span out of bounds | trap |
| invalid shift | trap |
| invalid alignment | trap |
| invalid pointer arithmetic | trap または compile error |
| uninitialized read | compile error |
| use-after-lifetime | trap または compile error |
| data race | compile error |
| unsupported unsafe operation | compile error |

詳細は [RULES.md](RULES.md)。

## 6. default-deny

安全意味論が定義され、実装が対応している機能だけを許可する。

代表的な既定禁止:

- inline assembly
- `reinterpret_cast` / `const_cast` / C-style cast
- 任意 pointer ↔ integer cast
- raw pointer arithmetic
- raw `new/delete`
- `malloc/calloc/realloc/free` の直接利用
- placement new / manual lifetime
- union type punning / inactive member access
- C varargs / `va_list`
- `setjmp/longjmp`
- `goto`
- MVP 標準プロファイルでの exceptions
- 未検証 shared mutable concurrency
- allowlist 外 intrinsic / builtin / vendor extension
- 危険な C memory/string API
- 未検証 format I/O
- 型・所有権を失う `void*`
- bounds を失う array-to-pointer decay
- 任意 raw byte reinterpretation

## 7. メモリ・所有権・寿命

Safe code では所有権を raw pointer と手動解放で表現しないことを既定とする。container、owner type、checked reference、span/view を利用する。

raw pointer dereference は、null、lifetime、alignment、型、bounds の必要条件を満たすことを証明するか runtime check を行う。

bounds check は安全を証明できた場合のみ削除できる。

use-after-lifetime を backend の UB として渡してはならない。静的に分かれば compile error、必要なら runtime trap を使用する。

## 8. 整数・浮動小数点・変換

signed overflow は既定 trap。unsigned overflow は modulo wrap。

負の shift count や型幅以上の shift は trap または compile error。

情報損失、object model 破壊、ownership/lifetime 消失を伴う cast は既定禁止。

浮動小数点は MVP では通常の target/IEEE 意味を保ち、fast-math、reassociation、NaN/Inf 不在仮定を既定禁止する。

## 9. 制御フロー

通常の `if`、loop、call/return は利用可能。

`goto` は既定禁止。`break`、`continue`、`return` は JSON で個別または `jump` group として禁止可能。

MVP 標準プロファイルでは `throw/try/catch` を禁止する。

## 10. C 由来の危険操作

C コードにも同じ safety policy を適用する。

代表例:

- `strcpy`, `strcat`, `sprintf`
- 未検証 `scanf`
- 動的・未検証 format string の `printf`
- 任意 `memcpy/memmove/memset`
- 型情報を失う `void*`
- 長さ不明 `char*`
- pointer arithmetic ベース iterator
- raw allocation/free
- varargs
- setjmp/longjmp
- union type punning
- 未初期化 aggregate の部分利用

## 11. 外部ライブラリと FFI

Safe C++ Compiler 管理外でビルドされた C/C++ library は trusted/FFI boundary とする。

raw ABI を Safe code へ直接露出する代わりに、検証済み wrapper を既定とする。wrapper は可能な範囲で型、nullability、buffer length、ownership、lifetime、thread-safety、error semantics を定義する。

`extern "C"` だけでは安全とはみなさない。

## 12. プリプロセッサ・マクロ

MVP は既存 frontend の preprocessing を利用してよい。

policy 判定は文字列検索ではなく preprocess/parse 後の AST と resolved call を中心に行う。診断には可能なら macro expansion location と spelling location を含める。

allowlist 外 pragma/attribute/builtin/vendor extension は禁止する。

## 13. 標準ライブラリ/API

ライブラリ名だけで安全扱いしない。API ごとに safety model を定義し、必要なら checked wrapper を用意する。

## 14. JSON policy

既定設定ファイルは project root の `safe-cpp.json`。`--config <path>` で上書き可能。

正式仕様は [CONFIG.md](CONFIG.md)。

- `ignore.rules` — rule 単位の除外
- `ignore.files` — glob 単位の除外
- `forbid` — 追加禁止
- `semantics` — trap/wrap/compile_error
- `optimizer` — optimizer safety constraint
- `target` — ABI/architecture

ignore 範囲は完全な Safe C/C++ 保証外。

## 15. 診断

安定した rule ID を必須とする。

```text
error[no_goto] src/main.cpp:18:5: goto statement is forbidden
note: ignored rules/files are outside the full safety guarantee
```

原則として severity、rule ID、file、line/column、message、必要な note/fix hint を含める。

## 16. Safe IR

Safe IR は backend に UB の自由を渡さないための中間意味論。

Invariant:

- undef/poison 相当を作らない
- UB 前提の unreachable を作らない
- trap を明示する
- checked arithmetic/memory access を表現する
- source/type/lifetime/bounds 情報を必要に応じて保持する

詳細は [IR.md](IR.md)。

MVP1 は Safe IR を完全 materialize せず AST から conservative LLVM IR へ直接 lower してもよいが、同じ invariant を守る。

## 17. LLVM lowering

証明なしに以下を付与・生成してはならない。

- `nsw` / `nuw`
- `getelementptr inbounds`
- UB を根拠にした `unreachable`
- 未証明 `llvm.assume`
- fast-math flags
- 未証明 nonnull/dereferenceable/alignment/noalias/provenance attributes

trap は optimization 後も意味が保存される形に lower する。

## 18. optimizer invariant

最適化は次を証明できる場合のみ許可する。

```text
SafeMeaning(before) == SafeMeaning(after)
```

「元の C/C++ では UB だから経路は存在しない」という証明は禁止。

Dead Code Elimination は結果未使用、副作用なし、trap 不可能をすべて証明した場合のみ許可。

speculative load、reorder、check elimination も trap/side-effect の順序を壊してはならない。

## 19. Portability

同じ source、`safe-cpp.json`、compiler version、target profile に対し、同じ Safe observable behavior を与えることを目標とする。

ABI、pointer width、endianness 等は明示 target profile に属する。

## 20. MVP

正式範囲は [MVP.md](MVP.md)。

**MVP1:** C/C++ → policy/safety check → conservative LLVM IR → LLVM backend。コンパイル速度優先。

**MVP2:** source-aware optimizer が元コードの意味を保持し、安全性を証明できない最適化を行わず machine code を生成。正しさ優先。

## 21. Conformance

最低限以下を test する。

- optimization level を変えても defined observable behavior が一致
- trap が消えない
- forbidden operation が正しい rule ID で拒否
- ignore.rules/files が指定範囲だけに作用
- C/C++ 混在 project
- LLVM IR に未証明 dangerous flags/attributes がない
- external FFI boundary が明示される

## 22. 文書一覧

- [SPEC.md](SPEC.md) — 全体仕様
- [CONFIG.md](CONFIG.md) — JSON policy
- [RULES.md](RULES.md) — rule catalog
- [IR.md](IR.md) — Safe IR / LLVM lowering
- [MVP.md](MVP.md) — MVP1/MVP2

## 23. ライセンス

MIT License。repository root の `LICENSE` を正とする。
