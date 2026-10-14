DRAFT — P2 study, written 2026-10-14, awaiting cross-check

# Tools and editing — subsystem study

Scope: tool surfaces and declaration cost, edit strategies, read/search output shaping, tool-call
repair, permissions, tool deferral. Strongest audited peers for edit reliability and tool-surface
economy: **codex** (apply_patch + deferred tools), **pi** (four-tool surface + argument repair),
**crush** (freshness enforcement + loop invariants), **aider** (edit formats as data + failure
reports). Zero Code's behaviour is specified in `docs/harness-engineering/spec/tools-editing.md`
(cited `spec`); claims marked (verified) were re-read at the pins in the read-only clones.

## 1. How each implementation works

### Codex (codex-rs) — patch grammar, capture-time caps, deferred discovery
- apply_patch grammar `*** Begin Patch` / Add / Update (+`@@` hunks, move) / Delete at
  `apply-patch/src/parser.rs:12-41`; heredoc shell form tolerated (`invocation.rs:164-183`);
  function-vs-freeform is per-model metadata in `models.json`.
- Matching is a monotone relaxation ladder: exact → trailing-whitespace-insensitive → both-ends
  trim; EOF-anchored hunks search from the end first; pattern-longer-than-file early-returns to
  avoid a past panic (`apply-patch/src/seek_sequence.rs:1-40`, verified).
- New content is derived fully in memory per file before any write; the unified diff is for display
  only (`file_update.rs:25-60,176`). The whole patch passes the same safety gate as exec —
  `assess_patch_safety → AutoApprove|AskUser|Reject` (`core/src/safety.rs:19-67`) — so an
  out-of-sandbox patch is an approval, never a silent write. A streaming parser streams a live diff
  to the UI *while patch tokens are still arriving* (`streaming_parser.rs`, `turn.rs:2638-2640`).
- Output caps happen at capture: 1 MiB per exec call, ≤10 000 streamed deltas, head+tail buffer,
  1/3–2/3 stdout/stderr rebalancing (`core/src/exec.rs:58,85,883-905`) — what the model saw is what
  the rollout records. Per-model `truncation_policy {mode: bytes|tokens, limit}` lives in the model
  table (`protocol/src/openai_models.rs:393`).
- Deferral: tiny always-on core; MCP/plugin tools searched by BM25 `tool_search`, exposed "for the
  next model call" (`tools/handlers/tool_search_spec.rs:94`, verified). Parallelism is one RwLock:
  readers fan out, mutators take the write lock (`tools/parallel.rs:46-51`).

### pi — four tools, two-tier matcher, argument-shape repair
- Default surface is exactly read/bash/edit/write (`core/tools/index.ts:164-171`); with
  grep/find/ls absent the prompt says "use bash for file operations" (`system-prompt.ts:103-118`)
  — absence is announced, not silent.
- edit matcher: exact `indexOf` → fuzzy space (NFKC, per-line trimEnd, smart quotes/dashes → ASCII)
  (`edit-diff.ts:34-55,221-222`); uniqueness enforced *inside fuzzy space* with an instructive
  "Found N occurrences … provide more context to make it unique" error (`:247-251,267-271`,
  verified); a fuzzy match is written back line-block-by-block so untouched lines keep original
  bytes (`:132-173`). No similarity tier anywhere.
- `prepareEditArguments` un-stringifies a JSON-string `edits` array, wraps a single edit object,
  folds legacy top-level `oldText/newText` — the comment names offending models (Opus 4.6,
  GLM-5.1) (`edit.ts:103-134`, verified). Length guard: `stopReason === "length"` fails *all* tool
  calls in the message with "re-issue with complete arguments" rather than executing salvaged
  partial JSON (`agent-loop.ts:264-270,478-503`).
- Deferral is transcript-native: prompt/tool declarations are transcript state;
  `declareToolChanges` synthesizes `toolsAdded`/`toolsRemoved` system messages so replay converges
  on the executable set (`agent-loop.ts:323-375`); exposure classes + `prepareLoadout` (`extensions/types.ts:510-516,629-638`).
- Permissions: none built in — full access; blocking is an extension returning `{block,reason}`.
  Output: 2000 lines OR 50 KB, spill to file, "Full output: <path>" printed into model text
  (`tools/bash.ts:348-352`).

