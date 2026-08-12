---
layout: page
title: "poc-logids — Log-Based SSH Brute-Force Detection"
permalink: /writeups/poc-logids/
---

*Repo: [github.com/tahovig/poc-logids](https://github.com/tahovig/poc-logids) — second in a series of portfolio projects supporting a pivot from software engineering into cybersecurity engineering.*

## The problem

A "mini IDS" for log analysis: a Go CLI that parses `auth.log`-style SSH logs and flags brute-force bursts — repeated failed-auth attempts from a single source within a time window. The interesting part isn't the detection logic itself (that's a straightforward threshold-over-a-window), it's what it took to make that logic hold up against real logs and real-time monitoring rather than a clean synthetic demo.

## Two modes, two different notions of "done"

**Batch mode** scans a static file and reports each burst once it's fully over — the right behavior for after-the-fact log analysis. **`-follow` mode** (`tail -f`-style) keeps watching a live file and needs the opposite behavior: it alerts the instant a source crosses `-threshold`, not after the burst ends, because waiting for an in-progress attack to stop before reporting it defeats the point of real-time monitoring. That's not a flag on the same detector — it's a genuinely separate streaming detector (`internal/detector.Live`), because the two use cases have incompatible definitions of "when do you know."

Real output from a full run against [loghub](https://github.com/logpai/loghub)'s public real-world Linux syslog dataset — genuine production data spanning 263.9 days, not synthetic fixtures:

```
SOURCE                                               ATTEMPTS  RATE       FIRST SEEN       LAST SEEN        USERS TRIED
150.183.249.110                                      80        50.5/min   Jul 10 16:01:43  Jul 10 16:03:18  root
220.82.197.48                                        80        53.3/min   Aug 29 07:22:24  Aug 29 07:23:54  root
85.17.1.3                                            64        45.2/min   Oct 15 15:49:47  Oct 15 15:51:12  root
...
(327 alerts total: 25 critical, 207 warning, 95 normal)
```

## Three bugs that only show up against real data

**Year inference across a real calendar boundary.** Classic syslog timestamps (`Jun 14 15:16:01`) carry no year. The parser increments an internal year counter whenever a line's month is earlier than the previous line's — but that heuristic is only trustworthy because the loghub dataset genuinely crosses a Dec→Jan boundary, which is exactly what surfaced and validated the logic: verified the computed gap across the real wrap comes out as seconds, not a false ~364-day jump.

**A ~50%-flaky race in log-rotation handling.** `-follow` has to survive both of logrotate's common strategies: rename+recreate, and in-place `copytruncate`. The truncation case had a genuine race — a truncate immediately followed by a rewrite can leave the file no smaller than the last read position by the time the next check runs, so a bare file-size comparison misses the rotation about half the time. Fixed with bounded trailing-content verification instead of trusting size alone.

**Severity that scales with configuration, not a hardcoded number.** A source counts as "critical" at ≥4x whatever `-threshold` is set to, not some fixed attempt count — so the tiers stay meaningful whether it's running at the default (5) or a much stricter one. Small decision, but the kind that's easy to get wrong by hardcoding the first number that worked in testing.

## Passive threat-intel enrichment, opt-in and clearly separated

`-cti` adds RDAP (network/org ownership) and ip-api.com (GeoIP/ASN) lookups for sources that repeat or exceed severity thresholds — both keyless public-registry lookups, never traffic to the source itself. It's off by default specifically because it makes outbound HTTP calls, and its output is tagged and routed to stderr (`[CTI] ...`, or a separate `"type":"cti"` JSON line), kept distinct from the primary alert stream on stdout so scripting against alerts doesn't have to filter it back out.

## Terminal output that degrades honestly

On a real terminal, rows are sorted worst-first (not chronologically) and colored by severity — critical in red, warning in yellow — so the findings most worth reviewing don't get buried in an equally-weighted list. That coloring and sorting emphasis is automatically skipped for piped/redirected output (`-json`, `| less`, a log file): no invisible ANSI escape bytes leaking into anything meant to be machine-read.

## Verification

Automated tests run against synthetic, deterministic fixtures — no live dependency. The demo/verification data is deliberately real instead: the full loghub Linux dataset for batch mode, and a live run of `-follow` against a file that had five real failed-login lines appended one every ~0.3s, to confirm the alert fires the instant the threshold is crossed rather than only appearing in a final summary. CI runs on every push.

## Try it

```bash
go build -o poc-logids ./cmd/poc-logids
./poc-logids -file auth.log -follow
```

Full README and source: [github.com/tahovig/poc-logids](https://github.com/tahovig/poc-logids).
