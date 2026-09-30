# Safe C++ Compiler 設計仕様

> **正本 (Authoritative Specification)**
>
> この文書は Safe C++ Compiler の設計に関する正本である。
> `docs/en/SPEC.md` はこの文書の英訳であり、内容に差異がある場合は本書を優先する。

## 1. 目的

Safe C++ Compiler は、C++ のコードをコンパイラ固有の危険な最適化からできるだけ切り離し、
同じ入力がコンパイラや最適化レベルの違いによって予期せず異なる意味になることを防ぐことを目的とする。

特に、未定義動作 (Undefined Behavior; UB) を「最適化のための自由」として利用しない。
Safe C++ Compiler の意味論では、対応対象となるすべての操作に明示的な意味を与える。

設計上の基本原則は次のとおりである。

1. **証明できない最適化は行わない。**
2. **未定義動作を最適化の前提として利用しない。**
3. **実行時エラー (trap) も observable behavior の一部として保存する。**
4. **コンパイラ固有の最適化規則を知らなければ正しく書けないコードを要求しない。**
5. **安全性を確認するための検査は、不要であることを証明できた場合にのみ削除する。**
6. **危険な言語機能・低レベル操作は default-deny とし、安全性を定義できたものだけ明示的に許可する。**

## 2. Safe C++ における「未定義動作なし」

Safe C++ では、言語として対応するすべての構文・演算を次のいずれかに分類する。

1. **Defined result** — 結果を言語仕様として定義する。
2. **Defined trap** — 実行時エラーとして定義し、明示的に停止する。
3. **Compile error** — 安全な意味を与えられない、またはポリシーで禁止された操作としてコンパイルを拒否する。

第四の状態として「何が起きてもよい」という未定義動作を残さない。

実装がまだ安全な意味を提供できない UB の種類は、暗黙に従来 C++ の UB へ落とすのではなく、
その時点では compile error とする。

### 2.1 代表的な扱い

| 操作 | Safe C++ の既定方針 |
| --- | --- |
| signed integer overflow | trap |
| unsigned integer overflow | wrap |
| division by zero | trap |
| `INT_MIN / -1` | trap |
| null dereference | trap |
| array out of bounds | trap |
| invalid shift | trap |
| invalid pointer arithmetic | trap または compile error |
| uninitialized read | compile error |
| use-after-lifetime | trap または compile error |
| data race | compile error（安全性を証明できる同期操作を除く） |
| invalid alignment | trap |
| unsupported unsafe construct | compile error |

## 3. JSON ポリシー

プロジェクトごとの許可・禁止事項と安全意味論は JSON ファイルで指定できる。
JSON は単なる最適化フラグではなく、使用可能な Safe C++ サブセットを定義するポリシーである。

例:

```json
{
  "language": "SafeCpp",
  "version": 1,
  "safety": {
    "feature_policy": "default_deny",
    "trusted_code": "runtime_only",
    "unsafe_escape_hatch": false
  },
  "forbid": {
    "statements": ["goto"],
    "statement_groups": [],
    "features": [
      "inline_assembly",
      "raw_memory_ownership",
      "raw_pointer_arithmetic",
      "reinterpret_cast",
      "const_cast",
      "c_style_cast",
      "placement_new",
      "manual_lifetime",
      "c_varargs",
      "setjmp_longjmp",
      "exceptions",
      "unchecked_concurrency",
      "compiler_intrinsics",
      "vendor_extensions"
    ]
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
  }
}
```

### 3.1 禁止文

`forbid.statements` には個別の文を指定できる。

例:

```json
{
  "forbid": {
    "statements": ["goto", "continue"]
  }
}
```

また、文のグループをまとめて禁止できる。

```json
{
  "forbid": {
    "statement_groups": ["jump"]
  }
}
```

`jump` グループは少なくとも次を対象とする。

- `break`
- `continue`
- `return`
- `goto`

禁止判定は単純な文字列検索ではなく、パース後の AST 上で行う。
そのため、マクロなどを経由した場合でも最終的な構文として判定する。

### 3.2 危険機能の default-deny

Safe C++ の標準安全プロファイルは **default-deny** とする。
つまり「C++ に存在するから使用可能」ではなく、Safe C++ が意味と安全条件を明示的に定義した機能だけを使用可能とする。

未知の構文、未対応の処理系拡張、意味論が安全に固定されていない低レベル操作は compile error とする。
MVP1/MVP2 の標準安全プロファイルには、ユーザーコードから安全規則を無効化する一般的な `unsafe` エスケープハッチを設けない。
必要な低レベル操作は compiler/runtime/検証済み標準ライブラリの **trusted code boundary** の内部に限定する。

