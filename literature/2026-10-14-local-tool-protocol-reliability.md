# Tool-call transport reliability for local coding agents

**Research note — 2026-10-14. Status: source-verified synthesis; proposed interventions UNMEASURED on Zero Code.**

**Question.** When a local coding agent misses, mangles or mishandles a tool invocation, how much of the loss belongs to the language model, the inference server, the wire format, or the harness—and which interventions should we test first?

**Relation to existing work.** [The provider/model-table study](../studies/2026-10-14-subsystems/providers-models.md) covers configuration, retries and engine lifecycle; [tools/editing](../studies/2026-10-14-subsystems/tools-editing.md) covers tool selection, edit application and output shaping. [Proven advances](2026-10-14-proven-advances.md) mentions JSON-schema/tool suppression and flags parallel open-model tool calls as unmeasured. This note addresses the *boundary between model generation and executable tool effects*: format compatibility, streamed extraction, execution authorization, result correlation and causal measurement. It does not claim to have run inference on the owner's hardware.

## Executive finding

**The actionable unit of compatibility is not a model family; it is a versioned tuple (weights/checkpoint, tokenizer + tool-use chat template, inference server + version, reasoning parser, tool-call parser, decoding mode and request options).** This follows directly from the contradictory-looking but individually valid primary-source recipes below. Treat it as a **design inference**, not a measured performance improvement.

A useful failure decomposition has five gates:

1. **Intent** — did the model choose a tool when a tool was needed, or abstain when one was not? Requires a semantic oracle.
2. **Emission** — did the model produce a complete call in its *native* token/text format? Requires raw pre-parser output or a faithfully captured stream.
3. **Extraction** — did the server recover `tool_calls`, including arguments, call IDs and finish reason? Requires a side-by-side raw vs normalized trace.
4. **Admission** — did the harness validate name, schema, arguments, permission, freshness, duplication and dependencies *before* execution? Requires dispatch telemetry.
5. **Settlement** — did the harness apply the right result to the right call exactly once, with an error surfaced to the model? Requires a persisted call/result ledger.

A flat “tool success rate” obscures these gates. Better model prompting cannot repair a mismatched server parser; better JSON validation cannot tell whether a necessary call was suppressed.

## 1. Source-backed implementation facts

### F1 — Native format depends on the exact checkpoint (source-verified; inference about risk)

- Qwen's general [Qwen3 function-calling guide](https://github.com/QwenLM/Qwen3/blob/7a2f61ffc7a20d47efcd2bf97f6f2bf52729042e/docs/source/framework/function_call.md#L43-L47) recommends **Hermes-style** tool use for Qwen3. Its vLLM example pairs Qwen3-8B with `--tool-call-parser hermes` ([lines 450–457](https://github.com/QwenLM/Qwen3/blob/7a2f61ffc7a20d47efcd2bf97f6f2bf52729042e/docs/source/framework/function_call.md#L450-L457)); the guide explicitly warns that malformed, unparseable calls can occur ([lines 525–532](https://github.com/QwenLM/Qwen3/blob/7a2f61ffc7a20d47efcd2bf97f6f2bf52729042e/docs/source/framework/function_call.md#L525-L532)).
- But SGLang's **Qwen3.8-27B** recipe requires `--reasoning-parser qwen3 --tool-call-parser qwen3_coder`; that checkpoint emits an inner `<function=…>` / `<parameter=…>` structure inside `<tool_call>`, whereas the Hermes parser expects JSON there. With the wrong parser the harness may see raw text instead of structured tool calls ([SGLang source, lines 348–367](https://github.com/sgl-project/sglang/blob/0cf683ea8d27041da981f7875e1755b9f7772799/docs/cookbook/autoregressive/Qwen/Qwen3.8-27B.mdx#L348-L367)).
- **Implication:** do not silently inherit “Qwen → Hermes” from a generic registry. An explicit, per-checkpoint compatibility manifest and round-trip smoke test should override family defaults. These examples involve different checkpoint generations and servers, so they are not an inconsistency within either implementation.

### F2 — Server format constraints and action-selection constraints are different (source-verified)

