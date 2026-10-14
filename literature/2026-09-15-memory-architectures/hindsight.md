# Hindsight (Vectorize) claim verification

Date of check: 2026-09-15. Method: `gh` API queries, shallow clone of the repository
(`git clone --depth 1`, HEAD d6c556f65dc028e75211bfb7ad55fb2a4bc58ef5), reading of the README,
the repo's own CLAUDE.md architecture inventory, the Docusaurus docs sources, the core Python
engine, the coding-agents TypeScript package, the arXiv PDF (text extracted with pypdf), and the
GitHub issue tracker. Nothing below is taken from marketing copy unless labelled as such.

Local clone: `/private/tmp/claude-501/-Users-zero-Documents-Dev/0012f4b7-2c66-4407-b505-d5ac12b403a3/scratchpad/hindsight`
Paper PDF and extracted text: `/private/tmp/claude-501/-Users-zero-Documents-Dev/0012f4b7-2c66-4407-b505-d5ac12b403a3/scratchpad/paper/`

## 0. Repository facts

| Item | Value | Source |
|---|---|---|
| Repository | https://github.com/vectorize-io/hindsight | `gh repo view` |
| Owner / copyright | Vectorize AI, Inc. | `LICENSE` |
| License | MIT (full text in `LICENSE`, "Copyright (c) 2025 Vectorize AI, Inc.") | https://github.com/vectorize-io/hindsight/blob/main/LICENSE |
| Stars / forks | 23,668 stars, 1,761 forks | `gh repo view --json stargazerCount,forkCount` |
| Created | 2025-10-30 | `gh repo view` |
| Last push | 2026-09-15T13:59:31Z | `gh repo view --json pushedAt` |
| Last commit on main | d6c556f65dc0, 2026-09-15 15:41 +0200, Nicolò Boschi, "fix(recall): search pg_search queries as pruned word terms, not the raw sentence (#4313) (#4403)" | `git log -1` |
| Open PRs | 25+ open; newest #4411 opened 2026-09-15T13:54Z; others on 09-14, 09-13, 09-11, 09-10, 09-03 | https://github.com/vectorize-io/hindsight/pulls |
| Merged today | #4403, #4402, #4401, #4400, #4399, #4397, #4395, #4394, #4393, #4392 all merged 2026-09-15 | `gh pr list --state merged` |
| Primary language | Python (engine); TypeScript (control plane, clients, coding-agents); Rust (CLI) | repo tree |
| Current version | `hindsight-api-slim` 0.10.0; `@vectorize-io/hindsight-coding-agents` 0.6.1 | `pyproject.toml`, `package.json` |
| Homepage / docs | https://hindsight.vectorize.io/ | README |
| Paper | arXiv 2512.12818, "Hindsight is 20/20: Building Agent Memory that Retains, Recalls, and Reflects" | https://arxiv.org/abs/2512.12818 |
| Demo paper | "Structured Agent Memory that Retains, Recalls, and Reflects", ACL 2026 demo track | https://aclanthology.org/2026.acl-demo.27.pdf |

The claim "open PRs as recently as September 12, 2026" is verified and in fact understated: PRs were
opened and merged on September 15, 2026.

Paper authors (from the arXiv HTML): Chris Latimer, Nicoló Boschi, Chris Bartholomew (Vectorize.io);
Andrew Neeser (The Washington Post); Gaurav Srivastava, Xuan Wang, Naren Ramakrishnan (Virginia Tech).

## 1. Claim: memory is separated into facts, experiences, synthesized entity knowledge and evolving beliefs, with retain / recall / reflect

Verdict: verified, with one correction about what the current code stores.

### 1a. Memory types in the paper vs the code

The paper (Section 3, extracted text lines 186 to 196) defines four networks: world (W) "objective
facts about the external world", experience (B) "biographical information about the agent itself,
written in the first person", opinion (O) "subjective judgments formed by the agent" with confidence
scores, and observation (S) "preference-neutral summaries of entities synthesized from multiple
underlying facts".

The current code stores three `fact_type` values on `memory_units`:

