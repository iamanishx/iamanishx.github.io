---
title: "Vectorization and Coalescing: The Two Ways to Move Memory Fast"
date: "2026-07-02"
tags: ["gpu", "cutlass", "cutedsl", "vectorization", "coalescing"]
---

# vectorization and coalescing: the two ways to move memory fast

two ideas that decides whether kernel is fast or slow at moving data and 
cute gives you one knob for each.

## why we care at all

the gpu is starving. its math units are so fast that most kernels spend
their time waiting for data from hbm (global memory). hbm is slow: a load
costs hundreds of cycles. so the whole game is: move the bytes you need in
as few and as wide trips as possible. two tricks do that, and they attack
two different parts of the problem.

## trick 1: vectorization (one thread, wide load)

a thread can load 1 element per instruction, or it can load 4, 8, 16 bytes
in a single instruction. same latency, more data. fewer instructions to
move the same tile.

```
scalar:      load a  |  load b  |  load c  |  load d      4 instructions
vectorized:  load [a b c d]                               1 instruction
```

the catch: the data has to be contiguous in memory (a,b,c,d next to each
other) and aligned. then one wide instruction grabs them all. this is
vectorization: make each thread's load as wide as the layout allows.

## trick 2: coalescing (many threads, adjacent addresses)

a warp is 32 threads issuing the same load instruction at the same time.
if those 32 threads ask for 32 addresses that are next to each other, the
hardware fuses them into a few big memory transactions. if they ask for 32
scattered addresses, it does 32 separate slow transactions.

```
coalesced:    thread0->addr0  thread1->addr1 ... thread31->addr31    (contiguous)
              hardware fuses into ~1 wide transaction                fast

scattered:    thread0->addr0  thread1->addr500 ... (all over)
              32 separate transactions                               slow
```

the rule: arrange it so adjacent threads touch adjacent addresses. this is
coalescing. it is about the shape of the thread-to-data mapping, not how
wide one thread's load is.

## the two are different (do not mix them up)

```
vectorization  = one thread reads a wide contiguous chunk
coalescing     = adjacent threads read adjacent addresses
```

one is "how fat is a single thread's load," the other is "do the 32
threads line up on memory." you want both: fat loads and lined-up threads.
together they turn a tile move into the fewest possible wide hbm
transactions.

## how cute does it: one knob each

cute splits these into exactly two calls.

```python
# 1) vectorization: how many bits one thread copies per instruction
atom = cute.make_copy_atom(cute.nvgpu.copyuniversalop(),
                           cutlass.float16, num_bits_per_copy=128)

# 2) coalescing: how the threads are laid out over the data (the tv layout)
tc = cute.make_tiled_copy_tv(atom, thr_layout, val_layout)

# then every thread copies its slice
thr = tc.get_slice(tidx)
cute.copy(tc, thr.partition_s(gmem), thr.partition_d(smem))
```

- `make_copy_atom(..., num_bits_per_copy=n)` sets the vector width. bump n
  from 16 to 128 and each thread moves 8 fp16 in one shot instead of one.
  that is vectorization.

- `make_tiled_copy_tv(atom, thr_layout, val_layout)` sets the thread
  layout (the "t"). you choose it so consecutive threads land on
  consecutive memory. that is coalescing. the "v" (value layout) says how
  many elements each thread owns and in what shape.

so: the atom controls the fat load, the tiled copy controls the thread
lineup.