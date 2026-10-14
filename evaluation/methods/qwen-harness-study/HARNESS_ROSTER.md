# Harness qualification roster

Compiled 2026-09-18.

## The eligibility gate

A harness qualifies only if it can be pointed at an arbitrary local OpenAI-compatible base
URL and model id **and** driven non-interactively so a task can be run and scored without a
human. Both halves matter: a harness whose GUI can reach a local endpoint but whose headless
path cannot is not eligible, because the headless path is what the benchmark drives.

Target: `ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit` on mlx-serve 26.9.2 at
`http://127.0.0.1:11234/v1`, which serves OpenAI Chat Completions and also exposes
`/v1/responses`. No cloud keys.

## Provenance of every fact below

| Tag | Meaning |
| --- | --- |
| **web** | read from the project's own repo or docs during this pass; URL cited |
| **local** | verified directly on this machine |
| **unverified** | reported by a secondary source or not confirmed within budget; marked UNKNOWN where it matters |

Nothing here has been confirmed by actually running a harness. Every verdict is a prediction
about eligibility that Stage 0 exists to test, and Stage 0 is expected to overturn some of
them.

## Summary

| # | Harness | Version seen | License | Local base URL | Native dialect | Headless | Verdict |
|---|---|---|---|---|---|---|---|
| 1 | Pi Coding Agent | ~0.74-0.75.x (unverified) | MIT | yes, `registerProvider` extension | OpenAI Chat Completions | `pi -p` | QUALIFIED |
| 2 | OpenCode | v1.18.30 (unverified) | MIT | yes, `provider.*.options.baseURL` | OpenAI Chat Completions | `opencode run` | QUALIFIED |
| 3 | OpenAI Codex CLI | UNKNOWN | Apache-2.0 | yes, `[model_providers.*]` | OpenAI **Responses** only | `codex exec` | QUALIFIED PENDING STAGE 0 |
| 4 | Claude Code | v2.1.275 (unverified) | proprietary | host only, via `ANTHROPIC_BASE_URL` | Anthropic Messages | `claude -p` | QUALIFIED WITH SHIM |
| 5a | DeepSeek Harness, Minimal | 0.1.0-rc.6 (unverified) | MIT | yes, `llm-pi-ai.providers.*` | selectable, incl. `openai-completions` | `sdk-minimal` profile | QUALIFIED WITH CAVEAT |
| 5b | DeepSeek Harness, Standard | same | MIT | same | same | `dsh --profile headless` | QUALIFIED |
| 5c | DeepSeek Harness, Code | same | MIT | same | same | selection mechanism UNKNOWN | BLOCKED PENDING VERIFICATION |
| 5d | DeepSeek Harness, Creator | same | MIT | same | same | n/a | EXCLUDED, out of scope |
| 6 | Qwen Code | 0.24.0 | Apache-2.0 | yes, `OPENAI_BASE_URL` | OpenAI Chat Completions | `qwen -p` | QUALIFIED |
| 7 | Aider | UNKNOWN | Apache-2.0 | yes, `OPENAI_API_BASE` | OpenAI Chat Completions via litellm | `aider --message` | QUALIFIED |
| 8 | mini-SWE-agent | ~2.4.6 (unverified) | MIT | yes, `model_kwargs.api_base` | OpenAI Chat Completions via litellm | `mini --exit-immediately` | QUALIFIED |
| 9 | OpenHands | UNKNOWN | MIT | yes, LiteLLM passthrough | OpenAI Chat Completions via litellm | `openhands --headless -t` | QUALIFIED |
| 10 | Goose | v1.51.0 (unverified) | Apache-2.0 | yes, `OPENAI_HOST` | OpenAI Chat Completions | `goose run -t` | QUALIFIED |
| 11 | Continue CLI | 1.5.47 npm (unverified) | Apache-2.0 | yes, `apiBase` in config.yaml | OpenAI Chat Completions | `cn -p` | QUALIFIED, project archived |
| 12 | Cline | 3.0.61 (unverified) | Apache-2.0 | yes, `--baseurl` | OpenAI Chat Completions | `cline --yolo --json` | QUALIFIED |
| 13 | Roo Code | archived 2026-05-15 | Apache-2.0 | GUI only, not the CLI | OpenAI Chat Completions | NDJSON CLI exists | BLOCKED |
| 14 | Zero Code | `2b47e4f9` | MIT fork | yes, `provider.mlx.options.baseURL` | OpenAI Chat Completions | `zero run` | QUALIFIED (local) |
| 15 | ForgeCode | v2.9.3 (unverified) | UNKNOWN | yes, `OPENAI_URL` | OpenAI Chat Completions | `forge -p` | CANDIDATE, unqualified |
| 16 | Grok Build | v1.0, Aug 2026 | Apache-2.0 | none found | UNKNOWN | `grok -p` | BLOCKED PENDING VERIFICATION |

