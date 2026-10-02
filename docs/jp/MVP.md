# MVP1 / MVP2 仕様

> 日本語版が正本。実装追跡は GitHub Issue を利用する。

## 1. MVP1

> **C/C++ を policy/safety check し、危険な仮定をできるだけ含まない conservative LLVM IR へ変換する高速な経路。**

```text
C/C++ source
   -> Clang frontend / AST
   -> Policy Checker
   -> Safety Analyzer / Rewriter
   -> Conservative LLVM IR
   -> LLVM backend
   -> Machine code
```

### 必須機能

- mixed C/C++ project
- safe-cpp.json
- stable rule ID
- ignore.rules / ignore.files
- AST-based checker
- resolved C API checker
- default-deny
- 初期 UB trap/error
- conservative LLVM IR
- LLVM backend
- optimization-level differential tests

### 最初の semantic set

- signed overflow -> trap
- unsigned overflow -> wrap
- division by zero -> trap
- signed MIN/-1 -> trap
- null dereference -> trap
- bounds violation -> trap
- invalid shift -> trap
- uninitialized read -> compile error（検出可能範囲）
- unsupported unsafe construct -> compile error

未対応 lifetime/data-race を黙って通常 UB として通してはならない。検出した未対応ケースは拒否する。

### LLVM で証明なしに禁止

nsw, nuw, inbounds, fast-math, UB-based unreachable, unproven llvm.assume, nonnull, dereferenceable, noalias, alignment。

### Definition of Done

1. stable rule ID diagnostics
2. ignore.rules が rule 単位で動く
3. ignore.files が file 単位で動く
4. 対応 UB が trap/error
5. conservative LLVM IR 出力
6. machine code 生成
7. test で O0/O1/O2/O3 等の Safe observable behavior が一致
8. dangerous LLVM flag/attribute test
9. C/C++ cross-call integration test

### 後回し可能

- 独自 parser
- 独自 backend
- 高度 lifetime/alias proof
- vectorizer
- exception support
- full concurrency model
- 全 standard library safety model
- Safe IR serialization

## 2. MVP2

> **元の C/C++ の意味を知る source-aware optimizer が、安全性を証明できない最適化を行わず machine code を生成する。**

```text
C/C++ source
   +-----------------------------+
   |                             |
   v                             v
AST / Semantic Graph          Safe IR
   |                             |
   +-------------+---------------+
                 |
                 v
       Source-aware Optimizer
                 |
                 v
           Low-level IR
                 |
                 v
            Machine code
```

LLVM backend を使うか独自 backend にするかは実装選択。ただし optimizer semantics の支配権は Safe C++ Compiler 側。

### optimizer rule

```text
SafeMeaning(before) == SafeMeaning(after)
```

証明できなければ最適化しない。

初期候補:

- constant folding
- copy propagation
- branch elimination
- safe DCE
- bounds-check elimination
- null-check elimination
- inlining
- simple loop optimization

safety check elimination を主要な性能源の一つとする。

## 3. 差

| 項目 | MVP1 | MVP2 |
| --- | --- | --- |
| 主目的 | 高速 compilation | correctness-first |
| frontend | Clang 利用可 | Clang/独自 |
| optimizer | LLVM を保守的利用 | source-aware 独自 |
| proof | 最小限 | 中心機能 |
| Safe IR | 論理 semantics でも可 | 中心表現 |
| machine code | LLVM backend | LLVM/独自 |
| 未証明最適化 | 危険仮定を渡さない | 実施しない |
| compile speed | より高速を目標 | 遅くてもよい |

MVP1 の「高速」は主にコンパイル速度。

## 4. テスト

MVP1: positive、forbidden syntax、C API、UB trap、ignore、mixed C/C++、LLVM IR pattern、optimization-level differential。

MVP2: 上記 + optimizer equivalence、proof failure skip、metadata preservation、randomized/differential、regression corpus。

## 5. Issue 運用

実装可能な単位になった機能は GitHub Issue にする。

最低限: goal、normative spec link、scope、non-scope、acceptance criteria、tests、unresolved questions。
