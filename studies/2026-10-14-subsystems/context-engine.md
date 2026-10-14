# Context engine — subsystem study (P2)

DRAFT — P2 study, written 2026-10-14, awaiting cross-check.
Scope: repo intelligence, retrieval, compaction, context assembly, prompt-cache placement, pruning. Method:
`PRIOR_WORK.md` first; competitor context subsystems read end-to-end in the head-pinned read-only clones
(`SOURCES.md` pins in §7); Zero Code behaviour from `spec/context-engine.md`, source read only where the spec
lacks detail (repomap wiring, archive gating). Strongest context-management audits were **gemini-cli, codex,
aider, pi** (crush as threshold reference); kilo-code was read and is weak on context, and
`02-COMPETITOR_AUDITS/oh-my-pi.md` does not exist — `pi.md` covers that lineage.

## 1. How each implementation works

**Gemini CLI** (`42c51822084a`, under `packages/core/src`). *Triggering*: measured prompt-token ground truth from
two sources — `usageMetadata.promptTokenCount` after every finish (`core/client.ts:857-865`) and a free
`countTokens` calibration at hot start (`context/contextManager.ts:470-493`). Budgets are tiered
(`context/config/profiles.ts:79-81,219-222`: retained 65k, normalized 100k, max 150k, coalescing 5k) and work
fires only when the deficit **grows past lastTriggeredDeficit + threshold** — hysteresis, reset when it falls
(`contextManager.ts:374-436`); a backward scan ages nodes past retained, protecting the last turn and pinned turn-0
environment context (`:329-372,440-468`). *Masking*: `context/toolOutputMaskingService.ts` — backward FIFO
over `functionResponse` parts; newest 50k tool tokens protected (`:25`), latest turn skipped (`:92-94`), batch
fires only at ≥30k prunable (`:26,152`); skill/ask-user/plan tools exempt (`:36-41`); output spills to
`tool-outputs/session-<id>/` while history keeps a preview — head/tail 250 chars, or for shell a structured
preview preserving Error/Exit Code (`:203-213,292-348`); idempotent via a tag (`:288-290`). Legacy
`ChatCompressionService` still runs behind a flag — two parallel systems.

**Codex** (`6786dccbe726`, `codex-rs/core/src`). *Triggering*: `session/context_window.rs:53-130` computes a
buffered auto-compact limit, a hard cap at `effective_context_window_percent` (90 % default), and a post-turn
percent threshold (`:111-117`). The scope selector `AutoCompactTokenLimitScope::{Total, BodyAfterPrefix}`
(:61-81) counts **only tokens added after the cached prefix** — cache-aware trigger: growth riding a warm prefix
is cheap; re-prefill is what costs. *Compaction*: engines local/remote-V2/token-budget behind one dispatcher;
on `ContextWindowExceeded` mid-compaction it trims **oldest-first to preserve the cached prefix** and retries
(`compact.rs:348-357`). Replacement history keeps verbatim user messages newest-first capped at 20k, truncating
at the boundary and marking `retained_sources_incomplete` (`:63,394-396,691-760`), re-materializes initial
context + WorldState baseline (`:84-99,404-410`); post-turn compaction errors on an empty summary (`:382-389`).
`ContextCompacted` is a persisted rollout item (replay matches what the model saw); a `comp_hash` change forces
compaction on model switch (`turn.rs:1374`); bulky retained payloads stay outside the transcript (`compact_retained_tool_outputs.rs`).