- `hindsight-api-slim/hindsight_api/engine/retain/types.py:331` has `fact_type: str  # "world", "experience", "observation"`.
- `hindsight-api-slim/hindsight_api/engine/response_models.py:16` has `VALID_RECALL_FACT_TYPES = frozenset(["world", "experience", "observation"])`.
- `hindsight-api-slim/hindsight_api/models.py:132` defaults `fact_type` to `"world"`.
- The extraction schema (`retain/fact_extraction.py:272`) asks the LLM for `"world"` or `"assistant"`; line 2062 maps `"assistant"` to `"experience"`. The prompt text at line 1075 defines "assistant" as actions the agent actually performed ("I changed X", "I discovered Y", "I debugged Z").

The opinion network from the paper was removed from the schema. Migration
`hindsight-api-slim/hindsight_api/alembic/versions/g2h3i4j5k6l7_remove_opinion_fact_type.py`
(Create Date 2026-04-02) says: "Remove the deprecated 'opinion' fact type: drop opinion-specific
indexes, update CHECK constraints, delete any remaining opinion rows, and drop the confidence_score
column". The original schema (`5a366d414dce_initial_schema.py:294`) allowed
`fact_type IN ('world', 'bank', 'opinion', 'observation')`. So "evolving beliefs" in today's code means
observations (consolidated, evidence-backed) plus mental models / knowledge pages, not a separate opinion
store with confidence scores.

