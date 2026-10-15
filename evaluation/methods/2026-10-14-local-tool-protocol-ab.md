# Same-model tool protocol A/B for local coding harnesses

**Status: protocol only; no inference runs claimed.** Companion to [the 2026-10-14 research note](../../literature/2026-10-14-local-tool-protocol-reliability.md). Run this experiment against the *exact local checkpoint and engine*. Never compare model A on engine X to model B on engine Y and call the difference a harness effect.

## Hypotheses (all untested locally)

- H1: For a model whose native call syntax conflicts with the configured parser, correcting the parser increases **extraction recall**, particularly among generations visibly containing native tool-call text.
- H2: Strict argument validation improves **executable validity** but cannot, by itself, guarantee the model chose a needed tool.
- H3: Serializing dependent actions reduces wrong ordering; parallel independent reads may reduce wall time without compromising state.
- H4: Bounded schema-specific feedback repairs some invalid calls at lower repeated-side-effect risk than blind retries.

## Experimental units and paired cells

Use *task×seed×configuration* as the unit. Fix input, tool schema, prior transcript, model hash, engine version, context window, decoding parameters, prompt template, machine load and evaluator. For every task create a baseline and intervention arm, interleave run order to reduce thermal/engine drift; do not present a fixed-seed run as deterministic unless engine confirms determinism.

| Cell | Constructed situation | Ground-truth requirement |
|---|---|---|
| C1 | Need exactly one read tool | right tool and valid argument |
| C2 | Question answerable without tools | abstain; no false call |
| C3 | Similar tools with overlapping names | choose correct tool |
| C4 | Required/optional/enum arguments | exact schema validity and semantic values |
| C5 | Two independent reads | both answers, correct call/result IDs |
| C6 | Edit followed by verification read | write precedes dependent read |
| C7 | Tool runtime error and suggested recovery | eventually correct external state |
| C8 | Tool-output truncation/stop boundary | never execute incomplete calls |
| C9 | Response stream interruption/reconnection | no duplicate non-idempotent action |
| C10 | Two-stage tools then structured final answer | necessary tool invoked AND final schema valid |

Prepare 10–20 *held-out* variants per cell using fixed repositories and external oracles. Start with 3 seeds per task/arm to diagnose instrumentation, then expand sample size using observed paired variance. Never claim a universal minimum run count before calculating uncertainty.

## Intervention matrix

1. **Baseline**: official checkpoint/template/parser tuple, auto tool choice, native call format, no forced parallel calls, no repair.
2. **Negative control**: intentionally mismatched parser in a disposable read-only fixture. Do not allow writes; this tests instrumentation and should not be scored as a deployment candidate.
3. **Tool-envelope constraint**: only where documented/supported by the exact engine and parser.
4. **Whole-response grammar**: if the server can actually combine it with tool calling; test for suppressed needed calls. A two-pass tools-then-final-schema arm isolates final answer formatting.
5. **Serial vs parallel**: only on C5/C6 with conflict-free/read-only tools unless external state isolation is guaranteed.
6. **Recovery policy**: no repair, one typed schema error + regeneration, or one blind retry; enforce identical wall/token budgets and measure duplicate effects.

Make parser × format × strictness a *controlled* matrix, not a full Cartesian product of unsupported combinations.

## Instrumentation schema (one JSON record per attempt)

```json
{
  "run_id": "example-id",
  "task_id": "C5-001",
  "arm": "baseline",
  "seed": 101,
  "model_hash": "<hash>",
  "tokenizer_hash": "<hash>",
  "template_hash": "<hash>",
  "engine_commit": "<sha>",
  "tool_parser": "<name>",
  "reasoning_parser": "<name>",
  "tool_choice": "auto",
  "parallel_tool_calls": false,
  "raw_capture_available": false,
  "raw_native_call_count": null,
  "normalized_call_count": 0,
  "valid_call_count": 0,
  "admitted_call_count": 0,
  "settled_call_count": 0,
  "duplicate_effect_count": 0,
  "needed_call": true,
  "correct_tool_selected": false,
  "final_state_pass": false,
  "output_schema_pass": null,
  "prompt_tokens": null,
  "completion_tokens": null,
  "cache_hit_tokens": null,
  "wall_ms": null,
  "parser_error": null,
  "tool_error": null,
  "environment": {"hardware": "", "power_state": "", "memory_pressure": ""}
}
```

Use null for unknown/unavailable, not zero. Raw text may contain secrets; keep raw traces locally with redaction and defined retention. Log engine/tokenizer/template revisions rather than only the model display name.

## Metrics and denominator rules

- **Needed-tool recall** = tasks with a necessary correct tool invocation / all tasks where the oracle requires it. Denominator must include turns with zero extracted calls.
- **No-tool specificity** = tasks without spurious dispatch / all C2 tasks.
- **Extraction recall** = correctly parsed native calls / observable raw native calls; report **N/A** when the engine exposes no raw capture and never infer model intent from a parsed absence.
- **Syntax validity** = parsed JSON argument objects / normalized calls. **Semantic validity** uses external expected name/argument values, not just JSON schema.
- **Admission integrity** = invalid or unauthorized calls blocked / all injected invalid or unauthorized calls.
- **Exactly-once effects** = effectful tasks with correct number of observed side effects / all attempted effectful tasks, including reconnect/crash cases.
- **Task completion** = external final-state oracle passed / all attempted tasks, counting timeouts and agent-aborted trials as failures.
- **Efficiency** = external passes / wall-hour, plus tokens per passed task; report unsolved trials in cost numerator to avoid survivorship bias.
- **Operational** = cache hits, TTFT, generation rate, calls/turn, attempts/call, loop rate, unsupported-mode errors and engine stalls.

Report paired per-task differences with bootstrap confidence intervals stratified by task; publish both absolute rate and delta. A green-format-metric with lower required-tool recall or worse external task success is not a win.

## Diagnostic decision tree

1. **Wrong model hash or incompatible parser?** Fix tuple; re-run C1/C2 read-only smoke.
2. **Native call seen but absent from structured response?** Investigate parser/stream/stop-token boundary before blaming model.
3. **Structured call exists, wrong tool selected?** Test schema descriptions and disambiguation, not just parser.
4. **Schema-valid call, wrong external state?** Evaluate semantics, dependency ordering and verifier.
5. **Duplicate side effects?** Fix durable admission/idempotent reconciliation before enabling retries or concurrent writes.
6. **Final answer valid but required tool not called?** Separate schema-constrained final formatting from tool-action turn.

## Go/no-go criteria

A candidate can advance only if it is supported by the actual engine, has no increase in unauthorized or duplicate side effects, retains at least baseline no-tool specificity, does not reduce held-out external completion outside uncertainty, and improves extraction/semantic accuracy or wall-hour efficiency on a clearly identified workload. If confidence intervals overlap materially, retain the simpler baseline. Any safety/integrity regression is a hard fail regardless of speed.

## References

- [Tool protocol research note](../../literature/2026-10-14-local-tool-protocol-reliability.md) with pinned upstream documentation.
- [Existing same-model comparison protocol](2026-09-16-harness-comparison.md).
- [Existing all-around leaderboard protocol](2026-09-17-all-around-leaderboard.md).
- [BFCL executable tests](https://github.com/ShishirPatil/gorilla/tree/6ea57973c7a6097fd7c5915698c54c17c5b1b6c8/berkeley-function-call-leaderboard).