### crush — freshness table, result-aware doom signature, no-dispatch-on-abnormal-finish
- Read-before-edit enforced in code: `filetracker.LastReadTime` zero → "you must read the file
  before editing it. Use the View tool first"; mtime newer than last read → rejection quoting both
  timestamps (`internal/agent/tools/edit.go:289-303`, verified). The tracker persists in the SQLite
  `read_files` table (`internal/filetracker/service.go`), surviving restarts and compaction.
- Edit matcher has two tiers: exact + whole-line whitespace-normalized (`edit_whitespace.go:26-66`);
  fails honestly rather than rewriting a 65 %-similar region.
- Doom loop: SHA-256 over (tool name, input, *result*) per step, paired by ToolCallID; >5 identical
  signatures in a 10-step window *halts* the turn (`loop_detection.go:11-13,41-70`, verified).
- Tool calls from an abnormal finish (length/content-filter/error) are recorded but never repaired
  or dispatched — truncated arguments never execute (`fantasy:agent.go:1612-1723`);
  `validateAndRepairToolCall` may make an extra model call to fix malformed args (`:1741`) — a cost the contended local engine cannot pay.
- Bash surface economy via refusal: curl/wget/ssh/sudo/package managers rejected with steering text
  (`bash.go:75-130`); auto-background after 60 s with `job_output`/`job_kill` (`:54,319-321`); 30k
  output cap (`:53`). LSP tools default-on; `lsp_replace_symbol` edits by documentSymbol range,
  removing text matching for whole-symbol changes. Permission fan-out is session-scoped: events
  carry a session ID, reach only clients viewing it, fail-open for API clients (`server/events.go:176-241`).

### aider — edit formats as data, failure reports as the real repair
- Per-model behaviour is a table: `resources/model-settings.yml`, 357 entries (324 diff / 35
  diff-fenced / 4 udiff / 4 whole), loaded at import (`models.py:154-158`); format, fence dialect,
  reminder placement and quirks are data, not if-trees.
- SEARCH/REPLACE cascade: exact line-tuple → uniform-indent outdent/reindent → drop spurious
  leading blank → `...` elision with exactly-once segment matching (`editblock_coder.py:157-187,190-240`);
  the 0.8-similarity `replace_closest_edit_distance` tier exists but is **dead code** behind a bare
  `return` (`:183-188`, verified). Zero-match tries every other chat file (`:55-65`). udiff goes
  furthest: git cherry-pick + diff-match-patch fallback (`search_replace.py:548-563`), progressive context shrinking preferring preceding context (`udiff_coder.py:282-310`).
- The actual weak-model repair is the failure report: failed block echoed verbatim, "Did you mean…"
  actual lines at ≥0.6 ratio ±5 lines, "REPLACE already present" detection, "don't re-send the
  applied ones" — fed back as the next user message, max 3 reflections (`editblock_coder.py:84-124,602-628`).

### Zero Code today (spec, this repo)
Nine-tier cascade with block-anchor Levenshtein ≥0.65 and disproportionate-span refusal
(`edit.ts:683-749`); near-miss errors return the best-matching region (≥60 % line similarity, 40
lines/4 KB cap) for in-turn fix (`edit/near-miss.ts:45-53,189-207`); apply_patch gated to `gpt-*`
models, hiding edit/write (`registry.ts:331-334`); repair pipeline alias → key alignment
(`file_path→filePath`) → JSON coercion → undeclared-name reroute to `tool_exec`
(`session/llm/repair.ts:17-57,274-343`); doom redirect at 3rd identical *settled* call
(`session/doom-loop.ts`); truncate service 2000 lines/50 KB with spill + reclaimable-bytes ledger
(`truncate.ts:76-143`); tool_search defers MCP at ≥8 tools or ≥5 % of context;
`ZERO_TOOL_PROFILE=coding-minimal`. W3 gaps (spec §Output shaping): no literal `(no output)`
marker, reclaim accounting diverges from the rewrite, grep's hard 100 cap has no pagination.

## 2. What is genuinely best, and why

1. **Freshness enforcement (crush).** The only audited implementation that *mechanically* refuses
   stale edits, at the cost of one table and one `stat`. In-house evidence it matters:
   `read-before-edit 1/4` on R4 (`06-EXPERIMENT_DESIGN.md` §R4) — our prompt asks, the model
   ignores, and stale `oldText` burns a cascade run.
