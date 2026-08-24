---
layout: page
title: "sketch-ops — Streaming Heavy-Hitter Sketch Benchmarking"
permalink: /writeups/sketch-ops/
---

*Repo: [github.com/tahovig/sketch-ops](https://github.com/tahovig/sketch-ops) — outside the cybersecurity-pivot series above; systems/algorithms engineering work, kept in its own category on the portfolio.*

## The problem

Streaming systems — network backbones, firewalls, exchange trading logs — often need to know which small fraction of items (an IP address, a ticker) is responsible for most of the traffic, in a single pass, using memory that's sub-linear in the size of the stream. That's the heavy-hitters problem, and there's no single best-known solution to it: Count-Min Sketch, Space-Saving, and HeavyKeeper trade off memory, accuracy, and throughput differently, and at wire speed even a 1% accuracy improvement per byte of memory is operationally significant.

`sketch-ops` is a two-phase project built around that tradeoff. Phase 1 is a Rust benchmark harness rigorous enough to trust its numbers. Phase 2 uses that harness to test whether a novel sketch design (PeelSketch) actually beats the established baselines — and reports the answer honestly, including when the answer is no.

## Phase 1: a benchmark trustworthy enough to make a claim on

Three algorithms — Count-Min Sketch, Space-Saving, HeavyKeeper — implemented behind one `HeavyHitterSketch` trait, benchmarked over seeded Zipfian streams against exact ground truth. The harness design spends most of its effort on the part that's easy to get subtly wrong: making a memory/accuracy comparison actually mean what it claims to mean.

- **Top-k accounting is charged against every algorithm's budget.** Only Space-Saving has top-k tracking built into its core design; Count-Min and HeavyKeeper both need an auxiliary min-heap bolted on. If that heap's memory weren't counted against the sweep's memory budget, CMS/HeavyKeeper would get a free advantage over Space-Saving at every point in the sweep, invalidating the whole comparison. Every `memory_bytes()` implementation includes it.
- **Hand-rolled hash functions, not an opaque hasher crate.** Hash cost is part of what's being measured, so a black-box hasher (e.g. SipHash, built for flood-resistance that's irrelevant here) would make that cost un-auditable.
- **Static dispatch in the hot insert loop.** The sweep matches an `Algorithm` enum once per run into a generic function, rather than `Box<dyn HeavyHitterSketch>` — a vtable call per item could mask exactly the cache effects the project exists to measure, especially at small memory budgets.
- **Fresh RNG per run, warmup excluded from timing, generation timed separately from insertion.** Each of these is a specific, named failure mode (a reused RNG silently desyncing a "reproducible" stream; a cold hashmap/heap skewing steady-state numbers; the Zipfian sampler's cost leaking into what's supposed to be pure insert throughput) that the harness design closes off explicitly rather than by accident.
- **Release-profile discipline.** LTO and a single codegen unit in the workspace's release profile, and a README that states plainly: debug-mode timing numbers aren't meaningful for cross-algorithm comparison, full stop.

```
$ cargo run --release --bin bench -- sweep \
    --cardinality 100000 --stream-length 2000000 \
    --skew 0.8,1.2 --memory-budgets 4096,65536,1048576 \
    --top-k 20 --trials 3 --seed 42 \
    --output results/sweep.csv
```

`--cardinality`, `--skew`, and `--memory-budgets` accept comma-separated lists and run as a full cross product against all algorithms, three trials each — three trials specifically because HeavyKeeper's probabilistic count-decay makes it non-deterministic given a fixed stream, unlike CMS or Space-Saving, and a single-trial run would silently pass off noise as signal.

## Phase 2: PeelSketch, and a rigorous negative result

Phase 2 designed and implemented a novel fourth sketch, PeelSketch — a statistics-only purity heavy-hitter sketch — wired into the same harness as a fourth `Algorithm` variant, with a pre-registered fallback-trigger criterion written into the design spec *before* the comparison ran: if PeelSketch isn't better than HeavyKeeper anywhere in the swept grid — dominated everywhere, or indistinguishable from trial-to-trial noise — the fallback trigger is considered met.

It was met. Across all six (skew, memory-budget) cells in the comparison grid, PeelSketch never beat HeavyKeeper by more than the noise floor: three cells were exact ties at F1 = 1.0 (both algorithms saturating at generous memory budgets, nothing left to differentiate), and the other three showed PeelSketch losing outright, worst at the tightest budget and lowest skew (F1 0.367 vs. HeavyKeeper's 0.850). At that same worst cell, PeelSketch also lost to plain Count-Min Sketch — not just the intended HeavyKeeper comparison — by roughly 3.3x on mean relative error.

Rather than stopping at "it didn't win," the results doc root-causes why, from the sweep's own diagnostic columns: PeelSketch's 12-byte cell (`fingerprint`, `vote_margin`, `raw_total`, all `u32`) buys roughly a third of Count-Min's columns-per-row at the same memory budget, and its purity-refinement logic only ever fires for *minority* candidates in a cell — the majority branch, which is what actually scores the heavy items that top-k accuracy depends on, returns the raw unfiltered count with none of the refinement applied. So on the metric that matters, PeelSketch behaves like a narrower, worse-provisioned Count-Min Sketch: all of the extra width cost, none of the purity benefit where it would count. That's a concrete, falsifiable explanation for the loss, not a shrug — and it's the input the next design (an auxiliary key-tracking structure, not a further tweak to the current statistics-only approach) has to budget its own per-cell cost against, or repeat the same mistake.

## Verification

`cargo test --workspace` runs unit and property tests on every push/PR (invariants like Count-Min never undercounting, Space-Saving's `estimate - error <= true_count <= estimate` bound, HeavyKeeper decay traced against a hand-computed small example) — deterministic, no dependency on a live sweep. The comparison numbers above are a real, `--release` sweep run (`results/phase2_comparison.csv`), not illustrative figures.

## What's not in this write-up

An adversarial "plateau" workload generator — meant to stress-test whatever auxiliary structure eventually replaces PeelSketch's approach — is in active design on a separate branch, spec and implementation plan only, not yet merged or built. Left out here rather than described as done.

## Try it

```bash
cargo run --release --bin bench -- sweep \
  --cardinality 100000 --stream-length 2000000 \
  --skew 0.8,1.2 --memory-budgets 4096,65536,1048576 \
  --top-k 20 --trials 3 --seed 42 \
  --output results/sweep.csv

python3 scripts/plot.py results/sweep.csv --out charts/
```

Full README and source: [github.com/tahovig/sketch-ops](https://github.com/tahovig/sketch-ops).
