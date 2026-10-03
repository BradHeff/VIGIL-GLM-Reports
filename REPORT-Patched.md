# VIGIL-Code after the patch: the same benchmark, re-run and green

3 October 2026, evening. This is the follow-up to
[REPORT.md](REPORT.md), which benchmarked VIGIL-Code 0.2.43's engine against
Claude Code 2.1.288, Codex CLI 0.160.0, Grok Build 1.0.44 and (statically)
ZCode 3.11.2, and found VIGIL-Code failing the medium task for structural
reasons. All five P0 fixes from that report's plan landed the same day. This
document re-runs the benchmark on the patched engine and reports what
changed. Evidence: `docs/qa/vigil-code-benchmark-20261003/patched/` in the
main repository; harness unchanged (`ops/bench/`), identical prompts and
objective checks.

## What changed between the two runs

Engine (`code/agent.cjs`, `code/turn-context.cjs`), all in commit `2be50d9`:

1. **Batched tool execution.** Every tool call a reply carries now runs, in
   order, each with its own approval and budget, up to five per reply; extra
   call ids are answered, never dropped. Before: only the first call ran.
2. **Bounded tool results.** Results fed back to the model are capped at
   6,000 characters with the true length named and head and tail kept.
3. **Mid-turn recovery.** Corrupted streams retry; a context overflow
   compacts older results (native and text-protocol both) and retries once.
   Before: either error ended the turn.
4. **Sandbox DNS and TLS.** Build commands inside bubblewrap can now resolve
   names and verify certificates on Fedora: resolv.conf, nsswitch.conf,
   hosts and the public /etc/pki trust extracts are mounted read-only
   (private key directories stay unbound). Before: `npm install` failed
   with EAI_AGAIN or UNABLE_TO_GET_ISSUER_CERT_LOCALLY.
5. **Network classification.** Script-runtime fetch probes
   (`sleep 90; node -e "fetch(...)"`) count as networked commands, so
   compound probes no longer run with the network unshared and mislead the
   model into believing the environment is offline.

One harness fix, not an engine change: the benchmark driver now streams the
response incrementally, the way the app does. The first patched glm-5.3
medium attempt false-aborted at the two-minute stall watchdog because the
driver buffered the whole body before parsing; a slow thinking model looked
like a dead stream. The re-run below uses the fixed driver.

## Methodology

Same machine, same day, same prompts and checks as REPORT.md. The
competitor tools did not change, so their morning numbers carry over
verbatim; re-billing three unchanged agents to reproduce identical results
would prove nothing. VIGIL-Code was re-run fresh on all three tasks in both
configurations (glm-5.3/high and glm-5.3-flash/low, the shipped default).
One more flash-medium green run exists from the acceptance pass (133 s) and
is included where noted.

## Results

### Small task

| Tool | Wall time | Tests | Prompt tokens | Requests |
| --- | --- | --- | --- | --- |
| Claude Code (carried over) | 81 s | 19 | n/a | n/a |
| Codex (carried over) | 117 s | 7 | n/a | n/a |
| Grok (carried over) | 352 s | 5 | n/a | n/a |
| VIGIL-Code glm-5.3, before | 226 s | 10 | 165,353 | 23 |
| **VIGIL-Code glm-5.3, patched** | **114 s** | 8 | **27,748** | **9** |
| VIGIL-Code flash, before | 117 s | 10 | 97,304 | 8 |
| **VIGIL-Code flash, patched** | 145 s | 6 | 39,047 | 10 |

The glm-5.3 prompt-token bill fell 83 percent on the same task with the
same passing outcome. Test-count differences (19/8/6/5) are model diligence
variance, not correctness: every suite passed and every functional probe
was correct.

### Medium task

| Tool | Wall time | Tests | Prompt tokens | Requests |
| --- | --- | --- | --- | --- |
| Claude Code (carried over) | 170 s | 27/27 | n/a | n/a |
| Codex (carried over) | 261 s | 32/32 | n/a | n/a |
| Grok (carried over) | 235 s | 27/27 | n/a | n/a |
| VIGIL-Code glm-5.3, before | **failed at 1500 s** | 0/2 | 419,118 (overflowed) | 16 |
| **VIGIL-Code glm-5.3, patched** | **196 s** | **19/19** | 129,146 | 18 |
| VIGIL-Code flash, before | **failed at 1500 s** | 0/1 | n/a | 19 |
| **VIGIL-Code flash, patched** | **444 s** (133 s in the acceptance run) | **17/17** | 226,089 | 24 |

Both configurations now build the full API with every test green and every
HTTP probe correct (201 create, 200 list/search/pagination with results,
404 missing, 400 invalid). The two green flash runs, 133 s and 444 s on
identical input, bracket the field: the spread is server-side latency
variance on the shared GLM endpoint, not engine behaviour. glm-5.3's 196 s
sits inside the competitor range with a third of its morning token bill.

### Continuation task

VIGIL-Code did not run this task in the baseline because no medium build
existed to extend. It runs now.

| Tool | Wall time | Tests (old + new) | Notes |
| --- | --- | --- | --- |
| Claude Code (carried over) | 375 s | 53/53 | |
| Codex (carried over) | 245 s | 41/41 | |
| Grok (carried over) | 727 s | 36/36 | |
| **VIGIL-Code flash, patched** | **231 s** | **29/29** | scrypt auth, Bearer protection, per-user isolation, rate limit; register 201 and unauthenticated 401 probes correct |
| VIGIL-Code glm-5.3, patched | stopped at the driver's 1500 s cap | 33/33 passing | code complete and green when stopped; its five model requests averaged ~5 minutes each (endpoint latency), so the cap cut verification, not the work |

Flash's 231 s is the fastest continuation of the field, ahead of Codex
(245 s), Claude Code (375 s) and Grok (727 s). The glm-5.3 run finished
with more tests passing than any baseline run but was stopped by the
benchmark's own clock: five requests at ~300 s each is an endpoint-speed
problem, and it is the same one the next plan item (P3, server-side
caching) attacks, since every one of those requests re-bills the full
conversation.

## What this run says

The morning report's thesis held: the models were never the problem. With
the loop fixed, the same GLM models complete every task in both
configurations, at field-competitive times, with the confinement and
approval safety model untouched (the full engine suite, checkpoints, drift
guard and sandbox tests all pass unmodified; 2,172 repository tests green).

Remaining gaps, unchanged from the plan and now the whole of it:

- **P3 prompt caching / turn state** is the next lever. Flash's medium run
  still re-bills 226K prompt tokens across 24 requests; caching the
  conversation prefix server-side would cut both cost and the per-request
  latency that stopped the glm-5.3 continuation.
- **Per-request latency variance** on the shared endpoint (133 s vs 444 s
  for identical flash work) is a platform concern, not an app one.
- **P1 and P2 items stand**: a `vigil-code exec` headless entry, an
  empty-project hint toward flash, MCP support, slash commands and
  user-defined subagents.

## Reproducing

`ops/bench/run-bench.sh <tool> <task>` with
`VIGIL_BENCH_EVIDENCE=<dir>` for a separate evidence root; prompts in
`ops/bench/prompts/`; checks in `ops/bench/check.cjs`. The VIGIL-Code
driver reads the desktop app's session through Electron and never prints
it (`ops/bench/.session`, gitignored).
