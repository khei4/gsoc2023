# Addressing Rust optimization failures in LLVM

<https://summerofcode.withgoogle.com/programs/2023/projects/a16FfPnb>

**Author**: [Kohei Asano](https://github.com/khei4)

**Organization**: [LLVM](https://llvm.org/)

**Mentor**: [Nikita Popov](https://github.com/nikic)

## Introduction

Over this summer I worked on bassically middle-end passes like InstCombine, SimplifyCFG, MemCpyOpt.

## Works

### 1. Constant Propagation for uniformly patterned aggregated types on InstCombine

(from proposal)
  This issue reports missed optimization opportunities for loads of global constant aggregates with the same elements. This particular opportunity could be handled by alignment-based and GEP-indexed-based analysis.
  At first, there are no implemneted constatnt folding which can handle array or global variables with variable index.

This is the first step for constant folding, which folds the uniformly initialized bits patterns.
<https://reviews.llvm.org/D144184>

second step is fold the cases that by using load instructions' alignment constraints, valid access patterns have all same results.
<https://reviews.llvm.org/D144445>

third step is to analize the gep indeces and checks the all possible indeces access are valid or not.
<https://reviews.llvm.org/D146622>

### 2. Removal of memcpy introduced on the attributed with nocapture, readonly and noalias on MemCpyOpt

(from proposal)
  LLVM function arguments can have attributes, noalias shows there is no other pointer variable that points to the same as the argument, and readonly shows that the argument is not modified in the function. Functions attributed with noalias and readonly at the same position show invariance during the execution of the functions. We can omit the memcpy of argument to give such functions. To completely fix this Rust issue, we also need to attach align attribute to the argument on the Rust side.

This is the patch to handle those case on LLVM side
<https://reviews.llvm.org/D150970>

and Rust side patch is this(althoguh it's not mine). That part was really hard for the tricky case for msvc corner case.
<https://github.com/rust-lang/rust/pull/112157>

### 3. Attaching wrapping flags for the switch to look up table conversion on SimplifyCFG

(from proposal)
This issue reports dropped nsw (no signed wrap) flags for arithmetic instructions on SimplifyCFG and InstCombine. (This is motivated by Rust Issue reported in the issue). This can be addressed by SimplifyCFG and InstCombine/InstSimplify. For the former SwitchToLookupTable is the place to handle this. And for the latter, simplifyBinOp in the InstSimplify is the one. We need to elaborate on when it’s safe to add nsw and/or nuw for both passes.

for this issue I choose to fix step by step again
<https://reviews.llvm.org/D146903>

### 4. Stack Move Optimization, which merge the allocas neither captured nor simultaneously used

(from proposal)
 Rust’s move semantics introduce more memcpy to optimize. We can use nocapture attribute and do a liveness analysis to find the necessity for memcpy. Redundant memcpy could happen frequently on Rust, but not on other languages. More details should be analyzed.

### Other patches

## Performance Analysis

### LLVM update for the Rust with tons of improvement for

This is combined improvement but this is the one I

### Profiling for Stack Move Optimization

## Future Work

### Dataflow sensitive Stack Move Optimization

###

### More missopportunities on Rust

## Acknoledgement