既定で禁止する対象には少なくとも次を含める。

| 分類 | 既定の扱い |
| --- | --- |
| inline assembly | compile error |
| `reinterpret_cast` | compile error |
| `const_cast` | compile error |
| C-style cast | compile error。安全な変換は明示的な安全 cast に置き換える |
| pointer ↔ integer の任意変換 | compile error |
| 無関係な型の pointer 間 cast | compile error |
| raw pointer arithmetic | compile error。範囲を証明できる checked pointer/index を使用する |
| 未検証 raw pointer dereference | compile error または checked access へ変換 |
| ユーザーコードでの raw `new` / `delete` | compile error |
| `malloc` / `calloc` / `realloc` / `free` の直接利用 | compile error |
| placement `new` | compile error |
| 明示 destructor 呼び出し・手動 lifetime 操作 | compile error |
| `std::launder` 等の低レベル lifetime 操作 | compile error |
| union を用いた type punning | compile error |
| inactive union member access | compile error |
| C 可変長引数 (`...`, `va_list`) | compile error |
| `setjmp` / `longjmp` | compile error |
| `goto` | compile error |
| exception (`throw` / `try` / `catch`) | MVP の標準安全プロファイルでは compile error |
| 未検証の thread/shared mutable state | compile error |
| data race の可能性を除去できない並行処理 | compile error |
| compiler intrinsic / builtin | allowlist にないものは compile error |
| vendor-specific attribute / pragma / extension | allowlist にないものは compile error |
| unchecked memory/string API | compile error または checked wrapper のみ許可 |
| function pointer の不正 cast | compile error |
| ABI を破る呼び出し規約 cast | compile error |
| 未定義・未規定の結果に依存するコード | 明示的に定義できなければ compile error |

ここで「禁止」は、機能そのものを永久に排除するという意味ではない。
Safe C++ 側で安全な意味論、必要な runtime check、最適化時の保存条件を定義できた機能は、将来 allowlist に追加できる。

### 3.3 raw memory の原則

Safe C++ のユーザーコードでは、メモリ所有権を raw pointer と手動解放で表現しないことを既定とする。

```cpp
int* p = new int[100];
delete[] p;
```

のようなコードは標準安全プロファイルでは拒否し、所有権と寿命が追跡可能なコンテナ、所有型、checked reference/view を使用する。

compiler/runtime 自身が内部で allocation を必要とすることは認めるが、その実装は trusted code boundary としてユーザーコードから分離する。
これにより `free` / `delete` 忘れ、double free、use-after-free、allocator mismatch を通常のユーザーコードから原則として排除する。

### 3.4 ライブラリ API も安全性検査の対象

危険操作は構文だけでなく、呼び出される API にも存在する。
そのため Policy Checker は AST の構文に加えて、解決済みの関数・メソッド呼び出しも確認する。

未検証の C memory/string API、危険な system API、compiler builtin 等は既定で拒否し、
Safe C++ 用に意味論を定義した checked wrapper または allowlist 済み API のみ許可する。

## 4. Safe IR

C++ の意味を LLVM IR や機械語へ直接落とす前に、未定義動作を持たない Safe IR を使用する。

概念例:

```text
%1 = checked_sadd i32 %a, %b
%2 = checked_index %array, %index
%3 = checked_load %2
%4 = checked_div_i32 %1, %3
```

各 checked operation は、成功時の値と失敗時の trap を明示的な意味として持つ。

Safe IR では、`undef`、`poison`、UB を前提とした `unreachable` のような、
後段の最適化器に「好きな結果を選べる」自由を不用意に与えない。

## 5. 最適化の原則

最適化は次の規則に従う。

```text
変形したい
   |
   v
変形前後の Safe C++ の意味が同一だと証明できるか
   |-- YES --> 最適化してよい
   `-- NO  --> 元の形を維持する
```

### 5.1 許可可能な最適化

安全性が証明できる場合に限り、次を許可できる。

- constant folding
- copy propagation
- branch elimination
- dead code elimination
- bounds-check elimination
- null-check elimination
- inlining
- simple loop optimization

Dead Code Elimination は「結果が使われない」だけでは不十分である。
削除対象が副作用を持たず、かつ trap しないことを証明できた場合のみ削除する。

### 5.2 原則として禁止する最適化・仮定

- signed overflow が起きないという仮定
- UB を利用した分岐削除
- strict-aliasing を根拠とする危険な変形
- 安全性を証明していない speculative load
- trap や副作用の順序を変える load/store reorder
- floating-point reassociation
- fast-math
- pointer provenance を根拠にした未証明の変形
- UB を理由にした `unreachable` 化

## 6. MVP1 — Fast / LLVM-oriented

MVP1 の目的は、

> **比較的安全な C++ を比較的安全な LLVM 入力へ変換すること**

である。

概念的なパイプライン:

```text
C++ source
   |
   v
