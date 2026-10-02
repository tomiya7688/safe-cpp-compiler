# Safe LLVM IR / Validator 仕様

> 日本語版が正本。

## 1. MVP1 output

MVP1 の正式出力は **Validated LLVM IR**。

```text
AST
 -> Safety Analyzer/Rewriter
 -> LLVM IR
 -> LLVM Verifier
 -> Safe LLVM Validator
 -> Validated LLVM IR
```

LLVM backend に渡す場合も、この validated IR を入力とする。

## 2. LLVM Verifier

最初に LLVM 自身の module/function verifier を通す。
構造・型・SSA 等の不正 IR は compiler error。

## 3. Safe LLVM Validator

LLVM verifier が受理する IR でも Safe C++ Compiler の方針上危険なものがあるため、追加 validator を実行する。

MVP1 の基本方針は **危険な optimization hint/flag を極力出さない**。

### 必須検査

- undef value を安全値として利用していない
- poison value を生成・利用していない
- 未証明 nsw/nuw がない
- 未証明 inbounds GEP がない
- fast-math flags がない
- UB 前提 unreachable がない
- 未証明 llvm.assume がない
- 未証明 nonnull/dereferenceable/noalias/alignment 属性がない
- runtime safety helper ABI が正しい
- trap branch/helper が消えていない
- unsafe raw LLVM construct が compiler-defined safe subset 外にない

## 4. Checked operations

signed overflow は overflow intrinsic + trap など、validator が識別可能な規定 lowering を使う。

division/shift/null/bounds も validator が確認できる canonical pattern または runtime helper を使う。

MVP1 では性能より検証容易性を優先し、危険操作を helper call に lower してもよい。

## 5. Validator の限界

MVP1 Validator は arbitrary third-party LLVM IR の完全 safety prover ではない。

**Safe C++ Compiler が生成する LLVM subset が、定義した lowering contract に従っているかを独立に確認するもの**。

外部 LLVM IR を入力として受理する機能は MVP1 の必須範囲外。

## 6. Failure

Validator failure は通常 source diagnostic ではなく compiler pipeline failure。

```text
compiler safety error: generated LLVM IR failed Safe LLVM validation
```

debug mode では該当 LLVM instruction と source location を表示する。

## 7. MVP2

MVP2 の optimizer output も Safe LLVM Validator または同等の machine-level validator を通す。