The README's list matches the code: World facts, Experiences, Observations, Mental models
(https://github.com/vectorize-io/hindsight#memory-types).

### 1b. What the three operations do (from source)

Retain (`hindsight-api-slim/hindsight_api/engine/retain/orchestrator.py`, 4,423 lines;
`retain/fact_extraction.py`, 3,492 lines): the LLM extracts facts, entities and temporal data as
structured JSON (Pydantic schemas built with `create_model`), entities are resolved
(`engine/entity_resolver.py`), facts are embedded and linked (`retain/link_utils.py`,
`retain/link_creation.py`), and an opt-in Memory Defense policy redacts or blocks secrets and PII
(`extensions/memory_defense`). After retain, consolidation runs as a background job
(`engine/consolidation/consolidator.py`): it "creates new observations from novel facts" or "updates
existing observations when new evidence supports/contradicts/refines them"; observations carry
`proof_count`, `source_memory_ids` and a `history` JSONB. A near-duplicate reconciliation step is on by
default at cosine 0.97 (`docs/developer/observations.mdx`).

Recall (`hindsight-api-slim/hindsight_api/engine/search/retrieval.py`): module docstring says
"4-way parallel search: 1. Semantic retrieval (vector similarity) 2. BM25 retrieval 3. Graph retrieval
(via pluggable GraphRetriever interface) 4. Temporal retrieval". Results are fused with reciprocal
rank fusion (`search/fusion.py`) and reranked by a cross-encoder (`search/reranking.py`). Temporal
extraction uses `dateparser` (`engine/query_analyzer.py`), not an LLM. I found no LLM call on the
recall path; the only LLM use inside `search/` is `think_utils.py`, which builds the disposition
prompt for reflect.

Reflect (`hindsight-api-slim/hindsight_api/engine/reflect/agent.py`): "agentic loop for reflection
with native tool calling. Uses hierarchical retrieval: 1. search_mental_models 2. search_observations
3. recall". Disposition traits (skepticism, literalism, empathy, 1 to 5) shape the answer. This means the
LLM used for reflect must support function calling.

Mental models and knowledge pages: stored in a separate `mental_models` table, refreshed by running
reflect over a bounded snapshot in "full" or "delta" mode (`engine/mental_model_refresh.py`). Delta
refresh represents the document as sections and blocks with stable ids
(`engine/reflect/structured_doc.py`) and the LLM emits edit operations against those ids
(`engine/reflect/delta_ops.py`): "zero ops equals identical doc", unchanged blocks are copied through
byte for byte. Knowledge pages are mental models with defaults (built from observations, refreshed after
consolidation, never read other pages) and can be projected to disk as markdown with `hindsight fs mount`
(`docs/developer/knowledge-pages.mdx`).

### 1c. Storage and models

- Database: PostgreSQL with pgvector (HNSW) for vectors, tsvector with GIN for BM25, JSONB, recursive CTEs for graph queries (`docs/developer/storage.md`). Optional pg_search (ParadeDB) and pgroonga text backends (`hindsight_api/_pg_search.py`, `_text_search.py`). Oracle AI Database 23ai is a second backend with feature parity (`engine/db/oracle.py`, `engine/sql/oracle.py`). There is no SQLite backend. Schema is managed by Alembic and migrations run on startup.
- Embedded Postgres: when `HINDSIGHT_API_DATABASE_URL` is unset the server starts pg0 (single binary Postgres with pgvector) on port 5555 and stores data under `~/.hindsight/pg0/` (`hindsight_api/pg0.py`, `pg0-embedded` extra).
- Embeddings: default local SentenceTransformers `BAAI/bge-small-en-v1.5` (`engine/embeddings.py:297`), downloaded from HuggingFace on first run. Alternatives: ONNX in-process, TEI, OpenAI-compatible, Cohere, Gemini, LiteLLM, ZeroEntropy, OpenRouter (`embeddings.py` provider switch at lines 2122 to 2292).
- Reranker: default local `cross-encoder/ms-marco-MiniLM-L-6-v2` (`engine/cross_encoder.py:139`). Alternatives: TEI, Cohere, flashrank, jina-reranker-v3 via MLX on Apple Silicon, LiteLLM, Google, Alibaba, RRF passthrough (no reranker).
- LLM providers (`engine/providers/`): openai (any OpenAI-compatible base URL), openai-responses, ollama, lmstudio, llamacpp (built-in llama.cpp with an auto-downloaded Gemma 4 E2B GGUF, `local-llm` extra), anthropic, gemini, vertexai, groq, bedrock, litellm, litellmrouter, claude-code (uses Claude Pro/Max subscription), openai-codex, github-copilot, xai-oauth, nous, fireworks, and `none`.
- `HINDSIGHT_API_LLM_PROVIDER=none` (`providers/none_llm.py`): "Retain uses chunks mode (no fact extraction), and reflect/consolidation are disabled"; a chunk store with semantic search only.

### 1d. Fully local?

Yes. Embedded pg0 plus local embeddings plus local reranker plus `ollama`, `lmstudio`, `llamacpp`,
or `openai` with `HINDSIGHT_API_LLM_BASE_URL` pointing at a local server. `.env.example` lines 146
to 160 give LM Studio (`http://localhost:1234/v1`) and Ollama (`http://localhost:11434/v1`)
examples. The docs page `developer/models.mdx` states "Set HINDSIGHT_API_LLM_PROVIDER=openai and
point HINDSIGHT_API_LLM_BASE_URL at the endpoint that serves /chat/completions". The published Docker
image does not bundle llama-cpp-python; `docker/docker-compose/local-llm/` adds it.

## 2. Claim: 39% to 83.6% on LongMemEval with the same 20B backbone vs full-context

Verdict: verified against the paper. Table 3 (extracted text, page 19), LongMemEval S setting,
500 questions, all values in percent:

| Question type | Full-context GPT-4o | Full-context OSS-20B | Zep GPT-4o | Supermemory GPT-4o | Supermemory GPT-5 | Supermemory Gemini-3 | Hindsight OSS-20B | Hindsight OSS-120B | Hindsight Gemini-3 |
|---|---|---|---|---|---|---|---|---|---|
| single-session-user | 81.4 | 38.6 | 92.9 | 97.1 | 97.1 | 98.6 | 95.7 | 100.0 | 97.1 |
| single-session-assistant | 94.6 | 80.4 | 80.4 | 96.4 | 100.0 | 98.2 | 94.6 | 98.2 | 96.4 |
| single-session-preference | 20.0 | 20.0 | 56.7 | 70.0 | 76.7 | 70.0 | 66.7 | 86.7 | 80.0 |
| knowledge-update | 78.2 | 60.3 | 83.3 | 88.5 | 87.2 | 89.7 | 84.6 | 92.3 | 94.9 |
| temporal-reasoning | 45.1 | 31.6 | 62.4 | 76.7 | 81.2 | 82.0 | 79.7 | 85.7 | 91.0 |
| multi-session | 44.3 | 21.1 | 57.9 | 71.4 | 75.2 | 76.7 | 79.7 | 81.2 | 87.2 |
| Overall | 60.2 | 39.0 | 71.2 | 81.6 | 84.6 | 85.2 | 83.6 | 89.0 | 91.4 |

Backbone: the paper says "For the Hindsight (OSS-20B) configuration, both the memory stack (TEMPR and
CARA) and the answer generation model are instantiated with GPT-OSS-20b" and "we use GPT-OSS-120b as the
judge LLM with temperature set to 0.0". Section 7.4: "HINDSIGHT with OSS-20B achieves 83.6% overall
accuracy, a +44.6 point gain over the Full-context OSS-20B baseline (39.0%), and even surpasses
Full-context GPT-4o (60.2%)". The abstract rounds 39.0 to "39%".

