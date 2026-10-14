# Verification report: code-graph tooling and context-rot claims

Date of check: 2026-09-15. Method: every claim treated as a hypothesis; primary sources fetched (arXiv HTML/abs pages, ACL Anthology, vendor blogs via curl, GitHub API, and the actual repositories cloned into the scratchpad and read). WebFetch summaries were cross-checked against raw HTML tables where numbers mattered. Em dashes inside quotations were replaced with commas.

Cloned for code reading:
- /private/tmp/claude-501/-Users-zero-Documents-Dev/0012f4b7-2c66-4407-b505-d5ac12b403a3/scratchpad/repos/LocAgent (commit 4935b55, 2026-08-12)
- /private/tmp/claude-501/-Users-zero-Documents-Dev/0012f4b7-2c66-4407-b505-d5ac12b403a3/scratchpad/repos/graphiti (commit c035afb, 2026-09-11, tag v0.30.2)
- CodeScaleBench README, technical reports, snapshot manifest and wins manifest fetched from raw.githubusercontent.com (branch `public`)

---

## Claim 1. "A June 2026 study found long-horizon agents terminate early or give uncertain answers as context grows; pruning/managing context materially improves performance"

Verdict: VERIFIED, with a scope caveat (web deep-search agents, not coding agents).

### The June 2026 study

- Title: "Diagnosing and Mitigating Context Rot in Long-horizon Search"
- Authors: Shijie Xia, Yikun Wang, Zhen Huang, Pengfei Liu (Shanghai Jiao Tong University, Fudan University, GAIR; Pengfei Liu corresponding)
- Venue: arXiv preprint only (no venue listed). arXiv 2606.29718, v1 submitted 29 June 2026, v2 4 August 2026.
- URL: https://arxiv.org/abs/2606.29718 (HTML: https://arxiv.org/html/2606.29718v2)
- Code: https://github.com/GAIR-NLP/ContextRot (12 stars, no license file listed by the GitHub API, last push 2026-08-04). Strategy names in the repo: react, discard, summary_semantic, summary_length, summary_turn, keep_k_latest, keep_k_latest_wo_any, keep_k_latest_wo_reasoning, fold.

The v1 abstract matches the claim's wording almost exactly: "extensive context causes models to directly give up or prematurely provide uncertain answers, and this issue is exacerbated as the context grows. Through pruning experiments, we demonstrate the relationship between the accumulated context and the rot phenomenon." The v2 abstract renames this "premature termination: under extensive context, models give up or provide uncertain incorrect answers long before exhausting the context window. By controlling for query difficulty, we show that the premature termination rate is positively correlated with context length."

What was actually measured:
- Four open-weight models: GLM-4.7, GLM-5.0, Qwen3.5-397B-A17B, MiniMax-M2.5.
- Three deep-search (web browsing QA) benchmarks: BrowseComp, BrowseComp-Plus, xbench-DeepSearch. No coding tasks, no SWE-style agents.
- Terminal states are classified (Table 1/2): Confident Correct, Uncertain Correct, Confident Incorrect, Uncertain Incorrect, Give Up, No Answer. "Premature termination" (PT) = give-up plus uncertain-incorrect.
- Seven context management methods in three categories: compaction (Summary by Length, by Turn, Semantic), trimming (Discard-all tool responses except the last, Keep Latest, Keep Latest with summary), isolation (FoldAgent: sub-agents return only summaries).

Numbers from Table 3 (parsed from the arXiv HTML directly), BrowseComp, accuracy / PT rate:
- Qwen3.5-397B-A17B: ReAct baseline 35.0% / 53.4%; Summary (Length) 46.6% / 2.4%; Discard 44.6% / 19.4%; Keep Latest (w/ sum.) 48.2% / 16.2%; FoldAgent 54.0% / 6.4%. Overall average across the three benchmarks: ReAct 54.4 vs FoldAgent 64.9.
- GLM-4.7: ReAct 33.8% / 58.2%; Summary (Length) 49.0% / 1.4%; Discard 46.0% / 33.6%; FoldAgent 44.0% / 3.8%. Overall: ReAct 51.8 vs Summary (Length) 60.4.
- Parallel sampling with "behavior-aware filtering" (drop give-up and uncertain trajectories before aggregation): "performance gain of 2.6% to 4.9% across the three aggregation methods".
- Stated selection principle: "for backbone models with strong agentic ability, context isolation using sub-agent calls can outperform other methods; whereas for models with weaker agentic ability, combining context compaction and context trimming achieves the best balance between performance and cost."
- The authors frame the management methods as "inherently test-time scaling strategies that reduce the premature termination rate to enable more exploration": tool-call counts rise sharply (e.g. Qwen3.5 ReAct 21.7 calls vs Summary (Length) 57.7 calls), so the gain is bought with more steps, not free.

