# 10 — Executive assessment

Fourteen direct answers. Each is traceable: numbers cite 06's obtained results (R1 ladder), mechanisms cite the
head-pinned audits in 02-COMPETITOR_AUDITS/, Zero facts cite 01. Where the honest answer is "unmeasured", it says so.

**1. How competitive is Zero Code today?** Top-quartile on mechanisms, mid-pack on shipping them. Its tool/edit/
recovery layer is the strongest surveyed (04 §B); its context engineering is #1–2 open-source with measured
justifications; but the *live* loop is the legacy monolith, the better V2 core is flag-off and incomplete, and the
long-horizon autonomy is half-shipped: a real done-ness driver (`/goal` + `--verify` oracle + continue/todo
drivers, default-on) and a measured auto review gate exist as **plugins**, but non-durably (restart kills the
driver and resets the cap), and nothing wakes an existing session after process exit (01 Part D). On identical local models, harness-side changes
measurably moved task quality in our own ladder (R1) — the capability is real; coherence is not.

**2. Strongest overall architecture?** Codex (open source): typed Op/EventMsg protocol, per-step immutable context,
RwLock parallel gate, OS sandbox with classified denials, rollout durability, single execution path. Claude Code is
strongest *as a product* (cache economics, tool surface, plugin ecosystem) but its internals are unavailable and
its loop is inferred. For Zero's purposes the reference architecture is Codex's kernel + OpenCode's shipped
durable-admission loop.

**3. Best agent loop?** Codex for the shipped loop. Zero V2 / OpenCode share the best *design* (durable admission,
boundary promotion, positive tool-call continuation, settleable interruption); only OpenCode ships it. Pi's is the
purest and most instructive but intentionally policy-poor.

**4. Most effective tool design?** As a *set*: pi (four tools, prompt-adaptive). As *engineering*: Zero Code's
(edit cascade + output shaping + repair + doom-loop + checkpointed shell jobs). As *interface for weak models*:
Codex's apply_patch ladder and aider's per-model formats — three valid answers, no same-model head-to-head exists
(that's experiment E2).

**5. Most intelligent context handling?** Gemini CLI's measured pipeline (API ground-truth triggering, masking with
protection window) is the most principled open implementation; Zero V1's compaction is the most *cache-intelligent*
(98% continuation cache, verbatim-tail sizing, retained user messages); Claude Code has the most sophisticated
*economic* handling, closed. Nobody has all three (08 §3 combines them).

**6. Best long-horizon execution architecture?** OpenHands SDK open-source (stateless step + event-sourced durable
status + fork); Antigravity closed-source and best-in-product (SQLite step snapshots, auto-resume of active turns,
unlimited headless runs, remote control). goose has the best *scheduled* story (recipe × scheduler); nobody yet
combines scheduling with durable session resume (07-S2's opening).

**7. Most useful multi-agent coordination?** Antigravity (declarative markdown agents, worktree branch/share
modes, roster injection, subagent trajectory correlation) — closed but thoroughly evidenced by changelog +
binary strings. Open: kimi's swarm (128 subagents, resume map) plus its access-declared conflict scheduler is the
only *safe parallel writes* machinery in the open survey.

**8. Most effective with weaker local models?** Zero Code, by construction and measurement (repair/coercion/doom-
loop/truncate-hint stack tuned on local ladder runs; harness overhead <1%). aider's per-model format tables and
kimi's trait flags are the best *data-driven* adaptations Zero should absorb; mini-SWE proves the low bar but only
with eval privileges Zero lacks.

**9. What did Google improve or change by moving to Antigravity CLI?** Not a rename: a Go rewrite on the shared
desktop engine that kept Gemini CLI's *migration surface* (config home, GEMINI.md, extension manifests, API-key
route) while replacing loop, runtime, session format, and vocabulary. What improved: durable per-step execution,
async/unattended posture (queue/auto-resume/remote-control/unlimited headless), a stable machine API (closed-vocab
NDJSON, structured errors, exit codes), permission UX (request-review default, pinned allow-suggestions, reviewing
agent), budget-partitioned context. What didn't demonstrably improve: task outcomes — never measured publicly, and
blocked for us by the auth wall (03 §6). Their source-verified predecessor still beats them in *published* context
machinery (ground-truth compaction, loop-recovery). Key lesson set: 03 §7–8.