## Detail

### 1. Pi Coding Agent
**web.** Repo `earendil-works/pi` (formerly `badlogic/pi-mono`), site pi.dev. MIT.
TypeScript/Node, `npm i -g @earendil-works/pi-coding-agent` (package scope moved from
`@mariozechner/*`). Local provider is registered from a TypeScript extension:
`pi.registerProvider(name, { baseUrl, apiKey, api: "openai-completions", models: [...] })`.
Headless `pi -p "prompt"` or `--print`, with `--mode json` for JSON-lines events,
`--no-session` for ephemeral runs, `--provider` / `--model` / `--thinking` flags.
Per-call token usage in the JSON stream is UNKNOWN. Version UNKNOWN beyond ~0.74-0.75.x.
Sources: pi.dev/docs/latest/custom-provider, pi.dev/docs/latest/usage, github.com/earendil-works/pi

**Note.** Ownership and naming moved between accounts and package scopes; confirm the
canonical repo before citing this as "Ronacher's agent". It is Mario Zechner's project with
Ronacher as close collaborator.

### 2. OpenCode
**web.** `sst/opencode`, MIT, TypeScript, client/server. Config `~/.config/opencode/opencode.json`:
`provider.<name>.npm: "@ai-sdk/openai-compatible"` with `options.baseURL` and a `models` map
whose keys are the model ids. Headless `opencode run "prompt"`, `--model provider/model`,
`--format json`, plus `opencode serve` for an HTTP-driven mode. Whether token usage is
itemised in the JSON stream is UNKNOWN. Version v1.18.30 is from a release aggregator, not
the releases page itself.
Sources: opencode.ai/docs/providers, opencode.ai/docs/cli, github.com/sst/opencode

**Relevance.** Zero Code is a fork of this at v1.18.30, so OpenCode is the single most
informative comparison in the study: it isolates what Zero Code's divergence actually bought.

### 3. OpenAI Codex CLI
**web.** `openai/codex`, Apache-2.0, Rust. `~/.codex/config.toml` takes
`[model_providers.<id>]` with `base_url`, `env_key`, `wire_api`, `name`, selected by
top-level `model_provider` and `model`. Headless `codex exec "prompt"` with `--json` for a
JSONL event stream and a meaningful exit status. Token usage fields in that stream UNKNOWN.

**The open question.** `wire_api = "chat"` was deprecated in December 2025 and removed in
February 2026 (**unverified**, secondary sources), leaving only `wire_api = "responses"`.
Our engine does expose `/v1/responses`, so Codex may reach it with no shim at all. Whether
mlx-serve's Responses implementation is complete enough for Codex, in particular tool calls,
reasoning items and streaming semantics, is exactly what a Stage 0 run settles. If it is not,
the fallback is a Responses-to-Chat-Completions shim and Codex becomes QUALIFIED WITH SHIM.
Sources: learn.chatgpt.com/docs/config-file/config-reference, developers.openai.com/codex/noninteractive

### 4. Anthropic Claude Code
**web.** Proprietary, closed binaries, public repo under all rights reserved.
`ANTHROPIC_BASE_URL` swaps the host and `ANTHROPIC_MODEL` passes a model id through
unvalidated when a custom base URL is set, but the endpoint must speak the **Anthropic
Messages API**: `ANTHROPIC_BASE_URL` changes the host, not the wire protocol. Headless
`claude -p` with `--output-format json|stream-json`, `--bare`, `--allowedTools`,
`--permission-mode`; `--output-format json` returns cost and a per-model breakdown, and full
sessions persist as resumable `.jsonl`.

**Consequence.** A Messages-to-Chat-Completions shim is mandatory (LiteLLM proxy or
`claude-code-proxy`). Section 8 of the brief applies in full here: the shim is a suspect, and
any Claude Code failure must be diagnosed as harness or as translation before it is
attributed to architecture.
Sources: code.claude.com/docs/en/env-vars, code.claude.com/docs/en/headless

### 5. DeepSeek Harness
**web, primary.** `deepseek-ai/deepseek-harness`, CLI `dsh`, MIT, Node/TypeScript, plugin
architecture ("Cordis"). Config `$DSH_HOME/settings.yaml`:

```yaml
llm-pi-ai:
  providers:
    local:
      apiKeyEnv: LOCAL_API_KEY
      api: openai-completions
      baseURL: http://127.0.0.1:11299/v1
      models:
        - id: ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit
```