So "materially improves performance" is supported (roughly +10 to +19 accuracy points on BrowseComp), but the setting is web search with open-weight models; extrapolation to coding agents is the reader's inference, not the paper's finding.

### The earlier evidence: Chroma "Context Rot" (July 2025)

- Title: "Context Rot: How Increasing Input Tokens Impacts LLM Performance"
- Authors: Kelly Hong, Anton Troynikov, Jeff Huber (Chroma). Published 14 July 2025.
- URL: https://research.trychroma.com/context-rot (301-redirects to https://www.trychroma.com/research/context-rot)
- Replication toolkit: https://github.com/chroma-core/context-rot (MIT, 305 stars, created 2025-07-14, last push 2025-09-03)

What was measured (18 models: Claude Opus 4, Sonnet 4, Sonnet 3.7, Sonnet 3.5, Haiku 3.5; o3, GPT-4.1 family, GPT-4o, GPT-4 Turbo, GPT-3.5 Turbo; Gemini 2.5 Pro/Flash, 2.0 Flash; Qwen3 235B/32B/8B):
1. Needle-in-a-haystack extension with controlled needle-question semantic similarity (cosine 0.445 to 0.829 bands over Paul Graham essays and arXiv papers). Finding: "performance degrades more quickly in input length with lower similarity needle-question pairs."
2. Distractors: baseline vs one vs four hand-written distractors. "Even a single distractor reduces performance relative to the baseline" and the effect grows with length.
3. Haystack structure: original vs shuffled sentences. Counterintuitive: "models perform worse when the haystack preserves a logical flow of ideas. Shuffling the haystack and removing local coherence consistently improves performance."
4. LongMemEval conversational QA: 306 prompts, full history (about 113k tokens) vs focused prompt (about 300 tokens of relevant turns). "Across all models, we see significantly higher performance on focused prompts compared to full prompts."
5. Repeated-words replication task (copy a sequence with one unique word), 12 lengths from 25 to 10,000 words. "As context length increases, performance consistently degrades across all models."

Not measured: agents, tool use, coding, summarization, multi-step reasoning. The report itself says real applications "demand significantly more processing and reasoning" than the tested tasks. The 2026 GAIR paper is the first of these two to measure agentic trajectories.

---

## Claim 2. Sourcegraph CodeScaleBench: "1,281 agent runs on 40+ large repositories ... grep/read broke down ... gains around 400K to 2M LOC ... Kubernetes example 6,000 s no answer to 89 s score 0.90"; "Sourcegraph 7 ... intelligence layer for agents"

Verdict: VERIFIED as to what Sourcegraph published; the quoted numbers all appear verbatim in a Sourcegraph marketing blog post. The benchmark's own technical report shows a much smaller aggregate effect and internally inconsistent size-band tables, so treat the headline numbers as selected anecdotes.

### Does CodeScaleBench exist? Yes.
- Repo: https://github.com/sourcegraph/CodeScaleBench (Apache-2.0, 36 stars, created 2026-02-01, default branch `public`, last push 2026-07-06, tag `v1-mixed371`). README badge says "Paper arXiv soon"; there is no arXiv paper yet.
- Dataset: https://huggingface.co/datasets/sgjarmak/CodeScaleBench (370-task problem layer).
- Technical report: https://raw.githubusercontent.com/sourcegraph/CodeScaleBench/public/docs/technical_reports/TECHNICAL_REPORT.md
- MCP vendor comparison: https://raw.githubusercontent.com/sourcegraph/CodeScaleBench/public/docs/technical_reports/MCP_COMPARISON_REPORT.md
- Runner: Harbor with the Claude Code agent harness; configs `baseline-local-direct` (full local code, built-in tools only), `mcp-remote-direct` (code reachable only through Sourcegraph MCP: keyword_search, nls_search, read_file, find_references, go_to_definition), plus `augment-local-direct` and `github-remote-direct`.
- Frozen snapshot `csb-v1-mixed371--haiku45--030326` (Claude Haiku 4.5, 371 tasks): mean reward 0.5357 baseline vs 0.5647 with Sourcegraph MCP; mean agent seconds 267.7 vs 158.1; mean cost per task $0.2994 vs $0.3307 (MCP costlier in this snapshot).

