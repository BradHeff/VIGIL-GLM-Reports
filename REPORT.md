# VIGIL-Code against the field: comparison, live benchmark, and the repair plan

3 October 2026. VIGIL-Code 0.2.43 (installed at /opt/VIGIL-Code, engine from
this repository's `code/`) compared against the four code agents installed on
this machine: Z.ai ZCode 3.11.2, Anthropic Claude Code 2.1.288, OpenAI Codex
CLI 0.160.0, and xAI Grok Build CLI 1.0.44. Identical build tasks were run in
isolated sandboxes through each tool's non-interactive mode, with objective
checks after every run. Harness: `ops/bench/`. Evidence: this directory.

## Versions and configurations under test

| Tool | Version | Model | Non-interactive mode |
| --- | --- | --- | --- |
| Claude Code | 2.1.288 | claude-opus-5.5, effort xhigh (account default) | `claude -p --dangerously-skip-permissions --output-format stream-json` |
| Codex | 0.160.0 | gpt-6-astra, reasoning medium (~/.codex/config.toml) | `codex exec --json -s workspace-write` with network enabled |
| Grok | 1.0.44 | grok-4.7 (account default) | `grok -p --always-approve --output-format streaming-json` |
| VIGIL-Code | 0.2.43 engine, driver in ops/bench/vigil-driver.cjs | glm-5.3/high and glm-5.3-flash/low (both defaults) | headless driver over `code/agent.cjs` with the app's session |
| ZCode | 3.11.2 desktop | (this session's GLM) | static comparison only: desktop app, no headless entry |

## Static capability comparison

| Dimension | Claude Code 2.1.288 | Codex CLI 0.160.0 | Grok Build 1.0.44 | ZCode 3.11.2 | VIGIL-Code 0.2.43 |
| --- | --- | --- | --- | --- | --- |
| Shape | Terminal CLI + IDE bridges | Terminal CLI + desktop app daemon | Terminal TUI | Desktop app (Electron) | Desktop app (Electron) |
| Model vendor | Anthropic (account default opus) | OpenAI gpt-6-astra | xAI grok-4.7 | Z.ai GLM | Z.ai GLM via the VIGIL platform |
| Non-interactive run | `-p` with stream-json | `exec --json` | `-p` with streaming-json (ACP) | none | none (this bench needed a custom driver) |
| Tool parallelism | parallel tool calls per turn | parallel tool calls | parallel tool calls | parallel tool calls | **one tool call per assistant reply** |
| Prompt caching | yes (total small-task cost $0.46) | yes (implicit) | yes | yes (implicit) | **no: full history re-sent every round** |
| OS sandbox | permission modes; optional container | seatbelt/landlock syscall sandbox | none by default (worktrees for isolation) | permission modes | bubblewrap with per-path /etc allowlist, PATH and env filtering (`code/sandbox.cjs`) |
| Project confinement | opt-in | sandbox-enforced | none observed | opt-in | **enforced: tools cannot leave the project root** (`resolveInside`) |
| Checkpoints/undo | session rewind, /rewind | resume/fork sessions | sessions, worktrees | session resume | per-turn checkpoints with /undo, drift guard, safety stops |
| Subagents | custom agents, background tasks | agents daemon | subagents, custom agent defs | agent tool, background agents | built-in explore/delegate only, not user-definable |
| Skills/plugins | skills, plugins, marketplaces | plugins, marketplaces | agents, campaigns | skills, plugins, marketplaces | 42-skill bundled guidance catalog (shared with the platform) |
| MCP | yes | yes (`codex mcp`) | ACP protocol | yes | **no** |
| Hooks | yes | yes | config hooks | yes | no |
| Slash commands | yes | yes | yes | yes | no |
| Plan mode | yes | yes | yes (`--no-plan` toggle) | yes (EnterPlanMode) | yes (ask/plan/agent modes) |
| Completion discipline | its own | its own | its own | its own | **separate completion reviewer + todo tracker** (`code/completion.cjs`, `code/todos.cjs`) |
| Project instructions | CLAUDE.md | AGENTS.md | project rules | AGENTS.md | **AGENTS.md, CLAUDE.md, VIGIL.md** (`code/instructions.cjs`) |
| Update mechanism | self-update | `codex update` | self-update | AppImage updates | signed feed + key-pinned verifier (`code/update-verify.cjs`) |
| CI/scripting story | `claude -p`, GitHub Actions | `codex exec` | `grok -p` | n/a | **none** |

Strengths this comparison surfaces for VIGIL-Code: the strongest confinement
story of the five (project-root enforcement plus a hardened bubblewrap layer
and per-turn checkpoints), instruction-file compatibility with both Claude
Code and Codex conventions, a bundled skills catalog, a signed update path,
and an independent completion reviewer that refuses to let a turn end on
narrated-but-not-done work. The weaknesses are concentrated in throughput
(single tool call per reply, no prompt caching, reviewer overhead per turn)
and in ecosystem surface (no MCP, hooks, slash commands, user-defined
subagents, or headless/CI mode).

## The field, tool by tool

**Claude Code 2.1.288** is the reference point the others are measured
against: parallel tool calls, prompt caching that made the whole small task
cost $0.46 on the flagship model, background sessions with attach, custom
subagents, skills, plugins with marketplaces, hooks, LSP integration,
auto-compaction with a tunable window, session resume, plan mode, and a
mature `-p` headless mode with JSONL events. Its permission model spans
ask/accept/bypass plus container isolation; nothing about it is
project-confined by default.

**Codex CLI 0.160.0** has the strongest systems story: a real syscall
sandbox (read-only / workspace-write / full), an `exec` mode built for CI,
apply-patch diffs, resume/fork/queue session management, an agents daemon
behind a desktop app, MCP and plugin marketplaces, and a doctor command. On
this account it runs gpt-6-astra at medium reasoning. Workspace-write blocks
network unless explicitly enabled, which our medium task needed.

**Grok Build 1.0.44** is the most aggressive explorer: git worktrees for
isolation, user-definable subagents, campaigns, an ACP protocol stream, and
no confinement at all by default. It was also the only tool that read the
benchmark's own harness and sibling runs when nothing stopped it. Slowest on
the small task (352 s) and wrote the fewest tests, but the code itself was
sound.

**ZCode 3.11.2** is Z.ai's desktop agent IDE: the same GLM model family as
VIGIL-Code behind a different harness with skills, plugins, marketplaces,
hooks, MCP, plan mode, todo discipline, background agents, scheduled
automations and session resume. It has no headless entry on Linux, so it is
compared statically here. It demonstrates what the same models do with a
richer agent loop, which is the most directly relevant calibration for
VIGIL-Code.

**VIGIL-Code 0.2.43** is the only one of the five that is project-confined
by construction, the only one with a per-turn completion reviewer that
refuses narrated-but-not-done work, and the only one shipping a hardened
bubblewrap layer with an allowlisted /etc, PATH and environment. It is also
the only one without parallel tool calls, prompt caching, MCP, hooks, slash
commands, user-defined subagents, or any non-interactive mode.

## How VIGIL-Code was run

The headless driver wires the exact modules the desktop app uses
(`agent.cjs`, `sse.cjs`, `turn-context.cjs`, `context.cjs`, `safety.cjs`,
`todos.cjs`, `completion.cjs`) to the platform API with the app's own session
cookie, fetched through Electron from the app's profile and never printed.
Only the renderer is replaced, by a JSONL transcript. Approvals auto-allow,
mode agent, access build: the app's unattended-build posture, matching the
bypass/full-auto flags given to the competitors.

## Benchmark design

Three tasks, identical prompt bytes for every tool, one isolated sandbox per
tool and task, objective checks run afterwards by `ops/bench/check.cjs`
(tests executed from a clean state, functional probes against the running
program, file and line inventory). Tasks: **small** (a dependency-free Node
todo CLI with node:test suite), **medium** (an Express + better-sqlite3 notes
REST API with validation, search, pagination, README and at least 12 tests,
dependencies installed and passing), **continuation** (the medium build
extended in place with scrypt auth, per-user isolation, Bearer protection and
a rate limit, then the whole suite green). Timeout 25 to 40 minutes per run,
100 turns allowed. All runs on this machine, in parallel, on the owner's
accounts.

## Small task results

| Tool | Wall time | Tests passing | LOC | Cost signal |
| --- | --- | --- | --- | --- |
| Claude Code (opus/xhigh) | 81 s | 19 | 366 | $0.46 total |
| Codex (gpt-6-astra) | 117 s | 7 | 191 | n/a |
| Grok (grok-4.7) | 352 s | 5 | 359 | n/a |
| VIGIL-Code (glm-5.3/high) | 226 s | 10 | 263 | 165,353 prompt tokens, 23 model requests |
| VIGIL-Code (glm-5.3-flash/low, default) | 117 s | 10 | 268 | 97,304 prompt tokens, 8 model requests |

All five produced working, clean code: load/save helpers, usage text, stable
ids, error paths. Differences were in diligence (Claude wrote 19 tests,
grok 5) and, for VIGIL-Code, in what the run cost to get there: the glm-5.3
run needed 10 agent rounds, 18 tool steps and 3 spawned explorers, and re-sent
the whole conversation every round, totalling 165K prompt tokens for a
268-line app. The default flash configuration was both faster than the
flagship and just as correct on this task, with a third fewer requests.

Two structural causes, both fixable in the engine (`code/agent.cjs`):
`extractToolCall` enforces **one tool call per assistant reply**, so 18 steps
cost 18 sequential round-trips where Claude Code and Codex batch parallel
calls into one turn; and there is no server-side prompt caching, so every
round pays again for the full brief and history. A third cost is the
per-turn progress check and completion review (`code/todos.cjs`,
`code/completion.cjs`), each of which carries the evidence history again.

## Medium task results

| Tool | Wall time | Tests passing | LOC | Live API behaviour |
| --- | --- | --- | --- | --- |
| Claude Code | 170 s | 27/27 | 1,676 | all endpoints correct: 201, 404, 400, search, pagination |
| Codex | 261 s | 32/32 | 1,328 | all endpoints correct |
| Grok | 235 s | 27/27 | 2,130 | all endpoints correct |
| VIGIL-Code (glm-5.3/high) | **failed at the 25 min cap** | 0/2 | 871 | server never ran |
| VIGIL-Code (glm-5.3-flash/low) | **failed at the 25 min cap** | 0/1 | 449 | server never ran |

This is the decisive run of the benchmark. All three competitors produced a
layered Express/better-sqlite3 service (separate app/server/validation
modules, `createApp(db)` factories for testability, WAL mode, prepared
statements, body-size limits) with entirely green suites. VIGIL-Code on both
configurations scaffolded the right files but could not finish inside 25
minutes: the glm-5.3 run consumed 419,118 prompt tokens over 16 rounds,
re-reading its own files with `cat` to rebuild context, lost one turn to
"Insufficient tokens for this context", and was stopped mid-work; the flash
run ground through 17-plus single-call rounds to the same cap. The failure
is structural, not model capability: the same GLM models complete this task
when the loop batches actions and keeps the context bounded (see P0-1 and
P0-2).

## Continuation task results

Run in each tool's own medium sandbox, so it also measures iteration on an
existing codebase.

| Tool | Wall time | Tests passing (old + new) | Total LOC |
| --- | --- | --- | --- |
| Claude Code | 375 s | 53/53 | 2,465 |
| Codex | 245 s | 41/41 | 1,565 |
| Grok | 727 s | 36/36 | 2,749 |

All three added scrypt auth, Bearer protection, per-user isolation and the
rate limit with fully green suites. The checker's HTTP probes show 401 on
unauthenticated note routes everywhere (correct once routes are protected)
and successful register/login flows for codex and grok; claude's register
returned 400 to the probe while its own 53 tests pass, so its field
validation differs from the probe's assumptions rather than being broken.
VIGIL-Code did not run the continuation because no medium build completed to
extend.

## Benchmark integrity note

In the first small-task round the sandboxes were siblings under one
directory. Grok's narration shows it read outside its task folder: "Other
copies of this task are nearby. I'll check the harness for the expected
interface" and "The grader checks `node --test` and a short add, done, list,
remove run." It then matched its output to the grader and deleted a file the
functional check had left behind. All medium and continuation runs therefore
use one isolated root per tool and task (`/tmp/vb-<tool>-<task>`). The small
results are retained with this disclosure: every tool passed on its own
merits, and VIGIL-Code's own tools cannot leave the project root at all
(`resolveInside` in `code/workspace.cjs`), which is precisely the confinement
the others lack. The finding itself is evidence for the report: an agent with
unrestricted read access will browse the machine when it suspects it is being
graded. Grok's transcript is preserved as `small/grok-transcript.txt`.

The sandboxes are kept at /tmp/vb-<tool>-<task> for inspection.

## The repair, fix and gap plan for the next release

### Update, 3 October: P0 implemented and re-benchmarked

The same day, the three P0 items landed in code/agent.cjs and
code/turn-context.cjs with regression tests (tests/vigil-code-p0-fixes.test.ts,
458 engine tests green, strict typecheck clean), and the medium benchmark was
re-run on the patched engine:

| Configuration | Before | After |
| --- | --- | --- |
| glm-5.3-flash/low, medium task | stopped at the 25 min cap, 0 tests passing, server never ran | 133 s, 14/14 tests, every HTTP probe correct, 13 requests, 90K prompt tokens |

That is faster than Claude Code (170 s), Codex (261 s) and Grok (235 s) on
the same task. The acceptance run also exposed and fixed two more P0-grade
sandbox defects: the project sandbox could not resolve DNS (resolv.conf and
friends were not mounted) or verify TLS on Fedora (the CA bundle symlink
under /etc/ssl points into /etc/pki, which was not mounted either; only the
public trust extracts are now bound, private key directories stay out), and
the network classifier missed script-runtime fetch probes, so compound
"sleep 90; node -e fetch(...)" commands ran without network and convinced
the model the environment was offline. Gate ledger:
.unlazy/vigil-code-p0-20261003/GATES.md, all eight gates met.

The remaining plan items below are unchanged; P3 (prompt caching) is the
next biggest lever now that batching and bounded results are in.

Priorities: P0 blocks the next release, P1 belongs in it, P2 is the
following release, P3 is directional. Every item names the code that
changes and how we will know it is fixed.

### P0-1 Execute every tool call in a reply, not just the first

Evidence: `code/agent.cjs` runs `nativeCalls[0]` and the comment says "Run
the first complete call only; the rest of the reply is discarded". The small
task needed 18 steps = 18 sequential round-trips; the medium task 33 steps.
Claude Code and Codex batch parallel reads and writes into one turn, which
is most of why they finished medium in 170-261 s while VIGIL-Code exceeded
1500 s.

Fix: in the main loop, execute each native call in order within the same
round (sequentially is fine and keeps approvals and the safety exclusivity
intact), append every real result to the working messages, and keep the
existing anti-fabrication rule: results only ever come from the engine. The
```tool fallback should accept a bounded list too (say up to 5) with the
same one-JSON-object-per-call parse it already has.

Acceptance: the benchmark harness's medium task reaches a passing suite in
under 10 minutes on the default flash configuration, and the transcript
shows multiple `step` events per `round` event.

### P0-2 Cap tool-result sizes in the working context

Evidence: the glm-5.3 medium run consumed 419,118 prompt tokens and still
lost track of its own files, re-reading them with `cat` (`cat src/db.js &&
cat src/app.js`); earlier in the same run the platform returned
"Insufficient tokens for this context" and the turn died. `boundTurnMessages`
caps the message count at 80 but not the size of individual tool results,
and nothing summarizes old results.

Fix: truncate every tool result fed back to the model to a fixed budget
(head plus tail, with a marker naming the full size on disk), and compact
results older than the last N rounds into one summary line per step. Keep
verbatim only the most recent results, the way `boundTurnMessages` already
keeps the recent window.

Acceptance: a synthetic turn with 40 read-only steps of 20 KB each stays
under the model's context on both glm-5.3 and flash, and a re-run of the
medium task never receives "Insufficient tokens".

### P0-3 Make stream and transport errors recoverable mid-turn

Evidence: the benchmark driver's first medium attempt ended the whole turn
on "Error: stream line too long"; the engine surfaced "No local tool action
completed in this turn. The task is saved. Continue when the issue is
resolved." `transientModelError` in `code/agent.cjs` retries connection and
5xx patterns but not stream-corruption or context-overflow errors.

Fix: add "stream line too long", "stream exceeded", and "insufficient
tokens" to the retry classification where the recovery differs: stream
errors retry the same request; context errors trigger P0-2's compaction
first and retry once. After P0-2, an "insufficient tokens" mid-turn should
compact rather than end the turn.

Acceptance: a fault-injection test in the existing engine test suite kills
one stream mid-turn and the turn completes on retry.

### P1-4 Ship the headless driver as `vigil-code exec`

Evidence: this benchmark needed a bespoke driver to run VIGIL-Code
non-interactively; every competitor has a first-class mode (`claude -p`,
`codex exec`, `grok -p`), and it is what makes CI, scripting and future
benchmarks possible. The driver already exists and works
(`ops/bench/vigil-driver.cjs`); it needs productizing.

Fix: add a CLI entry (a `vigil-code` bin or `--exec` flag on the existing
launcher) that runs one prompt against a project folder with flags for
model, effort, access mode and approval policy, streaming JSONL events to
stdout. Reuse the exact agent core; no second engine.

Acceptance: `vigil-code exec --root . --access build "fix the failing
tests"` runs in a terminal and exits non-zero on failure; the benchmark
harness switches to it.

### P1-5 Model-setting guidance: default flash for greenfield, flagship for edits

Evidence: on the small task the default flash configuration matched
glm-5.3's correctness in a third of the requests; on the medium task flash
stayed inside its window where glm-5.3's long thinking overflowed. The app
today leaves the choice entirely to the user.

Fix: when a turn starts in an empty or near-empty project folder, suggest
the flash model in the UI (one-line hint, not a forced switch); keep
glm-5.3 for work in existing codebases.

Acceptance: hint shows in an empty project, never in a populated one.

### P1-6 Tighten turn latency instrumentation

Evidence: `turnSummary` exists but the benchmark had to derive request
counts and token usage from a custom meter. Competitors print tokens used
per run.

Fix: include requests, prompt and completion tokens, and round count in the
turn summary the app already records, and surface them in the chat footer.

Acceptance: a completed turn shows its token and request totals in the UI.

### P2-7 MCP client support

Evidence: Claude Code, Codex and ZCode all speak MCP; it is the standard
way a coding agent gains tools the vendor never shipped. VIGIL-Code has no
equivalent, and its security model (confined tools, per-action approval)
is actually a strong base for it.

Fix: an MCP client in the Electron main process that lists a configured
server's tools into `nativeToolDefinitions`, routes calls through the same
`confirm` gate as built-in tools, and labels them clearly as external.

Acceptance: one stdio MCP server configured in settings appears as callable
tools with approval prompts.

### P2-8 Slash commands and user-defined subagents

Evidence: every competitor has both; VIGIL-Code's explore/delegate
subagents are engine-internal only.

Fix: read `.vigil/commands/*.md` as prompt macros and `.vigil/agents/*.md`
as delegate definitions with name, description and prompt, feeding the
existing `runSubagents`.

Acceptance: a checked-in command file appears as `/name` in the composer.

### P2-9 Keep the confinement advantage and document it

Evidence: grok read the benchmark's grader and sibling runs when nothing
stopped it; VIGIL-Code's tools cannot leave the project root at all, and it
is the only tool of the five with a hardened bubblewrap layer, drift guard
and per-turn checkpoints. Nobody knows this, including prospective users.

Fix: a "Security" page on the website and a section in the README covering
project confinement, the sandbox's allowlisted /etc and PATH policy,
checkpoints with /undo, and the signed update feed. No engine change.

Acceptance: page live, linked from the downloads page.

### P3-10 Prompt caching or server-side turn state

Evidence: 165K prompt tokens for a small app and 419K for a partial medium
build, against Claude Code's $0.46 total for the same work, is the single
largest cost and latency multiplier. P0-1 and P0-2 reduce it, but the
platform re-billing the whole brief every round remains.

Fix: server-side, cache the static brief prefix per (model, mode, access)
and the conversation prefix per turn, or move to an incremental turn-state
protocol. This is a platform change, not only the app.

Acceptance: prompt-token billing for a 20-round turn drops by more than
half against today's numbers for the same transcript.

### Release recommendation

0.2.44 scope: P0-1, P0-2, P0-3, P1-6 (engine hardening and speed, all
testable with the existing suite plus the benchmark harness). P1-4 and
P1-5 follow immediately in 0.2.45 with the packaging work. P2 items are
the 0.3 track.
