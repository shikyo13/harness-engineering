# 02 — Competitor audit: Aider

Audited 2026-10-13. Repo: `Datasets/harness-research/repos/aider`, head `5dc9490bb35f9729ef2c95d00a19ccd30c26339c` (2026-05-22, "Merge PR #5173 — anthropic models-list add opus-4-5"), version `0.86.3.dev` (`aider/__init__.py:3`). Method: direct reads of the Python implementation under `aider/`; every `file:line` claim is source-verified unless tagged **[doc]**.

Branch check: `git ls-remote --heads origin` shows only `main` (at audited head) and `debug-works` (`466df1f8`, 2025-03-28, single "noop" commit, strictly behind main — diff vs main is 190 files of *deletions*). No active development branch holds newer work. Clone is shallow (`git rev-parse --is-shallow-repository` → true, 1 commit), so commit-cadence claims are **[doc]**: the head merge is a model-catalog YAML addition, consistent with the well-known post-2025 slowdown to catalog/config maintenance.

## 1. Architecture and main loop

Everything is one synchronous Python object, `Coder` (`aider/coders/base_coder.py:88`), selected by `edit_format` string via a registry scan over `coders.__all__` (`base_coder.py:124-201`, registry in `coders/__init__.py:17-34` — 14 coder classes). Entry: `Coder.run` (`base_coder.py:876-892`) is a REPL: `get_input` → `run_one` → repeat; KeyboardInterrupt once = interrupt current turn, twice within 2 s = exit (`base_coder.py:986-1000`).

`run_one` (`base_coder.py:924-944`) is the whole agent loop — a `while message:` that sends the message, then re-sends only if `self.reflected_message` was set (reflection, §7), capped by `max_reflections = 3` (`base_coder.py:101`, enforced at `:939-941`). There is no tool-call loop at all: one LLM call per cycle, edits are parsed out of prose, and continuation happens only through explicit reflection. A turn is `send_message` (`base_coder.py:1419-1623`):

1. Append user msg to `cur_messages`; build full prompt via `format_messages` → `format_chat_chunks` (§5); pre-flight token check `check_tokens` (`:1396-1417`) warns and asks to proceed if over `max_input_tokens`.
2. Streaming send with exponential retry: `retry_delay` starts 0.125 s, doubles, stops at `RETRY_TIMEOUT = 60` (`models.py:26`); retryability per exception type comes from a hard-coded table (`exceptions.py:13-56`); `ContextWindowExceededError` breaks out to `show_exhausted_error` (`:1628-1679`) and ends the turn — **no automatic compaction**; the user must `/clear`/`/drop`.
3. `FinishReasonLength` (output truncation, `:1894-1898`): if the model supports assistant prefill, the partial content is re-appended as an assistant message with `prefix=True` and the request is re-issued to continue (`:1492-1505`); otherwise treated as exhausted.
4. Post-response: file-mention detection in the reply offers to `/add` (`:1560-1567`, `check_for_file_mentions:1761-1781`); then `reply_completed()` hook (`:1625`, overridden by architect/context coders); then `apply_updates` (`:2296-2336`): `get_edits` → `apply_edits_dry_run` → `prepare_to_edit` (permission + dirty-commit gate) → `apply_edits`. Any `ValueError` from parsing/applying becomes `self.reflected_message = str(err)` (`:2305-2316`) — the LLM gets its own failure report back as the next user message.
5. After successful edits: auto-commit (§6), then auto-lint (`auto_lint=True` default, `:318`; `lint_edited:1681-1696`, tree-sitter syntax lint per `linter.py:203`), offer "Attempt to fix lint errors?" → sets reflection message; then suggested shell commands are offered and optionally run with output added to chat (`run_shell_commands:2434-2448`); then optional `--test-cmd` auto-test with the same reflection path (`:1616-1623`).

"Done" detection is absence-based: no reflection raised and edits applied → turn ends. The assistant's committed-hash reply is injected into history as `"I committed the changes with git hash {hash} & commit msg: {message}"` (`base_prompts.py:4`, used at `base_coder.py:2387-2390`), and a zero-edit response gets `"I didn't see any properly formatted edits in your reply?!"` (`base_prompts.py:8`).