- vLLM distinguishes *whether a call is required*, *whether a structural envelope is constrained*, and *whether its individual argument schema is constrained*. In the pinned documentation, `auto` leaves tool use optional; the strictness of one tool does not necessarily constrain every other tool's arguments, and `--tool-strict-level` can set a server-wide floor ([vLLM lines 112–157](https://github.com/vllm-project/vllm/blob/5d5c3d7c4360d777a4bb9c75b6be250dc8d812d1/docs/features/tool_calling.md#L112-L157)).
- A separate, paper-reported mechanism is **tool suppression** under simultaneous tool calling and whole-response JSON-schema generation: grammar masks can render the tool-opening token unreachable. The authors propose doing tool execution before a schema-constrained final response ([Constraint Tax, arXiv:2606.25605](https://arxiv.org/abs/2606.25605); research preprint, not reproduced here).
- **Important distinction:** vLLM's *tool-envelope-aware* constraint facility and a generic *whole-response JSON grammar* are not equivalent. It would be incorrect to infer that every strict-tool mode suppresses tools. Test the actual grammar and route, and log schema conformance **and** needed-tool recall, not schema compliance alone.

### F3 — A syntactically emitted call can disappear during extraction (source-verified)

- vLLM's documented Hugging Face response-template parser emits a streamed tool call once a complete region parses. If an incomplete call fails to parse, it is dropped; that parser also has limitations on constrained/named/parallel modes ([vLLM lines 189–197](https://github.com/vllm-project/vllm/blob/5d5c3d7c4360d777a4bb9c75b6be250dc8d812d1/docs/features/tool_calling.md#L189-L197)). **Scope:** this is specifically the documented `hf` response-template parser, not proof that every vLLM parser behaves identically.
- llama.cpp explicitly distinguishes *native* and *generic* handlers, notes the generic fallback may cost more tokens, and says that multiple/parallel calls are available only on some models and are disabled by default unless `"parallel_tool_calls": true` is passed ([llama.cpp lines 1–25](https://github.com/ggml-org/llama.cpp/blob/a0e6dc7384edb8f5d927155e1d26c10b4a310e6d/docs/function-calling.md#L1-L25)).
- **Implication:** the absence of `tool_calls` in the OpenAI-compatible message does **not** prove that the model never attempted one. Preserve a local, redacted raw-output trace at least during experiments. Do **not** execute free-form raw text on the assumption that it was a suppressed call; record it as a diagnostic and prefer a safe corrective turn.

### F4 — Tool formatting has an empirical effect, but no single universal winner (measured in external study)

- Berkeley's [BFCL V4 format-sensitivity study](https://gorilla.cs.berkeley.edu/blogs/17_bfcl_v4_prompt_variation.html) varied return formats, function-description formats, tool tags, prompt presentation and style across 39 models. The reported aggregate trend favors Python/JSON return formats over XML variants, especially for smaller models.
- That observation is **not** a prescription to override the native `qwen3_coder` checkpoint format with generic JSON: BFCL varies *prompted* formats in its study and does not establish that a non-native template will outperform a compatible native one for Qwen3.8 in SGLang.
- **Implication:** measure the exact on-device format/template/parser combination. A model's documented native format should be the baseline; consider a custom format only after an apples-to-apples A/B that includes no-tool and multi-turn cells.

### F5 — Stateful and multi-call correctness are separate from formatting (external benchmarks)

- [Berkeley BFCL](https://github.com/ShishirPatil/gorilla/blob/6ea57973c7a6097fd7c5915698c54c17c5b1b6c8/berkeley-function-call-leaderboard/README.md#L33-L44) explicitly spans simple, parallel, multiple and multi-turn function calls, and supports targeted case IDs for local evaluation ([lines 163–199](https://github.com/ShishirPatil/gorilla/blob/6ea57973c7a6097fd7c5915698c54c17c5b1b6c8/berkeley-function-call-leaderboard/README.md#L163-L199)). It can detect model-level call/argument errors.
- [ToolSandbox](https://aclanthology.org/2025.findings-naacl.65/) includes stateful execution and implicit dependencies, evaluated at intermediate and final milestones. A perfect parser can still execute the wrong sequence of writes or supply a result against the wrong prior state.
- **Implication:** a coding-harness probe needs both *call-level* scoring and *final-state* scoring; BFCL alone is not a coding-task completion benchmark.

## 2. Proposed architecture (original synthesis; not yet validated)

### 2.1 Versioned compatibility profile

Store one immutable-at-session-start profile, with provenance to its calibration run:

~~~yaml
profile_id: qwen38-27b-sglang-tool-v1
model_checkpoint: "<exact model revision / hash>"
quantization_or_adapter: "<artifact hash>"
tokenizer_revision: "<hash>"
chat_template_hash: "<hash>"
engine: "sglang"
engine_revision: "<commit or container digest>"
tool_call_parser: "qwen3_coder"
reasoning_parser: "qwen3"
tool_choice_mode: "auto"
parallel_tool_calls: false
whole_response_json_grammar: false
thinking_mode: "<explicit setting>"
calibration_id: "<saved fixture result identifier>"
~~~

**Behavior:** reject or warn on an uncalibrated tuple; never silently switch the parser when a model changes. Keep the original unmodified provider payload and any local-engine raw-generation capture *only where the engine exposes it*. An OpenAI-compatible response alone may be insufficient to reconstruct raw tokens.

### 2.2 Canonical call ledger and admission gate

Normalize to one internal record *after* server extraction but *before* tool dispatch:

~~~text
Call {run_id, turn_id, call_id, index, source_generation_id,
      tool_name, arguments_json, schema_version, emitted_at,
      status, effect_class, dependency_ids, idempotency_key}
status: proposed → validated → admitted → executing → settled
                         ↘ rejected(error)
~~~

Required invariants:

1. A `call_id` is unique within a turn; every tool-result part references an admitted call ID. Missing or duplicate IDs are protocol failures, not new tasks.
2. Validate JSON parsing, exact tool name, schema, workspace scope and permissions before dispatch. Distinguish syntax error, unknown tool, wrong arguments, authorization denial and runtime failure.
3. For *read-only independent* calls, parallel execution is eligible after validation. For mutating calls, or a read dependent on a prior write, serialize or require a proven conflict-free effect graph. This is an engineering safety policy, **not** an experimentally proven speed optimum.
4. Persist admission and terminal result, keyed by run+turn+call ID; do not replay a non-idempotent call merely because the provider streamed twice, a connection dropped, or the process restarted. For a crash between side effect and commit, report an **unknown outcome** and reconcile state instead of blindly retrying.
5. On a model/provider parse failure, avoid speculative execution. Supply a bounded, model-visible correction only when enough evidence exists; otherwise re-prompt for the intended action or surface an actionable error.
6. Completion is gated on settlement of every admitted call (or an explicit recorded cancellation/unknown outcome), not on the presence of a natural-language final answer.

The ledger complements existing work on durable admission in [agent-kernel](../studies/2026-10-14-subsystems/agent-kernel.md); it is specifically **per tool call**, not a replacement for the session WAL. A real implementation must define idempotency at the tool boundary: an idempotency key is insufficient unless the callee honors it or the harness can reconcile effects.

### 2.3 Failure-triggered repair rather than universal retry

| Fault class | Observable evidence | Safe first intervention | Avoid |
|---|---|---|---|
| Wrong parser/template | raw native call present; normalized `tool_calls` absent | fix/checkpoint-pin parser and rerun pre-dispatch calibration | changing system prompt first |
| Incomplete emitted call | raw region truncated, stop/length signal, parser diagnostic | one targeted continuation/re-generation within budget | executing truncated arguments |
| Valid JSON, wrong semantics | callable parsed, wrong tool/argument according to oracle | clarify tool contract or require prerequisite observation | claiming schema constraints solve semantics |
| Invalid argument schema | parser succeeded; local schema validation failed | bounded error with expected schema + offending fields | broad blind retries |
| Duplicate call after transport retry | repeated logical call ID/operation | dedupe or reconcile effect state | rerunning writes |
| Missing tool result | admitted call has no terminal settlement | settle from persisted job state or record unknown | fabricating a successful tool result |
| Unneeded tool use | no-tool oracle; model called anyway | tool-choice/abstention calibration | forcing `required` globally |
| Whole-response grammar suppression | needed call disappears only with response JSON grammar | decouple action turn and constrained final formatting | assuming tool API failure |

**Cost principle:** a repair that improves format validity but increases false invocations or wall-time per solved task is not automatically a win. The relevant optimization target is **verified completions per wall-clock hour**, with safety and task quality as guardrails.

## 3. Priority experiments

Full executable protocol in [tool-protocol A/B](../evaluation/methods/2026-10-14-local-tool-protocol-ab.md).

**E1 (first): parser/template round trip.** For the exact Qwen3.8 checkpoint, use the vendor-documented SGLang tuple as the baseline. Run one tool needed, no tool needed, malformed-argument opportunity, and two independent tools. Record *raw native call existence → extracted call → schema-valid call → correct execution*. An intentionally mismatched parser is a **negative control**, not a candidate improvement.

**E2: streaming parity.** Run matched cases streaming on/off, truncation/stop pressure and output-token caps. Question: are failures generated by the model or introduced by incomplete server-side extraction? Do not assume token-identical outputs across streaming modes.

**E3: constraints interaction.** Compare unconstrained `auto`, supported native *tool-envelope* strictness, whole-response JSON grammar (if supported), and two-pass tools-then-final-JSON. Keep format validity **and** required-tool recall. Verify what strictness modes the server actually implements before enabling an arm.

**E4: parallel call policy.** Compare forced serial vs permitted parallel read-only calls (plus a dependency-sensitive case). Optimize elapsed time but fail the arm if final-state oracle, exactly-once settlement or tool/result pairing regresses.

**E5: recovery.** Compare no repair, schema-specific bounded repair, and fixed retry once, on realistic invalid-input/error fixtures. Count extra model tokens, repeats of writes, loop rate and successful final-state resolution.

**Order:** E1 and E2 first. Only after transport/extraction are clean can a “model is poor at tool use” claim be meaningfully interpreted. The existing repository's same-model, fixed-seed evaluation protocol remains the control design.

## 4. Conclusions and limits

**Defensible now:**
- Format/parser compatibility is a real engineering failure mode, documented in upstream recipes.
- Structured tool calling involves distinguishable decisions about obligation, envelope and arguments; not all grammar modes mean the same thing.
- Multi-turn, no-tool and stateful evaluation are required to measure more than JSON well-formedness.

**Not established here:** a quantitative gain for Zero Code; a general ranking of vLLM vs SGLang vs llama.cpp; that native formatting always beats custom formatting; that sequential tool execution always helps; or a new result for Qwen3.8. Those require the controlled runs above.

**Predicted, falsifiable outcome:** correcting mismatched template/parser tuples will most strongly improve *extraction recall* where raw calls already exist; schema-specific repair should improve *execution validity* without improving *intent accuracy*; the effect of parallel execution should depend on task dependency structure. If that partition does not appear in traces, revisit the five-gate taxonomy.

## Primary sources and verification

Source reads on 2026-10-14. GitHub documents are pinned to commits read; these are docs claims, not a new audit of server internals.

| Source | Exact provenance | Role |
|---|---|---|
| SGLang Qwen3.8-27B recipe | [sglang `0cf683ea`, docs/cookbook/autoregressive/Qwen/Qwen3.8-27B.mdx L348–367](https://github.com/sgl-project/sglang/blob/0cf683ea8d27041da981f7875e1755b9f7772799/docs/cookbook/autoregressive/Qwen/Qwen3.8-27B.mdx#L348-L367) | model-specific parser mismatch |
| Qwen3 tool-use guide | [Qwen3 `7a2f61f`, docs/source/framework/function_call.md L43–47, L450–457, L525–532](https://github.com/QwenLM/Qwen3/blob/7a2f61ffc7a20d47efcd2bf97f6f2bf52729042e/docs/source/framework/function_call.md) | Hermes baseline for another checkpoint |
| vLLM tool calling | [vLLM `5d5c3d7`, docs/features/tool_calling.md L112–157 and L189–200](https://github.com/vllm-project/vllm/blob/5d5c3d7c4360d777a4bb9c75b6be250dc8d812d1/docs/features/tool_calling.md) | obligation/envelope/schema; streaming-parser scope |
| llama.cpp function calling | [llama.cpp `a0e6dc7`, docs/function-calling.md L1–25](https://github.com/ggml-org/llama.cpp/blob/a0e6dc7384edb8f5d927155e1d26c10b4a310e6d/docs/function-calling.md#L1-L25) | native/generic handlers, parallel request flag |
| Berkeley BFCL | [Gorilla `6ea5797`, README L33–44, L163–199](https://github.com/ShishirPatil/gorilla/blob/6ea57973c7a6097fd7c5915698c54c17c5b1b6c8/berkeley-function-call-leaderboard/README.md) and [format sensitivity study](https://gorilla.cs.berkeley.edu/blogs/17_bfcl_v4_prompt_variation.html) | format sensitivity, controlled call-level comparisons |
| ToolSandbox | [Lu et al., NAACL Findings 2025](https://aclanthology.org/2025.findings-naacl.65/) | stateful and dependent evaluation |
| Constraint Tax | [Li et al., arXiv:2606.25605](https://arxiv.org/abs/2606.25605) | joint JSON grammar/tool suppression, two-pass idea; preprint |

**License/scope:** Research summary and novel proposals only. No third-party code reproduced or vendored. External benchmark values are intentionally not transported into a local performance claim.
