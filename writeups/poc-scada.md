---
layout: page
title: "poc-scada — DNP3 Protocol Deep-Packet Inspection"
permalink: /writeups/poc-scada/
nav_exclude: true
---

*Repo: [github.com/tahovig/poc-scada](https://github.com/tahovig/poc-scada) — third in a series of portfolio projects supporting a pivot from software engineering into cybersecurity engineering, grounded in real critical-infrastructure-protection (energy generation/transmission/distribution, EMS) experience.*

## The problem

A Rust CLI that performs deep-packet inspection on DNP3 traffic — the protocol SCADA/ICS systems use to poll and control field equipment — reading captured `.pcap` files offline and flagging two specific, security-relevant patterns rather than trying to be a general DNP3 analyzer:

- **Dangerous function codes** — Cold Restart, Warm Restart, Direct Operate, Direct Operate No Ack. Rare in normal polling traffic (which is dominated by Read/Response), so their presence is a meaningful signal on its own.
- **Select-before-operate violations** — an Operate not immediately preceded by a Select from the same master/outstation pair, breaking the safety pattern real utility control systems are built around.

```
$ cargo run -- data/dnp3-iti/cold_restart_and_response.pcap
poc-scada — DNP3 DPI report
file: data/dnp3-iti/cold_restart_and_response.pcap
------------------------------------------------------------
  [!] dangerous-function-code    t=1422552945 192.168.60.1:49423 -> 192.168.60.130:20000 ColdRestart function code seen (dnp3 dest=10, src=1)

2 packet(s) analyzed, 1 finding(s)
```

## Why Rust, and why offline-only

Rust was a deliberate choice over the faster-to-ramp-up Python/`scapy` route: parsing untrusted, potentially malformed binary protocol data from a pcap is exactly the class of problem where memory-corruption bugs are a realistic risk in C/C++, and where Rust's safety guarantees are a genuine differentiator for a security-tooling portfolio piece. There's no mature Rust DNP3 parser, so the parser is hand-rolled — just enough of the link, transport, and application layers to support the two detections above; object-header parsing and multi-frame transport reassembly are explicitly out of scope for v1, documented as such rather than half-built.

The tool is offline/batch-only by design, not as a limitation to fix later: unlike `poc-logids`'s SSH honeypot, there's no equivalent live ICS network available to tap here, and staying offline also sidesteps the packet-capture privilege requirements (root/`CAP_NET_RAW`) live capture would need.

## A UI detour, reverted on purpose

A Tauri desktop UI was prototyped on top of the analysis backend, then abandoned in favor of a richer terminal presentation instead — findings colored by rule, a background spinner while a file is analyzed. The spinner itself is a deliberate design constraint: it's "still working" feedback, not a fake progress bar. `analyze_pcap` doesn't report incremental progress, so the spinner just ticks on a background thread for as long as the call is in flight and joins the moment it returns — building a real per-packet progress bar wasn't worth the API surface it would add, and doing it anyway would repeat the same mistake the Tauri attempt taught: don't add motion or polish that isn't earning its keep for what's fundamentally a fast batch tool.

## Validation data had to be evaluated, not assumed

The public 4SICS ICS Lab captures were checked as a possible detection-validation source and mostly rejected: one has zero DNP3 traffic, one's port-20000 traffic turned out to be scanner noise (Oracle TNS, raw HTTP) rather than DNP3, and the third has only 36 genuine DNP3 frames — all link-layer status probes with no application-layer payload, so no Select/Operate/Restart traffic exists in it at all. It's kept as a non-DNP3-heavy background-noise fixture (confirming the tool doesn't false-positive on realistic traffic that isn't DNP3), while the actual detections are validated against [ITI/ICS-Security-Tools](https://github.com/ITI/ICS-Security-Tools)'s per-function DNP3 pcaps instead — purpose-built, one-function-per-file captures.

## Design decisions worth calling out

- **Select-before-operate deliberately doesn't flag Direct Operate.** Direct Operate (function code 5) bypasses the select/operate handshake by design — that's what the function code means, not a bug in a captured session — so flagging it under both rules would double-count the same event. It's covered once, by the dangerous-function-code check.
- **Zero-copy, streaming parsing, even though the current fixtures don't require it.** `LinkFrame` borrows from the original packet bytes instead of allocating per frame, and pcap reading decodes one packet at a time as an iterator rather than loading the whole capture into memory first. Not needed for the small fixture pcaps in the repo today, but the honest way to build a DPI tool meant to eventually point at a multi-hour real SCADA capture that shouldn't need to fit twice in RAM just to be scanned once.
- **Link/block CRCs are stripped during reassembly but not validated** — a documented, known limitation rather than a silent gap. Fine for well-formed fixture data; a real capture with bit errors or a deliberately malformed frame could currently parse as something other than what was actually on the wire.

## Verification

CI runs unit and integration tests against the ITI fixtures on every push. "2 packet(s) analyzed" on a file that looks non-trivial on disk is expected, not a bug — the packet counter only counts TCP/UDP packets carrying a non-empty payload, correctly excluding TCP handshake/ACK/teardown packets from purpose-built single-function captures.

## Try it

```bash
cargo build --release -p poc-scada-cli
./target/release/poc-scada data/dnp3-iti/cold_restart_and_response.pcap
```

Full README and source: [github.com/tahovig/poc-scada](https://github.com/tahovig/poc-scada).