Parser / AST
   |
   v
Policy Checker + Safety Checks
   |
   v
Safe IR
   |
   v
Conservative LLVM IR
   |
   v
LLVM backend
   |
   v
Machine code
```

MVP1 は MVP2 より高速なコンパイルパイプラインを優先する。
独自の高度な証明型最適化器を持たず、可能な範囲で LLVM backend を利用する。

ただし、Safe C++ 側で証明されていない情報を LLVM へ「事実」として渡してはならない。
例えば signed overflow が起きないことを証明していない加算へ、安易に `nsw` を付けない。
同様に `inbounds`、`nuw` その他の強い仮定も、意味論上の証明なしには生成しない。

MVP1 の目標は「完全無欠」ではなく、
従来 C++ よりコンパイラ最適化由来の危険を大きく減らしながら、速いコンパイルを実現することである。

## 7. MVP2 — Correctness-first / Source-aware optimizer

MVP2 の目的は、

> **元の C++ コードと意味論を知っている最適化器を持ち、安全性を壊す変形を許さずに機械語へ変換すること**

である。

概念的なパイプライン:

```text
C++ source
   |
   +--------------------+
   |                    |
   v                    v
AST / Semantic Graph   Safe IR
   |                    |
   +---------+----------+
             |
             v
   Source-aware Safe Optimizer
             |
             v
        Low-level IR
             |
             v
   Instruction Selection
             |
             v
        Machine code
```

MVP2 の「バグを許さない」とは、少なくとも **最適化による意味変更を仕様上許さない** ことを意味する。
変形の正しさを証明できなければ、その最適化は実施しない。

これは「コンパイラ実装そのものに絶対に実装バグが存在しない」と数学的に保証することとは別である。
MVP2 が保証対象とする中心は、最適化規則による意味の破壊を許容しないことである。

### 7.1 元の C++ の情報を捨てない

MVP2 の optimizer は低レベル IR だけを見るのではなく、可能な限り次の情報を保持・参照する。

- 元の AST
- 型
- 変数とオブジェクトの寿命
- 配列サイズ
- 所有関係
- nullability
- alignment
- 元の式
- 制御構造
- ソース位置
- policy によって与えられた意味論

これにより、安全性チェックそのものを証明によって削除できる。

例:

```cpp
for (int i = 0; i < 100; ++i) {
    sum += a[i];
}
```

配列 `a` の長さが 100 であり、ループ条件から常に `0 <= i < 100` を証明できるなら、
各 iteration の bounds check を削除してよい。

重要なのは「おそらく安全だから」ではなく、
**Safe C++ の意味論上 trap が起きないことを証明できるから削除する**ことである。

## 8. MVP1 と MVP2 の差

| 項目 | MVP1 | MVP2 |
| --- | --- | --- |
| 主目的 | 高速なコンパイルと安全性改善 | 正しさを最優先 |
| 出力経路 | 安全寄りの LLVM IR を経由 | 独自最適化から機械語へ |
| optimizer | LLVM を保守的に利用 | 元 C++ を知る独自 optimizer |
| 安全性証明 | 最低限 | 中心機能 |
| 証明できない最適化 | LLVM に危険な仮定を渡さない | 実施しない |
| trap の扱い | 保存する | 保存し、不要性を証明した場合のみ削除 |
| コンパイル速度 | MVP2 より高速を目標 | 解析・証明のため遅くてもよい |

「MVP1 の方が高速」は主としてコンパイルパイプラインについての目標である。
生成コードの実行速度について、常に MVP1 が MVP2 より高速であることを保証するものではない。

## 9. Portability Rule

同じ Safe C++ ソース、同じ JSON policy、同じ明示的 target profile に対して、
適合する Safe C++ Compiler は同じ observable behavior を与えることを目標とする。

backend や optimizer による差を、プログラマが暗黙に意識しなければならない設計にしない。

target に依存する事項（整数幅、ABI、endianness など）が意味に影響する場合は、
暗黙のコンパイラ差として扱わず、明示的な target profile の一部として扱う。

## 10. プロジェクトの短い定義

**MVP1**

> 危険な C++ を危険な LLVM IR にしないための、高速で保守的な変換器。

**MVP2**

> C++ の元の意味を忘れず、安全性を証明しながら、意味を壊す最適化を許さずに機械語を生成するコンパイラ。

## 11. ライセンス

本プロジェクトは MIT License の下で公開する。
ライセンス本文はリポジトリ直下の `LICENSE` を正とする。