`api` selects the protocol per provider from `openai-completions`, `openai-responses` or
`anthropic-messages`, so no shim is needed. Headless `dsh --profile headless "job"`, exit 0
on completion. Other profiles: `sdk`, `sdk-minimal`, `web`, `acp`. Whether a key can be
omitted for an unauthenticated local server is UNKNOWN. Structured tool-call and token
capture for the headless profile is UNKNOWN; the CLI streams reasoning to stderr and prints
final text to stdout.

**unverified.** The mode taxonomy (Minimal keeps only shell and a file editor; Standard is
the full agent; Code generates a TypeScript SDK and has the model write a program against it
instead of doing function-call round trips; Creator modifies the harness itself) comes from
secondary blogs, not a primary modes document. The research pass also found heavy SEO
blogspam and several forks with injection-like descriptions around this project; none of it
was used as a source.

**Why the split matters.** Minimal against Standard against Code is the cleanest natural
experiment in the whole roster: three tool-surface philosophies inside one codebase with one
prompt stack and one provider layer. It isolates tool surface far better than any
cross-project comparison can. Confirming how to select Code mode non-interactively is
therefore worth real effort in Stage 0.
Sources: github.com/deepseek-ai/deepseek-harness, its apps/cli/README.md and docs/user/guide/providers.md

### 6. Qwen Code
**web.** `QwenLM/qwen-code`, Apache-2.0, Node/TypeScript, `npm i -g @qwen-code/qwen-code`,
binary `qwen`, version 0.24.0 published the same day as this pass. Env vars in `.qwen/.env`
or `~/.qwen/.env`: `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_MODEL`. Headless
`qwen -p "query"` or stdin, with `--output-format json` returning usage and token stats and
`stats.tools.totalCalls`, `--output-format stream-json`, `--yolo`, `--max-wall-time`,
`--max-tool-calls`. A `model.baseUrl` settings.json field exists but the docs say not to set
it by hand.

**Caution.** Same model family as the fixed model. The brief explicitly forbids assuming it
is therefore best, and its inclusion is to test that assumption, not to honour it.
Sources: github.com/QwenLM/qwen-code, qwenlm.github.io/qwen-code-docs/en/users/features/headless/

### 7. Aider
**web.** Apache-2.0, Python, `aider-chat` on PyPI, built on litellm. `OPENAI_API_BASE` plus
`--model openai/<name>`, or `openai-api-base` in `.aider.conf.yml`, or
`AIDER_OPENAI_API_BASE` in `.env`. Headless `aider --message "..." --yes-always`, with
`--auto-test --test-cmd` available. Transcript persists to `.aider.chat.history.md` and
per-turn tokens print as `Tokens: N sent, M received`; there is no structured JSON trajectory.
Version UNKNOWN.
Sources: aider.chat/docs/llms/openai-compat.html, aider.chat/docs/scripting.html

