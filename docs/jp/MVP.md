# MVP1 / MVP2 仕様

> 日本語版が正本。

## 1. MVP1

MVP1 の目的:

> **最新安定版 C/C++ を解析し、危険操作を error/warning/trap として処理し、検証済み LLVM IR を出力する。**

基準:

- C23 / ISO/IEC 9899:2024
- C++23 / ISO/IEC 14882:2024

pipeline:

```text
C23/C++23
 -> frontend/AST
 -> Safety Analyzer/Rewriter
 -> LLVM IR
 -> LLVM Verifier
 -> Safe LLVM Validator
 -> Validated LLVM IR
```

### MVP1 Definition of Done

1. C23/C++23 mixed project を受理
2. 一般的な安全コードを過剰に拒否しない
3. unsafety error / unsafety warning を出せる
4. signed overflow / division / shift / null / bounds 等の主要 UB を trap/error 化
5. ignore.rules / ignore.files
6. LLVM IR 生成
7. LLVM Verifier 合格
8. Safe LLVM Validator 合格
9. dangerous flags/attributes を validator が検出可能
10. validated LLVM IR を .ll または .bc として出力可能

machine code 生成は MVP1 の必須完成条件ではない。smoke test として LLVM backend に渡してよい。

## 2. MVP2

MVP2 は validated LLVM IR までの安全性に加え、source-aware optimizer と machine code generation を含む。

元 C/C++ AST / type / lifetime / bounds / ownership 情報を保持し、

```text
SafeMeaning(before) == SafeMeaning(after)
```

を証明できる最適化だけ適用する。

MVP2 は compilation speed より correctness を優先する。

## 3. MVP1 と MVP2

| 項目 | MVP1 | MVP2 |
| --- | --- | --- |
| C/C++ standard | C23/C++23 | 同じ基準から開始 |
| diagnostic | unsafety error/warning | 同じ |
| output | validated LLVM IR | machine code |
| optimizer | 最小・保守的 | source-aware |
| LLVM validator | 必須 | 必須/同等検証 |
| compile speed | 高速優先 | correctness 優先 |

## 4. 実装 Issue

MVP1 は GitHub Issue #1 で追跡する。