### The blog post that the claim paraphrases
"Why coding agents fail in large codebases (and what to do about it)", Stephanie Jarmak, 8 May 2026, https://sourcegraph.com/blog/why-coding-agents-fail-large-codebases (WebFetch is blocked with 403; fetched with curl and a browser user agent). Exact quotes:
- "Data from 1,281 agent runs across 40+ large open source repos reveals five repeatable failure patterns in coding agents"
- "CodeScaleBench, a benchmark of software engineering tasks evaluated across 40+ of the largest open source repositories spanning 9 programming languages. The observations come from 1,281 scored agent runs"
- Kubernetes: "tracing the Dynamic Resource Allocation (DRA) system through 1.4 million lines of Go spread across 22,000 files ... At 6,000 seconds (over an hour and a half), the agent has produced nothing. No analysis. No output. Zero score." Then with index-backed tools: "Eight keyword searches and six semantic searches identify the relevant packages. One find-references call maps the cross-package dependency chain. In 89 seconds, the agent produces a comprehensive analysis and scores 0.90 out of 1.0."
- "Agents with only local tools (grep, file read, glob) begin to struggle systematically when codebases exceed roughly 400,000 lines of code." Table in the post: "< 400K: reward delta -0.080, Tools add overhead; grep typically works fine" and "400K-2M: +0.259, Strongest positive effect". No row above 2M is shown in the post.
- Other figures: Strata refactor 0.32 (2 of 7 files) vs 0.80; "96 tool calls over 84 minutes (including 6 complete reversals of approach) and scored 0.32" vs "5 targeted search calls in 4.4 minutes and scored 0.68"; "30% cheaper ($0.51 per task vs. $0.73 baseline) and 38% faster"; tool usage 7,993 keyword vs 2,449 semantic vs 57 deep-search calls; multi-repo +0.209 F1 vs +0.085 single-repo.
- The five failure patterns: lost in the codebase, wrong file/wrong symbol, partial completion, tool thrashing, context overflow.
Secondary coverage: Tessl, "What 1,281 agent runs reveal about coding agent failure in large codebases", Paul Sawers, 20 May 2026, https://tessl.io/blog/coding-agent-failure-patterns-large-codebases/ (repeats the same numbers).

### What the technical report actually says (analysis export of 3 March 2026)
- "Valid scored rows in export: 1,281"; 370 paired baseline/MCP tasks; 20 suites (9 SDLC + 11 Org).
- Overall paired reward delta: +0.0349 (95% CI [+0.0130, +0.0579]). SDLC +0.0363 (CI [-0.0083, +0.0835], includes zero). Org +0.0339.
- Several suites are negative with MCP: csb_sdlc_secure -0.0500, csb_sdlc_debug -0.0372, csb_sdlc_test -0.0113, csb_org_platform -0.0287, csb_org_crossrepo -0.0242, csb_org_domain -0.0165. Largest positives: sdlc_understand +0.1148, sdlc_refactor +0.1029, org_incident +0.1125, org_security +0.1057.
- Retrieval quality: Precision@10 0.095 to 0.313, Recall@10 0.120 to 0.272, F1@10 0.091 to 0.240. Cost per task $0.7333 to $0.5121 (-30.16%), mean wall-clock delta -36.22 s, mean agent-execution delta -101.06 s.
- Two different size-band tables exist and disagree:
  - Section 11.5, F1@10 delta by GitHub repo-size proxy: <400K (n=15) +0.2503; 400K-2M (n=31) +0.2618; 2M-8M (n=143) +0.1796; 8M-40M (n=74) +0.0719; >40M (n=3) +0.0242.
  - Section 11.9, reward delta by cloc LOC: <400K (n=50) +0.079; 400K-2M (n=169) +0.009; 2M-8M (n=129) +0.000; 8M-40M (n=47) +0.017; >40M (n=3) +0.108. The report's own summary sentence under this table ("strong positive lift in 400K-2M") contradicts the +0.009 in the table.
  - The blog's "-0.080 / +0.259" pair does not appear in either table of the public report.
- By language: TypeScript is the "strongest negative outlier" (n=7, -0.140). By difficulty: expert tasks -0.057.
- Correlation between retrieval MRR delta and reward delta: Spearman +0.1295, p=0.1533, "no statistically significant retrieval-outcome correlation signal yet".
- The Kubernetes 89 s / 0.90 anecdote is not traceable in the public wins manifest (59 MCP wins, models cc_haiku45, cc_sonnet46, oh_sonnet46). The closest public entry is `k8s-dra-scheduler-event-fix-001` (OpenHands + Sonnet 4.6): baseline 0.0, MCP 0.73. No public win has an MCP score of exactly 0.90.
- MCP comparison report: on Servo (10M LOC Rust, 133K files) Claude Code + Sourcegraph scored 0.83 (Opus) while Augment and GitHub MCP scored 0.00 across all trials; Cursor native and Cursor + Sourcegraph also 0.00.

### "Sourcegraph 7"
Real. "A new era for Sourcegraph: The intelligence layer for AI coding agents and developers", Graham Mcbain, 25 February 2026, https://sourcegraph.com/blog/a-new-era-for-sourcegraph-the-intelligence-layer-for-ai-coding-agents-and-developers. Quotes: "Today, we're announcing Sourcegraph 7.0, a release that marks the beginning of a new chapter"; "doubling down on being the intelligence layer that developers and AI agents rely on to navigate, understand, and operate on large codebases." Contents: Deep Search via an MCP server, MCP usage analytics, a versioned Sourcegraph API, code navigation inside Deep Search, Svelte UI, and removals (Cody Web for Deep Search customers, Search Notebooks). No quantitative claims in that post.