### 8. mini-SWE-agent
**web.** `SWE-agent/mini-swe-agent`, MIT, Python, litellm-based. YAML config:
`model.model_name`, `model.model_kwargs.custom_llm_provider: "openai"`,
`model.model_kwargs.api_base`. Cost tracking for unregistered local models is silenced with
`cost_tracking: "ignore_errors"` or `MSWEA_COST_TRACKING`. Headless via `--exit-immediately`
and `-o/--output` for a `.traj.json` trajectory holding the full message history plus
`model_stats` with `api_calls`. An open issue (#303) says local-model support has known gaps.
The exact one-shot invocation was not fetched from the usage docs and is **unverified**.

**Role.** This is the minimal-agent baseline. Its value is as a floor, not as a contender.
Sources: mini-swe-agent.com/latest/models/local_models/, .../usage/output_files/, github.com/SWE-agent/mini-swe-agent

### 9. OpenHands
**web.** MIT, Python, `pip install openhands-ai`, LiteLLM-based, org moved from
`All-Hands-AI/OpenHands` to `OpenHands/OpenHands`. Settings UI exposes "Base URL" and
"Custom Model"; env or `config.toml` equivalents exist but the exact key spelling was not
confirmed and is UNKNOWN. Headless `openhands --headless -t "task"` or `-f task.txt`, with
`--json` streaming event-level JSONL covering actions and observations; headless is always
auto-approve. Token and cost reporting UNKNOWN. Version UNKNOWN, sources conflicted between
0.36.1 and 1.11.0.
Sources: docs.openhands.dev/openhands/usage/llms/llms, .../usage/cli/headless

### 10. Goose
**web.** Apache-2.0, Rust, donated by Block to the Agentic AI Foundation in April 2026; repo
moved to `aaif-goose/goose`, docs to goose-docs.ai. `GOOSE_PROVIDER=openai`, `OPENAI_HOST`
(host only, with `OPENAI_BASE_PATH` defaulting to `v1/chat/completions`), `OPENAI_API_KEY`
as a placeholder, `GOOSE_MODEL` for the id. A custom-provider JSON system exists under
`~/.config/goose/custom_providers/`. Headless `goose run -t "instruction"`, `--no-session`,
`--recipe file.yaml --params k=v`, exit-code signalling confirmed. Session transcript format
and token capture UNKNOWN.

**Config wrinkle.** `OPENAI_HOST` plus `OPENAI_BASE_PATH` splits what every other harness
takes as one base URL. Getting `http://127.0.0.1:11299/v1/chat/completions` out of that pair
is a Stage 0 detail to nail down rather than assume.
Sources: goose-docs.ai/docs/getting-started/providers/, .../tutorials/headless-goose/

### 11. Continue CLI
**web.** Apache-2.0, `@continuedev/cli`, binary `cn`, Node 20+. `config.yaml`:
`models: [{ provider: openai, model: <id>, apiBase: <url>, apiKey: <key> }]`, plus `--model`
and `--config` flags. Headless `cn -p "prompt"`, explicitly recommended for CI and no-TTY.
Transcript and token capture UNKNOWN. The repository is archived and read-only, which makes
pinning trivial and the project frozen.
Sources: docs.continue.dev/cli/quickstart, docs.continue.dev/customize/model-providers/top-level/openai

### 12. Cline
**web.** Apache-2.0, npm `cline`, prebuilt platform binaries, shares its agent core with the
VS Code extension. CLI flags: `-P openai-native --apikey ... --modelid ... --baseurl <url>`
(the docs note the base must end at `/v1`, not `/v1/chat/completions`). Headless
`cline "prompt"`, `--yolo` to skip approvals, `--json` for NDJSON. Token metrics in the NDJSON
stream UNKNOWN.
Sources: github.com/cline/cline/blob/main/apps/cli/README.md, docs.cline.bot/provider-config/openai-compatible

### 13. Roo Code
**web. BLOCKED.** Archived read-only on 2026-05-15. The GUI extension has an
"OpenAI Compatible" provider with base URL, key and model fields, but the headless
`@roo-code/cli` path does not: issue #11917, "Roo Code CLI support for OpenAI compatible
endpoints", was opened 2026-03-12, never answered, never implemented, and the repo is now
archived. Our gate requires the non-interactive path itself to reach the endpoint, so it
fails on the half that matters.
Sources: github.com/RooCodeInc/Roo-Code/issues/11917, github.com/RooCodeInc/Roo-Code

### 14. Zero Code
**local, verified on this machine, not web-researched.** Commit
`2b47e4f94b6052c7c76df94c9e27ef454ad97d1e` on branch `zero-code-baseline` in
`/Users/zero/Documents/Dev/zero-code-next`, read-only worktree at
`/Users/zero/Documents/Dev/zero-code-baseline`. MIT-licensed fork of OpenCode v1.18.30,
TypeScript on Bun 1.3.14. Reaches the engine through `@ai-sdk/openai-compatible` speaking
OpenAI Chat Completions natively. Headless:
`bun run --cwd packages/zero src/index.ts run "<prompt>" --agent build`, `--format json`
available. Typecheck clean across five packages and 879 tests passing at the freeze.
Full record in `HARNESS_PROFILES/zero-code-baseline.md`.

### 15. ForgeCode
**web, screening pass only.** `antinomyhq/forge`, forgecode.dev, version ~v2.9.3
(**unverified**), license UNKNOWN. Role-specialised agents built into the core loop rather
than bolted on as optional subagents: `forge` implements, `sage` is read-only research,
`muse` plans into `plans/`. Positions itself as harness-first with schema and tool-call
optimisation and parallel tool execution. `OPENAI_URL` and `OPENAI_API_KEY` point it at any
OpenAI-compatible provider; headless `-p/--prompt`.

**Why it is worth a slot.** Role specialisation inside the loop is architecturally distinct
from everything else on this list, which is the brief's stated bar for adding a candidate.
It is listed as an unqualified candidate because nothing about it has been verified beyond a
single screening pass.
Source: github.com/antinomyhq/forgecode

### 16. Grok Build
**web, screening pass only. BLOCKED PENDING VERIFICATION.** `xai-org/grok-build`,
Apache-2.0, open-sourced July 2026, v1.0 on 2026-08-07. Rust runtime, subagents on by
default, Agent Client Protocol support. Headless `-p/--prompt`, `--prompt-json`,
`--format json`. No custom base URL flag or env var was found: `-m/--model` selects from a
fixed `grok models` list. Fails the gate unless such a mechanism is found.
Source: github.com/xai-org/grok-build

### Considered and rejected in screening
Muse Code (Meta): novel architecture, but bound to a closed hosted model with no custom
endpoint. Antigravity CLI (Google): recent but architecturally similar to harnesses already
covered, local backend support vague. Warp, Amp, Amazon Q Developer CLI, Devin CLI, Factory
Droid: closed or cloud-tethered with no local-model story. SWE-agent, smolagents,
AutoCodeRover: predate the window and are covered conceptually by mini-SWE-agent.

## Qualified arms for Stage 0

Each line is what the Stage 0 runner must produce. `$BASE` is the metering proxy for that
cell, `http://127.0.0.1:11299/v1`, never the engine directly. `$MODEL` is
`ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit`. `$KEY` is a placeholder string.

| Arm | Configuration | Headless command |
|---|---|---|
| zero-code | `provider.mlx.options.baseURL = $BASE` in `~/.config/zero/zero.json` | `bun run --cwd packages/zero src/index.ts run "$TASK" --agent build` |
| opencode | `provider.local.npm=@ai-sdk/openai-compatible`, `options.baseURL=$BASE`, `models.{$MODEL}` | `opencode run --model local/$MODEL "$TASK" --format json` |
| pi | extension calling `pi.registerProvider("local", { baseUrl: $BASE, apiKey: $KEY, api: "openai-completions", models: [{ id: $MODEL }] })` | `pi -p "$TASK" --provider local --model $MODEL --mode json --no-session` |
| codex | `~/.codex/config.toml`: `[model_providers.local] base_url=$BASE, env_key=..., wire_api="responses"`; `model_provider="local"`, `model=$MODEL` | `codex exec --json "$TASK"` |
| claude-code | `ANTHROPIC_BASE_URL` at a Messages-to-Chat-Completions shim in front of `$BASE`, `ANTHROPIC_MODEL=$MODEL` | `claude -p "$TASK" --output-format stream-json --verbose` |
| dsh-standard | `$DSH_HOME/settings.yaml`: `llm-pi-ai.providers.local` with `api: openai-completions`, `baseURL: $BASE`, `models[0].id: $MODEL` | `dsh --profile headless "$TASK"` |
| dsh-minimal | same settings | `sdk-minimal` profile, exact one-shot form to be confirmed |
| qwen-code | `.qwen/.env`: `OPENAI_BASE_URL=$BASE`, `OPENAI_MODEL=$MODEL`, `OPENAI_API_KEY=$KEY` | `qwen -p "$TASK" --output-format json --yolo` |
| aider | `OPENAI_API_BASE=$BASE`, `OPENAI_API_KEY=$KEY` | `aider --message "$TASK" --yes-always --model openai/$MODEL` |
| mini-swe-agent | YAML `model.model_name=$MODEL`, `model_kwargs.custom_llm_provider=openai`, `model_kwargs.api_base=$BASE`, `cost_tracking=ignore_errors` | `mini --exit-immediately -o <traj>` |
| openhands | LiteLLM-style base URL and model settings, exact key names to confirm | `openhands --headless -t "$TASK" --json` |
| goose | `GOOSE_PROVIDER=openai`, `OPENAI_HOST=http://127.0.0.1:11299`, `OPENAI_BASE_PATH=v1/chat/completions`, `GOOSE_MODEL=$MODEL`, `OPENAI_API_KEY=$KEY` | `goose run --no-session -t "$TASK"` |
| continue-cli | `config.yaml` model entry with `provider: openai`, `apiBase: $BASE`, `model: $MODEL` | `cn -p "$TASK"` |
| cline | flags carry the config | `cline -P openai-native --apikey $KEY --modelid $MODEL --baseurl $BASE --yolo --json "$TASK"` |

Excluded from Stage 0: Roo Code (blocked), Grok Build (blocked pending verification),
DeepSeek Harness Code (selection mechanism unconfirmed) and Creator (out of scope),
ForgeCode (candidate, not yet qualified).

Fourteen arms is already above what the compute budget can screen at one trial per cell
across six tasks. Stage 0 exists partly to cut it: arms that cannot be made to reach the
model cleanly, or whose translation layer visibly mangles tool calls, drop out before
Stage 1 rather than consuming replication budget.
