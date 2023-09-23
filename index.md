# Addressing Rust optimization failures in LLVM

<https://summerofcode.withgoogle.com/programs/2023/projects/a16FfPnb>

**Author**: [Kohei Asano](https://github.com/khei4)

**Organization**: [LLVM](https://llvm.org/)

**Mentor**: [Nikita Popov](https://github.com/nikic)

## Introduction

In Rust, there have been numerous reports of optimization failures, as we can see the case that [naive translations from C++ can result in decreased performance.](https://www.reddit.com/r/rust/comments/10dpw5r/c_vs_rust_which_is_faster_x86_assembly_inside/). In this project, we aim to address and rectify several of these reported issues. Over the summer, I focused primarily on middle-end passes like InstCombine, SimplifyCFG, and MemCpyOpt.

## Works

### 1. Constant Propagation for uniformly patterned aggregated types on InstCombine

Originally there are no implemneted constatnt folding which can handle global aggregate-type variables with variable index. <https://llvm.godbolt.org/z/s97K4bG3z>

### Steps for Constant Folding

1. **Initial Step**: Fold aggregate variables with uniform bit patterns (either all ones or all zeros).
   - <https://reviews.llvm.org/D144184>
2. **Second Step**: Focus on cases where using load instruction alignment constraints ensures that all valid access patterns yield identical results.
   - <https://reviews.llvm.org/D144445>
3. **Third Step**: Analyze the GEP indices to determine whether all possible access indices are valid.
   - <https://reviews.llvm.org/D146622>

Although the issue on the Rust side remains unresolved for the embedded vector local manipulation for global array constants, we could finaly fold the constant global aggregate loads accordingly.

### 2. Removal of memcpy introduced on the attributed with readonly, noalias and nocapture on MemCpyOpt

LLVM function arguments can have attributes, noalias shows there is no other pointer variable that points to the same as the argument, and readonly shows that the argument is not modified in the function. Functions attributed noalias and readonly at the same position show invariance during the execution of the functions. If it's also attributed with nocapture We can omit the memcpy of argument to give such functions.

The patch to handle those case on LLVM side
<https://reviews.llvm.org/D150970>

and Rust side patch is this(althoguh it's not mine).
<https://github.com/rust-lang/rust/pull/112157>

and original motivational issue is closed.

### 3. Attaching wrapping flags for the switch to look up table conversion on SimplifyCFG

This issue reports dropped nsw (no signed wrap) flags for arithmetic instructions on SimplifyCFG and InstCombine. (This is motivated by Rust Issue reported in the issue). This can be addressed by SimplifyCFG and InstCombine/InstSimplify. For the former SwitchToLookupTable is the place to handle this.

motivated transformation: <https://alive2.llvm.org/ce/z/GZs_dj>

1. add nsw/nuw on SwitchToLookupTable in SimplifyCFG
  a. add nsw on SwitchToLookupTable index calculation on MinCaseVal subtraction <- this revision will be

  <https://alive2.llvm.org/ce/z/WSFRK5>

  b. add nuw/nsw on BuildLookuptable BitMap shiftwidth calculation <https://reviews.llvm.org/D150838>

  <https://alive2.llvm.org/ce/z/5Cmc72>

  c. add nsw on BuildLookuptable LinearMap calculation <https://reviews.llvm.org/D150943>

<https://alive2.llvm.org/ce/z/cKBQfs>
preserve nsw on InstCombine for the cases. <https://reviews.llvm.org/D152088>
transformation: <https://alive2.llvm.org/ce/z/ZWmZLv>

### 4. Stack Move Optimization, which merge the allocas neither captured nor simultaneously used

(from proposal)
 Rust’s move semantics introduce more memcpy to optimize. We can use nocapture attribute and do a liveness analysis to find the necessity for memcpy. Redundant memcpy could happen frequently on Rust, but not on other languages. More details should be analyzed.

single-BB
<https://reviews.llvm.org/D153453>

multi-BB
<https://reviews.llvm.org/D155406>

dataflow-sensitive
<https://reviews.llvm.org/D159075>

### Other patches

- [[ConstantFold] use StoreSize for VectorType byte checking](https://reviews.llvm.org/D150515)
- [[ConstantFolding] fold integers whose bitwidth is greater than 64.](https://reviews.llvm.org/D150422)
- [[LangRef] fix the function result attributes location explanation and example](https://reviews.llvm.org/D151772)
- [[InstCombine] add overflow checking on AddSub `C-(X+C2) --> (C-C2)-X`](https://reviews.llvm.org/D152068)

## Performance Analysis

### LLVM update for the Rust with tons of improvement for

This is combined improvement but this is the one I
<https://github.com/rust-lang/rust/pull/114048#issuecomment-1654195018>

## Future Work

### Extinding sensitive Stack Move Optimization

current landed are theoretically incomplete and there are rooms for more patterns

so need to be done

### Profiling for Stack Move Optimization

It's effect is measured cocretely by original author for the patch,

### More missopportunities on Rust

There are more issues on both on Rust side and llvm side, with A-LLVM,

## Acknoledgement