---

## Claim 3. LocAgent (arXiv 2503.09089, github.com/gersteinlab/LocAgent)

Verdict: VERIFIED. Paper, venue, repo, numbers all check out; the code reveals important simplifications.

### Paper and repo
- "LocAgent: Graph-Guided LLM Agents for Code Localization", Zhaoling Chen, Xiangru (Robert) Tang, Gangda Deng, Fang Wu, Jialong Wu, Zhiwei Jiang, Viktor Prasanna, Arman Cohan, Xingyao Wang. arXiv 2503.09089 (v1 12 March 2025, v2 29 April 2025): https://arxiv.org/abs/2503.09089
- ACL 2025 main conference long paper: https://aclanthology.org/2025.acl-long.426/ (pages 8697-8727, DOI 10.18653/v1/2025.acl-long.426).
- Repo: https://github.com/gersteinlab/LocAgent. Apache-2.0. 627 stars, 98 forks, 16 open issues. Created 2025-03-12. Last commit 4935b55 on 2026-08-12 ("Add note about CodeNib LocAgent integration"). Benchmark: https://huggingface.co/datasets/czlll/Loc-Bench_V1. Fine-tuned models: czlll/Qwen2.5-Coder-7B-CL and czlll/Qwen2.5-Coder-32B-CL.

### How the graph is built (read from dependency_graph/build_graph.py, VERSION 'v2.3')
- Language: Python files only (`file.endswith('.py')`); the paper states "we focus our study on Python repositories and leave codebases with other programming languages as future work."
- Parser: Python's standard-library `ast` (ast.parse, an ast.NodeVisitor subclass `CodeAnalyzer`, ast.get_source_segment). Tree-sitter is NOT used for the graph. `tree-sitter`/`tree-sitter-languages` are in requirements.txt and are used only by `repo_index/codeblocks` (a moatless-style chunker with python.scm/java.scm queries) for the BM25/embedding chunk index. There is a `handle_edge_cases()` string-rewriting hack (e.g. `except X, e` to `except X as e`, `print ` to `yield `) so Python-2-era files parse; files that still fail to parse are skipped.
- Storage: a `networkx.MultiDiGraph`, pickled per benchmark instance to `$GRAPH_INDEX_DIR/{instance_id}.pkl` (repo_ops.py). Node attributes: `type`, full `code` text, `start_line`, `end_line`. Node ids: `/` and relative directory paths for directories, relative file paths for files, `path/to/file.py:Class.method` for classes and functions (nested names joined with dots).
- Node types: directory, file, class, function (methods are function nodes contained by class nodes).
- Edge types: `contains` (dir to dir/file, file to class/function, class to method), `imports` (file to file/class/function, with an `alias` attribute; resolved by `resolve_module` against the repo tree), `invokes` (class/function to class/function), `inherits` (class to class).
- Call and inheritance resolution is name-based, not type-based. `find_all_possible_callee` walks up the `contains` chain collecting sibling and inner nodes, then the file's imports (following `__init__.py` re-export chains and aliases). Callee names are bare identifiers from `ast.Call` (`Name.id` or `Attribute.attr`). With `fuzzy_search=True` (the default) every candidate sharing the short name gets an `invokes` edge; with `global_import=True` (used by the runtime) any unresolved name is matched against a repo-wide short-name dictionary, so `foo()` links to every `foo` in the repository. Result: high recall, many false edges, no dynamic-dispatch or type resolution.
- Test files are excluded from search and traversal by a path heuristic (`is_test_file`: any path segment starting with "test").

### Indexes and search
- "Sparse Hierarchical Entity Indexing" in the paper corresponds to: (1) a global short-name dictionary (exact and lowercase) over node ids, (2) a BM25 index (bm25s / llama-index BM25) over entity names and over code chunks, (3) RapidFuzz fuzzy matching over node names as a last resort. An optional FAISS embedding index exists in `repo_index` but the agent path uses BM25 plus the graph.
- `search_entity` order: exact node-id match; `Class.method` splitting against the name dictionary; BM25 (restricted to include_files first, then global); fuzzy.

