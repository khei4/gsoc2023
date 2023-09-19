# Addressing Rust optimization failures in LLVM

<https://summerofcode.withgoogle.com/programs/2023/projects/a16FfPnb>

**Author**: [Kohei Asano(khei4)](https://github.com/khei4)

**Organization**: [LLVM](https://llvm.org/)

**Mentor**: [Nikita Popov](https://github.com/nikic)

## Introduction

- Rust is popular
- But current LLVM does have a lot of miss-opportunity for the opimitzation

- One of them is to remove memcpy introduced by the move semantics.

## Patches

### Constant Propagation for uniformly patterned aggregated types on InstCombine

### removal of memcpy introduced on the attributed with nocapture, readonly and noalias on MemCpyOpt

### Attaching wrapping flags for the switch to look up table conversion on SimplifyCFG

### Stack-move Optimization

## Performance Analysis

### LLVM update for the Rust with tons of improvement for

This is combined improvement but this is the one I

### stack-move optimization profiling

## Future Work

### forward dataflow based stack-move optimization

###

### More missopportunities on Rust

## Acknoledgement