**10. Five biggest mistakes in current coding-harness design?**
(i) Treating "finish=stop" as completion — only Zero Code's own (non-durable) goal plugin verifies done-ness
in-product (01 D1; everywhere a
premature-success anecdote is really this gap).
(ii) Estimating context occupancy instead of measuring it, and compacting globally instead of by budget floors —
eviction bugs ("skills got compacted away", the reported Antigravity class; Zero's own prune-before-compaction risk).
(iii) Two execution paths coexisting (Zero V1/V2, Cline's XML/native migration, Gemini's dual context systems,
Antigravity's admitted 1.1.17 merge) — every fix twice, drift guaranteed.
(iv) Paying schema tokens for rarely-used tools via multiple overlapping discovery layers instead of one deferral
mechanism (six teams converged; Zero runs four layers).
(v) Trusting the model's self-report of irreversible actions instead of OS-level enforcement + typed denials
(Codex/Gemini/Kilo sandbox lane is the minority position it should not be).

**11. Adopt / adapt / reject?** **Adopt**: durable admission + replay-derived status (V2/OpenHands/Qwen), per-step
checkpoints with fork (Codex), token-ground-truth triggering + output masking (Gemini), exact-first edit cascade +
freshness enforcement (Zero + crush), one deferral mechanism (pi), wake recipes + session→recipe (goose), done-ness
oracle at idle (omO — which Zero's goal plugin already is, plugin-side: port and make durable), closed machine
contract (Antigravity), model-tables-as-data (aider/kimi/qwen), FTS5 indexing/ranking over the shipped text
search. **Adapt**: PTC as presentation mode (DeepSeek discipline over Zero's interpreter, gated by E1d); cheap
reviewer-agent boundary calls (Guardian/Auto pattern, priced by E6 before arming); worktree subagent modes
(Antigravity) + access-declared scheduling (kimi). **Reject**: role-theater pipelines (zero measured benefit);
next-speaker continuation (killed by its own upstream); whole-file-first editing; benchmark-prose-driven design;
Cline's stash-as-checkpoint; aider's litellm-style provider enumeration; Coder/god-object runtimes; DeepSeek-style
dual TS+Python runtimes.

**12. Keep Zero Code's core or adopt a different foundation?** Keep it (Option B in 08). Evidence: the differentiating
value is the tool/recovery/context layer — none of the candidate runtimes (pi, codex-core, OpenHands, goose) has
it; V2 already embodies the right durable design; OpenCode demonstrates the same design shipping successfully;
the engine seams (Location services, event store, native LLM protocols) are where the missing pieces (wake,
objective, portfolio, ModelTable) attach without replacement. A rewrite would re-debug the edit/permission/
checkpoint layer that competitors demonstrably lack, to gain nothing the porting plan doesn't give.

**13. Which combination can outperform every individual system?** 07's stack, named by its edge: durable admission
+ scheduled wakes + a **durable done-ness oracle** (S2) makes autonomy honest and unattended — no audited system
has all three; the **context portfolio** (S1) maximizes useful information per token at every window size;
**verification-in-depth** (S5) makes false completion a measured rate; **model-aware orchestration** (S4) lets a
local primary buy frontier judgment only at priced boundaries (aider's economics + Zero's measured review
pricing). The individual pieces are proven elsewhere; the *compositions* close exactly the gaps each source
audit documented — that's the whole argument, and every gate is a zero-eval number, not a belief.

**14. What should be built first?** Phase 1: port V1's reliability machinery into V2 and give it a real async
surface, then flip the default and delete the monolith (the "1.1.17 moment"). In parallel, the quick wins W1–W6
(eval-boot fix, archive-path surfacing, grep paging, ledger feedback loop, FTS5, repomap refresh discipline) and
E1–E3 so Phases 2–3 land against measurements, not opinions. First end-to-end proof: the Phase-2 vertical slice —
`long-qa-spec` at forced 64k context with ≥80% recall, halved prefix-breaks, and hidden-test pass, reported in
`docs/harness-research/`.

## Closing judgment

Zero Code is one coherent-architecture push away from field leadership. The research found no magic loop —
the field converged on a seven-part skeleton (§05 meta-finding); Zero already owns the best tool layer and the
second-best context layer in the open set, and its own measurements already price the hard tradeoffs. What it
lacks is making its shipped plugin organs durable (goal verify/continues, design gate, guard lints), arming the
executive on evidence, and a wake mechanism for existing sessions. Those are buildable in the order
09 gives, each behind an experiment that can also say "no".