**Aider** (`5dc9490bb35f`). *Repo map*: tree-sitter def/ref tags cached on disk keyed by fname + mtime
(`repomap.py:233-264`); personalized PageRank with battle-tuned weights — mentioned ident ×10, long-cased name
×10, leading `_` ×0.1, >5-definer ×0.1, chat-referencer ×50, `sqrt(refs)`, personalization 100/N
(`:481-525`); binary-search fit of top-N tags to the budget with 15 % tolerance and a sampled tokenizer
(`:666-703,89-101`). *Refresh cost is the lesson*: `refresh="auto"` caches the rendered map on
(chat, other, budget, mentions) **only if the last build took >1 s** (`:592-613`) — small repos rebuild every
turn, big repos only when the mention set changes. *Placement*: system → examples → readonly → repo-map → done →
chat_files → cur → reminder, with cache marks on stable chunks and a ~5-min keep-alive ping
(`chat_chunks.py:16-47`, `base_coder.py:1340-1394`) — map before history so the provider prefix covers it.
*Summarization*: background thread during think-time, cap `min(max(input/16,1024),8192)`, recursive
split-in-half keeping a tail to an assistant boundary (`history.py`).

**Pi** (`3daa20280f6a`, `packages/coding-agent/src/core/`). *Triggering*: hybrid — real `usage` from the last
valid assistant message plus chars/4 for later messages (`compaction/compaction.ts:196-224`), usage invalidated
by a later `context_edit`/`compaction` (`:226-262`); fires at `tokens > window − reserve`, defaults reserve
16 384 / keepRecent 20 000 (`:126-130,267-270`). *Compaction*: never cuts mid tool-call (`:351-406`); walks
backward to keepRecent (`findCutPoint :446-501`); a turn split merges a turn-prefix summary into the history
summary (`:994-1033`); on re-compaction the previous summary is merged forward via an UPDATE prompt
(`:540-579,717-733`); the read/modified file ledger survives N compactions in `CompactionEntry.details`
(`:51-88,1056-1069`); length-truncated summaries are rejected, never persisted (`:585-593`); summary calls send
`cacheRetention: "none"` so one-off summaries never pay cache-write (`:629-633`). *Cache economics*:
`cache-warmer.ts` refreshes at 90 % of TTL (10 s margin, `:29-32`) only when `P(continuation) × missCost −
warmCost ≥ $0.05` (`:19-26`), with replayability checks (`:55-58`) and a decision record (`:91-106`).

**Crush** (`d6edc3222231`, `internal/agent/agent.go`), threshold reference: auto-summarize when
`remaining ≤ threshold`, 20k tokens if window >200k else 20 % (`:54-58,1095-1116`), **skipped when the window is
unknown** — a local-model guard (`:1098-1101`). Triggers on the previous step's usage arithmetic, so one huge
tool result overshoots (audit weakness 3); cache-control marks the last system boundary, last two messages and last tool (`:720-726,893-907`).

**Zero Code today** (spec `context-engine.md`; only gaps read in source): estimate-driven overflow with engine
ground truth when reported; summary-as-turn cache continuation (98 % vs 0 %, `01-ZERO_CODE_BASELINE.md:28,521`,
`compaction.ts:698-770`); serialized fit+detail ladder; retained verbatim user messages (20k/2k/10 %,
newest-first with cut pointers) in `compaction_context`; pressure-gated prune (≥75 % and >20k prunable, 40k
protect, `3d338bc44f`); disk-persisted frozen prefix + tool freeze. Gaps confirmed in source: the full-transcript **archive is written only on overflow** and the refusal/ladder paths
(`compaction.ts:681-682,797,863,916`) — a plain successful compaction leaves no pointer (unlanded W2, `09:12-13`);
the flag-off **repomap has no cache or refresh gate** (`repomap.ts` `build()` only) but is frozen into the system
prefix with a 45 s timeout (`prompt.ts:1419-1447`), so full rebuilds land on every freeze reset — and every compaction resets the freeze.

## 2. What is genuinely best, and why (evidence)

- **Zero's cache-continuation compaction** is the best local-engine summarization found anywhere: 98 % cache vs
  0 % serialized (`01` :521), compaction cut to 1-2 min from 2-4, and the after arm recalled every requirement in
  every 64k run across 7-10 compactions (`2026-09-23-long-horizon-results.md`). Codex's oldest-first
  trim-to-retry (`compact.rs:348-357`) independently validates it: compaction must spend cached prefill.