### Tools the agent gets (plugins/location_tools/repo_ops/repo_ops.py, `__all__`)
Code names (paper names in parentheses, Table 2 of the paper):
- `search_code_snippets(search_terms, line_nums, file_path_or_pattern)` (SearchEntity): keyword/entity search returning results at three detail levels (fold, preview, complete) with line-numbered code.
- `explore_graph_structure(start_entities, direction, traversal_depth, entity_type_filter, dependency_type_filter)` and `explore_tree_structure(...)` (TraverseGraph): BFS over the graph, direction upstream/downstream/both, hop limit (default 2; -1 means 20), node and edge type filters, output as an ASCII tree, a DOT/pydot string, or JSON.
- `get_entity_contents(entity_names)` (RetrieveEntity): full code and metadata for entity ids.
The agent loop is OpenHands-style CodeAct (`<execute_ipython>` blocks) or native function calling via litellm.

### Reported numbers (parsed from the arXiv HTML tables)
- Table 4, SWE-bench-Lite localization. LocAgent Qwen2.5-32B(ft): file Acc@1/3/5 = 75.91 / 90.51 / 92.70; module Acc@5/10 = 85.77 / 87.23; function Acc@5/10 = 71.90 / 77.01. LocAgent Claude-3.5: 77.74 / 91.97 / 94.16; 86.50 / 87.59; 73.36 / 77.37. LocAgent Qwen2.5-7B(ft): 70.80 / 84.67 / 88.32; 81.02 / 82.85; 64.23 / 71.53. Baselines (file Acc@5 / module Acc@10 / function Acc@10): OpenHands Claude-3.5 90.15 / 83.58 / 70.07; SWE-agent Claude-3.5 90.15 / 78.10 / 64.60; Agentless Claude-3.5 79.56 / 68.98 / 58.76; CodeRankEmbed 84.67 / 78.83 / 58.76; BM25 61.68 / 52.92 / 36.86.
  So "up to 92.7% file-level accuracy" = fine-tuned Qwen2.5-Coder-32B, file-level Acc@5 (ground-truth file anywhere in the top 5), on SWE-bench-Lite. The graph's edge over OpenHands is small at file level (92.70 vs 90.15) and larger at function level (77.01 vs 70.07).
- Table 5, cost: LocAgent Claude-3.5 $0.66 per example, 7 rounds; Qwen2.5-32B(ft) $0.09, 9 rounds; Qwen2.5-7B(ft) $0.05. "Approximately 86% reduction" = 1 - 0.09/0.66.
- Table 6, ablation (Qwen2.5-7B ft; file Acc@5 / module Acc@10 / function Acc@10): full 88.32 / 82.85 / 71.53; w/o TraverseGraph 86.13 / 78.47 / 66.06; relation types restricted to contain 86.50 / 79.56 / 66.42; traverse hops 1: 86.86 / 80.29 / 66.79; w/o RetrieveEntity 87.59 / 81.39 / 69.34; w/o SearchEntity 68.98 / 61.31 / 53.28; w/o BM25 index 75.18 / 68.98 / 60.22. Reading: the entity search index carries most of the value; graph traversal adds about 2 (file) to 5.5 (function) points.
- Table 7, Loc-Bench (560 instances: 242 bug reports, 150 feature requests, 29 security, 139 performance; collected after October 2024): LocAgent Claude-3.5 file Acc@5 83.39, module Acc@10 70.89, function Acc@10 59.29; Qwen2.5-7B(ft) 78.57 / 63.04 / 51.43; OpenHands Claude-3.5 79.82 / 68.93 / 59.11; CodeRankEmbed 74.29 / 63.21 / 43.39.
- Table 8, downstream repair (Agentless repair on top of different localizers): Agentless localization Claude-3.5: Acc@5 58.39, Pass@1 26.31, Pass@10 33.58; LocAgent Qwen2.5-32B(ft): 69.34 / 26.79 / 36.13; LocAgent Claude-3.5: 73.36 / 27.92 / 37.59. The "12%" is the relative Pass@10 gain 33.58 to 37.59 (+4.01 points absolute, 11.9% relative) with Claude-3.5 localization; Pass@1 moves only 1.6 points.

---

## Claim 4. Graphiti (github.com/getzep/graphiti)

Verdict: VERIFIED (descriptive); details below from README and code.

- Repo: https://github.com/getzep/graphiti. Apache-2.0. 30,895 stars, 3,138 forks, 506 open issues. Created 2024-08-08. Last push 2026-09-11 (commit c035afb). Package `graphiti-core` 0.30.2 (tag v0.30.2). Paper: "Zep: A Temporal Knowledge Graph Architecture for Agent Memory", https://arxiv.org/abs/2501.13956. Docs: https://help.getzep.com/graphiti.

