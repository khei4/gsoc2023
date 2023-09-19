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

### 2. Removal of memcpy introduced on the attributed with nocapture, readonly and noalias on MemCpyOpt

(from proposal)
  LLVM function arguments can have attributes, noalias shows there is no other pointer variable that points to the same as the argument, and readonly shows that the argument is not modified in the function. Functions attributed with noalias and readonly at the same position show invariance during the execution of the functions. We can omit the memcpy of argument to give such functions. To completely fix this Rust issue, we also need to attach align attribute to the argument on the Rust side.

### 3. Attaching wrapping flags for the switch to look up table conversion on SimplifyCFG

(from proposal)
This issue reports dropped nsw (no signed wrap) flags for arithmetic instructions on SimplifyCFG and InstCombine. (This is motivated by Rust Issue reported in the issue). This can be addressed by SimplifyCFG and InstCombine/InstSimplify. For the former SwitchToLookupTable is the place to handle this. And for the latter, simplifyBinOp in the InstSimplify is the one. We need to elaborate on when it’s safe to add nsw and/or nuw for both passes.

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
