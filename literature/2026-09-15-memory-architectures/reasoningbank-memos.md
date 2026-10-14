# ReasoningBank and MemOS: claim verification report

Date: 2026-09-15. Method: read the arXiv HTML of both papers (v1 and latest), cloned the official repos (`google-research/reasoning-bank`, `MemTensor/MemOS` at commit de80694, 2026-09-08) and read the code, pulled GitHub / PyPI / npm metadata through the APIs, and searched for independent reproductions. Nothing below is taken from marketing text unless labeled as such. Local copies: `scratchpad/rb_v1.txt`, `scratchpad/rb_v2.txt`, `scratchpad/memos_v1.txt`, `scratchpad/memos_v4.txt`, `scratchpad/reasoning-bank/`, `scratchpad/memos/`, `scratchpad/memos_releases.json`, `scratchpad/memos_commits_2026.tsv`.

---

## Part A. ReasoningBank

### A1. Paper identity (verified)

- Title: "ReasoningBank: Scaling Agent Self-Evolving with Reasoning Memory". arXiv 2509.25140. https://arxiv.org/abs/2509.25140
- Versions: v1 submitted Monday 29 September 2025; v2 revised 16 March 2026. arXiv comments field: "Accepted to ICLR 2026; Code available at: https://github.com/google-research/reasoning-bank". The repo BibTeX cites ICLR 2026 with OpenReview id `jL7fwchScm` (https://openreview.net/forum?id=jL7fwchScm). I could not fetch OpenReview directly (bot check, and the public notes API returned nothing), so the acceptance is verified only through arXiv metadata and the repo BibTeX.
- Authors (17): Siru Ouyang, Jun Yan, I-Hung Hsu, Yanfei Chen, Ke Jiang, Zifeng Wang, Rujun Han, Long T. Le, Samira Daruki, Xiangru Tang, Vishy Tirumalashetty, George Lee, Mahsan Rofouei, Hangfei Lin, Jiawei Han, Chen-Yu Lee, Tomas Pfister. Affiliations in the HTML: Google Cloud AI Research (11 affiliation tags) and University of Illinois Urbana-Champaign (1).
- Full text used: https://arxiv.org/html/2509.25140v2 (and v1 for diffing; the result tables are identical between v1 and v2, the v2 changes are text and the appendix).

### A2. What a memory item is (verified against paper section 3.2, appendix A.2, and code)

Paper, section 3.2 "Memory Schema": "Each memory item specifies three components: (i) a title, which serves as a concise identifier summarizing the core strategy or reasoning pattern; (ii) a description, which provides a brief one-sentence summary of the memory item; and (iii) the content, which records the distilled reasoning steps, decision rationales, or operational insights extracted from past experiences."

Appendix A.2 "ReasoningBank Storage": "We maintain ReasoningBank in a JSON format, and each entry of ReasoningBank consists of a task query, the original trajectory, and the corresponding memory items. All memory items are stored with the schema {title, description, content}. The embedding is pre-computed for each given query and stored in another JSON file."

Code confirms the on-disk shape (`WebArena/induce_memory.py`, `third_party/src/minisweagent/run/extra/swebench.py`): one JSONL line per finished task:

```
{"task_id": ..., "query": ..., "think_list": [...], "action_list": [...],
 "status": "success" | "fail", "memory_items": [<markdown chunks>], "template_id": ...}
```

Note that `memory_items` is literally the LLM's Markdown output split on blank lines (`response.split("\n\n")`); there is no structured parse of title/description/content in the released code. A second JSONL (`<site>_embeddings.jsonl`) holds `{"id": task_id, "text": query, "embedding": [...3072 floats]}`.

Important detail: retrieval is keyed on the task query embedding, not on the memory item text. The paper says "identify the top-k relevant experiences and their corresponding memory items"; the code (`select_memory`) ranks past task queries by cosine against the current query and returns all memory items of those experiences.

### A3. Extraction from successful AND failed trajectories (verified)

Pipeline (paper 3.2 and A.2; code `WebArena/induce_memory.py`, `WebArena/autoeval/`, `third_party/src/minisweagent/run/extra/swebench.py`):

1. Correctness signal: an LLM-as-a-judge with the same backbone as the agent, temperature 0.0, returns a binary Success/Failure "given the query and trajectory without any ground-truth reference". No gold labels are used at test time (the code also has a `--criteria gt` switch for the WebArena ground-truth reward, used for analysis).
2. Extraction: the same backbone LLM at temperature 1.0 is prompted with the query and trajectory plus one of two system instructions (success vs failure). "For each trajectory, at most 3 memory items could be extracted."
3. Consolidation: "newly generated items are directly added without additional pruning" (no merge, no dedupe, no forgetting).

Exact extraction prompts from the official code (`WebArena/prompts/memory_instruction.py`; the SWE-Bench variant in `third_party/src/minisweagent/memory/instruction.py` differs only in the domain sentence and the "Do not mention specific websites, queries, or string contents" line):

SUCCESSFUL_SI (system instruction; user message is `**Query:** ...\n\n**Trajectory:**\n<think>...</think>\n<action>...</action> ...`):

```
You are an expert in web navigation. You will be given a user query, the corresponding trajectory that represents **how an agent successfully accomplished the task**.

## Guidelines
You need to extract and summarize useful insights in the format of memory items based on the agent's successful trajectory.
The goal of summarized memory items is to be helpful and generalizable for future similar tasks.

## Important notes
  - You must first think why the trajectory is successful, and then summarize the insights.
  - You can extract *at most 3* memory items from the trajectory.
  - You must not repeat similar or overlapping items.
  - Prefer concrete, actionable procedures over abstract principles. Do not embed specific product names, queries, or literal string contents from the task.

## Output Format
Your output must strictly follow the Markdown format shown below:

# Memory Item i
## Title <the title of the memory item>
## Description <one sentence summary describing when or when NOT to use the memory item>
## Content <1-3 sentences describing the insights learned to successfully accomplishing similar tasks in the future>
```

FAILED_SI (same skeleton; the differences are):

```
... the corresponding trajectory that represents **how an agent attempted to resolve the task but failed**.
...
  - You must first reflect and think why the trajectory failed, and then summarize what lessons you have learned or strategies to prevent the failure in the future.
  - You can extract *at most 3* memory items from the trajectory.
  - You must not repeat similar or overlapping items.
  - Prefer concrete, actionable recovery procedures over abstract principles. Do not embed specific product names, queries, or literal string contents from the task.
...
## Content <1-3 sentences describing the insights learned to avoid such failures and successfully accomplishing similar tasks in the future>
```

Judge prompts:

- WebArena (`WebArena/autoeval/prompts.py`, `build_text_eval_prompt`, inherited from the AWM/autoeval line of work): system message defines three task types (information seeking, site navigation, content modification), then a "Strictness rules" block (Completeness, Grounding, Right target) ending with: "When uncertain on any of these, mark failure. A false success is more harmful than a false failure, because memory induction amplifies it into future behavior." Output format: `Thoughts: ...` then `Status: "success" or "failure"`. Input: user intent, action history, final page state (accessibility tree text), bot response.
- SWE-Bench (`run/extra/swebench.py`, `llm_judge_status`): a much weaker judge. System: "You are a helpful assistant that judges whether the agent successfully completed the task." User: `Task: {task}\n\nTrajectory:\n{all non-system messages}\n\nDid the agent successfully complete the task? Answer with 'success' or 'fail' only.` Temperature 0.0; success if the substring "success" appears in the reply. No tests are run for the judge; the actual resolve rate is computed later with the official SWE-Bench harness.

MaTTS prompts (`PARALLEL_SI`): "You will be given a user query and multiple trajectories showing how an agent attempted the task. Some trajectories may be successful, and others may have failed... Use **self-contrast reasoning**... You can extract *at most 5* memory items from all trajectories combined" (temperature 0.7 in `induce_scaling.py`). Sequential: `SEQUENTIAL_PROMPT` = "Important: Let's carefully re-examine the previous trajectory, including your reasoning steps and actions taken... If you find inconsistencies, correct them. If everything seems correct, confirm your final answer." followed by "Let's check again." on later rounds.

### A4. Retrieval and injection (verified)

Appendix A.2: "we embed each task query using gemini-embedding-001, accessed via Vertex AI. Similarity search is conducted over the memory pool using cosine distance. We select memory items of the top-k most similar experiences (default k=1; ablation study in section 5)." Code: `select_memory(n=1, ...)` in `WebArena/run.py` and `select_memory(1, ...)` in `swebench.py`; `output_dimensionality=3072`, task type `RETRIEVAL_DOCUMENT`; the query side is wrapped as `Instruct: <task description>\nQuery: <query>`. A Qwen3-Embedding-8B path exists in `memory_management.py` but the instruction vector is still computed with Gemini, so it is not a working local alternative as shipped.

Injection: retrieved items are appended to the system prompt, "each item represented by its title and content", preceded by: "Below are some memory items that I accumulated from past interaction from the environment that may be helpful to solve the task. You can use it when you feel it's relevant. In each step, please first explicitly discuss if you want to use each memory item or not, and then take action." (verbatim in `third_party/src/minisweagent/agents/default.py` line 77).

Ablation (appendix C.1, WebArena-Shopping, Gemini-2.5-flash): no memory 39.0; k=1 49.7; k=2 46.0; k=3 45.5; k=4 44.4. More retrieved experiences hurt.

### A5. Memory-aware test-time scaling, MaTTS (verified)

Section 3.3: scaling factor k = number of trajectories (parallel) or refinement steps (sequential). Parallel: k trajectories for the same query under the same retrieved memory; memory is extracted by self-contrast across all k (PARALLEL_SI); final answer for scoring is Best-of-N chosen by the same LLM (Figure 12). Sequential: self-refinement inside one trajectory; intermediate notes feed extraction. Results (section 4.3, WebArena-Shopping, Gemini-2.5-flash): parallel 49.7 (k=1) to 55.1 (k=5); sequential 49.7 to 54.5; without memory, parallel fluctuates between 39.0 and 42.2 and sequential between 37.4 and 40.6. Pass@5 62.1 with MaTTS vs 52.4 without memory (C.3). The "+MaTTS" rows in Table 1 are parallel scaling k=5, pass@1 (BoN).

### A6. Benchmarks and numbers (verified from the tables in the HTML)

Backbones: Gemini-2.5-Flash, Gemini-2.5-Pro, Claude-3.7-Sonnet, all via Vertex AI; ReAct-style agents in BrowserGym (WebArena, 30-step cap, temperature 0.7) and bash-only mini-SWE-Agent (SWE-Bench). Baselines: No Memory, Synapse (raw trajectories as memory), AWM (workflows), reimplemented by the authors with the same retrieval and consolidation so that only extraction differs.

Table 1, WebArena overall (684 tasks, Map excluded), success rate / average steps:

| Backbone | No Memory | Synapse | AWM | ReasoningBank | + MaTTS |
|---|---|---|---|---|---|
| Gemini-2.5-flash | 40.5 / 9.7 | 42.1 / 9.2 | 44.1 / 9.0 | 48.8 / 8.3 | 51.8 / 7.9 |
| Gemini-2.5-pro | 46.7 / 8.8 | 47.7 / 8.5 | 47.6 / 8.7 | 53.9 / 7.4 | 56.3 / 7.1 |
| Claude-3.7-sonnet | 41.7 / 8.0 | 42.6 / 7.9 | 40.8 / 8.9 | 46.3 / 7.3 | 48.8 / 7.2 |

Per-subset ReasoningBank rows (SR): flash Shopping 49.7, Admin 51.1, Gitlab 40.6, Reddit 67.0, Multi 13.8; pro 51.9 / 56.6 / 44.4 / 80.2 / 13.8; claude 44.9 / 53.3 / 41.1 / 57.5 / 3.4. The "up to 20% relative improvement" is 48.8 vs 40.5 (flash); "up to 16% fewer steps" is 7.4 vs 8.8 (pro).

Table 2, SWE-Bench-Verified (500 instances, mini-SWE-Agent, bash only), resolve rate / average steps:

| Backbone | No Memory | Synapse | ReasoningBank |
|---|---|---|---|
| Gemini-2.5-flash | 34.2 / 30.3 | 35.4 / 30.7 | 38.8 / 27.5 |
| Gemini-2.5-pro | 54.0 / 21.1 | 53.4 / 21.0 | 57.4 / 19.8 |

So the SWE gain is +4.6 points (23 of 500 instances) for flash and +3.4 points (17 instances) for pro. No Claude backbone, no AWM ("action space in mini-SWE-Agent is open-ended"), no MaTTS on SWE, single run per cell, no variance or confidence intervals anywhere in the main tables. The numbers are unchanged between v1 and v2.

Table 3, Mind2Web (1341 tasks: cross-task 252, cross-website 177, cross-domain 912), EA / AF1 / SSR / SR:

| Backbone, method | Cross-Task | Cross-Website | Cross-Domain |
|---|---|---|---|
| flash, No Memory | 46.0 / 59.1 / 40.3 / 3.3 | 39.8 / 45.1 / 31.7 / 1.7 | 35.8 / 37.9 / 31.9 / 1.0 |
| flash, ReasoningBank | 52.1 / 60.4 / 44.9 / 4.8 | 44.3 / 52.6 / 33.9 / 2.3 | 40.6 / 41.3 / 36.6 / 1.6 |
| pro, No Memory | 49.3 / 60.2 / 44.4 / 3.5 | 41.2 / 49.8 / 34.8 / 3.4 | 37.9 / 37.7 / 35.0 / 1.4 |
| pro, ReasoningBank | 53.6 / 62.7 / 45.6 / 5.1 | 46.1 / 54.8 / 36.9 / 3.8 | 42.8 / 45.2 / 38.1 / 1.7 |

Task-level SR on Mind2Web is 1 to 5 percent for every method; the headline gains are on step-level metrics.

Extra results: Gemma-3-12B-Instruct on WebArena-Shopping (Table 6): No memory 17.1, Synapse 16.0, AWM 21.4, ReasoningBank 24.1 (steps 13.7 / 14.0 / 12.5 / 11.8). Token cost (Table 5): No Memory 50,847 tokens per task, ReasoningBank 53,055 (+4.3%), Synapse 58,515, AWM 59,373.

Verdict on the claim "The paper specifically evaluates it on software-engineering tasks and finds that accumulated reasoning memories improve agents": verified, with the caveat that SWE is the smallest experiment (one table, two Gemini backbones, two baselines, no MaTTS) and the gains are 3 to 5 points from single runs.

### A7. Official code and reimplementations

Official: https://github.com/google-research/reasoning-bank. Apache-2.0. Python. Created 2026-02-26 (five months after v1). 585 stars, 74 forks, 6 commits total; last commit 2026-05-19 ("Update BibTeX"), last push 2026-08-05. README disclaimer: "not an officially supported Google product... intended for demonstration purposes only". Contents: WebArena pipeline (browsergym 0.14.1 pinned, vendored patched `third_party/webarena` with corrected shopping annotations and evaluator fixes, autoeval judge, `induce_memory.py`, `pipeline_memory.py`, `pipeline_scaling.py`, `induce_scaling.py`), and SWE-Bench via a vendored mini-swe-agent fork (`third_party/src/minisweagent/run/extra/swebench.py` and `agents/default.py` carry the memory hooks). Mind2Web code is not released. Only Vertex AI (Gemini, Claude) clients are wired for the memory path.

Code-quality observations for a skeptic: README says `pip install -r requirements.txt` but there is no `requirements.txt` (only `pyproject.toml` with `requires-python >= 3.13` and `uv.lock`); root `main.py` is a hello-world stub; `WebArena/induce_scaling.py` has inverted reward logic (`if reward == 0: status = "success"`) which is harmless only because the parallel prompt never sees the status; `minisweagent/memory/induce_memory.py` is a vestigial copy of the WebArena inducer (it reads browsergym logs) while the real SWE loop lives in `run/extra/swebench.py`.

Third-party reimplementations (GitHub search 2026-09-15, none endorsed by the authors, internals not audited): Lanerra/reasoning-bank-slm (108 stars, no license, last push 2025-10-14, small models) https://github.com/Lanerra/reasoning-bank-slm ; thakshak/ReasoningBank (64, MIT, 2026-02-09) https://github.com/thakshak/ReasoningBank ; budprat/ReasoningBank (17, MIT, 2025-10-19) https://github.com/budprat/ReasoningBank ; hanw39/ReasoningBank-MCP (10, MIT, 2025-10-31) https://github.com/hanw39/ReasoningBank-MCP ; iamtaehyunpark/reasoning-bank (Apache-2.0, 2026-08-04, vLLM local models) https://github.com/iamtaehyunpark/reasoning-bank ; mjm114514/claude-reasoning-bank (no license, 2026-06-08, Claude Code, BM25, per repo) https://github.com/mjm114514/claude-reasoning-bank ; ramankrishna/reasoning-bank (Apache-2.0, 2026-05-31) https://github.com/ramankrishna/reasoning-bank .

---

## Part B. MemOS (MemTensor)

### B1. Identity and metadata (verified)

- Paper: "MemOS: A Memory OS for AI System", arXiv 2507.03724, v1 4 July 2025, v4 3 December 2025, 39 authors, MemTensor (Shanghai) with university coauthors. https://arxiv.org/abs/2507.03724 . A shorter precursor is arXiv 2505.22101.
- Repo: https://github.com/MemTensor/MemOS . License Apache-2.0 (root `LICENSE`); the TypeScript local plugin declares MIT in its own `package.json`. 11,336 stars, 1,037 forks, 74 open issues. Created 2025-07-06. Last push 2026-09-09; last commit on main 2026-09-08 `de80694` "fix: restore OpenClaw installation and automatic memory hooks (#2350)". At least 1,200 commits since 2026-01-01 (API pagination cap reached).
- Language: GitHub reports TypeScript first by bytes (9.5 MB TS vs 5.5 MB Python) because `apps/` vendors large TypeScript projects, including a full copy of the Openwork Electron app. The core library is Python: `src/memos`, PyPI package `MemoryOS` 2.0.33 (uploaded 2026-09-03, 39 releases), `requires-python >= 3.10`. The local plugin is TypeScript: npm `@memtensor/memos-local-plugin` 2.0.19 (2026-09-08; first publish 2026-04-27).
- Repo homepage and docs: https://memos.openmem.net , https://memos-docs.openmem.net . Releases: https://github.com/MemTensor/MemOS/releases (57 GitHub releases, 39 core `vX.Y.Z` plus local-plugin tags).

### B2. What the Python core actually implements today (read from `src/memos`)

Memory types:

- Plaintext (`memories/textual/`): `NaiveTextMemory` (JSON file, search = word-overlap count, no vectors), `GeneralTextMemory` (LLM extraction + Qdrant vector search), `TreeTextMemory` and `SimpleTreeTextMemory` (graph database backed, scopes `WorkingMemory` / `LongTermMemory` / `UserMemory` with default caps 20 / 1500 / 480), `PreferenceTextMemory` (Milvus), and skill memory (`mem_reader/read_skill_memory/`: LLM extracts SKILL.md style packages from conversations, stored to Alibaba OSS or a local dir; first commit 2026-01-28, PR #964).
- Activation (`memories/activation/`): `KVCacheMemory` builds a HuggingFace `DynamicCache` from text with an in-process HF model, pickles it, moves it to GPU on use; `VLLMKVCacheMemory` only stores prompt strings so vLLM prefix caching can preload them. Both exist since the initial commit (2025-07-06). Off by default (`MOS_SCHEDULER_ENABLE_ACTIVATION_MEMORY=false`).
- Parametric (`memories/parametric/lora.py`): explicit placeholder. The file header says "This file currently serves as a placeholder... Please do not use this as a functional module yet" and `dump()` writes the bytes `b"Placeholder"`. Parametric memory is not implemented.

MemCube (`mem_cube/general.py`): `GeneralMemCube` is a container holding one textual, one activation and one parametric memory backend (plus preference memory), with `load`/`dump` to a directory or a remote HF repo, and per-user cube registration and sharing through the API. It is a configuration bundle, not the paper's MemCube with a metadata header of lifecycle, permission and storage policy. The paper's MemLifecycle / expiration / MemOperator are not present as modules.

Textual memory item schema (`memories/textual/item.py`): `TextualMemoryItem{id, memory, metadata}` with metadata fields `user_id, session_id, status (activated|resolving|archived|deleted), is_fast, evolve_to, version, history[ArchivedTextualMemory{update_type: conflict|duplicate|extract|unrelated|feedback, ...}], working_binding, type, key, confidence, source, sources[SourceMessage provenance], ...`.

Retrieval (`memories/textual/tree_text_memory/retrieve/`): `Searcher` and `GraphMemoryRetriever.retrieve` do "hybrid memory retrieval" over the graph store: graph neighborhood recall, vector recall (`search_by_embedding` in the graph DB), BM25 recall (`EnhancedBM25`, rank_bm25 plus TF-IDF rerank, first commit 2025-10-29 PR #414), fulltext recall (`search_by_fulltext` on PolarDB / Neo4j / Postgres indexes; inserted 2025-12 PR #567, keyword search 2026-02 PR #1073, toggle `search_strategy.fulltext`), then reranking (`http_bge`, `http_bge_strategy`, `cosine_local`, `noop`), working-memory path, optional internet retrievers (Bocha, Tavily, Xinyu), and search modes fast / fine / mixture. Vector-only `GeneralTextMemory` uses Qdrant top-k only.

Scheduling (`mem_scheduler/`): `GeneralScheduler` / `OptimizedScheduler` consume add/search events (in-process, or Redis streams / RabbitMQ), reorder working memory, and can promote hot plaintext memories into KV cache via `activation_memory_manager.py`. Disabled by default (`MOS_ENABLE_SCHEDULER=false`).

Consolidation: `GraphStructureReorganizer` clusters nodes and writes LLM summary parent nodes (`MOS_ENABLE_REORGANIZE=false` by default); feedback-driven merges and version history; the "Dream" plugin (`src/memos/dream`, first commit 2026-05-08, beta, disabled unless `MEMOS_ENABLED_PLUGINS=dream`) reflects on recently added memories off the request path and writes Context nodes, insights and a diary.

Forgetting: no decay, TTL or eviction routine exists in the core. `grep` for forget / decay / expire / evict finds only MMR candidate pruning in the search handler and the Dream archive writer. Deletion is soft (status archived / deleted) plus `TreeTextMemory.drop(keep_last_n)` backup rotation. The paper's lifecycle model is aspirational relative to the code.

Feedback and correction (`mem_feedback/feedback.py`, class `MemFeedback`): LLM judges a natural-language feedback message against retrieved memories, then adds / updates (with versions and archived history) / deletes, with keyword-replace shortcuts and a strict info filter. First commit 2025-12-02 (PR #541), shipped in v2.0.0 on 2025-12-24; refined in 2026 (thresholds PR #1723, "feedback experience retrieval" PR #1634, May 2026).

Backends: LLMs `openai` (any OpenAI-compatible `api_base`), `openai_new`, `deepseek`, `qwen`, `minimax`, `ollama`, `hf`, `vllm`; embedders `universal_api` (OpenAI-compatible `/v1/embeddings` or Azure), `ollama`, `sentence_transformer`, `ark`; vector DBs `qdrant`, `milvus`; graph DBs `neo4j`, `neo4j_community`, `polardb`, `postgres` (pgvector); user manager `sqlite` or `mysql`. Full option list: https://github.com/MemTensor/MemOS/blob/main/docker/.env.example-full .

Interfaces: FastAPI service (`memos.api.server_api:app`, routes such as `/product/create_cube`, `/product/add`, `/product/search`), an MCP server (`src/memos/api/mcp_serve.py`, FastMCP, transports stdio / http / sse, tools: chat, create_user, create_cube, register_cube, unregister_cube, search_memories, add_memory, get_memory, update_memory, delete_memory, delete_all_memories, clear_chat_history, dump_cube, share_cube, get_user_info, control_memory_scheduler; first commit 2025-07-20, PR #127), a CLI, and a Python SDK (`MOS`).

Coding-harness integrations: none for Claude Code, Codex or Cursor. The `.claude/agents`, `.codex/agents`, `CLAUDE.md` and `AGENTS.md` at the repo root are contributor tooling for developing MemOS itself, not product integrations. What ships: OpenClaw plugins (cloud: `apps/MemOS-Cloud-OpenClaw-Plugin`; local v1: `apps/memos-local-openclaw`, first commit 2026-03-07; local v2: `apps/memos-local-plugin`, first commit 2026-04-09), Hermes Agent (Python provider talking JSON-RPC to the Node core), DeepSeek Harness `dsh` (Cordis bundle, added 2026-08-14), the generic MCP server, and `apps/openwork-memos-integration`, a vendored fork of the Openwork Electron desktop agent (which drives the OpenCode CLI) with a small `services/memory.ts` shim; only 5 files there mention MemOS, so it is an experiment, not an OpenCode integration. Docs also describe MCP use from CozeSpace.

### B3. The four "added during 2026" claims

| Claim | Verdict | Evidence |
|---|---|---|
| Local memory | Verified for 2026, but it is a separate TypeScript plugin, not the Python core | OpenClaw local plugin v1 PR #1182 in release v2.0.8 (2026-03-09) https://github.com/MemTensor/MemOS/releases/tag/v2.0.8 ; v2 "Reflect2Evolve" rewrite PR #1511 in v2.0.14 (2026-04-23) https://github.com/MemTensor/MemOS/releases/tag/v2.0.14 ; npm first publish 2026-04-27 https://www.npmjs.com/package/@memtensor/memos-local-plugin . Storage: SQLite via better-sqlite3, vectors as float32 BLOBs with brute-force cosine in JS (`core/storage/vector.ts`), FTS5 trigram tables (`core/storage/keyword.ts`), local embeddings `Xenova/all-MiniLM-L6-v2` (384-dim) via `@huggingface/transformers`. |
| Hybrid lexical / vector retrieval | Partly: it predates 2026 in the core; 2026 extended it and shipped it in the local plugin | Core BM25 recall first commit 2025-10-29 (PR #414); fulltext path PR #567 in v2.0.0 (2025-12-24) https://github.com/MemTensor/MemOS/releases/tag/v2.0.0 ; keyword search PR #1073 in v2.0.6 (2026-02-12) https://github.com/MemTensor/MemOS/releases/tag/v2.0.6 ; fulltext toggle PR #1096 (v2.0.7), `search_by_fulltext` optimizations (v2.0.8), CJK tokenization PR #2040 (v2.0.23, 2026-07-09). Local plugin: FTS5 + vector + RRF + MMR (v1, March 2026), three-tier retrieval with RRF fusion (`core/retrieval/ranker.ts`, v2). |
| Experience / skill evolution | Verified for 2026 | Core skill memory PRs #964 to #990 in v2.0.4 (2026-01-30) https://github.com/MemTensor/MemOS/releases/tag/v2.0.4 ; upload skill PR #1560 and "feedback experience retrieval" PR #1634 in v2.0.15 (2026-05-11) https://github.com/MemTensor/MemOS/releases/tag/v2.0.15 ; SkillMemory extraction fix PR #2342 in v2.0.33 (2026-09-03). Local plugin: L1 traces, L2 policies, L3 world models, skill crystallization with Beta posterior lifecycle (`core/skill/`, `core/memory/l2`, `core/memory/l3`), idle skill archival PR #2209 (v2.0.32, 2026-08-28). |
| Feedback and correction | Partly: introduced 2025-12, extended in 2026 | `mem_feedback` first commit 2025-12-02 (PR #541), released v2.0.0 on 2025-12-24 https://github.com/MemTensor/MemOS/releases/tag/v2.0.0 ; 2026 follow-ups: PR #771 / #833 (v2.0.1), feedback threshold PR #1723 (v2.0.17, 2026-05-26), tests PR #1715. Local plugin `core/feedback` (classifier, revisor, decision repair) since April 2026. |

### B4. Benchmark claims

Paper (v4, section 6.1, Table 3, LoCoMo, all systems on GPT-4o-mini, LLM-judge score): MemOS-1031 overall 75.80 (single-hop 81.09, multi-hop 67.49, temporal 75.18, open-domain 55.90, F1 45.27, 1589 context tokens) vs Memobase 72.01, Mem0 64.57, MIRIX 64.33, Zep 59.22, MemU 56.55, Supermemory 55.34. LongMemEval (Table 4): MemOS-1031 77.8 overall vs Memobase 72.4, Mem0 66.4, Zep 63.8, Supermemory 58.4, MIRIX 43.49, MemU 38.4. Also PreFEval, PersonaMem, an API latency table (Table 7) and KV-cache TTFT results (Table 8: "Qwen2.5-72B achieves a 91.4% reduction in TTFT under long-context, short-query conditions", HF transformers, identical outputs). The v1 paper reported a different model version (MemOS-0630) against LangMem, Zep, OpenAI-Memory and Mem0.

README (September 2026): "LoCoMo 88.83, LongMemEval 89.20, PersonaMem v2 40.58, HaluMem 80.91, BEAM-10M 56.75, GDPVal 62.07, LiveCodeBench 64.96, OmniMath 61.00, SWE-Bench 38.46, BrowseComp-Plus 23.85", "OpenClaw improves average task completion from 36.63% to 50.87%", plus "72% lower token usage" (cloud plugin) and "35.24% token savings" (repo description). These numbers are not in the paper, are 13 points above the paper's own LoCoMo score, and come from MemTensor's own harness OmniMemEval (https://github.com/MemTensor/OmniMemEval , 60 stars, created 2026-06-24) with no stated backbone, judge or prompt in the README.

Are the numbers self-reported only? Yes. The baselines were run by MemTensor themselves ("we give unofficial implementations for the following memory frameworks: zep, mem0, memobase, supermemory, memu", docs/en/open_source/evaluation/overview.md; vendor API keys in `.env.example-full`; "the configuration for each method is selected based on its best validation performance"). I found no independent reproduction of 88.83 or 75.80. The LoCoMo benchmark itself is contested: Mem0 re-evaluated Zep's 84% claim to 58.44% (https://github.com/getzep/zep-papers/issues/5), and an audit reports the standard LLM judge accepts a large share of deliberately wrong answers and that the answer key is partly corrupted (https://dev.to/gde03/the-ai-memory-benchmark-everyone-quotes-forbids-saying-i-dont-know-o1n). Treat all LoCoMo leaderboard numbers, including MemOS's, as vendor-run.

### B5. Running fully local, and use beside a TypeScript / Bun harness with local Qwen

Python core, local stack: LLM through `MOS_CHAT_MODEL_PROVIDER=openai` with `OPENAI_API_BASE=http://localhost:<port>/v1` (vLLM, llama.cpp server, LM Studio, Ollama's OpenAI endpoint) and the same for `MEMRADER_*`; embeddings through `MOS_EMBEDDER_BACKEND=universal_api` with `MOS_EMBEDDER_API_BASE` pointing at a local `/v1/embeddings` server (for example Qwen3-Embedding or bge-m3 under vLLM or llama.cpp) and `EMBEDDING_DIMENSION` matched, or `ollama` / `sentence_transformer`; reranker `cosine_local` to avoid the BGE HTTP reranker. Storage still needs services: `MOS_TEXT_MEM_TYPE=general_text` requires Qdrant only; `tree_text` requires Neo4j community (or Postgres + pgvector, or PolarDB) and the docker compose brings Neo4j + Qdrant. There is no embedded, zero-service mode in the Python core (NaiveTextMemory is a JSON file with word-overlap search). Integration from a TS/Bun harness would be plain `fetch` against the REST API, or spawning `python -m memos.api.mcp_serve --transport stdio` as an MCP server. Risks: every add runs several extraction and judge LLM calls (MemReader, preference extractor, feedback model, scheduler) with prompts written for GPT-4o-mini and Qwen cloud models; JSON-mode reliability with a small local Qwen is the main practical risk; large dependency surface (transformers, torch optional, fastapi, sqlalchemy).

TypeScript local plugin core (`apps/memos-local-plugin`, MIT, Node >= 20): the ARCHITECTURE doc explicitly invites new adapters. Two integration paths: (1) in-process, import `createMemoryCore` from `core/` and map the harness's turn start / turn end / tool outcome / feedback events onto the `MemoryCore` interface (`agent-contract/memory-core.ts`); (2) out-of-process, spawn `bridge.cts` and speak line-delimited JSON-RPC 2.0 over stdio (`session.open`, `turn.start`, `turn.end`, `feedback.submit`, `memory.search`, `skill.list`, `events.subscribe`). Config keys for a local Qwen: `llm.provider: openai_compatible`, `llm.endpoint: http://localhost:<port>/v1`, `llm.model: <qwen id>` (apiKey relaxed for local endpoints since PR #1921), `embedding.provider: openai_compatible` with `embedding.endpoint` and `dimensions`, or `embedding.provider: local` (MiniLM, 384-dim, ONNX download about 23 MB). Caveats: better-sqlite3 is a native N-API addon (works on Node 20 to 25 per their fixes; Bun compatibility was not tested here and `bun:sqlite` is not a drop-in); the design is chat-agent shaped (turns, episodes, "reward" from a rubric LLM, L2 policy induction, L3 world models, skill crystallization), so each turn can trigger several background LLM calls; telemetry is on by default (`telemetry.enabled: true`, opt-out); the docs and some prompts are partly Chinese; `package.json` in the repo (2.0.16-beta.1) and the hand-maintained CHANGELOG lag the npm release (2.0.19) and release notes are bot-generated; the project moves fast (57 GitHub releases in 14 months) with many "Fix #NNNN" commits on the local plugin in June to August 2026 (bridge races, native module ABI mismatches, CPU 100% from synchronous vector scans, ONNX memory leaks).

Bottom line for a TS/Bun harness: MemOS gives you a large chat-memory product, not a small library. If the goal is a "lessons from failures" feature, ReasoningBank's schema and prompts are a few hundred lines to reimplement (JSONL store, one embedding call per task, cosine top-1, two extraction prompts, one judge prompt); the MemOS local plugin's `core/` is the closest ready-made TypeScript code (SQLite + FTS5 + brute-force cosine + RRF/MMR ranker) if you want to borrow retrieval plumbing.

---

## Sources

- ReasoningBank arXiv: https://arxiv.org/abs/2509.25140 , HTML v2: https://arxiv.org/html/2509.25140v2 , HTML v1: https://arxiv.org/html/2509.25140v1
- ReasoningBank code: https://github.com/google-research/reasoning-bank (files cited: `README.md`, `WebArena/prompts/memory_instruction.py`, `WebArena/autoeval/prompts.py`, `WebArena/induce_memory.py`, `WebArena/induce_scaling.py`, `WebArena/run.py`, `WebArena/memory_management.py`, `third_party/src/minisweagent/run/extra/swebench.py`, `third_party/src/minisweagent/agents/default.py`, `third_party/src/minisweagent/memory/instruction.py`, `third_party/src/minisweagent/memory/memory_management.py`)
- MemOS arXiv: https://arxiv.org/abs/2507.03724 , HTML v4: https://arxiv.org/html/2507.03724v4 , HTML v1: https://arxiv.org/html/2507.03724v1
- MemOS repo: https://github.com/MemTensor/MemOS ; releases: https://github.com/MemTensor/MemOS/releases ; local plugin: https://github.com/MemTensor/MemOS/tree/main/apps/memos-local-plugin ; env reference: https://github.com/MemTensor/MemOS/blob/main/docker/.env.example-full ; evaluation docs: https://github.com/MemTensor/MemOS/blob/main/docs/en/open_source/evaluation/overview.md ; plugin comparison: https://github.com/MemTensor/MemOS/blob/main/docs/en/openclaw/plugin_compare.md
- PyPI: https://pypi.org/project/MemoryOS/ ; npm: https://www.npmjs.com/package/@memtensor/memos-local-plugin ; OmniMemEval: https://github.com/MemTensor/OmniMemEval
- LoCoMo credibility: https://github.com/getzep/zep-papers/issues/5 , https://dev.to/gde03/the-ai-memory-benchmark-everyone-quotes-forbids-saying-i-dont-know-o1n , https://github.com/memodb-io/memobase/blob/main/docs/experiments/locomo-benchmark/README.md
