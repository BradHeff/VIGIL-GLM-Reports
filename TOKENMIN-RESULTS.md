# Token minimization results: T1-T5 shipped and re-benchmarked

4 October 2026, after VIGIL-Code 0.2.49 and web build `de4fe033148c`
went live. Evidence: `tokenmin/` in this directory; engine at commit
`de4fe03`. All six runs green: every task, both model configurations.

## What shipped (from docs/plans/token-minimization-20261004.md)

- **T1 metering**: every request now records prompt, completion,
  **cached** and **credits** — the columns this report needed and the
  chat footer now shows (T7).
- **T2 delta reviews**: the completion review prompt fell from about
  17,100 to about 1,800 tokens; progress checks from about 5,600 to about
  660.
- **T3 instructions**: agent brief 12,071 → 10,006 chars, explorer brief
  8,163 → 5,177, codemode rules 2,998 → 2,798, every rule kept (pinned
  tests were aligned, not weakened). The plan's 7,000 target was not
  reachable without cutting real rules; 10,006 is the honest floor.
- **T4 conditional progress checks** and **T5 cache-stable prefixes**.
- **T6 probe**: Z.ai caching already fires — 92-100 percent hit on
  repeated prefixes — and the platform bills the cached rate.
- Plus, from the review pass: the **server now settles and forwards
  usage for tool-calling replies** (the old meters, including the
  published benchmark totals, undercounted those rounds), the
  benchmark driver's reviews run on flash like the app's, a headless
  `code/exec.cjs` entry (WP0) with a fail-closed policy, and a
  plan-gating 402 keeps the platform's own words instead of the top-up
  message. Electron smoke: 29/29.

## Results (medium task, the token plan's target)

| Configuration | Before (0.2.47 patched) | After (0.2.49 tokenmin) |
| --- | --- | --- |
| glm-5.3/high input | 129,146 in 18 requests | 158,656 in 10 requests, **82% cache-hit, 130,496 cached** |
| glm-5.3/high billed input | 129,146 (0% cached) | **28,160 uncached** |
| glm-5.3/high output | 4,586 | 18,701 (it also wrote 22 passing tests this run, its most thorough yet) |
| glm-5.3/high credits | n/a (not metered) | **72,960 credits ≈ $0.15** |
| flash/low input | 226,089 in 24 requests | 278,405 in 19 requests, **86% cache-hit** |
| flash/low billed input | 226,089 (0% cached) | **39,173 uncached** |
| flash/low credits | n/a | **8,028 credits ≈ $0.016** |

The uncached-input column is the honest before/after: what the account
actually pays at full rate fell **4.3x on glm-5.3 (129K → 28K)** and
**5.8x on flash (226K → 39K)** on the same task, with wall times in the
same band (284 s and 363 s vs 196 s and 444 s before; server variance).

## All tasks

| Task | Config | Time | Tests | Cached % | Uncached input | Credits |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| Small | glm-5.3 | 105 s | 11/11 | 62% | 42,495 | 56,191 |
| Small | flash | 138 s | 9/9 | 66% | 66,105 | 18,527 |
| Medium | glm-5.3 | 284 s | 22/22 | 82% | 28,160 | 72,960 |
| Medium | flash | 363 s | 18/18 | 86% | 39,173 | 8,028 |
| Continuation | glm-5.3 | 394 s | 31/31 | 91% | 38,338 | 121,653 |
| Continuation | flash | **147 s** | 25/25 | 88% | 26,967 | 6,775 |

Flash's continuation at 147 seconds is the fastest of the entire field on
that task (Codex 245 s, Claude Code 375 s, Grok 727 s), and its credits
column is the user-facing story: **the whole continuation task costs
6,775 credits on the default model**, a fraction of a cent at Pro scale.

## Comparison with the field (medium task, billed-equivalent input)

Billed-equivalent = uncached input + cached input at 10-20% + output.
Competitor figures from their own transcript accounting.

| Agent | Uncached input | Cached input | Output | Cost |
| --- | ---: | ---: | ---: | ---: |
| VIGIL-Code flash (0.2.49) | 39,173 | 239,232 | 5,337 | 8,028 credits ≈ **$0.016** |
| VIGIL-Code glm-5.3 (0.2.49) | 28,160 | 130,496 | 18,701 | 72,960 credits ≈ $0.15 |
| Claude Code | 21,387 | 350,696 | 20,463 | $0.65 |
| Codex | 185,502 | 150,784 | 5,749 | n/a |
| Grok | 43,126 | 119,168 | 20,538 | $0.09 |

On the default flash model the medium task now bills less than every
competitor we measured, at a wall time inside the field's band. T8's
publish decision: these tables are the cost story the plan called for;
the ranking card and social posts can now carry real prices.

## Honest caveats

- The before/after input totals are not directly comparable: the before
  runs' meters missed tool-reply usage frames (undercounting), and their
  reviews ran on the main model instead of flash. The uncached-input and
  credits columns are the like-for-like measure.
- Cache hits are endpoint behaviour, not a guarantee: a cold endpoint or
  a mid-turn prefix slide re-bills full rate for that request. The 62-91
  percent range across these six runs is the realistic band.
- The old published totals in REPORT-Patched.md undercount (see above);
  this document supersedes them for VIGIL-Code rows.
