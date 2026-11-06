# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->451.724<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.008<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->434579.0<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12671.1<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4615<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2227.192 ms, 2 threads: 2000.909 ms, 4 threads: 2503.356 ms, 8 threads: 2176.993 ms. Fastest: 2 threads
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version spends its time in the Python interpreter, not in arithmetic: every multiply-add of the 256 x 256 dot products is a bytecode dispatch on boxed floats, one scalar at a time. The vectorized version hands whole matrices to the compiled BLAS library NumPy links against, which runs tight loops over contiguous memory, uses SIMD (AVX) instructions that process several floats per instruction, and blocks the work to stay in cache. Same arithmetic, 448x faster (451.7 vs 1.0 ms).
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve stops improving at 2 threads: 2227 ms with one thread, 2001 ms with two, then 2503 ms with four. Two threads gain only 10%, because Colab's 2 vCPUs are likely two hyperthreads of one physical core, sharing its execution units and cache. Beyond 2, threads outnumber vCPUs: the OS time-slices them, and every matmul ends with a barrier where all threads must finish, so they wait on each other. The 8-thread value (2177 ms) is within run-to-run noise of a 3-repeat median.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A training step is dominated by large dense matrix multiplies (32 sequences x 256 tokens x 256 channels per layer) where every output element is independent. The T4 runs them on thousands of CUDA cores executing the same instruction in lockstep (SIMT), with far more memory bandwidth than the CPU: 100 steps took 12.7 s instead of 434.6 s (34x). Token by token generation at batch size 1 would not benefit: each step is a tiny matmul, so kernel launch and host-device synchronisation dominate.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment hit a different limit.

Vectorization removed interpreter overhead, not arithmetic (451.7 to 1.0 ms). At 1.0 ms the 256 x 64 problem is too small to keep the SIMD units busy, so call overhead and memory access now bound it.

The thread sweep was bound by hardware I did not have, then by coordination. Two vCPUs gave only 10% (2227 to 2001 ms), and asking for 4 threads made a step 25% slower (2503 ms) because the extra threads only add synchronisation.

The GPU gave 34x (434.6 s to 12.7 s), well below what its peak suggests. By my estimate a step does about 280 GFLOP, so 127 ms per step is roughly 2 TFLOPS, a quarter of the T4's 8 TFLOPS FP32 peak. The remainder goes to work that cannot be divided and data that has to move: the Python training loop, launching many small kernels, and copying each batch from host to device.

Parallelism stops helping when the remaining time is no longer the parallel part
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
Sweep the GPU batch size from 8 to 256 and record milliseconds per token. If throughput keeps rising with batch size, the GPU was starved at batch 32 by launch and transfer overhead. If it flattens near 32, the step was already compute-bound and my overhead explanation is wrong.
<!--/answer-->
