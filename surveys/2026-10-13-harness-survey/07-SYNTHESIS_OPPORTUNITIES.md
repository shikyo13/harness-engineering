# 07 — Synthesis opportunities

Seven combinations that create capability no audited system has alone (≥5 required). Each names its contributing
implementations (all evidence-backed in `02-COMPETITOR_AUDITS/`), the new behavior, why the combination beats the
sources, architectural requirements, disadvantages, the test, and the complexity guard. Ranked by expected
improvement in *successful autonomous coding per unit of inference/time/compute*.

**S1 — The context portfolio: aider's ranked repo map × pi's one-deferral-mechanism × Gemini's token ground truth.**
- *Contributors*: aider's personalized-PageRank repo map with documented weight constants and binary-search token
  fitting (repomap.py); Zero's already-ported but flag-off variant (repomap.ts, seed-boost); pi's
  `ToolExposure/prepareLoadout` (one mechanism replacing Zero's four visibility layers); Gemini's
  usageMetadata-ground-truth deficit triggering (`client.ts:859-867` → `contextManager.ts:377-418`) and tool-output
  masking with a 50k protection window; Antigravity's named budget floors (20k rules that cannot be evicted, 1.2.7).
- *New behavior*: every turn composes a *budget-optimized portfolio* — ranked repo map (fit to remaining budget by
  binary search), deferred tools loaded on demand, masked-but-recoverable old tool outputs, floors for
  instructions/skills/subagents — with *measured* token occupancy driving every lever instead of estimates.
- *Why better than any source*: aider has the map but static budgets and no deferral; pi has deferral but no map;
  Gemini has ground truth but no ranking; Zero has all the parts scattered across flags. Nobody maximizes useful
  information per token end-to-end. The pieces are proven separately; the composition is arithmetic, not research.
- *Requirements*: single context-budget service owning floor/reserve/fit math; repomap on by default with aider's
  constants reborrowed; the four visibility layers collapsed into one deferral seam; occupancy fed from provider
  usage with estimate fallback (already exists).
- *Disadvantages*: map staleness on big repos (aider refreshes every turn unless <1 s build — expensive for
  Zero's local prefill: cap to change-triggered rebuilds); one deferral mechanism risks over-hiding tools weak
  models need (mitigate: pin the 4 core tools always-on, pi-style).
- *Test*: E1 + E3 arms; success = ≥25% fewer declared tokens with pass parity, and E3 recall ≥80% at 64k window.
- *Complexity guard*: if the portfolio service exceeds ~1 module of policy code, ship floors+deferral first and
  keep the map flag-gated behind its own A/B.

**S2 — Durable long-horizon runtime: V2 admission × OpenHands-style replay-derived status × goose's scheduler ×
Kilo's wake-parking × omO's done-ness predicate.**
- *Contributors*: Zero V2's durable `session_input` + steer/queue promotion + failInterruptedTools; OpenHands'
  replay-derived execution state (no separate status table to drift) and condensation-as-replayable-event; Qwen
  MQ2c's wake fencing/watermarks; goose's recipes × cron scheduler × agent-callable `manage_schedule` (the only
  complete async loop in the survey); Kilo's `schedule_wakeup`/process-wait parking with hard caps; omO's durable
  done-ness predicate checked at `session.idle` — **which Zero Code's own goal plugin already implements
  in-process (`/goal --verify` exit-0 oracle + marker-vs-open-todos hold + continue drivers, live-verified in its
  own session DB, 01 D1 — non-durable, count-budgeted)**; the desktop recurrence scheduler (01 D6.1) and engine's
  job-ended wake primitive (`memory.ts:91-160`, D3); Codex's rollout meta-line for faithful resume.
- *New behavior*: a session becomes a *schedulable job*: it can park ("retry in 20 min", "when the build finishes",
  "until the API is up"), be woken by time/process/events after restart, evaluate its objective against a durable
  oracle at every idle boundary, and expose cross-process status derived from the event log alone. Hours-to-days
  autonomy without a process babysitting it — which **no** audited harness delivers (Antigravity has the pieces
  closed, goose has scheduling without durable admission, V2 has admission without wake, and Zero's own
oracle lives in a restart-fragile plugin Map instead of at durable boundaries (01 D1) — S2 is therefore a
**port-and-durable-ize of organs Zero already ships**, not an invention.
- *Why better*: each source fails alone exactly where the others are strong: goose's scheduled runs aren't
  resumable sessions; V2 dies with the process; omO's oracle can't wake itself. The composition closes the loop:
  admit durably → drain → park with reason + wake condition → wake evaluates oracle → done, continue, or escalate.
- *Requirements*: `session_wake` table (kind: timer|process|event|manual, fencing watermark); durable `objective`
  + `done_predicate` rows (command/verify contract, seeded from todowrite/plan files); status *derived* by replay
  (never stored); startup sweep re-arming eligible wakes; zero-eval's `--verify` contract promoted from eval
  fixture to session primitive.
- *Disadvantages*: wake storms (cap + coalesce, V2's pendingWake precedent); oracle gaming (verify must be
  outside the agent's editable tree — cleanroom pattern exists); more moving parts on the V2 path that must ship
  behind the same flag.
- *Test*: new fixture `long-park-resume` (agent must wait on a file/server, restart mid-wait, resume and finish) +
  E6 + E8. Success: task survives a kill+restart with zero duplicated provider calls and honest done-state.
- *Complexity guard*: no new services beyond two tables + one scheduler fiber; if wake semantics need more than
  these kinds, the design drifted toward workflow-engine — stop.

**S3 — One capability, two tool surfaces: DeepSeek-PTC discipline × Zero's existing codemode interpreter × the
recovery layer.**
- *Contributors*: DeepSeek Harness's PTC mode (executor-enforced single `run_code`, fresh sandboxed process per
  run with hard caps, sub-calls re-entering the *full* native tool pipeline with durable dispatch event pairs, only
  curated output to the transcript, "never auto-replay a program"); Zero's 3.5k-line codemode interpreter +
  truncate/repair machinery; opencode's live CodeMode as feasibility proof.
- *New behavior*: crowded toolsets and fan-out patterns (50 MCP tools, "query all services and diff") run as typed
  code with intermediate values *never entering context*; strong models can use native calls, weak models too
  (code-gen is their strength per Zero's own ladder: code output is what the local model does best), and every
  sub-call still passes Zero's permissions, truncation, and event journal.
- *Why better*: Zero's current codemode is MCP-only, uncapped, and flag-off; DeepSeek's is well-engineered but
  unmeasured and attached to their own runtime; opencode's unproven economically. Attaching PTC discipline (caps,
  executor-enforced mode, durable dispatch events, curated-output rule) to Zero's interpreter reuses the strongest
  tool-recovery layer in the survey (04 §B) and makes the *first measured* PTC-vs-native A/B (E1 arm d) — evidence
  neither DeepSeek nor Codex has published.
- *Requirements*: per-run limits wired (120 s/512 MiB/64 MiB-style); presentational mode flag on the agent turn
  (executor rejects direct calls in PTC mode); dispatch events in the V2 journal; generated typed SDK prompt from
  the live registry (DeepSeek's flavor approach beats opencode's hand-maintained spec).
- *Disadvantages*: another model-facing contract to teach (mitigated: same tool names/args as native surface);
  debuggability of multi-call programs; interpreter maintenance.
- *Test*: E1 arm (d) on MCP-heavy synthetic fixtures + token-per-solve delta; ship only if fan-out tasks improve
  ≥20% or declared-token cost drops ≥40% at parity.
- *Complexity guard*: if PTC needs its own error taxonomy rather than reusing tool-failure channels, stop.

**S4 — Model-aware orchestration with typed frontiers: aider's architect/editor economics × kimi/qwen trait tables ×
Zero's recovery stack × oh-my-pi economics hooks.**
- *Contributors*: aider's shipped+measured architect(cheap editor, empty history, map_tokens=0); kimi's trait-ized
  per-model quirks and completion-budget auto-sizing; qwen's capability presets + `disableField` one-knob
  arbitration; Zero's repair/coercion/doom-loop layers and per-model `.txt` forks (the fork-code to fix);
  oh-my-pi's `solutionSpace` openness field + `cache_warming_decision` cost exposure; the convergent
  cheap-reviewer-agent pattern (Codex Guardian, Antigravity Auto, ladder design-review +blind scores).
- *New behavior*: the harness *knows* the local primary's profile (quirks, window, reasoning-budget economics) and
  routes only typed boundary events to a stronger frontier model: plan approval, done-ness verification, risky
  irreversible-action triage, post-failure strategy change. Subtasks get effort budgets from a solution-openness
  estimate; review calls carry minimal context (aider's empty-editor lesson) and measured token cost per review.
  Locals do the volume; the frontier bills only leverage points.
- *Why better*: aider splits once per task (wasteful for strong models); Anthropic/Codex/Antigravity reviewers are
  closed or risk-only; kimi/qwen tune requests but not orchestration. Nobody composes per-model data tables with
  economics-gated delegation, and Zero's own ladder already priced the two best review points (design review
  +7 blind at ~2×; robustness instructions +tests at 2.5×) — the pricing mechanism exists, the routing does not.
- *Requirements*: move per-model behavior from prompt forks into a declarative table (traits, formats, budgets,
  repair flags) — aider's YAML pattern; executive finally *wired* with typed decisions (the dead core Executive
  service + `SessionRunnerModel.resolve(override)` seam already exist); frontier calls as durable events (auditable).
- *Disadvantages*: routing bugs waste the expensive model (mitigate: caps + ledger kill-switch); two-provider
  privacy posture (documented boundary, opt-in per repo); the executive must stay advisory until E6 prices it.
- *Test*: E5 + E6, plus a new `ladder-t4-meals` A/B: local-only vs local+frontier-at-boundaries; success = within
  blind-review noise of frontier-only at <40% of frontier-only frontier-tokens.
- *Complexity guard*: exactly three boundary kinds at first (approve/verify/triage); any fourth proposal must beat
  a simpler re-scoping of the first three.

**S5 — False-completion defense in depth: SWE-agent's reviewer-gated submit × zero-eval's hidden-test &
recall-probe machinery × Gemini loop-recovery × measured verification pricing.**
- *Contributors*: SWE-agent's submit gate + trajectory tooling; zero-eval's `verified_after_timeout` diagnostic and
  recall answer-keys; omO's `<promise>` oracle and degradation monitor; Gemini's detect→recover-not-abort;
  Antigravity's denial-respect rule (1.2.15); the ladder's measured robustness-instruction effect.
- *New behavior*: tasks declare (or the agent writes) an acceptance contract at start (commands + constraints +
  "pitfall checklist" from the measured unstated-pitfall taxonomy); "done" requires the contract to pass in a
  clean copy; the verifier's *claims* are re-checked by a second cheap model (blind-review protocol, industrialized
  from the ladder's judging); stall triggers strategy-change nudges rather than abort. False completion becomes a
  *measured rate* per task class, not an anecdote.
- *Why better*: most harness cores treat completion as finish=stop; Zero's plugin layer already verifies at idle
(`/goal --verify`, design review gate) but non-durably and with no claim audit (01 D1/D4); eval-side checks never run inside
  the product loop. The ladder proved in-product verification moves blind scores — this industrializes that into
  the autonomy story (the assignment's "preventing premature success" requirement).
- *Requirements*: acceptance-contract artifact (superset of `--verify`); cleanroom runner already exists (ladder
  cleanroom.ts); claim-audit prompt template + scoring hook; S2's idle-boundary oracle uses the contract.
- *Disadvantages*: contract-writing overhead on small tasks (auto-skip below a size gate); verifier cost (price via
  E6); gaming if tests live inside the editable tree (cleanroom copies).
- *Test*: E6; also retro-score the ladder runs: current false-completion rate becomes the baseline number.
- *Complexity guard*: contract schema is commands + text constraints only; no formal-spec ambitions.

**S6 — Safe parallelism by declaration: kimi's access-declared scheduler × Antigravity's worktree workspace modes ×
durable child transcripts.**
- *Contributors*: kimi's conflict-aware tool scheduler from declared file accesses; Antigravity's `branch`/`share`
  subagent workspace modes + roster auto-injection + `conversation_id`/`log_uri` trajectory correlation; Claude
  Code's anti-injection hand-back envelope; Kilo's wave orchestrator and worktree Agent Manager; Zero's task tool
  (already denies nesting by default, inherits permissions, background-injects results).
- *New behavior*: parallel *writers* become safe: tools declare read/write paths, the scheduler serializes on
  conflicts (kimi) and forks worktrees per write-heavy subagent (Antigravity) — while every child transcript is a
  durable, indexed session the parent (and humans) can cite after the fact (message-text search already ships,
`session/session.ts:203-213`; the delta is FTS5 ranking + citations), fixing Claude's
  non-persistence gap. Read-only fan-out (explore/search) keeps today's cheap shared-read model.
- *Why better*: Zero races tools entirely (04 §F); Antigravity has modes but closed, no conflict scheduling; kimi
  schedules but has one workspace. The pair covers intra-session tool safety and inter-session code isolation
  with the same declared-access primitive.
- *Requirements*: access declarations on file tools (paths are already args — schema-visible without model burden);
  worktree-per-subagent default for `background: true` writers; child sessions inherit the V2 durability stack;
  `@conversation:`-style FTS search (SQLite FTS5 is trivially added to Zero's event store).
- *Disadvantages*: worktree merge friction for related edits; declaration honesty (sandbox verifies, post-hoc).
- *Test*: new two-writer fixture (same package, disjoint files, then clashing files); success = zero lost updates,
  wall time <2× serial for disjoint case; measure merge-conflict rate for clashing case.
- *Complexity guard*: declare-access + worktree-mode only; no distributed-lock system.

**S7 — Cache-as-invariant context stack: Zero's frozen prefixes × Claude's measured breakpoint economics ×
aider's cache-ordered placement × Qwen's prompt_cache_key discipline.**
- *Contributors*: Zero's disk-persisted frozen system+tool snapshots with the 7–12-min-refill justification and
  98%-vs-0% continuation-summary cache-hit measurement; Claude's 1h-ephemeral writes + snapshot/hoist flags
  (observed, priced); aider's repomap-before-history ordering; Qwen's DashScope stable-prefix marks + explicit
  session prompt_cache_key with breakpoints; oh-my-pi exposing warmCost/missCost to policy.
- *New behavior*: every context mutation (compaction, deferral, steering injection, memory updates) is *cost-checked
  against the prefix-cache delta before it is applied*, with per-provider cache accounting including the implicit
  providers (today's gap: cache accounting exists only for Anthropic-style explicit fields, and the telemetry
  ledger's `prefix_broken` is recorded but unread). The memory block and repo map become *append-only-at-tail* or
  breakpoint-pinned, so daily-variation content (dates, memory, free RAM) stops poisoning the prefix — currently
  frozen at session start partly to hide this.
- *Why better*: Zero is the only open harness that *measures* these effects but treats cache as a property to
  preserve, not an optimization axis to trade (e.g., it refuses live memory updates rather than placing them at
  tail); Claude/Antigravity monetize the axis in closed product loops; aider/qwen got placement right by hand.
  Making it an explicit invariant + policy input is a small, high-yield generalization.
- *Requirements*: cache accounting per provider family (implicit cache proxy: prefix-hash hit rate via engine
  timings where available, else prefix_broken heuristics); placement policy table (what may sit where, at what
  mutation frequency); `cache_warming_decision`-style hook already has a seam in Zero's executive.
- *Disadvantages*: overfitting to one engine's cache behavior (keep policy per provider family); more bookkeeping
  in the assembly path (measure with existing `context_assembly_ms`).
- *Test*: E3/E5 read `prefix_broken`/hit-rate as a first-class outcome; success = halve prefix breaks per long run
  at equal recall.
- *Complexity guard*: placement rules stay a data table; no new cache layers (no cross-session sharing) until
  data says otherwise.

## Combinations deliberately rejected

- *Roo-style role pipeline × anything*: role prose showed zero measured benefit (roo audit); the enforceable half
  (capability profiles) is absorbed into S4's table instead.
- *Antigravity runtime clone (single-binary everything)*: distribution posture costs Google a standing CPU-quirk
  changelog; Zero's desktop/SEA path can adopt the artifact shape later without bending the harness design for it.
- *next-speaker continuation × durable admission*: the LLM side-query was killed by its own upstream on cost; V2's
  positive signal wins on both price and determinism.
- *Mini-SWE radicalism as product*: its minimalism buys five privileges (no human, no resume, sandbox, fixed
  horizon, cheap rerun) that an interactive harness doesn't have (mini-swe audit); adopting its *loop size* while
  discarding those privileges loses durability (see baseline verdict Option B).
- *DeepSeek-style full TS/Python dual runtimes now*: one interpreter already exists; dual flavors only if E1(d)
  shows Python-side wins on specific model families.

## Cross-check against the assignment's examples

The illustrative combos from §8 of the assignment map as: "Aider repo intelligence + Pi minimal loop + Codex
execution patterns" → **S1** (+S3 for execution); "OpenHands durability + DeepSeek composability + Zero autonomy"
→ **S2 + S3**; "Antigravity async + Pi session control + persistent objectives" → **S2**; "model-aware tool
interface + fine-grained observability + adaptive context" → **S4 + S7**. All four were viable enough to promote;
the synthesis adds what none of them cover — completion honesty (**S5**) and safe parallel writes (**S6**) — which
the survey found to be the two universally missing capabilities.
