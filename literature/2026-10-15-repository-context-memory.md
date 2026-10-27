# Repository retrieval and cross-task memory

**Status:** technical study and proposed evaluation; no new local runs.

## Distinct failure modes
Cold retrieval finds relevant code, tests and call sites for the current task. Cross-task memory recovers relevant lessons from *earlier tasks*. These are different from within-task compaction and revision-bound evidence, already studied in `context-engine.md` and `literature/2026-10-14-execution-state-evidence-memory.md`. Mixing them hides causality.

## Evidence
Agent Retrieval Bench (https://arxiv.org/abs/2607.24882) isolates file retrieval failures and evaluates retrieval seeding. VibeMemBench (https://arxiv.org/abs/2609.23570) finds that a useful memory exists more often than an integrated memory system successfully helps. Neither implies that adding vector search improves patch completion.

## Retrieval design
Compare lexical grep and filename search, symbol/RepoMap retrieval, local embeddings and a hybrid candidate union with bounded reranking. Each returned hit must contain path, repository revision, line range and evidence snippet. Freeze the exact initial repository snapshot and task wording; test file recall@k and time to first relevant file **separately** from patch correctness. Avoid judging retrieval only by model-written explanations.

## Memory design
Compare memory-off, raw historical transcripts, compact verified facts, and oracle-selected useful memories (upper bound). Entries include source task, repository commit, affected paths/symbols, test or verifier receipt, expiration/invalidation rules and explicit uncertainty. Keep the memory store out of the evaluator and prohibit retrieving gold solution patches from held-out tasks. Split by repository and chronology, not merely randomly by text chunk.

## Evaluation
Use the same local model, inference engine, context-token cap, hidden verifier and task snapshots across arms. Record retrieval precision/recall, distractor count, stale-memory incidence, context length, cache behavior, indexing and query overhead, wall-clock time and verified task resolution. Include an irrelevant-but-similar-memory negative control and external file mutations. If retrieval F1 improves but final-state completion does not, label it a retrieval gain only.

## Recommendation
Start with revision-aware lexical+symbol search and small verified memory records. Add embedding search only when traces show persistent missed relevant files. Reject systems that consume more local inference time than they save.