Caveats a skeptic should keep:

- The 89.0 and 91.4 rows use GPT-OSS-120b for the memory stack; Gemini-3 Pro is only the final answer generator in the 91.4 row.
- Zep and Supermemory scores are taken from vendor reports, and the README says "Other scores are self-reported by software vendors".
- The extracted paper text still contains unfilled placeholders: "Token budgets for retrieval are set to <add> tokens for LongMemEval and <add> tokens for LoCoMo".
- Scoring is LLM-as-judge (GPT-OSS-120b).
- On LoCoMo (Table 4) Backboard scores 90.00 overall against Hindsight's best 89.61; the abstract's "vs. 75.78% for the strongest prior open system" refers to Memobase, so Hindsight is not top of that table.
- The benchmark harness is in the repo: `hindsight-dev/benchmarks/longmemeval/longmemeval_benchmark.py` and `scripts/benchmarks/run-longmemeval.sh`; live results at https://benchmarks.hindsight.vectorize.io/.
- The paper describes an opinion network that the shipped code no longer has (see 1a), so "research version" and current code are not identical.

Paper URL: https://arxiv.org/abs/2512.12818 (PDF https://arxiv.org/pdf/2512.12818, HTML https://arxiv.org/html/2512.12818).

## 3. Claim: explicit coding-agent support with per-repo memory from git history and sessions, living knowledge pages, hooked into Codex, Claude Code, OpenCode, Cursor

Verdict: verified for everything except "historical lessons", which is not a named feature.

Feature location: `hindsight-integrations/coding-agents/` (npm `@vectorize-io/hindsight-coding-agents`,
version 0.6.1). Docs page: https://hindsight.vectorize.io/sdks/integrations/coding-agents (generated from
that README by `hindsight-docs/scripts/sync-coding-agents-doc.mjs`). README on GitHub:
https://github.com/vectorize-io/hindsight/blob/main/hindsight-integrations/coding-agents/README.md

Install command: `npx @vectorize-io/hindsight-coding-agents install all` or `install claude-code`,
`install codex`, `install opencode`, `install cursor-cli`, and so on. There is no separate ingest command;
the README section "Ingestion internals (no CLI)" says the deepen engine (`dist/deepen.js`) "is spawned by
every session start and does only the missing work: bank configuration, conversation import (dedup by
document id), the one-time gitlog seed, the next per-commit diff batch (newest first, bounded per run),
then knowledge pages once extraction has drained".

How it integrates, per harness (README "Per agent" and `src/harness/registry.ts`, `src/harness/hook-lifecycle.ts`):

