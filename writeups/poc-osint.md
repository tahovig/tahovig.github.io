---
layout: page
title: "poc-osint — Automated Subdomain Recon & Attack-Surface Drift Detection"
permalink: /writeups/poc-osint/
nav_exclude: true
---

*Repo: [github.com/tahovig/poc-osint](https://github.com/tahovig/poc-osint) — first in a series of portfolio projects supporting a pivot from software engineering into cybersecurity engineering.*

## The problem

Every org's real external attack surface tends to drift ahead of its inventory: a `dev.` or `staging.` subdomain someone spun up for a demo two years ago, still resolving, still running whatever was on it the day it was abandoned. That's the "shadow IT" gap `poc-osint` targets — it models the first thing a threat-intel analyst or pentester actually does against a target domain: find what's publicly reachable, check what's alive, and see what it's exposing.

The tool is a Python CLI with three phases — subdomain enumeration, liveness checking, and header/fingerprint analysis — plus a fourth piece that turns single scans into an ongoing signal: saved-scan comparison, so "what changed on our attack surface since last time" has a real answer instead of a guess.

**Authorized use only.** It performs active checks (HTTP requests to discovered hosts), not just passive lookups — the target is always an explicit CLI argument, never a bundled or default list, and the README says so up front.

## How it works

**1. Subdomain enumeration.** Queries [crt.sh](https://crt.sh)'s certificate transparency logs — passive, public data, no interaction with the target itself. This is where the project's most interesting engineering decision shows up (below).

**2. Liveness check.** Concurrent async HEAD checks against ports 80/443, behind a `asyncio.Semaphore`-bounded `--max-concurrency` (default 10) and a configurable `--delay` between requests. This isn't just a performance knob — it's a deliberate ethics/engineering point: nothing about mapping a domain's attack surface requires hammering it.

**3. Header analysis.** For live hosts: a fixed five-header security checklist (`CSP`, `HSTS`, `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`) plus `Server`/`X-Powered-By` fingerprint matching against a short signature list (Apache, nginx, IIS, PHP, ASP.NET, Express, Cloudflare). Deliberately scoped — a fixed checklist and a short signature list, not an open-ended Wappalyzer clone. Broader fingerprinting is documented as future work rather than half-built now.

Real output, a genuine run against `example.com` (`✓`/`✗`/`-` = present / missing / not-applicable-because-dead):

```
HOST                 | PORT | LIVE | STATUS | SERVER     | CSP | HSTS | XFO | XCTO | RP | FINGERPRINT
---------------------+------+------+--------+------------+-----+------+-----+------+----+------------
dev.example.com      | 80   | no   | -      | -          | -   | -    | -   | -    | -  | -
example.com          | 80   | yes  | 200    | cloudflare | ✗   | ✗    | ✗   | ✗    | ✗  | Cloudflare
www.example.com      | 443  | yes  | 200    | cloudflare | ✗   | ✗    | ✗   | ✗    | ✗  | Cloudflare
```
(trimmed — full run also flags `m.`, `products.`, and `support.` as dead via DNS resolution failure)

## crt.sh is flaky — so don't just retry, route around it

Development testing hit real crt.sh HTTP-frontend flakiness: alternating 200/502 responses seconds apart against the *same* query, confirmed independent of the client via raw `curl`. Retry-with-backoff is the obvious fix and it's in there — but it's a mitigation, not a solution, for an upstream that's just unreliable sometimes.

So the project went looking for a second path to the same data rather than accepting degraded reliability:
- **Spyse** — dead since 2022, ruled out immediately.
- **Google's undocumented CT Report endpoint** — used by some OSINT tools, but no stability guarantee, rejected as a foundation for something meant to demonstrate engineering rigor.
- **crt.sh's own public read-only Postgres instance** (`guest`, no password, trust auth, port 5432) — real, documented, the same approach tools like `quickcert` use specifically to bypass the flaky HTTP frontend while reading identical, authoritative data.

The Postgres path is wired in as an automatic fallback: `get_subdomains()` tries HTTP first, falls through to Postgres only on failure, and raises (naming both failures) only if neither works. Verified live, twice — once as a direct call, once by forcing the HTTP path to fail and confirming the fallback actually engages end-to-end, landing on the identical result set (after filtering out the CA-name and email-SAN noise the looser Postgres full-text query surfaces, which the HTTP path doesn't return).

## Built to be tested

Fetch/network code (`crtsh.py`, `liveness.py`) is kept separate from pure parsing/logic (`headers.py`, `compare.py`) specifically so the test suite never depends on live network or a real target — `respx` mocks the HTTP layer, a fake connection object mocks `asyncpg`. 63 unit tests, 0 network calls, all passing. A second, smaller suite (6 tests) runs against real Docker containers — a `healthy-host` fixture with the full header checklist and a `vulnerable-host` fixture with none — to verify the pipeline behaves correctly against something real, not just mocks. Both suites are wired into CI on every push (`unit` on a 3.11/3.12 matrix, `integration` against the Docker fixtures), and both were confirmed green on GitHub, not just locally.

## A real bug: a status message that could never render

While adding more specific progress messaging (e.g. surfacing the crt.sh → Postgres fallback live), a message set immediately before calling into a function that itself sets a *different* message — with no `await` between them — turned out to be unable to ever actually render. Python's asyncio has no preemption between `await` points, so the spinner task never got a scheduling opportunity to draw the first message before it was overwritten.

The fix was to consolidate into one message emitted right before the first genuine `await` (the real Postgres connect), guaranteeing a visible window. One case was left as a documented limitation rather than "fixed": the header-analysis stage message, since that work is synchronous over already-fetched data with no natural slow point — inserting an artificial delay just to make a cosmetic status flash would be dishonest UX, not a fix. Verified for real under a pseudo-tty (`script -qec`, since piped/captured output isn't a tty and would silently skip the spinner entirely) — frames genuinely cycle, a live "10/12 complete" counter is caught mid-run, and piped `--json` output produces zero stderr bytes.

## Turning single scans into a drift signal

`lookup --save scans/2026-01-01.json` always persists the result as JSON regardless of the display format; `compare old.json new.json` diffs two saved scans and reports what actually changed:

```
+ ADDED    www.example.com:80
+ ADDED    www.example.com:443
- REMOVED  legacy.example.com:80
~ CHANGED  example.com:443  missing headers: 0 -> 5
```

This is the tool's direct answer to the shadow-IT problem it opened with — not a bolted-on visualization, but the same `(host, port)`-keyed comparison an analyst would do by hand, automated. `compare_reports()` deliberately works on plain loaded-JSON dicts rather than reconstructed model objects, since dict equality is all a diff needs.

## What's deliberately not done

- Broader server/CMS fingerprinting beyond the current short signature list — documented as future work rather than scope-crept in now.
- A real-target demo beyond `example.com` — evaluated real options (OWASP has no VDP covering owasp.org itself; Tesla's Bugcrowd program only offers partial safe-harbor with a 24h disclosure obligation, a poor fit for repeated demo runs; NASA's VDP is legitimate but scoped to a specific target list) and deferred, since a live, real-DNS smoke target (`example.com`) already proves the full pipeline end to end.

## Try it

```bash
cd code
python3 -m venv poc-osint-venv
poc-osint-venv/bin/pip install -e ".[dev]"
poc-osint-venv/bin/poc-osint lookup example.com
```

Full README and source: [github.com/tahovig/poc-osint](https://github.com/tahovig/poc-osint).