```
user input ─→ preproc (commands / @file mentions / URL scrape)      base_coder.py:912-922
   │
   ▼ run_one loop (≤3 reflections)                                   :924-944
send_message: chunks = system+examples+readonly+repomap+done+files+cur+reminder  :1226-1331
   │  token gate :1396 · retry/backoff :1449-1488 · prefill continue :1492-1505
   ▼
reply ─→ @file add offers :1560 ─→ get_edits ─ apply_edits          :2296-2336
   │        (ValueError ⇒ reflected_message ⇒ loop)
   ├─ auto_commit(edited) via weak-model msg + Co-authored-by        :2375 / repo.py:131
   ├─ auto_lint  (errors ⇒ reflection after confirm)                 :1599-1607
   ├─ suggested shell cmds, output→chat optional                      :2434-2485
   └─ auto_test  (errors ⇒ reflection after confirm)                  :1616-1623
```

## 2. Edit formats

All formats implement `get_edits()` (parse reply → list of `(path, ...)` edits) and `apply_edits()`; failures raise `ValueError` with an LLM-facing report that drives reflection.

- **whole** (`wholefile_coder.py`, `edit_format="whole"`): reply contains filename line + fenced full-file copies. Filename recovery is layered (`:32-122`): fence-preceding line stripped of `*`/`` ` ``/`#`/`:`; bogus `path/to/` prefix stripped if basename is in chat; a backtick-quoted filename anywhere in prose (`saw`); sole chat file as last resort — else `ValueError` "No filename provided before fence". Duplicate blocks for one file resolved by source priority `block > saw > chat` (`:108-122`). Writes whole file. This is the `ModelSettings` dataclass default (`models.py:131`) but *all* modern entries in the catalog use diffs.
- **diff** = SEARCH/REPLACE edit blocks (`editblock_coder.py`, `edit_format="diff"`; the default for 324 of 357 catalog entries in `aider/resources/model-settings.yml`). Markers tolerate 5-9 `<`/`=`/`>` (`HEAD/DIVIDER/UPDATED` regexes, `:386-388`) so the model can use conflict-marker variants. Application cascade in `replace_most_similar_chunk` (`:157-187`): exact line-tuple match → same-content-with-uniform-indent-offset (outdents SEARCH/REPLACE, reindents, `:243-293`) → drop spurious leading blank line → `...` elision matching (`try_dotdotdots:190-240`, requires each elided segment to occur exactly once) → fuzzy: best `SequenceMatcher` chunk within ±10 % length window, threshold 0.8 (`replace_closest_edit_distance:296-329` — dead code, unreachable `return` before it at `:183`-`:188`). Zero-match recovery inside the same call: if the named file misses, try **every other chat file** for the same SEARCH block (`:55-65`). On failure the reflection text lists each failed block, appends `find_similar_lines` "Did you mean…" (ratio ≥ 0.6, ±5 lines context, `:602-628`), warns if REPLACE content already present, and instructs "reply with fixed versions of the blocks above that failed; don't re-send the applied ones" (`:84-124`). Shell blocks (```bash etc.) are intercepted and executed, not applied as edits (`:451-485`, `shell.py`).
- **diff-fenced** (`editblock_fenced_coder.py`, 10 lines): same engine, prompts use `<source>` XML fences instead of backticks (chosen for Gemini 2.5 Pro family, 35 catalog entries).
- **udiff** (`udiff_coder.py`, `edit_format="udiff"`; 4 entries, gpt-4-turbo era): parses ```` ```diff ```` blocks into hunks (`find_diffs/process_fenced_block:312-400`). Application: normalize hunk → `directly_apply_hunk` via `flexible_search_and_replace` (search_replace.py: preproc matrix of strip-blank/relative-indent combos + a *real git cherry-pick* fallback that commits before/after in a temp repo and cherry-picks `--minimal` (`search_replace.py:548-563`), plus `dmp_lines_apply` diff-match-patch) → if the whole hunk fails, `make_new_lines_explicit` rewrites elided context against real file content (`:209-240`), then `apply_partial_hunk` tries progressively fewer context lines, preferring preceding over following (`:282-310`). Guards: refuses tiny (<10 non-ws char) context that matches multiple times (`:268-272`); ambiguity raises `SearchTextNotUnique` with an "add more context lines" message (`:29-39`). `udiff-simple` is a prompt-variant subclass (`udiff_simple.py`).
- **patch** (`patch_coder.py`, opt-in): port of OpenAI codex `apply_patch` (add/update/delete/move chunks, `:549-640`), with a fuzz ladder 0 (exact) / 1 (rstrip) / 100 (strip) and a 10 000 penalty for matching an EOF-marked chunk away from EOF (`find_context_core:59-95`). Not in the shipped catalog (0 entries).
- **func** variants (`wholefile_func_coder.py`, `editblock_func_coder.py`) use OpenAI function calling; whole-file-func now raises `RuntimeError("Deprecated…")` in `__init__` (`wholefile_func_coder.py:47`).
- **ask / help / context / architect / editor-\*** are prompt-only or orchestration variants (see §4).

Fence selection is content-aware: `choose_fence` (base `:609-635`) scans all in-chat + read-only content and picks the first fence from `all_fences` (triple-backtick, quad-backtick (issue #2879), `<source>`, `<code>`, `<pre>`, `<codeblock>`, `<sourcecode>`, `base_coder.py:77-85`) that appears in no file; system prompt then injects a quad-backtick warning reminder when needed (`:1203-1208`).

Model→format selection is **data, not code**: `aider/resources/model-settings.yml` has 357 model entries (counts: 324 `diff`, 35 `diff-fenced`, 4 `udiff`, 4 `whole` (grok-3-mini + gemini-2.5-pro-preview aliases), 300 with `weak_model_name`, 338 with `use_repo_map: true`, 145 with `editor_model_name`), loaded at import (`models.py:154-158`). Exact-match wins; else `apply_generic_model_settings` pattern-matches model-name substrings (`models.py:437-598`, e.g. `"/o3-mini"` → diff + `"Formatting re-enabled. "` prefix; `deepseek v3` → diff + `reminder=sys`; `gpt-4-turbo` → udiff; `qwen3-235b` → diff + `/no_think` prefix); else dataclass default whole. Also per-model: `lazy`/`overeager` prompt injections (`base_prompts.py:12-19`, injected via `fmt_system_prompt:1174-1179`), `reminder` placement `"sys"` vs appended-to-last-user-message (`:1320-1329`), `examples_as_sys_msg`, `send_undo_reply`, temperature/streaming quirks.

## 3. Repo map (`repomap.py`) — the crown jewel

`RepoMap.__init__` defaults: `map_tokens=1024`, `map_mul_no_files=8` (args default 2 via `--map-multiplier-no-files`, `args.py:264-268`; `Coder.__init__` signature default 8, `base_coder.py:323`), `refresh="auto"` (`args.py:254-262`), `max_context_window = model max_input_tokens` (`base_coder.py:493-508`).

Pipeline per turn (`Coder.get_repo_map`, `base_coder.py:709-748`): extract `mentioned_fnames` (paths/basenames mentioned in current user message) and `mentioned_idents` (all words ≥5 chars, `:678-707`, plus identifier-equals-filestem matching `:684-707`); split repo files into chat vs other; call `RepoMap.get_repo_map(chat, other, mentions)`; if empty, retry unhinted global map; prepend `repo_content_prefix` ("Here are summaries… treat them as *read-only*… ask me to add them", `base_prompts.py:45-48`) and wrap as a user+assistant exchange ("Ok, I won't try and edit those files without asking first.", `base_coder.py:750-761`). Inserted into the prompt *after* read-only files and *before* chat history (§5 order).

**Tag extraction** (`get_tags_raw:279-363`): per file, tree-sitter parse + `<lang>-tags.scm` queries (32 language-pack `.scm` files under `aider/queries/tree-sitter-language-pack/`, 27 under the legacy `tree-sitter-languages/`; `get_scm_fname:805-829`) capturing `name.definition.*` → `Tag(kind="def")` and `name.reference.*` → `Tag(kind="ref")` with `(rel_fname, fname, line, name, kind)`. Languages that emit defs but no refs get refs backfilled by pygments `Token.Name` lexing (line=-1) (`:338-363`). Tags cached in a diskcache SQLite dir `.aider.tags.cache.v4` keyed by absolute fname, validated by mtime (`TAGS_CACHE_DIR:43`, `get_tags:233-264`); SQLite corruption falls back to in-memory dict (`tags_cache_error:177-215`).

**Graph ranking** (`get_ranked_tags:365-574`): a `networkx.MultiDiGraph` of *files*; for each identifier defined in one or more files, edge `referencer → definer` with weight (lines 481-514):
- base mul 1.0; **×10** if identifier appears in user's mentioned idents; **×10** if identifier is snake/kebab/camelCase and ≥8 chars (long distinctive names matter more); **×0.1** if starts with `_`; **×0.1** if defined in >5 files (common names discounted); **×50** extra if the referencer file is in the chat;
- edge weight = mul × `sqrt(num_refs)` (dampen high-frequency references, `:512`).
- defs with zero refs get a 0.1 self-edge so they survive PageRank (`:472-479`).
- Personalization: chat files, mentioned files, and files whose path components match a mentioned ident get `personalize = 100/len(fnames)` (100× the networkx default uniform 1/N; `:383`, `:424-445`), passed as both `personalization=` and `dangling=` (`:519-525`); `ZeroDivisionError` retries unpersisted, then returns `[]`.
- PageRank (`nx.pagerank(G, weight="weight")`, `:525`) ranks *files*; then each file's rank is distributed across its out-edges proportional to weight to score `(file, identifier)` pairs (`:533-545`); ranked defs (skipping chat files) form the tag list, followed by remaining files in descending pageRank order as bare filenames, then tag-less files (`:547-572`) — i.e., full files → signature snippets → filename-only, by rank.

**Token budgeting** (`get_ranked_tags_map_uncached:629-706`): binary search over "top-N ranked tags" so that rendered map tokens hit the budget: start `middle = min(max_map_tokens // 25, num_tags)`, accept if `tokens <= budget` and maximal, or within `ok_err = 15 %` of budget (early stop, `:689-696`). Token counting on map text is *sampled*: for texts ≥200 chars, take every `len//100`-th line, tokenize, scale (`token_count:89-101`). Budget scaling: with no files in chat, budget becomes `min(max_map_tokens × map_mul_no_files, max_context_window − 4096)` (`get_repo_map:122-132`); `ContextCoder` multiplies by 8 and forces refresh (`context_coder.py:17-19`). Important root files (README/LICENSE/requirements/package.json/pyproject/… ~150-entry list, `special.py:3-…`, `filter_important_files`) are force-prepended even if unranked (`:656-662`).

**Rendering** (`to_tree/to_render:710-784`): per file, TreeContext from `grep_ast` with lines-of-interest = ranked def lines, expanded to enclosing scopes (`add_context`, no line numbers, margin 0); lines hard-truncated to 100 chars (minified-file guard, `:782`); chat files excluded.

**Update strategy** (`get_ranked_tags_map:576-627`): modes via `--map-refresh`: `auto` (default) — cache keyed on (chat files, other files, budget, mentions) *only used if last build took >1.0 s*, i.e. small repos rebuild every turn, big repos only when the mention-set changes; `always` never caches; `files` always caches (ignores mentions in key); `manual` returns the last map until `/map refresh`. Result also memoized in `map_cache` and `last_map`. Per-`RepoMap` instance, process lifetime.

## 4. Architect / editor two-model mode

`ArchitectCoder` (`architect_coder.py`, format `"architect"`) subclasses `AskCoder` — the architect has **no apply path**; its system prompt is plan-oriented. Everything happens in `reply_completed` (`:11-48`): on any non-empty reply, confirm "Edit the files?" (or auto-accept via `--auto-accept-architect`, default True, `args.py:179`, `main.py:1005`), then build a second coder: `editor_model = main_model.editor_model or main_model` (`:23`; catalog gives 145 models an `editor_model_name`, e.g. gemini 2.5 pro pairs a flash editor), `edit_format = main_model.editor_edit_format` — which defaults to `"editor-" + editor_model.edit_format` (`models.py:640-643`), i.e. stripped-down edit-only prompts without chat pleasantries (`editor_editblock_coder.py`, `editor_whole_coder.py`, `editor_diff_fenced_coder.py`). The editor starts with **empty chat history** (`cur_messages=[] done_messages=[]`, `architect_coder.py:38-39`), no repo map (`map_tokens=0`, `:28`), no shell suggestions, and receives the architect's plan verbatim as its single user message (`run(with_message=content, preproc=False)`). Cost and commit hashes roll up to the architect; chat records "I made those changes to the files." Mode-switching at runtime (`/ask`, `/architect`, `/code`) constructs a new coder via `Coder.create(from_coder=…)` which copies chat/read-only files and — when the edit format changes — **summarizes the whole history through the weak model** so old assistant messages in the old format can't contaminate the new LLM (`base_coder.py:154-169`).

## 5. Context management

- **Editable vs read-only**: two disjoint sets, `abs_fnames` (editable, full content fenced into `chat_files` chunk with "Trust this message as the true contents…" prefix, `base_prompts.py:24-27`) and `abs_read_only_fnames` (`/read`; sent as its own chunk with "Do not edit these files!", `base_coder.py:763-787`, `base_prompts.py:50-52`). Editing a file not in chat triggers an interactive "Allow edits to file that has not been added to the chat?" and adds it (`allowed_to_edit:2191-2240`). `/add` supports globs, dirs, creation of new files (`commands.py:799+`); `/drop` clears all or substring-matched files incl. read-only (`commands.py:912+`). Adding ≥4 files totaling >20 k tokens warns once (`check_added_files:2244-2267`).
- **Per-turn message assembly** (`format_chat_chunks:1226-1331`, order fixed in `chat_chunks.py:16-27`): `system → examples → readonly_files → repo-map → done(summary-capped history) → chat_files → cur → system_reminder`. Repo map and read-only files sit *before* history so the provider prefix cache covers them; `add_cache_control_headers` marks examples/system + readonly/repomap + chat_files chunks as cacheable (`chat_chunks.py:28-47`), and an optional background thread re-pings the provider every ~5 min (`AIDER_CACHE_KEEPALIVE_DELAY`, `base_coder.py:1340-1394`) to hold Anthropic-style caches warm. The `system_reminder` (e.g. the 1 600-char SEARCH/REPLACE ruleset, `editblock_prompts.py:120+`) is re-appended each turn — either as a system message or glued onto the last user message, per `model.reminder`, and only if a token check says it fits (`:1313-1329`).
- **History summarization** (`history.py`): cap = `min(max(max_input/16, 1024), 8192)` tokens (`models.py:356-358`). When `done_messages` exceed the cap, a **background thread** summarizes during user thinking time (`summarize_start/summarize_worker:1002-1034`); algorithm is recursive split-in-half: keep a tail ≤ half-budget ending at an assistant boundary, summarize the head (up to summarizer's own context) into one user message with `summary_prefix`, recurse if still too big (`summarize_real:33-95`). Summarizer model chain: `[weak_model, main_model]` (`base_coder.py:510-513`).
- **Token counting**: real tokenizers everywhere via litellm (`models.py:647-658`); failure → 0 which silently disables budget gates; image token cost modeled OpenAI-style (170/token tile, `:704-718`); map text uses line-sampling estimator (§3).

## 6. Git discipline

- **Auto-commit after AI edits**: `auto_commit(edited)` → `GitRepo.commit(fnames=edited, context=cur_messages dump, aider_edits=True)` (`base_coder.py:2375-2395`, `repo.py:131-318`): stages *only* the edited paths (`git commit -- -- <paths>`), generates the message with the weak-model chain over `git diff HEAD -- fnames` (`get_commit_message:326-373`, skips models whose context can't fit), commits with `--no-verify` unless `--git-commit-verify` (`args.py:493`), and attributes via `GIT_AUTHOR_NAME`/`GIT_COMMITTER_NAME = "<user> (aider)"` or (default since args default `attribute_co_authored_by=True`, `args.py:482-490`) a `Co-authored-by: aider (<model>) <aider@aider.chat>` trailer (`repo.py:246-267`, `291-308`). The hash+message are fed back into chat so the model can reference its own commits.
- **Dirty-files policy**: before applying edits, any target file that is dirty in git gets force-committed *first* (`check_for_dirty_commit:2175-2189` → `prepare_to_edit → dirty_commit:2411-2423`, `aider_edits=False`: user's work preserved in a separate committer-attributed commit) — rationale in code comment `:2183`: "/undo needs a committed copy of the file".
- **Undo** (`commands.py:553-656`): `/undo` only reverts a head commit whose hash is in the session's `aider_commit_hashes`; refuses merge commits, dirty affected files, files absent from the parent tree, and already-pushed heads; then `git checkout HEAD~1 -- <each file>` + `git reset --soft HEAD~1` — restores *files and moves HEAD back*, leaving the parent as head. Weak models optionally receive a canned undo-success assistant reply (`send_undo_reply`, `commands.py:654`, set for gpt-4/llama3-70b eras, `models.py:511-524`).
- Non-AI file watcher (`watch.py`) detects user edits mid-session and re-sends refreshed file contents.

## 7. Weak-model handling (the reflection/retry machinery)

Layered, all source-verified:
1. **Transport retries**: typed table, exp backoff ≤60 s (§1).
2. **Format forgiveness**: tolerant marker regexes (5-9 fence chars), filename recovery heuristics, fence collision avoidance, `...`-elision support, whitespace outdenting, cross-file SEARCH relocation, udiff context-shrinking and context-expansion, patch fuzz ladder — all above §2.
3. **Reflection loop**: any parse/apply `ValueError` becomes the next user message, max 3 per turn (`base_coder.py:101, 936-944, 2305-2316`). Failure reports are engineered: exact failed blocks echoed, "Did you mean these actual lines?" from near-match search (≥0.6 ratio), "REPLACE lines are already in the file" detection, and "don't re-send applied blocks" (`editblock_coder.py:84-124`).
4. **Lint/test reflection**: lint or test output becomes a reflection message after confirmation (`base_coder.py:1599-1623`).
5. **Output-limit continuation** via assistant prefill (§1.3); **context-window exit** is honest and advisory only (`show_exhausted_error` with 0.7-fudge heuristic, `:1628-1679`).
6. **Model-level guardrails in data**: `lazy` prompt for lazy models (gpt-4o `lazy: true` in yml), `overeager: true` prompt for Gemini (stop it editing unrelated code), temperature forced off for reasoning models, `system_prompt_prefix` like `"Formatting re-enabled. "` for o-series and `/no_think` for qwen3.
7. **Weak model as utility**: summarization, commit messages, and (per `commit_message_models()` fallback chain) anything cheap; main-model-fits checks before the weak model is tried.

There is no *automatic* downgrade to a cheaper edit format on repeated malformed responses — that stays with the user (`/edit <format>`) — and there is no edit-format self-repair beyond re-prompting in the same format.

## 8. Strengths, brittleness, what is stale

Genuinely strong (source-verified):

- The repo map: best-in-class, cheap, cache-tiered, honest about its approximate token math.
- Edit-application cascades with *specific, model-actionable* failure reports (echoed block + actual lines + applied-don't-resend).
- Data-driven per-model behavior table — zero per-model if-trees inside the loop; quirks live in YAML.
- Prompt layout engineered for provider prefix caching: stable chunk order, cache_control headers on the stable prefix, background warming pings.
- Git discipline: pre-edit dirty-file protective commits, path-scoped commits, hash-tracked session undo.
- One synchronous loop with almost no machinery — the whole agent is auditable in an afternoon (~13.4 k LOC in `aider/` + 6.9 k in `coders/`).

Brittle / stale:

- Name-substring model matching in `models.py:437-598` (`if "sonnet-4-" in model` style) rots with every model release; only the YAML half is maintainable.
- litellm as a monolithic dependency, with an exception table that *raises at import* if litellm ships an error class aider hasn't enumerated (`exceptions.py:66-76`).
- Dead code that reviewers keep "fixing": the fuzzy `replace_closest_edit_distance` is unreachable behind a bare `return` (`editblock_coder.py:183-188`).
- Pin debt: `networkx<3.5` (`requirements/requirements.in:40`), dual tree-sitter grammar backends with version-dependent tag semantics (`repomap.py:266-277, 472-475`).
- No async, no durability (process death loses the in-flight turn), no structured tool use in the main path, no parallel sessions, single global repo-map cache per process.
- Context overflow strategy is only "warn, or summarize away history"; no compaction that preserves recent turns verbatim.
- Reflection cap 3 gives up silently; no automatic fallback edit format.
- Undo is strictly LIFO over session-tracked aider commit hashes: no cross-session undo, refuses merges and pushed heads.
- 2 485-line `Coder` god-object; mutable class-attribute state shared across instances (e.g. `tree_cache = dict()` at `repomap.py:708`, `warned_files`/`warning_given`); `Coder.create` deep-kwargs plumbing between modes is fragile (comment-laden reconciliations at `base_coder.py:154-188`).

Activity **[doc]**: head merge is a model-catalog YAML addition; version `0.86.3.dev`; the only other remote branch is 14 months stale; upstream releases slowed through 2025-2026 into maintenance/catalog mode. Treat the algorithms as mature and the codebase as frozen.

## 9. What Zero Code should learn — and not copy

**Adopt (repomap specifics, in priority order):**
1. The **def/ref graph + personalized PageRank over files** with the exact weight constants that were battle-tuned: mentioned-ident ×10, long-cased-name ×10, leading-underscore ×0.1, >5-definer name ×0.1, chat-referencer ×50, `sqrt(ref_count)`, personalization 100/N for chat+mentioned files, and the ×0.1 self-edge for ref-less defs (repomap.py:487-514). Zero's tree-sitter tagging exists; the ranking+personalization is the missing half.
2. **Binary-search token fitting** of the map against the budget with 15 % tolerance and a sampled tokenizer (repomap.py:666-703, 89-101) — cheap and robust vs. our fixed-size attempts.
3. **Mention-driven personalization from the current user message only** (not the whole history), with filename→ident path matching (base_coder.py:709-747), and the 3-level fallback to an unhinted map.
4. **mtime-keyed on-disk tag cache** + adaptive rebuild (rebuild-per-turn unless the last build cost >1 s) — the `refresh=auto` heuristic is a good default for interactive latency.
5. Force-include manifest/README files (`special.py`) in every map; truncate map lines at 100 chars; render def-site snippets via tree-sitter enclosing scope, not the whole file.
6. **Repo map as read-only user+assistant pair placed *ahead of history*** for prefix-cache stability, and budget ×2-8 when no files are in chat (repomap.py:122-132).

**Adopt (edit-format selection):** keep our matcher cascade but move selection into a *data table per model* — format, prompt dialect (fenced vs backtick), reminder placement, lazy/overeager injections, temperature/prefill quirks (models.py + model-settings.yml pattern). The two dialects that matter today: SEARCH/REPLACE (`diff`) for most, `diff-fenced` for models that over-fence (Gemini), whole only as floor for the weakest. Add `--attribute-co-authored-by`-style attribution default and pre-edit dirty-file protective commits — both cheap and directly usable by Zero's shadow-git design; our undo should keep aider's "only revert commits I made, refuse pushed/dirty" checks.

**Adopt (reliability mechanics):** failure reports that quote the offending block + "did you mean" actual-lines + "already applied — don't resend" (the single highest-leverage weak-model artifact found in any audited harness); the bounded (3) reflection loop with *typed* transport retry table; background-thread history summarization while the user types; cache-warming pings; the architect/editor split (cheap editor with empty history + map_tokens=0 — proven pattern for our model tiers).

**Do NOT copy:** litellm-dependency and its exhaustive exception enumeration; the God-object Coder and class-attribute mutable state; name-substring model matching (use our registry + explicit metadata); whole-file edits as default for anything above toy models; single-process synchronous execution (no durability — exactly what Zero V2 durable admission fixes); summarizing-away-history as the only context-overflow strategy (aider never compacts mid-conversation); interactive confirmations as the security boundary (any-`confirm_ask` everywhere is unusable for our autonomous mode); func-calling edit formats (aider itself deprecated them).

## 10. Claim ledger

| Claim | Status |
|---|---|
| All file:line mechanics in §1-§7 (loop, formats, repomap weights/budget, architect handoff, context assembly, git discipline, reflection) | source-verified at head 5dc9490b (local reads this session) |
| Catalog counts (357 entries; 324 diff / 35 diff-fenced / 4 udiff / 4 whole; 300 weak-model; 338 use_repo_map; 145 editor_model) | source-verified (`grep -c` on `aider/resources/model-settings.yml`) |
| 32 + 27 language tag `.scm` query files | source-verified (`aider/queries/*`) |
| LOC counts (13 359 / 6 923) | source-verified (`wc -l`) |
| `debug-works` branch behind main, single stale commit | source-verified (fetched shallow, diffed) |
| Project-wide activity/decay ("catalog mode", release cadence) | **[doc]** — shallow clone blocks local git-log evidence; inferred from head commit content, version string, and remote branch state |
| Aider benchmark/Bench results, user counts, community standing | not assessed (out of scope; no docs reads) |