### What the graph stores (graphiti_core/nodes.py, edges.py)
- EpisodicNode: `source` (EpisodeType message | json | text), `source_description`, raw `content`, `valid_at` (the caller's reference_time), `entity_edges` (uuids of facts derived from it), `episode_metadata`. Episodes are the provenance layer; every fact points back to episode uuids.
- EntityNode: `name`, `labels` (custom Pydantic entity types), `name_embedding`, LLM-written `summary`, `attributes` dict.
- EntityEdge (a "fact"): `name` (relation label), `fact` (natural-language sentence), `fact_embedding`, `episodes` (provenance list), `created_at`, `valid_at`, `invalid_at`, `expired_at`, `reference_time`, `attributes`. This is bi-temporal: valid_at/invalid_at are event time, created_at/expired_at are transaction time. Contradictions are resolved in `resolve_edge_contradictions`: an older overlapping edge gets `invalid_at = new_edge.valid_at` and `expired_at = now`; nothing is deleted.
- Also: EpisodicEdge (MENTIONS: episode to entity), CommunityNode/CommunityEdge (optional Leiden-style communities with LLM summaries), SagaNode with HasEpisodeEdge/NextEpisodeEdge (ordered episode sequences with rolling summaries, newer addition).

### Database
- Neo4j 5.26+ is the default and a hard dependency of the pip package (`neo4j>=5.26.0` in pyproject).
- FalkorDB 1.1.2+ via extra `graphiti-core[falkordb]`; embedded `falkordblite` via extra (Python 3.12+, redis<9 pin).
- Kuzu 0.11.2+ via extra, but DEPRECATED: "the upstream Kuzu project is no longer maintained ... will be removed in a future release".
- Amazon Neptune (Database or Analytics) plus an OpenSearch Serverless collection for full-text.
- Drivers live in graphiti_core/driver/ (neo4j_driver.py, falkordb_driver.py, kuzu_driver.py, neptune_driver.py).

### LLM and embedding calls during ingestion (`Graphiti.add_episode`, graphiti_core/graphiti.py lines 1043 onward; utils/maintenance/*)
Per episode, in order:
1. Fetch the last N previous episodes for context (DB only).
2. `extract_nodes`: one LLM call (`extract_nodes.extract_message` / `extract_text` / `extract_json` depending on source), optionally with custom entity types and exclusions.
3. `resolve_extracted_nodes`: embed all new entity names (`embedder.create_batch`), hybrid search for candidate duplicates, then an LLM `dedupe_nodes.nodes` call for the unresolved ones.
4. `_extract_and_resolve_edges`: one LLM call `extract_edges.edge`; then for each edge: `fact_embedding` via the embedder, candidate lookup, an LLM `dedupe_edges.resolve_edge` call (duplicate detection plus contradiction detection against existing edges), an LLM `extract_edges.extract_timestamps` call to fill valid_at/invalid_at when absent, and `extract_edges.extract_attributes` for custom edge types. A verbatim-match fast path skips the LLM when the same fact already exists.
5. `extract_attributes_from_nodes`: LLM `extract_nodes.extract_attributes` for typed entities plus batched summary generation (`extract_summaries_batch` / `extract_entity_summaries_from_episodes`).
6. Optional `update_communities` (LLM `summarize_nodes.summarize_pair` and `summary_description`) and optional saga summaries (`summarize_sagas.summarize_saga`).
There is also a combined single-call extraction path (`combined_extraction.py`, `extract_nodes_and_edges.extract_message`, `extract_timestamps_batch`). Rough count: a minimum of 3 to 5 LLM calls per episode, growing linearly with the number of extracted edges; plus 1 embedding batch for names and 1 embedding per fact.
- Retrieval (graphiti_core/search): hybrid of cosine similarity over embeddings, BM25 full-text (database index), and breadth-first graph expansion; rerankers RRF, node_distance, episode_mentions, MMR, cross_encoder.

### Provider support and fully-local feasibility
- LLM clients: OpenAI, Azure OpenAI, Anthropic, Gemini, Groq, `OpenAIGenericClient` for any OpenAI-compatible endpoint (README names Ollama, vLLM, llama.cpp, LM Studio, DeepSeek, Together, OpenRouter), and `GLiNER2Client` (local 205M to 340M parameter NER model on CPU, entity extraction only; it must wrap another LLM client for edges, dedupe and summaries).
- Embedders: OpenAI (and any OpenAI-compatible base_url, e.g. Ollama `nomic-embed-text`), Azure OpenAI, Gemini, Voyage.
- Cross-encoders: OpenAI reranker (logprob trick), Gemini reranker, `BGERerankerClient` (local sentence-transformers `BAAI/bge-reranker-v2-m3`).
- Fully local is possible in principle: Ollama or vLLM through `OpenAIGenericClient` + Ollama embeddings through `OpenAIEmbedder(base_url=...)` + local BGE reranker + FalkorDB in Docker (or embedded falkordblite on Python 3.12+, or a local Neo4j). The README warns that the pipeline "depends on structured (JSON) output ... Very small models frequently emit JSON that doesn't match the requested schema, which surfaces as extraction failures", offers a `structured_output_mode="json_object"` fallback, and recommends low `SEMAPHORE_LIMIT`. Telemetry goes to PostHog by default; opt out with `GRAPHITI_TELEMETRY_ENABLED=false` (posthog is a hard dependency). The `openai` package is also a hard dependency even when unused.
- Note for a code-graph use case: Graphiti stores LLM-extracted entities and natural-language facts; it is not a code parser and has no notion of files, symbols or call edges unless you define them as custom entity/edge types and let the LLM extract them.

---

## Claim 5. Augment Code: "100M-line repositories, sub-200ms search, >99.9% fidelity to exact vector search"

Verdict: VERIFIED as quoted; unaudited vendor engineering blog, no public benchmark or named repository.

- Post: "How we made code search 40% faster for 100M+ line codebases using quantized vector search", Luke Paulsen (Senior Software Engineer, Augment Code), published 11 June 2025 (page shows an update stamp of 18 June 2026). URL: https://www.augmentcode.com/blog/repo-scale-100M-line-codebase-quantized-vector-search
- Exact quotes:
  - "implementing an approximation of context search that shrinks the search space by orders of magnitude while maintaining more than 99.9% fidelity to the exact results. As a result, the latency of real-time features like code completions seamlessly improved by over 40% for our most challenging codebases."
  - "Here's the hard data: we reduced memory usage by 8x (from 2GB to 250MB for a 100M LOC codebase), cut search latency from 2+ seconds to under 200ms, and maintained 99.9% accuracy on typical queries. For the 0.1% of edge cases, usually involving very recent code changes or extremely rare patterns, we have automatic fallbacks."
  - "effective embedding models for code context can require 20+ bytes per LOC to store and 20+ nanoseconds per LOC to search ... a codebase with 100 million LOC would require loading 2 GB of embeddings into RAM and spending 2 seconds of CPU work for every user operation."
  - "quantization, reducing a large embedding vector to a much smaller bit vector that efficiently represents that vector's general 'neighborhood'. By first searching the quantized representation to generate an initial list of candidate embeddings and then searching those candidates using the full embedding similarity computation, we can speed up retrieval by a factor of tens to hundreds while maintaining better than 99.9% parity of results with the original approach in practice."
  - "we recently upgraded it to seamlessly support codebases up to 100 million LOC and beyond." Conclusion: "with under 200 milliseconds of additional latency."
- Caveats: the "2 GB / 2 s" baseline is arithmetic from the "20+ bytes and 20+ ns per LOC" rule of thumb, not a measured run on a named repo; the 99.9% figure has no stated query set, k, or metric definition; "40% faster" refers to end-to-end completion latency, while "2+ s to <200 ms" refers to the search step; no bit-width, candidate count or re-ranking depth is disclosed. The word "demonstrated" in the claim overstates what the post shows.

---

## Bonus. Local-first code-graph indexers usable from TypeScript / Bun

Two that fit best (tree-sitter, local, MCP plus library API), plus alternatives. GitHub numbers as returned by the API on 2026-09-15. Neither README mentions Bun; both target Node.

1. lzehrung/codegraph, https://github.com/lzehrung/codegraph
   - TypeScript. MIT. npm `@lzehrung/codegraph` (2.3.27) and a slim library `@lzehrung/codegraph-core`. Node >= 22.16. 5 stars, created 2025-10-16, pushed 2026-09-15 (active).
   - Pipeline: tree-sitter parse ("extract imports, exports, definitions, bindings, and scopes"), module resolution to project files or external nodes, forward and reverse dependency indexes plus a semantic symbol index. A Rust native runtime (`packages/codegraph-native`) does precise parsing; without it the tool "falls back to reduced graph-only and regex recovery".
   - Languages: JS, TS, Python, PHP, Go, Java, C#, Ruby, Rust, Kotlin, Swift, Zig, C, C++, plus SQL and template formats.
   - Library API: `buildProjectIndex`, `getDependencies`, `getReverseDependencies`, `goToDefinition`, `findReferences`, `analyzeImpactFromDiff`, `createAgentSession`; exports JSON, Mermaid, DOT, SQLite. MCP server included. Honest limits section: "not a compiler or type checker. Reflection, generated code, macros, overload behavior, and dynamic dispatch can be missed."

2. colbymchenry/codegraph, https://github.com/colbymchenry/codegraph
   - MIT. npm `@colbymchenry/codegraph`; Rust kernel with compiled tree-sitter grammars; SQLite store with FTS5 at `.codegraph/codegraph.db`; nodes = functions/classes/methods, edges = calls/imports/extends/implements, then a resolution pass (calls to definitions, imports to files, inheritance, framework routes); file-watcher auto-sync. 70,958 stars per the GitHub API (created 2026-01-18, pushed 2026-09-15). GitHub lists the primary language as C (vendored grammars).
   - MCP: a single default tool `codegraph_explore` (source + call paths + blast radius); `codegraph_callers`, `codegraph_callees`, `codegraph_impact` etc. are unlisted but available. Library: `CodeGraph.init/open`, `indexAll`, `searchNodes`, `getCallers`, `getImpactRadius`, `buildContext`; the embedded-library path requires Node 22.5+ for built-in `node:sqlite`. Publishes measured "fair coverage" per language (e.g. TS/JS 95.8%, Go 96.6%, Rust 86.7%). Telemetry on by default, opt-out `codegraph telemetry off`.

Others seen: sdsrss/code-graph-mcp (Rust, MIT, 75 stars, 10 languages); tirth8205/code-review-graph (Python, MIT, 31,459 stars, local-first MCP); CodeGraphContext/CodeGraphContext (Python, MIT, 4,191 stars, needs a graph database); denzharkov/codegraph-mcp (pure JS + WASM tree-sitter, MIT, 1 star, very new).

---

## What a lightweight local project map (file descriptions, regex symbols, import graph with PageRank) is missing compared with LocAgent's graph

1. Containment hierarchy with line ranges: directory to file to class to method, each node carrying start/end lines and full source, so a hit can be expanded to exactly the enclosing entity and a line number can be mapped back to its function (`get_module_name_by_line_num`).
2. Invocation edges: who calls what, scope-aware (imports, `__init__` re-exports, aliases). An import graph only says file A depends on file B; it cannot answer "who calls `allocate`" or "what does this method touch", which is the query LocAgent's ablation credits with most of the function-level gain (77.0 vs 66.1 without TraverseGraph on the 7B model).
3. Inheritance edges (class to base class), needed for method-override and subclass reasoning.
4. Entity-level import edges (file to a specific class/function, with alias), rather than file-to-file.
5. Bidirectional bounded traversal as a tool: upstream (dependents) and downstream (dependencies) with hop limits and node/edge type filters, returned in a compact tree or DOT form. PageRank gives a global centrality score; it does not give a local neighbourhood on demand.
6. A layered entity search: exact id, short-name dictionary (case-insensitive, `Class.method` splitting), BM25 over entity names and code, fuzzy fallback. Regex symbol lists have no ranking; LocAgent's ablation shows removing the search index costs 19 file-level points, far more than removing the graph.
7. Test-file exclusion and three retrieval detail levels (fold / preview / complete) so the agent controls token spend.
8. Agentic access rather than a static dump: the map is queried multi-hop by the model, not pasted into the prompt.

Also worth knowing before copying LocAgent: it is Python-only, uses the stdlib `ast` (no tree-sitter for the graph), resolves calls by bare name with repo-wide fuzzy fallback (over-approximate edges, no types), keeps the whole graph in one pickled NetworkX object, and was evaluated with a lenient Acc@5 file metric on SWE-bench-Lite.

---

## Source index
- arXiv 2606.29718 (context rot in long-horizon search): https://arxiv.org/abs/2606.29718 ; code https://github.com/GAIR-NLP/ContextRot
- Chroma Context Rot: https://research.trychroma.com/context-rot ; code https://github.com/chroma-core/context-rot
- Sourcegraph failure-patterns post: https://sourcegraph.com/blog/why-coding-agents-fail-large-codebases
- Sourcegraph 7.0 post: https://sourcegraph.com/blog/a-new-era-for-sourcegraph-the-intelligence-layer-for-ai-coding-agents-and-developers
- CodeScaleBench repo: https://github.com/sourcegraph/CodeScaleBench ; technical report https://raw.githubusercontent.com/sourcegraph/CodeScaleBench/public/docs/technical_reports/TECHNICAL_REPORT.md ; MCP comparison https://raw.githubusercontent.com/sourcegraph/CodeScaleBench/public/docs/technical_reports/MCP_COMPARISON_REPORT.md ; dataset https://huggingface.co/datasets/sgjarmak/CodeScaleBench ; Tessl coverage https://tessl.io/blog/coding-agent-failure-patterns-large-codebases/
- LocAgent: https://arxiv.org/abs/2503.09089 ; https://aclanthology.org/2025.acl-long.426/ ; https://github.com/gersteinlab/LocAgent
- Graphiti: https://github.com/getzep/graphiti ; https://arxiv.org/abs/2501.13956
- Augment Code: https://www.augmentcode.com/blog/repo-scale-100M-line-codebase-quantized-vector-search
- Indexers: https://github.com/lzehrung/codegraph ; https://github.com/colbymchenry/codegraph ; https://github.com/sdsrss/code-graph-mcp ; https://github.com/tirth8205/code-review-graph ; https://github.com/CodeGraphContext/CodeGraphContext ; https://github.com/denzharkov/codegraph-mcp
