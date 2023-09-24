# Addressing Rust optimization failures in LLVM

<https://summerofcode.withgoogle.com/programs/2023/projects/a16FfPnb>

**Author**: [Kohei Asano](https://github.com/khei4)

**Organization**: [LLVM](https://llvm.org/)

**Mentor**: [Nikita Popov](https://github.com/nikic)

## Introduction

In Rust, there have been numerous reports of LLVM-level optimization failures, as sometimes we can see the case that [naive translations from C++ can result in decreased performance.](https://www.reddit.com/r/rust/comments/10dpw5r/c_vs_rust_which_is_faster_x86_assembly_inside/). In this project, we aim to address and rectify several of these reported issues. Over the summer, I focused primarily on issues related LLVM middle-end passes like InstCombine, SimplifyCFG, and MemCpyOpt.

## Works

### 1. [Constant Propagation for uniformly patterned aggregated types on InstCombine](https://github.com/llvm/llvm-project/issues/66868)

Originally, no implemented constant folding can handle constant global aggregate-type values with variable index access. This was the first issue I tackled on LLVM. Throughout the project, I decided to split the problem step by step and make the patch as small as possible.

### Steps for Constant Folding

1. **Initial Step**: Fold aggregate variables with uniform bit patterns (either all ones or all zeros).
   - <https://reviews.llvm.org/D144184>
2. **Second Step**: Focus on cases where using load instruction alignment constraints ensures that all valid access patterns yield identical results.
   - <https://reviews.llvm.org/D144445>
3. **Third Step**: Analyze the GEP indices to determine whether all possible access indices are valid.
   - <https://reviews.llvm.org/D146622>

Although the issue on the Rust side remains unresolved for [the embedded vector manipulation for global array constants](https://github.com/rust-lang/rust/issues/107208#issuecomment-1677404374), we could finally fold the constant global aggregate loads with variable indices for the unique result on LLVM-IR levels.

### 2. [Removal of memcpy introduced on the attributed with readonly, noalias and nocapture on MemCpyOpt](https://github.com/rust-lang/rust/issues/107436)

LLVM function arguments can have attributes, i.e. noalias shows there is no other pointer variable that points to the same as the argument, and readonly shows that the argument is not modified in the function. Functions attributed noalias and readonly at the same position show invariance during the execution of the functions. If it's also attributed with nocapture, we can omit the memcpy of argument to pass for such functions. But to completely remove that memcpy, we need to attach alignment attributes to the arguments.

1. **Initial Step**: Use memcpy source directly if dest is known to be immutable from attributes.
   - <https://reviews.llvm.org/D150970>
2. **Second Step**(Rust side and other person's patch): Add alignment to indirectly-passed by-value types, correcting the alignment of byval on x86.
   - <https://github.com/rust-lang/rust/pull/112157>

For the Second step, Nikita and erikdesjardins worked hard on the corner cases, and we finally could get the issue closed.

### 3. [Attaching wrapping flags for the switch to look up table conversion on SimplifyCFG](https://github.com/rust-lang/rust/issues/107436)

This issue reports dropped nsw (no signed wrap) flags for arithmetic instructions on SimplifyCFG and InstCombine. (This is motivated by Rust Issue reported in the issue). This can be addressed by SimplifyCFG and InstCombine/InstSimplify. For the former SwitchToLookupTable on SimplifyCFG is the place to handle this.

1. **Initial Step**: add nsw/nuw on SwitchToLookupTable in SimplifyCFG
   - Add nsw on SwitchToLookupTable index calculation on MinCaseVal subtraction
      - <https://reviews.llvm.org/D146903>
   - add nuw/nsw on BuildLookuptable BitMap shift width calculation
      - <https://reviews.llvm.org/D150838>
   - add nsw on BuildLookuptable LinearMap calculation
      - <https://reviews.llvm.org/D150943>
2. **Second Step**: Preserve nsw on InstCombine for the cases.
   - <https://reviews.llvm.org/D150838>

Although SimplifyCFG patch was wrong and required [fixup patch](https://github.com/llvm/llvm-project/pull/65882), we could resolve the original issue.

### 4. Stack Move Optimization, which merges the allocas neither captured nor simultaneously used

 Rust’s move semantics introduce memcpy when Clone type values are 1) rebind to the variables, 2) passed by value to the functions. But for most cases, also due to the property of the Rust references, both pointers aren't used simultaneously. By using alias/dataflow analysis, we could find such patterns on LLVM-IR. Originally, liveness-analysis approach was proposed, but it stalled for a long time. So I pushed forward by splitting it's patch.

1. **Initial Step**: Single basic block Stack Move Optimization
   - <https://reviews.llvm.org/D153453>
2. **Second Step**: Multi basic block Stack Move Optimization
   - <https://reviews.llvm.org/D155406>
3. **Third Step**: Dataflow sensitive Stack Move Optimization(WIP)
   - <https://reviews.llvm.org/D159075>

Due to the lifetime span reconstruction complexity, so many reverts happened. And now, it is still in progress.

### Other patches

#### LLVM

- [[ConstantFold] use StoreSize for VectorType byte checking](https://reviews.llvm.org/D150515)
- [[ConstantFolding] fold integers whose bitwidth is greater than 64.](https://reviews.llvm.org/D150422)
- [[LangRef] fix the function result attributes location explanation and example](https://reviews.llvm.org/D151772)
- [[InstCombine] add overflow checking on AddSub `C-(X+C2) --> (C-C2)-X`](https://reviews.llvm.org/D152068)

#### Rust

- [Add a -Z print-codegen-stats option to expose LLVM statistics.](https://github.com/rust-lang/rust/pull/113723)

## Performance Analysis

### LLVM update for the Rust with tons of improvement for

Although is the combined improvement of all of Rust-LLVM updates, I could see the significant compile-time/binary size improvements on Rust by updating LLVM.
<https://github.com/rust-lang/rust/pull/114048#issuecomment-1654195018> That was a very impressive moment to see those results. And I'm looking forward to seeing the results for the complete Stack Move Optimizations also.

## Future Work

### Extending Stack Move Optimization

Current optimization lacks the ability to reconstruct lifetime spans and has miss opportunities to merge potential memory locations, i.e., for the copy of byval attributed arguments. So, I'll do more work to realize more robust optimizations.

### Profiling for Stack Move Optimization

The original patch author measured improvements for Stack Move Optimization <https://arewestackefficientyet.com/>, but it'll be better to profile again to see the scope and abilities of Stack Move Optimization.

### More issues on Rust and LLVM

On Rust issues, there are more issues related to LLVM middle-end, labeled with `A-LLVM`; I'll also fix the issues reported for my above patches.

## Acknoledgement

I deeply appreciate the guidance and support from my mentor, Nikita Popov, throughout this project. His passion is evident, and he swiftly reviewed numerous patches, mine included. His assistance went beyond this project, positively influencing my broader journey as a software developer. Observing and working alongside him has been truly an invaluable experience. Furthermore, many LLVM contributors took the time to review my patches, even when they were less than perfect. I genuinely appreciate the entire LLVM community.