- **Gemini's ground-truth triggering with hysteresis** is the best trigger — nobody else's beats usage + chars/4
  (pi) or previous-step arithmetic (crush). It kills estimator drift and the compaction-that-doesn't-compact
  class; the coalescing threshold stops per-turn churn that pi/crush pay unconditionally.
- **Gemini's masking is the best cheap lever**: fires only at ≥30k prunable, protects newest 50k, exempts
  high-signal tools, keeps a preview + file pointer — recoverable, cache-neutral when batched, composable with Zero's spill files.
- **Aider's repo map** is the best-quality cheap map (tuned constants, budget-fit search), and its refresh
  discipline (rendered-map cache keyed on inputs; mtime-incremental parse; rebuild-per-turn only when cheap) is the best answer to map cost.
- **Pi's incremental summary merge + file ledger** is the best continuity design: summaries survive N compactions
  without summarizing summaries, and the file set stays alive; Zero's `compaction_context` carries todos +
  archive but no file ledger. **Pi's warming rule** (expected-savings ≥ floor at a TTL fraction) is the only
  quantified policy for when a cache mutation pays — the template for cost-checking any mutation.
- **Crush's unknown-window guard and percent-scaled thresholds** are the right constants for local models;
  Zero's fixed 20k/40k constants don't scale down.

## 3. What to combine

1. **Token-ground-truth triggering + masking (gemini pattern)**, window-scaled: occupancy from provider usage
   with estimate fallback (exists), deficit hysteresis, tiered budgets as data; masking ahead of summarization.
   Gemini's absolutes (50k/30k on 150k) must become fractions (~30-40 % window protection, ~15-20 % batch floor)
   — at 64k the R5 cell shows the current stack managing context more than using it (72 compactions, hit 90→34 %, `06` R5).
2. **Cache-as-invariant placement** (07 S7): frozen head (system+tools), append-only history, volatile content
   (pressure notes already pinned-at-index; memory block, dates, free RAM) moved to tail or breakpoint-pinned
   instead of frozen at session start; every mutation cost-checked against its prefix-cache delta. Codex's
   `BodyAfterPrefix` scope is the same invariant as a trigger: charge pressure against tokens that must be
   re-prefilled. Pi's `cacheRetention:"none"` is a free win on Anthropic-style providers; on the local engine the
   warm-cost is prefill contention, so warm only into gaps, if at all.
3. **Aider's refresh-cost lesson, adapted**: change-triggered rebuild only. Zero's freeze already amortizes the
   map across turns; the remaining cost is rebuild-at-every-freeze-reset (compaction, restart). Fix: content-hash
   the file set (git status/watch) and rebuild only on change; carry the rendered map across the freeze when the repo hasn't changed.
4. **Symbol graph vs repomap vs LSP — the lost owner ideas** (`drops.md:52`). *SQLite AST/symbol graph: earns a
   place, but not in the prefix.* LocAgent's own ablation: the entity **search index** carries the value (−19
   file-level points without it) while traversal adds ~2-5.5 (`code-graphs.md`); CodeScaleBench's public report
   shows +0.035 aggregate reward, TypeScript the strongest *negative* outlier, gains above ~400k LOC. Zero's 64k
   measurement is mixed (map arm fastest on qa-spec, timeout on inventory). As prompt payload, neither PageRank
   maps nor symbol graphs beat grep+read for a capable local model on repos this size; as an agent-queried tool
   with detail levels, a durable index could — defer behind E3/E1. Ship the crush-gated LSP (file-type ∧
   root-marker ∧ PATH) first for definition/references precision. *Context-quality metrics: earn the place now.*
   `prefix_broken` is recorded but unread (W4) and per-component prompt-token breakdown never existed
   (`01` Part C §6; `06` §5 names it a prerequisite for reading E1/E3 claims). Without attribution, portfolio
   budget decisions are unfalsifiable; this is a W-scale add that gates §5.

