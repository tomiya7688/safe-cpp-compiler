# Safe IR / LLVM Lowering 仕様

> 日本語版が正本。MVP1 が Safe IR を materialize しない場合でも、この invariant を守る。

## 1. 目的

Safe IR は frontend の意味を、backend が UB として自由に利用できない形へ固定する中間意味論。

## 2. Invariant

1. IR 自体に言語レベル UB を残さない。
2. 未初期化値を arbitrary value として流さない。
3. poison 相当を意味論に入れない。
4. trap を明示 control effect とする。
5. side effect と trap の相対順序を保存する。
6. bounds/null/lifetime/alignment 情報を必要に応じて保持する。
7. source location を保持できる。
8. optimizer proof に必要な provenance を保持できる。

## 3. 概念命令

```text
%r = checked_sadd i32 %a, %b
%r = checked_sdiv i32 %a, %b
%p = checked_index %base, %index, %length
%v = checked_load %p
checked_store %p, %v
check_nonnull %p
check_alive %object
check_align %p, 4
trap out_of_bounds
```

syntax は実装時に変更可能。意味を normative とする。

## 4. Arithmetic

signed add/sub/mul は overflow なら trap。proof があれば normal arithmetic に lower 可能。

unsigned は modulo wrap。LLVM nuw で意味を置き換えない。

division は divisor != 0 と signed MIN/-1 を check。

## 5. Memory

memory access は必要に応じて null、bounds、lifetime、alignment、read/write permission、type/representation compatibility を check。

proof による check elimination は Safe semantics 上 failure 不可能な場合のみ。

## 6. Trap

trap は noreturn の defined control effect。

代表 reason:

- signed_overflow
- division_by_zero
- signed_div_overflow
- null_dereference
- out_of_bounds
- invalid_shift
- invalid_alignment
- use_after_lifetime

observable side effect と競合する場合、trap 順序を変更禁止。

## 7. LLVM lowering

overflow は `llvm.*with.overflow` または等価 check sequence を使用可能。

division は安全条件 check 後に div instruction。

null/bounds 等は branch + trap。proof がある場合だけ省略。

## 8. 証明なしに LLVM へ渡さない情報

- nsw
- nuw
- inbounds
- nonnull
- dereferenceable
- 強い alignment
- noalias/provenance
- llvm.assume
- fast-math
- UB-based unreachable

証明できた場合だけ付与可能。MVP1 は多くを常に付与しない実装から開始してよい。

## 9. unreachable

noreturn trap/call 直後など構造的到達不能だけに使用可能。C/C++ UB を理由に使用禁止。

## 10. reorder / DCE

trap、external call、volatile/atomic、observable side effect の順序を壊す reorder 禁止。

DCE は value 未使用、副作用なし、trap 不可能、lifetime/control effect なしをすべて証明した場合のみ。

## 11. check elimination

bounds/null/lifetime check を削除するには、その execution path で failure 不可能なことを証明する。

## 12. source-aware metadata

MVP2 で保持候補:

- original AST/source range
- C/C++ type
- object identity
- lifetime
- bounds
- ownership
- nullability
- alignment
- control region
- originating rule/policy

## 13. MVP1

最短実装:

```text
Clang AST
  -> Policy Checker
  -> Safety Rewriter
  -> conservative LLVM IR
```

Safe IR を file/IR object として構築しなくても、各 operation は Safe IR の defined result/trap/error 意味に対応すること。

## 14. MVP2

source semantic graph + Safe IR を optimizer の中心に置き、

```text
SafeMeaning(before) == SafeMeaning(after)
```

を証明できる変形だけを適用する。
