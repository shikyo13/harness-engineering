# 05 — Agent engineering findings

The deeper lessons from 18 head-pinned audits, Zero Code's own baseline, and prior local ladder experiments.
Each finding states the mechanism, the evidence, and the strength of the causal claim. Where the survey found
*convergent independent invention* (same mechanism built twice by unaffiliated teams), the claim is strong; where
praise outran evidence, it is called out.

## 1. Designs that demonstrably improve agent effectiveness

**F1 — Durable admission before execution (strong; five independent inventions).**
OpenCode (live), Zero V2, Qwen's hosted MQ2c journal, Antigravity's queuedMessages/auto-resume, and omO's
WAL-journaled DAG all converge on: persist the request with a monotone sequence, decide promotion (steer/queue) at
explicit safe boundaries, and let execution be interruptible/recoverable *because* admission is durable.
Gemini CLI and pi show the negative case — quality loops whose runs still die with the process. The pattern's
details matter: Qwen's key simplification is deriving pending inputs *purely from event replay* (wake-intake.ts:12-52
— no separate status table to go stale); Codex freezes base_instructions into a rollout meta-line so resume is
faithful; Gemini's unanswered-tool-call closure before accepting new input (geminiChat.ts:528-535) prevents the
class of provider pairing errors that otherwise force synthetic repairs. Causal story: long-horizon failures are
mostly boundary failures (crash mid-turn, steering lost, duplicate send). Durability at the admission boundary is
the mechanism that shrinks that failure surface.

**F2 — Exact-first edit cascades with actionable failure text (strong; universal convergence).**
Every serious edit tool is now exact-match-first with graded fallbacks and refusal, not fuzzy-first: Zero's 9-stage
cascade with disproportionate-span refusal, Gemini's 4-tier with strategy telemetry, Codex's fuzz ladder with
`seek_sequence` relaxation, aider's elision/recovery heuristics, pi's exact→normalized. The measured differentiator
is not the matcher but the **failure response**: aider's "did you mean" real lines + "already applied, don't resend",
Zero's near-miss excerpt with line numbers, Gemini's "re-Read before retry". Weak models recover within one turn
when the error text contains the actual file state; they loop when it doesn't. The reflection-loop cap (aider: 3)
bounds the cost. Corollary with contrary evidence: crush *enforces* read-before-edit via a tracked read table with
mtimes (edit.go:288-302) — the only implementation that removes the hallucinated-region failure mode at the source;
everybody else leaves it to prompt convention.

**F3 — Output shaping is an intelligence lever, not cosmetics (moderate-strong; measured locally by Zero,
corroborated everywhere).** Zero's own ladder data (2026-10-09): harness-side rewrite overhead was 0–6% of decode
while *tool output policy* determined whether long tasks survived. Gemini's token-ground-truth triggering removes
estimator drift; its masking service protects 50k of recent tool tokens and evicts older bulk to disk in batches —
cheap context without summarization loss. Antigravity partitioned budgets (20k rules floor that skills/MCP/subagents
cannot evict, 1.2.7) after Gemini-CLI-style global eviction produced the reported "skills got compacted away" class
of bugs. mini-SWE's head5k/tail5k-with-count survives SWE-bench because *elision that says it elided* keeps the
model honest. Conversely, empty output that is literally `"(no output)"` (Zero) vs silent empty is the cheapest
reliability trick in the set, and only Zero and SWE-agent practice it deliberately.

**F4 — One mechanism for tool visibility (strong; six inventions, one anti-pattern).**
Claude (ToolSearch), Codex (BM25 tool_search), Kimi (`select_tools`), Antigravity (meta-tool dispatcher +
`alwaysLoad` opt-in), pi (`ToolExposure`/`prepareLoadout`), Gemini-adjacent (omO's MCP gating) — all converged on a
*single* deferred-load path with a schema cost that is measured (Zero measured built-in declarations at ~2,100 of
~6,000 tokens, tools.ts:41-45). Zero Code currently runs **four** overlapping layers (registry visibility →
repair aliases → tool_search/tool_exec → codemode), which the baseline flagged as finding #6. The evidence says:
schema tokens spent on rarely-used tools are tokens not spent on history; every convergent implementation reports
this as the reason.

**F5 — Loop detection must recover, not abort (moderate; two strong implementations).**
Gemini's heuristic→LLM-double-check→*feedback turn* (`client.ts:1300`, "take a step back") and oh-my-pi/Zero's
doom-loop redirect (answer the third identical call with the previous result + redirect text) are the only
mechanisms in the survey that turn a stall into progress rather than a failure. Codex Guardian (agent-triaged
approvals) and Antigravity Auto (reviewing agent for sandbox escapes) extend the same "intervene cheaply, keep
going" posture. Abort-on-loop (aider's cap-then-give-up; mini-SWE's hard exit) is adequate only in eval contexts
where a human retries.

**F6 — Checkpoint granularity = undo granularity = trust granularity (strong; three convergences).**
Shadow-git-with-alternates per step (Zero), git-dir snapshots (Gemini), SQLite step snapshots with sweep-forward
recovery (Antigravity), per-step `StepContext` + fork-truncate (Codex) — all independent. Kline's counter-example:
Cline *abandoned* shadow-git for plain `git stash` in its rebuild (checkpoint-restore.ts:45-66) and now cannot
checkpoint untracked-then-committed flows per-step; Roo's orchestrator shipped without checkpoints. Users allow
agents more autonomy in direct proportion to reliable per-step undo. This is also the cheapest insurance against
the fuzzy-edit corruption class (F2).

**F7 — Separate model decisions from host execution, with typed interfaces (moderate-strong).**
Codex's Op/EventMsg protocol, pi's hook-carried policy, DeepSeek's PTC sub-dispatch through the *full* native
pipeline, kimi's XState machines refolded from journal, OpenHands' stateless step() — all make "what the model
decided" and "what the host did" distinct durable records. Where teams skipped it (aider's god-object Coder,
Gemini's config god-object, Cline's 2,910-line agent-runtime), testability and mid-turn feature work measurably
suffered (their own bug-churn trails). But note the cost: DeepSeek's ceremony and omO's "80% of a harness built
through OpenCode's event surface, then they forked the loop anyway" both show the seam is only worth having if the
loop itself stays *simple*. The winning shape is a small kernel with typed boundaries — not middleware sprawl.