## 4. Proposed design for the new core

- **ContextPortfolio as data.** One budget service per session holds floors (instructions, skills, plan, reserved
  output — antigravity's non-evictable-20k-rules idea), reserves and fit math as a table, not code paths; each
  turn composes frozen head + (map if earned) + history tail + masked pointers. Occupancy = last ground truth +
  estimated delta, invalidated by any `context_edit`-style mutation (pi `:226-262`); deficit crossing a tier with hysteresis selects a lever.
- **One pruning policy with the measured pressure gate.** Replace today's three gates (prune-gate
  `compaction.ts:502-557`, overflow, fit ladder) with one ordered ladder over measured occupancy: (1) **mask**
  tool outputs older than the protection window in ≥batch groups (preview + pointer, high-signal exempt);
  (2) **prune** old completed tool results to placeholders when masking wasn't enough and occupancy ≥ the gate
  fraction of the usable window (the 30k-prune / 126k-re-read ≈ 6-min measurement stands); (3) **compact** via
  summary-as-turn, serialized ladder last. Below every floor, history is byte-stable and the cache stays warm.
- **Archive pointer on every compaction (close W2).** Write `<data>/compaction/<session>/<msg>.txt` always, not
  overflow-only (`compaction.ts:681-682` is the gap), pointer in `compaction_context`; recall must improve or tie
  (W2's own exit criterion). Pair with a pi-style file ledger so re-read loops shrink.
- **Compaction as a persisted, replayable event** with a summary that *merges forward* (pi UPDATE prompt) rather
  than re-summarizing; length-truncated summaries rejected (pi `:585-593`; Zero's blank-summary recovery
  checkpoint stays); post-compaction non-empty-summary check (codex `:382-389`).
- **Placement table as an invariant**: frozen head; volatile blocks only at tail or pinned breakpoints; the
  in-memory context-cache experiment (spec "GAP — no test") graduates only if per-component metrics show it pays.
- **Metrics before defaults**: repomap stays off until the change-triggered rebuild (W6) + attribution show a
  navigation win at 64k; LSP on with crush gating + `lsp_replace_symbol`.

## 5. Measurable targets (baselines cited)

Baseline cells: qa-spec@64k today — pass, 1 compaction, 91.1 % hit, 2 breaks, 10/10 recall, 2809 s
(`2026-10-14-p0-qa-spec.md/.json`); 2026-09-23 qa-spec@64k — after 3 compactions/4 min, tool-search 1/1 min;
inventory@64k baseline 9 compactions/~27 min → after 7/12 min, tool-search 3-4/7-9 min; R5 24k — 72
compactions, 34 % hit, timeout (`06` R5); long-horizon prefix-hit range 0.78-0.91 over 14 runs, warm first turn
1.3-1.5 s (`SYNTHESIS-2026-09-29.md`). Read at the E3 cell (`06`: recall ≥80 % and pass parity vs native window at 64k; below → priority bump):

| metric (64k, n≥3, quiet engine) | baseline | target |
|---|---|---|
| compactions per qa-spec / inventory run | 1 / 7-10 | ≤1 / ≤4; ≤2 min each |
| compaction share of wall (inventory) | ≈16 % (12/73 min) | ≤10 % |
| recall probe at 64k | 10/10, 12/12 post-fix | ≥80 % floor (E3), hold 100 % on the three fixtures |
| prefix hit at 64k | 91.1 % (P0); range 0.78-0.91 | floor ≥90 %; ≥0.60 at the 24k stress cell |
| prefix breaks per long run | 2 (P0 qa-spec) | ≤1 (halve — S7 exit criterion) |
| archive pointer post-compaction | overflow-only (gap) | 100 %, new test, recall ≥ baseline |
| compaction-that-doesn't-compact | unmeasured | 0 after ground-truth trigger (hysteresis logged) |

## 6. Beyond / correcting the audits

- The gemini audit understates its own system: ground truth has a second source (hot-start `countTokens`, no
  generation — `contextManager.ts:470-493`); the trigger is deficit-*growth* hysteresis, not a plain crossing
  (`:380-403`), with a second `normalizedTokens` tier (`:408-436`) and turn-0/last-turn protection (`:440-468`).
  Masking keeps previews and shell exit codes in history (`toolOutputMaskingService.ts:203-213,292-348`) —
  "replaced by file references" (audit adopt-item 4) is too strong — and its token math is estimated (`:121`); only the trigger is ground truth.
- The codex audit missed that the trigger itself can be scoped to post-prefix tokens
  (`context_window.rs:61-81`) — the strongest published example of cache-as-invariant feeding the pressure model,
  directly liftable for Zero's overflow test; also the 20k newest-first retained-user cap with a truncation
  marker (`compact.rs:63,691-760`), which matches Zero's `compaction_context` budget almost exactly — convergent design, so treat 20k as validated.
- Aider's ">1 s" rule is the rendered-map cache key (`repomap.py:599-613`); tag parsing is separately
  mtime-incremental (`:233-264`). W6's "change-triggered rebuild" must account for the freeze interaction
  (§3.3): the rebuild lands on compaction, not on turns.
- pi.md's anchors are stale; the live file `packages/coding-agent/src/core/compaction/compaction.ts` has
  defaults :126-130, trigger :267-270, hybrid occupancy :196-224, usage invalidation :226-262, findCutPoint
  :446-501, split-turn merge :994-1033, merge prompt :540-579/717-733, file ledger :51-88/1056-1069,
  length-guard :585-593, `cacheRetention:"none"` :629-633 (audit cited 428-470/636-666/49-83 same mechanisms).
  Two missed behaviors: usage invalidation after `context_edit`; never cutting at a tool result (`:351-364`).
- Crush's trigger reads the *previous step's* usage (`agent.go:1103`), so a single huge result overshoots before
  any gate acts — the exact case masking (run on the fresh result) exists for; argument for lever ordering, not a third trigger.
- `02-COMPETITOR_AUDITS` has no oh-my-pi file; `09` W1's "oh-my-pi argument canonicalization" was not re-verified
  here (loop-detect scope). On the lost owner ideas (`drops.md`): metrics yes-now; symbol graph yes-later and as a tool, not in the prompt (§3.4).

## 7. Sources and commit pins

Clones READ-ONLY under `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/` (`SOURCES.md`):
gemini-cli `42c51822084a`, codex `6786dccbe726`, aider `5dc9490bb35f`, pi `3daa20280f6a`, crush `d6edc3222231`,
kilo-code `4ee8d1b3b1f8` (read, out of scope), oh-my-pi `bcd11cb1ccb` (clone; no audit). File:line above re-read
at these heads this session; crush/kilo anchors otherwise carry the audits' verification.
Zero Code: `docs/harness-engineering/PRIOR_WORK.md`; `spec/context-engine.md`; `spec/drops.md`;
`01-ZERO_CODE_BASELINE.md` (98/0 measurement); `05` F3; `06` R4/R5 + E3; `07` S1/S7;
`09-IMPLEMENTATION_ROADMAP.md` W2/W4/W6 + Phase 2. Measurements: `docs/evals/2026-09-23-long-horizon-results.md`,
`2026-09-17-zero-stable-prefix-ab.json` (null result — both arms 95.1 % hit on a 2-request task; the
frozen-prefix case rests on warm-vs-cold engine numbers, not this A/B), `SYNTHESIS-2026-09-29.md`,
`2026-10-14-p0-qa-spec.md/.json`. Prior research: `docs/research/2026-09-15-memory-architectures/`
(code-graphs.md — LocAgent ablations, CodeScaleBench report, context-rot papers; titans/reasoningbank/hindsight
are memory-architecture lines, none bolt-on today — Titans ships no code/weights at ≤760M scale; the others inform the memory study, not this one).