2. **Failure-report engineering (aider, extended by our near-miss).** aider's echoed-block +
   did-you-mean + don't-resend report is the highest-leverage weak-model artifact in any audit;
   Zero's near-miss embeds it in the same error — one fewer round trip on a slow local decode.
3. **Two-tier predictability (pi/crush) vs nine-tier recall (Zero).** Nobody in production replaces
   at 65 % similarity: pi has no similarity tier, crush fails honestly, aider's is dead code
   (verified). Zero is the only live 65 % implementation anywhere — our fuzzy tiers are an
   unmeasured asset-or-liability, which is exactly why E2 arm (b) exists.
4. **Deferral (pi's mechanism, codex's ranking).** Both converge on one vocabulary: tiny always-on
   core + one search seam. Our measurement supports the payload half: `2026-09-17-tool-search-ab.md`
   −36 % schema bytes/req, −3 005 first-turn tokens, accuracy neutral; `2026-09-17-pi-tool-profile.md`
   −62 % median wall / −52.6 % schema with four tools (one task).
5. **Never dispatch a length-truncated call (pi, crush/fantasy).** For models that truncate tool
   calls, a salvaged partial patch hunk applying silently is the worst failure mode; both peers
   refuse execution outright. This belongs as an invariant, not a repair.

## 3. What to combine

- **The one deferral seam.** Adopt pi's transcript-native loadout (declarations +
  `toolsAdded`/`toolsRemoved` transcript deltas) as the single exposure mechanism, codex-style BM25
  `tool_search` behind it, and retire the four overlapping reach-the-right-tool layers (baseline
  finding #6: registry × repair × search × code-mode). Keep the measured trigger (catalog ≥5 % of
  window) and rare-builtin deferral (already default on).
- **Edit format decision for LOCAL MODELS.** Our failure modes are misspelled argument keys and
  truncated calls (spec §repair). Truncation is catastrophic for patch grammar — a patch emitted
  without its `*** End Patch` either fails whole (cheap) or mis-anchors (corrupting) — while a
  truncated `edit` call costs one re-issue. Decision: function-call `edit {path, edits[]}` stays
  the **sole default for all local models**, protected by the length-guard invariant (drop, never
  execute, length-finished calls) plus existing key-alignment/coercion repair; apply_patch is
  offered only via the aider-style *model metadata table* (per-entry `editFormat`, replacing the
  `registry.ts:331-334` `gpt-*` if-code); fuzzy tiers ship only where E2 shows rescue > corruption,
  with the disproportionate-span guard retained either way.