## 2. Commonly praised features that are mostly cosmetic or unproven

- **Role-theater multi-agent (architect/coder/reviewer as prompts).** Roo's roles decompose into an enforceable
  half (permission profiles — real) and a role-play half with **zero in-repo evidence of measured benefit** and an
  orchestrator changelog that is pure bug-churn. Anthropic's own product moved the other way (Claude Code's newest
  models *remove* Todo tools; advisor is a routing field, not a persona). Adopt declarative capability profiles;
  reject role prose as a quality mechanism — it costs prefix-cache bytes on every turn.
- **LLM next-speaker continuation.** Shipped by Gemini CLI and inherited by qwen-code; **defaulted off** by Gemini
  (`config.ts:1288`) — an extra LLM call per turn to decide what "no tool calls" means. Tool-call-driven
  continuation (every other harness) won on cost; qwen-code still pays it.
- **Whole-file editing as default.** aider keeps it as fallback; the survey shows nobody who measured corruption
  rates chose it first for weak models.
- **MCP breadth as a selling point.** Five of the strongest loops gate MCP behind search/deferral or budget floors;
  two (pi, aider) skip MCP almost entirely without observable task loss. Unrestricted MCP census measurably evicts
  context (Antigravity's 1.2.7 fix is the receipts).
- **Benchmark-claim velocity.** pi's "beats Claude Code/Codex" is talk-only (transcript, no reproducible config);
  DeepSeek explicitly refuses an "unconditional savings" claim for PTC; SWE-agent's retirement memo concedes ACI
  elaborateness was mostly weak-model compensation. Exclude leaderboard prose from design decisions.
- **Subagent *counts* as capability.** kimi's 128-agent swarm is real engineering (resume map, templates) but the
  survey's long-horizon leaders (Codex, Claude, aider) ship one-strong-agent-plus-tools and delegate sparingly.
  Evidence for parallel agents beating one strong agent is concentrated in *read-only fan-out* (explore/search),
  not parallel writes (where only Antigravity/kimi built the worktree/access-conflict machinery that would make it
  safe — F-extra).

## 3. What specifically helps local/weak models

Ordered by evidence strength:

1. **Recovery layers at the tool boundary**: alias tables, argument coercion, JSON repair, invalid-args rewrite
   text, alias-routed MCP names (Zero A, kimi's call-ID normalization, Antigravity 1.2.4 returning schema-validation
   failures *to the model*). Local models emit messier calls; fixing them at the boundary costs nothing on strong
   models and saves whole turns on weak ones.
2. **Per-model behavior as data** (aider's YAML tables; kimi's trait flags; qwen's capability presets +
   `disableField` arbitration after "qwen3.8 rejects reasoning_effort with thinking_budget"). Zero's nine prompt
   `.txt` forks are the same idea expressed as fork-code — works, but every new model costs a file, and drift
   between forks is invisible.
3. **Window honesty + budget math tuned down**: small windows make compaction quality load-bearing — verbatim
   tails, retained user messages, todo carry-over (Zero), reserve-based triggers (pi 16k/20k, kimi 85%+50k).
   Zero's `--context` override exists precisely to test this (see the live 32k run for 06).
4. **Explicit completion-budget sizing**: kimi's budget = window − used; Antigravity's medium-default verbosity
   grouping; both reduce truncation-induced amnesia at small windows.
5. **Examples in tool descriptions** (Zero's shell prompt with good/bad quoting examples per shell family is the
   best instance in the survey); **lower format bars** where parse failure risk is high (mini-SWE's single-fenced
   action for weak models beats demanding XML).
6. **Cheap-model subtasks with measured economics** (aider architect→cheap-editor with empty history is measured;
   pi's cache-warmer shows its cost math; oh-my-pi's `cache_warming_decision` exposes warmCost/missCost to policy).
   Local-heavy + occasional frontier review is viable *where the review boundary is a typed artifact* (diff, test
   report), not vibes.
7. What did **not** help in local experiments (docs/research/2026-10-09 ladder): low reasoning effort (slower *and*
   worse on the local engine); design-review subagents (+blind-score gap, ~2× time); the failure mode that
   dominated was *unstated pitfalls*, which prompt robustness clauses fixed at ~2.5× time — i.e., weak-model
   improvement is bought with tokens, and the harness's job is making that purchase opt-in per task class.

## 4. What increases long-horizon reliability

Ranked by surveyed evidence: durable admission + replay-derived pending state (F1) → per-step checkpoints (F6) →
partitioned context budgets + ground-truth triggering (F3) → steering that is durable and boundary-promoted (V2/
OpenCode semantics, with Antigravity's 1.3.1 "send-immediately silently queued" bug as the regression-test
cautionary tale) → stall detection with recovery turns (F5) → scheduled wakes for genuinely asynchronous work
(goose's recipe×scheduler, Kilo's goal parking — nothing else in the survey closes the loop between "agent finished
its turn" and "something later should happen again") → done-ness as a durable predicate checked at idle (omO) —
the single highest-leverage missing mechanism in *every core loop* — corrected 2026-10-14: Zero Code already ships
this predicate **plugin-side** (the goal plugin's `--verify`/marker check with open-todo hold at idle, live-verified
in its own session DB, 01 D1), so the true gap is durability + cost-budgeted continuation + strategy change, not
the concept (core executive remains designed-but-dead; Claude's advisor observed-but-advisory; Antigravity's
reviewing-agent arrived 1.3.5).

Against the assignment's Ralph-loop question: raw Ralph iteration (re-inject the same prompt) is distinguishable
from real autonomy by three properties the good systems have and loops don't: (a) state the re-injected prompt can
*see* (todos/ledger/plan files), (b) a done-ness oracle that can fail the loop (omO's `<promise>` oracle,
zero-eval's hidden-test `verified_after_timeout`), (c) budget accounting with strategy change on stall (nothing
ships this yet — open field, see 07-S5).

## 5. Patterns Zero Code should explicitly avoid

1. **Two execution paths in production** (own finding #1; Antigravity needed a headline entry to fix theirs).
2. **Fuzzy edit tiers without freshness enforcement** (crush's read-table; also don't let 2000-char line caps and
   "use `cut`" advice fight "don't use bash for files" — Zero's own contradiction).
3. **Four tool-visibility layers** (converge to one; keep aliases as error-recovery only).
4. **God-object runtimes** (Cline 2,910 lines, aider Coder, Gemini config.ts ~4k) — the survey's own bug trails.
5. **LLM next-speaker continuation** (dead on arrival upstream; V2's positive tool-call signal is right).
6. **Role-prose multi-agent as default architecture**; also subagent fan-out for write-heavy tasks until
   access-conflict scheduling (kimi) or worktree modes (Antigravity) land.
7. **Benchmark-driven architecture claims** (DeepSeek's discipline is the standard; run zero-eval A/Bs instead).
8. **Cost accounting that ignores the actual constraint**: local cost=$0; measure *tokens and prefill* as the cost
   currency (telemetry ledger already has the fields; nothing consumes them).
9. **Stash-as-checkpoint regressions** (Cline) and **summarize-only overflow** (aider) as "good enough".
10. **Decorative dashboards**: the ledger must feed loop policy (e.g., stop warming caches when hit-rate collapses),
    not just charts.

## 6. Meta-finding: the survey's shape

Tier-1 leaders have converged on the same seven-part skeleton (durable admission; typed decision/execution split;
exact-first edits with actionable failures; measured context management with budgets/floors; OS-level sandbox or
equivalent enforcement; checkpointed steps; one deferred-tool mechanism). None of the eight audited Tier-1 systems
ships *all seven* **durably**: Zero V1+V2+plugins come closest at ~6.5/7 (OS sandbox absent; done-ness oracle
exists but only as non-durable plugin state, 01 D1; most of the rest
flag-gated); Antigravity ~6/7 closed; Codex ~5.5/7; OpenCode ~6/7 at head. The differentiation that remains is not
loop structure — it is (a) long-horizon *policy* (progress evaluation, done-ness, strategy change — universally
weak), (b) local-model economics (frontier-first teams under-invest here, and Zero's own ladder shows the harness
can close measured gaps), and (c) composition of repository intelligence with verification. Those are exactly where
07 puts its synthesis proposals.
