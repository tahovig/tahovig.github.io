---
layout: page
title: "poc-emstime — Grid Timing Anomaly Detection"
permalink: /writeups/poc-emstime/
nav_exclude: true
---

*Repo: [github.com/tahovig/poc-emstime](https://github.com/tahovig/poc-emstime) — fourth in a series of portfolio projects supporting a pivot from software engineering into cybersecurity engineering. Long-term goal: satellite clock (GPS) anomaly detection for power-grid time synchronization infrastructure.*

## The problem

A Python ML pipeline for sub-second power-grid time-series data: ingest real micro-PMU (µPMU) synchrophasor data, inject labeled timing-fault anomalies (GPS jitter, signal dropout, clock-step offsets, time-quality-flag corruption), engineer rolling/gap-aware features, and score an Isolation Forest anomaly detector against the injected ground truth. Unlike the three CLI-only projects before it, this one also builds a full web app layer on top — a FastAPI backend and a React/TypeScript SPA for running the pipeline and reviewing results.

This write-up focuses on the part that makes this project different from the others: it doesn't stop at "the detector works," it goes on to independently validate that claim against real data the detector wasn't tuned on — and reports honestly what that validation actually found.

## Phase 1: does it work on data it was tuned against?

Run against ~1.13 hours of real LBNL `a6_bus1` µPMU data (487,667 rows, 120Hz), with one injected fault of each type:

| fault type | caught at all (window level) |
|---|---|
| `timestamp_jitter` | yes |
| `tq_corruption` | yes |
| `dropout` | yes |
| `clock_step` | yes |

Getting to that table surfaced two real bugs, not tuning:

1. **sklearn's default row-subsampling made rare faults nearly invisible.** `IsolationForest`'s default caps each tree's training subsample at 256 rows; against 487,667 rows, a 3-4-row fault has under a 1-in-500 chance of landing in any tree's subsample, so almost no tree ever learns to split around it — even for a fault ~400 standard deviations from baseline. Raising `max_samples` to the full dataset took detection on two fault types from 0% to 100%.
2. **Two fault types weren't undetected — they were mislabeled.** Checking the model's raw anomaly scores (not just the thresholded flags) showed the `dropout` and `clock_step` boundary rows ranked 1st and 2nd most anomalous out of all 487,667 rows — near-perfect catches the scoring was simply looking one row away from, since a gap or clock-step's evidence only appears once time resumes to normal, one sample after the labeled window's `end`. Fixed by padding the fault labels themselves, not by loosening the detector.

Scaled up to a full real day (10,367,998 rows) with no rework needed beyond the compute cost (~10 min single-threaded fit, unchanged detection quality) — memory was the actual constraint (the full 120M-row two-channel dataset OOM'd at the ingestion/merge step alone on this project's 7.6GB-RAM environment), addressed by scoping to a bounded one-day slice rather than a bigger rewrite.

## Phase 1.5: does it work on data it's never seen?

Every result above uses fault labels this project injected itself — the detector was tuned and graded on the same synthetic ground truth. To test whether it actually generalizes, it was run against the [Grid Event Signature Library](https://gesl.ornl.gov) (GESL), an ORNL/PNNL-hosted, DOE-funded repository of real, independently-labeled US utility PMU data: 14 real `Instrument::Timing::*`-tagged signatures (genuine GPS/clock-error events) against 15 non-Timing signatures as a negative control.

**First result:** 14/14 Timing signatures flagged (100% recall) — and 15/15 negative controls also flagged (100% false-positive rate). Not a validated detector; a validated blind spot. Root cause: `IsolationForest(contamination=0.01)` was being refit independently per channel per signature, and contamination is relative, not absolute — it forces roughly the most-extreme 1% of *that channel's own data* to be called anomalous regardless of whether the channel is actually broken. Any channel with enough rows gets flagged by construction.

**Follow-on fix:** fit each measurement type's model once on a held-out "known normal" reference, then score every test channel against that fixed baseline instead of refitting per-channel. Recall dropped to 7/14 (50%) — some real clock-error signatures no longer cleared the fixed bar — but the negative-control false-positive rate didn't move: still 15/15. Digging into per-channel flag rates (not just "any channel") found why: current-angle channels were flagged near-universally (98.7%) because their absolute values are site/load-relative and don't pool meaningfully across 20 different physical PMU locations, while genuinely grid-wide quantities like frequency sat in a plausible 18-25% range.

**Second follow-on, two changes isolated separately:**

| | `any()`, raw pooling | graded threshold, raw pooling | graded threshold + per-channel normalization |
|---|---|---|---|
| Timing recall | 7/14 (50%) | 7/14 (50%) | 14/14 (100%) |
| Negative-control FP rate | 15/15 (100%) | **11/15 (73%)** | 15/15 (100%) |

The graded threshold alone (scoring by *fraction* of channels flagged instead of a single `any()` verdict) was real, verified progress — same recall, false-positive rate down from 100% to 73%. Per-channel normalization looked like it fixed everything (100% recall) but was verified, not assumed, to have done so for the wrong reason: it saturated *every* channel type to 89-99% flagged on negative controls, including ones that never had the current-angle problem in the first place, because normalizing every channel to its own mean/std throws away the fact that frequency and voltage magnitude *are* legitimately comparable in absolute terms across sites. **That change was reverted** rather than shipped, and the actually-correct fix — normalizing only the site-relative channels, leaving grid-wide ones pooled as raw values — is scoped as unattempted follow-on work rather than left looking finished.

## Why this is the interesting part

Three consecutive rounds of "does this fix work" testing here, and the honest answer twice was "partially, and here's specifically what's still broken" rather than declaring victory at the first improved number. That's the same standard the rest of the portfolio holds to (verify against real data, don't paper over a limitation) — this project just had a result complex enough to make the discipline visible.

## The app layer

A FastAPI backend and React/TypeScript SPA sit on top of the Phase 1 pipeline — this series' first project with a real UI instead of a CLI. A single background worker thread processes one run at a time from a bounded queue (concurrent ~10-minute model fits don't fit this environment's RAM), and progress streams live over Server-Sent Events rather than a blocking spinner, since a real run takes real time. Chart data is server-side decimated once, right after scoring, with one representative anomalous row force-included per bucket so no anomaly-containing region is lost to downsampling — verified against a real 10.37M-row run end-to-end in a headless browser, including catching and fixing a scaling bug where the decimator's anomaly-inclusion logic scaled with `contamination × row count` instead of the intended bucket count.

![Run detail page showing a completed 10.37M-row run, metrics, fault-type table, and anomaly chart](https://raw.githubusercontent.com/tahovig/poc-emstime/main/docs/screenshots/run-detail.png)

*Real screenshot — a completed run against the full 1-day, 10.37M-row dataset, captured by driving the actual running app in a headless browser.*

## Try it

```bash
cd code
python3 -m venv poc-emstime-venv
source poc-emstime-venv/bin/activate
pip install -e ".[dev]"
pytest
```

Full README, GESL validation detail, and source: [github.com/tahovig/poc-emstime](https://github.com/tahovig/poc-emstime).