- **Freshness enforcement (crush read-table).** One durable `file_reads` table keyed
  (session, path) → last-read mtime, written by read and every successful edit; edit/write/
  apply_patch refuse when `lastRead` is zero or mtime moved, crush's message shape. Compaction
  carries the live-path set (pi's `readFiles/modifiedFiles` ledger) so post-compaction edits aren't
  falsely stale.
- **Result-aware doom signature (crush) + our redirect** — merged in §4. **Capture-time caps
  (codex)** — cap bytes where they are born (shell reader, rg stream), never rewrite stored output.

## 4. Proposed design for the new core

1. **Single dispatch seam.** One ordered pipeline, the only place tool policy lives: salvage-parse
   args → length-guard (drop whole message) → repair (alias, key-align, coerce, reroute) →
   permission gate → freshness gate → per-file mutation queue → execute → capture-time cap →
   doom-signature. Built-ins, MCP tools and future code-mode calls all traverse it; parallelism
   follows codex's RwLock rule (mutators exclusive, rest shared), per-file queue as write-lock.
2. **Declaration budget in tokens.** Budget *declared-tool tokens* (model tokenizer, not bytes —
   local prefill is token-priced) against a window share (start 5 %, matching the measured
   `TOOL_SEARCH_AUTO_SHARE`). Exposure ranking: core four always; previously-used tools next
   (session frequency); the rest behind `tool_search` with pi-style transcript deltas so
   prefix-cache stability and replay are preserved.
3. **W3 issues resolved.** (a) *Literal no-output marker*: the single representation of an empty
   result is the literal string `(no output)`; no spill file, no rewrite — tests assert the exact
   string in the transcript. (b) *One reclaim accounting*: truncate computes the rewritten text
   once and derives reclaimed-bytes from that same string, so ledger and artifact cannot diverge;
   compaction re-trim updates the ledger through the same function. (c) *Grep pagination*: replace
   the hard 100-cap message with a `continue_token` (rg `--max-count` page + after-line cursor),
   same shape as read's `offset=` hint so the model learns one pattern.
4. **Freshness and repair as data.** Per-model metadata table (aider/codex pattern) carries
   `editFormat`, fuzzy-tier allowance, key-alias additions; behaviour branches on the row, not the id.

## 5. Measurable targets (E2 metrics and neighbours)

- **Tool-error rate** (E1 ledger: invalid args, duplicate read, noop edit): ≤5 % of calls *reaching
  the model as errors* after repair; repair counts logged per strategy so dead strategies die.
- **Declared tokens/req**: coding-minimal arm ≥30 % below full (byte baseline 21 188 → 10 038);
  deferred-catalog arm keeps the ≥36 % reduction measured 2026-09-17; declared share ≤5 % of window.
- **Edit-attempt success by tier** (E2 telemetry, `js-cli-json-flag`/`py-duration-bug`, n=3):
  first-attempt success ≥85 %; fuzzy rescue/corruption ratio >1 to keep a tier; near-miss
  in-turn-fix rate ≥50 %; corrupted-region incidents = 0 for any retained tier.
- **Freshness**: read-before-edit 100 % enforced (replaces R4's observed 1/4); stale-edit refusals
  tracked as a quality signal, not an error. **Doom**: loops terminated by ≤4 calls; zero false
  terminations on polling shapes (new result ⇒ not counted).

## 6. Beyond / correcting the audits

- **crush audit lesson #3 ("include results in the doom signature — strictly better than
  input-only") is stale for Zero Code**: our doom loop already keys on same-args + same-*settled
  result* (`spec/tools-editing.md` §doom loop). The real crush differences are its window/count
  aggregation (10-step window, >5 repeats, `loop_detection.go:11-13`, verified) and that it *halts*
  where we redirect — ours is stronger; take only the configurable window.
- **aider audit §2 overstates its cascade**: `replace_closest_edit_distance` is unreachable behind a
  bare `return` (`editblock_coder.py:183-188`, verified). Corrected: no audited harness has shipped
  similarity-based replacement in production; E2's fuzzy-disarmed arm tests an unbenchmarked position.
- **pi audit §9 ("no name-alias repair") is right but incomplete**: pi's uniqueness-count error
  (`edit-diff.ts:267-271`, verified) *is* a repair mechanism — it states the exact corrective
  action. Our near-miss errors should echo occurrence counts the same way, merging aider's report
  style with crush's terse refusal.
- **codex's capture-time truncation** (`exec.rs:58`) answers our W3 reclaim divergence: cap at the
  byte source and stop rewriting stored output — the transcript holds exactly what the cap emitted.
- **crush's bash ban-list** (`bash.go:75-130`) is an under-rated tool-surface economy: refusing six
  tools inside bash's error text is cheaper than deferring them and teaches the substitution
  in-place — copy as guidance text on refusal, not as a hard ban.

## 7. Sources and commit pins

- Repos (read-only under `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/`, pins
  per `docs/harness-research/SOURCES.md`): codex `6786dccbe726` (2026-10-13); pi `3daa20280f6a`;
  crush `d6edc3222231` (2026-10-13); aider `5dc9490bb35f` (2026-05-22). Crush loop internals cited
  `fantasy:` = `charm.land/fantasy` v0.45.1 Go module cache per the crush audit header.
- Audits `02-COMPETITOR_AUDITS/{codex,pi,crush,aider}.md`; Zero behaviour
  `docs/harness-engineering/spec/tools-editing.md`; evidence
  `docs/evals/2026-09-17-pi-tool-profile.md`, `docs/evals/2026-09-17-tool-search-ab.md`,
  `06-EXPERIMENT_DESIGN.md` R4/R5 + E1/E2, `docs/harness-engineering/PRIOR_WORK.md`.
- Re-verified at the pins this session: `codex-rs/apply-patch/src/seek_sequence.rs:1-40`;
  `crush/internal/agent/tools/edit.go:289-303`, `internal/agent/loop_detection.go:11-70`;
  `pi/.../tools/edit.ts:103-134`, `edit-diff.ts:247-271`; `aider/coders/editblock_coder.py:155-188`.