- Claude Code: 3 hooks in `~/.claude/settings.json` (SessionStart, UserPromptSubmit, Stop), MCP server via `claude mcp add` (user scope), and a companion skill at `~/.claude/skills/hindsight-coding-agent`.
- Codex CLI: 3 hooks in `~/.codex/hooks.json` plus `[mcp_servers]` in `config.toml` (requires `codex_hooks = true`).
- opencode and opencode 2, Kilo: a plugin entry in `~/.config/opencode/opencode.json`; "native tools, no MCP needed". The opencode runtime is built in `src/index.ts` / `src/opencode2.ts` / `src/kilo.ts`.
- Cursor CLI: hooks in `~/.cursor/hooks.json`, `~/.cursor/mcp.json`, and the skill.
- Also: GitHub Copilot CLI, Grok Build, Qwen Code, Factory Droid, ZCode, Antigravity CLI, Devin CLI, Cline CLI (native plugin), pi and Prime Agent (extensions), DeepSeek Harness (Cordis plugin), DeepAgents Dcode (Agent Plugin with Hooks V2).

The hook lifecycle contract is `sessionStart`, `prompt`, `stop` (`hook-lifecycle.ts`: `export type HookLifecycle = "sessionStart" | "prompt" | "stop"`). The prompt hook returns `hookSpecificOutput.additionalContext` (Claude Code's UserPromptSubmit protocol). The MCP tools are defined in `src/core/knowledge-tools.ts`: `hindsight_sync_status`, `hindsight_diagnose`, `hindsight_search_knowledge_pages`, `hindsight_list_knowledge_pages`, `hindsight_read_knowledge_page`, `hindsight_reflect`, `hindsight_capture_initiative`, `hindsight_ingest_document`.

Per-repo bank: one bank per repository named `coding-agent::{gitProject}` shared by every agent (README "Migrating from the per-agent plugins"); `src/core/git.ts` `repoNameOf` is worktree-aware so all worktrees map to one bank. `bankIdTemplate` can change the naming.

Git history ingestion (`src/core/git.ts` header): two paths, "ingestGitLog / gitLogText: the last N commit MESSAGES ONLY, aggregated into ONE document (default). Cheap (one extraction op total)" with `DEFAULT_SEED_LIMIT = 300` (`src/core/seed.ts`), and "ingestGit / retainCommit: per-commit, full message + full diff (opt-in, --diffs). Expensive (one extraction op per commit)". The gitlog document id is `gitlog:<repo>` and a `gitlog-head:<sha>` tag marks freshness.

Codebase survey (`src/core/survey.ts`): on a cold repo the SessionStart hook spawns a detached, read-only headless run of the harness's own CLI (`claude -p ... --mcp-config`, `codex exec --sandbox read-only`, `agy -p --mode=plan`, `opencode run --agent hindsight-survey`) that ingests four fixed documents: `repository-component-map`, `repository-core-concepts`, `repository-conventions-and-patterns`, `repository-tech-stack-and-features`. It re-runs after `surveyRefreshCommits` (default 20) new commits.

Previous sessions: the Stop hook writes the session transcript back to the bank (per-harness transcript readers in `src/core/transcript-*.ts`). `install <harness> --import-conversations` re-imports historical local transcripts for Claude Code, Codex, DeepSeek Harness, pi and Prime Agent (only where the session file records its working directory); opencode, Cursor, Cline, Copilot, Devin are skipped because their SQLite schemas are unversioned.

Knowledge pages (`src/core/missions.ts` `PAGE_TAXONOMY`), each a `source_query` that the server synthesizes from observations:

1. "Component map" (tag `knowledge:component`)
2. "Core concepts" (`knowledge:concept`)
3. "Conventions and patterns" (`knowledge:convention`)
4. "Key decisions and rationale" (`knowledge:decision`)
5. "Initiatives and enhancements" (`knowledge:feature-work`), plus one page per captured initiative under an "Initiatives" folder created by `hindsight_capture_initiative` (`src/core/hindsight.ts:844-870`). This is the "work in progress" part.

Each query gets a scope rule appended: "Scope this page to <project> ITSELF ... when it is about a dependency, leave it out". A grep for "lesson" across the package source, README, skill and docs page found only a code comment, so there is no "historical lessons" page. The nearest things are "Key decisions and rationale" and the `experience` fact type (first-person agent actions) that consolidation folds into observations.

Injection: `src/core/session-start.ts` injects a one-line banner plus the live knowledge-page roster and a guidance preamble; with `autoReflect` (default true) the first prompt receives a one-time reflect synthesis; the page roster is re-injected every `pageRefreshEveryTurns` (default 10) turns. Page refresh policy `pageTriggerType` defaults to `"cron"`.

Where memory lives (README table): `cloud` (Hindsight Cloud, the default, needs an API token), `self-hosted` (URL), or `daemon` (a local `hindsight-embed` started on demand at `127.0.0.1:9077`; needs `uv` on PATH and an LLM for extraction; on macOS the README says a current Rust toolchain is also needed for litellm). The default being Cloud is worth noticing.

The older per-agent plugins (`hindsight-integrations/claude-code` with `.mcp.json`, `settings.json` hooks and a marketplace entry in `.claude-plugin/marketplace.json`; `hindsight-integrations/codex`, `cursor`, `opencode`) still exist but the README says they "are superseded by this package".

Blog PRs currently open (#4091 "CLAUDE.md Doesn't Scale", #4092 "What We Got Wrong About Agent Memory", #4087 "When Your Coding Agent Goes Down, Don't Start Over") show the positioning, but they are marketing, not code.

## 4. Claim: legitimate issues around stale/incorrect memories and lifecycle management

Verdict: verified. Concrete issues:

1. https://github.com/vectorize-io/hindsight/issues/3509 (open, 2026-08-15) "Feature: per-fact delete / correction-expiry for retained memory units (false facts cannot be retracted, only corrected)". A self-hosted user had five retained items claiming a KeePass integration worked when it was never installed; a correcting retain left the originals recallable. Maintainer Nicolò Boschi answered "why don't you just use invalidate?"; the reporter confirmed `PATCH /v1/default/banks/{bank_id}/memories/{memory_id}` with `{"state": "invalidated"}` quarantined 360 units. The thread then turned into a design discussion of "invalidated vs superseded vs active" and a "review date" per fact; the maintainer rejected TTL-based expiry as incorrect by construction.

2. https://github.com/vectorize-io/hindsight/issues/3475 (closed 2026-08-19) "Knowledge Pages restate retired facts as current". With api 0.9.0 and coding-agents 0.3.3 on Claude Code, the auto-seeded "Conventions and patterns" page presented a chezmoi `autocommit=false` setting as a current convention one day after a commit reversed it, and four of five pages named an abandoned dependency as current for 26 hours. The reporter's point: neither invalidation (fact was never wrong) nor the supersession rule (facts do not contradict) covered "true then, retired now".

3. https://github.com/vectorize-io/hindsight/issues/1094 (closed 2026-06-11) "Edit or delete incorrect memories?". Ingesting sessions with Gemma 4 31B produced memories with the wrong subject (agent instead of user). A second commenter reported an agent refusing to use a working web_search tool because a stale "tool is broken" memory kept being injected. Resolved by PR #1976 (reversible curation), designed in RFC https://github.com/vectorize-io/hindsight/issues/1951.

Supporting issues: https://github.com/vectorize-io/hindsight/issues/2696 (Correct and Forget operations, closed), https://github.com/vectorize-io/hindsight/issues/3877 (closed 2026-08-31: a tag filter returned untagged documents, the coding-agents gitlog sweep deleted 53 documents and ~1,445 memory units three times, cascade wiped their facts), https://github.com/vectorize-io/hindsight/issues/3249 (near-duplicate observations across retain calls, proof_count 1 each), https://github.com/vectorize-io/hindsight/issues/3470 (invalidate returned HTTP 500 on Hindsight Cloud for a day because of a missed migration).

What exists in code today for lifecycle: reversible per-fact invalidation with reason (`api/http.py:2595-2675`, `PATCH .../memories/{memory_id}`, states `valid` and `invalidated`), invalidated units moved to an `invalidated_memory_units` archive (`engine/memories/pg/curation.py`), retraction of citations from mental models when a source fact disappears (`engine/reflect/retractions.py`), observation refinement with preserved history, near-duplicate reconciliation, document delete and replace with cascade and re-consolidation, bulk delete by fact type. There is no TTL, no "superseded" state, and no review-date field.

## 5. Practical fit next to a TypeScript/Bun harness with a local Qwen model on an OpenAI-compatible endpoint

What you would run:

- A separate Python 3.11+ FastAPI service (`hindsight-api-slim`, `requires-python = ">=3.11"`). There is no TypeScript implementation of the engine. The TypeScript packages are a generated REST client (`@vectorize-io/hindsight-client`, `hindsight-clients/typescript`, fetch-based, has Deno tests) and a lifecycle wrapper (`@vectorize-io/hindsight-all`, Node >= 22, requires `uv`/`uvx` on PATH to download and run the Python `hindsight-embed` daemon). The CLI is Rust.
- PostgreSQL 15+ with pgvector 0.5+, or the embedded pg0 that the service starts itself (data in `~/.hindsight/pg0/`). No SQLite option.
- An LLM with structured JSON output for retain and consolidation, and function calling for reflect and knowledge-page refresh. A Qwen model behind an OpenAI-compatible endpoint is supported with `HINDSIGHT_API_LLM_PROVIDER=openai`, `HINDSIGHT_API_LLM_BASE_URL=http://<host>:<port>/v1`, `HINDSIGHT_API_LLM_MODEL=<name>` (or the native `ollama`, `lmstudio`, `llamacpp` providers; `.env.example` line 21 mentions handling of thinking blocks from vLLM, Ollama, llama.cpp, TGI). Recall itself needs no LLM. `HINDSIGHT_API_LLM_PROVIDER=none` degrades to a chunk store.
- Embedding and reranker models: by default local HuggingFace downloads (`BAAI/bge-small-en-v1.5`, `cross-encoder/ms-marco-MiniLM-L-6-v2`) via the `local-ml` extra (torch >= 2.11, sentence-transformers >= 5, transformers >= 5.5) or the lighter `local-onnx` extra; or point `HINDSIGHT_API_EMBEDDINGS_PROVIDER=openai` at any endpoint that serves `/v1/embeddings`. `pip install hindsight-api` equals `hindsight-api-slim[all]` which is `local-ml,local-onnx,embedded-db`.
- Simplest path: `docker run -p 8888:8888 -p 9999:9999 -e HINDSIGHT_API_LLM_PROVIDER=openai -e HINDSIGHT_API_LLM_BASE_URL=http://host.docker.internal:PORT/v1 -e HINDSIGHT_API_LLM_MODEL=... -v hindsight-data:/home/hindsight/.pg0 ghcr.io/vectorize-io/hindsight:latest`. Then talk REST from Bun, or mount the per-bank MCP endpoint `http://localhost:8888/mcp/{bank_id}/` (`docs/developer/mcp-server.md`, open by default, API-key tenant extension optional).
- The coding-agents package assumes specific hosts. For a custom harness you would either wire Claude Code style hooks yourself (it speaks `hookSpecificOutput.additionalContext`) or add a HarnessAdapter as `src/harness/registry.ts` describes. Note that the registry already contains a harness id `zcode` with config at `~/.zcode/cli/config.json`; unrelated unless it is your project.
- Cost of first start: the embed daemon README says the first run "downloads dependencies and loads ML models - can take 1-3 minutes"; every retain and every consolidation is at least one LLM call, and knowledge-page auto-refresh is "one synthesis per page per consolidation" (README config table).

License compatibility: MIT is a permissive license and is compatible with AGPL-3.0-only. You may copy code into an AGPL-3.0-only project provided the MIT copyright and permission notice is preserved for the copied portions (for example in a third-party notices file); the combined work can be distributed under AGPL-3.0-only. Ideas and architecture are not restricted by copyright at all. MIT carries no patent grant, which does not affect AGPL compatibility. The README marks "Hindsight" with a trademark symbol, so do not reuse the name for your own feature.

## 6. Three ideas from the code worth borrowing for a lightweight project-memory feature

1. Status handoff: treat the handoff document as a structured doc of sections and blocks with stable ids and let the model emit edit operations against ids instead of rewriting prose (`engine/reflect/structured_doc.py`, `engine/reflect/delta_ops.py`: "Sections and blocks not mentioned by any op are physically copied through unchanged", "zero ops equals identical doc", invalid ops are dropped and logged). Record what each block was based on and prune sentences whose sources no longer exist (`engine/reflect/retractions.py`). The "Initiatives and enhancements" page plus one page per captured initiative (`hindsight_capture_initiative`) is a clean model for in-flight work, and `hindsight_sync_status` gives a readiness contract ("synced": true) so the agent knows whether the handoff is current.

2. Lessons from failures: keep first-person agent actions ("I changed X", "I discovered Y", "I debugged Z") as a distinct `experience` type from world facts (`retain/fact_extraction.py:1075`), then consolidate repeated evidence into observations that carry `proof_count`, exact supporting quotes, `source_memory_ids` and a `history` that is refined rather than overwritten (`consolidation/consolidator.py`). Add a reversible `invalidated` state with a reason instead of hard delete (`api/http.py` PATCH memories, `memories/pg/curation.py`), and consider the three-state taxonomy the community converged on in #3509: active, superseded, invalidated.

3. Project map: seed cheaply from git before doing anything expensive: one aggregated document of the last 300 commit messages (one extraction), then per-commit diffs newest first in bounded batches (`src/core/git.ts`, `src/core/seed.ts`), a `gitlog-head:<sha>` marker plus `git rev-list --count sha..HEAD` to know how stale the map is, and a read-only headless survey run under the agent's own CLI that writes fixed-id documents (component map, core concepts, conventions, tech stack) re-run every 20 commits (`src/core/survey.ts`). Name the bank by the worktree-aware repo name so all worktrees share one memory, and append a scope rule that excludes facts about dependencies ("Scope this page to <project> ITSELF", `src/core/missions.ts`).

## 7. Source index

- Repo: https://github.com/vectorize-io/hindsight
- README: https://github.com/vectorize-io/hindsight/blob/main/README.md
- Repo CLAUDE.md (architecture inventory): https://github.com/vectorize-io/hindsight/blob/main/CLAUDE.md
- Engine: https://github.com/vectorize-io/hindsight/tree/main/hindsight-api-slim/hindsight_api/engine
- Fact types: https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/engine/retain/types.py
- Opinion removal migration: https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/alembic/versions/g2h3i4j5k6l7_remove_opinion_fact_type.py
- Retrieval: https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/engine/search/retrieval.py
- Reflect agent: https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/engine/reflect/agent.py
- Consolidation: https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/engine/consolidation/consolidator.py
- Delta ops: https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/engine/reflect/delta_ops.py
- Embeddings: https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/engine/embeddings.py
- Storage docs: https://github.com/vectorize-io/hindsight/blob/main/hindsight-docs/docs/developer/storage.md
- Models docs: https://github.com/vectorize-io/hindsight/blob/main/hindsight-docs/docs/developer/models.mdx
- Knowledge pages docs: https://github.com/vectorize-io/hindsight/blob/main/hindsight-docs/docs/developer/knowledge-pages.mdx
- Coding agents package: https://github.com/vectorize-io/hindsight/tree/main/hindsight-integrations/coding-agents
- Coding agents docs page: https://hindsight.vectorize.io/sdks/integrations/coding-agents
- Page taxonomy: https://github.com/vectorize-io/hindsight/blob/main/hindsight-integrations/coding-agents/src/core/missions.ts
- Harness registry: https://github.com/vectorize-io/hindsight/blob/main/hindsight-integrations/coding-agents/src/harness/registry.ts
- Paper: https://arxiv.org/abs/2512.12818
- Benchmarks site: https://benchmarks.hindsight.vectorize.io/
- Issues: https://github.com/vectorize-io/hindsight/issues/3509 , https://github.com/vectorize-io/hindsight/issues/3475 , https://github.com/vectorize-io/hindsight/issues/1094 , https://github.com/vectorize-io/hindsight/issues/1951 , https://github.com/vectorize-io/hindsight/issues/2696 , https://github.com/vectorize-io/hindsight/issues/3877
