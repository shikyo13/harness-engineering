# Improvement priorities, 2026-09-29

What to change next in Zero Code for long unattended tasks, ranked by measured cost, each with
the evidence, the change, and how to tell whether it worked. Read with
`SYNTHESIS-2026-09-29.md`, which covers everything measured in `docs/evals/` up to today.

## Fixed today

**A long session died on "This prompt does not fit in GPU memory".** The engine sends that
refusal as an ordinary SSE data line inside a 200 response, carrying a flat OpenAI error object.
The stream-error parser only knew the `{ type: "error", error }` shape, so it fell through to a
fatal `UnknownError`. Now it is classified as context overflow, which the session already
recovers from by compacting and carrying on; compaction retries a refused summary with half the
history it last sent, up to three times; and if memory stays short through all of that, the
session stops in a state that resumes with its full history. `30e8ed8873`, `fa2796497e`.

**Why the engine refused.** 162,265 tokens needed ~14.9 GB with 12 GB free, although 161,023 of
them were already cached. An uncommitted `serve.sh` change had raised `--prefill-chunk` from 4096
to 8192, replacing the comment that said not to. The KV was ~5.4 GB, so ~9.5 GB of the estimate
was chunk working memory, roughly half of which 4096 gives back. The working tree's default is
back to 4096 with the measured reason; the running engine picks it up on its next start.

**Every session-ending error on record** (`zero-main.db`): 31 user aborts, 3 five-minute timeouts
(09-17 to 09-22, since removed), 1 memory refusal (fixed above), 0 connection failures.

## A correction

The 2026-09-19 handover called "~3.4 s per turn between deciding to call the model and the first
token" harness overhead and the best lead for making Zero Code faster. It is not harness overhead.
Across 2,021 requests, request preparation is 1 ms at the median and 6 ms at p90, context assembly
1 ms, and the first stream event arrives 172 ms after the turn starts. The time is the engine
restoring cached prefixes from its SSD tier before prefill, which `prompt_ms` does not count: 512
restores, p50 108 ms, p90 1.1 s, worst 36 s. That is engine tuning, not Zero Code code.

## 1. Compaction is the largest avoidable cost in a long run

Codex's long-horizon runs spent 7 to 21 minutes of 48- to 90-minute runs compacting. Across 83
compaction requests in the engine log the median is **108 s**: 42 s of prefill and **63 s of
decode for a median of 2,262 generated tokens**.

### 1a. Summaries think with no limit

Compaction uses the model's first variant, `low`, as its lightest. On this engine that is backwards.
`medium` is capped: the engine enforces an 8,192-token thinking budget in-stream. `low` only
injects a sentence into the template ("Reasoning effort is set to low. Keep your thinking brief
and focused") and sets no budget, and the engine's default budget is unlimited
(`--reasoning-budget`, "default: unlimited"). The one lever meant to make compaction light is the
only effort level without a hard cap.

- **Change:** turn thinking off for the summary request on a local engine, the same way titles
  already are (`c52fd56cab`). Earlier user messages are carried across compaction verbatim
  (`8b3c9b6f1a`), so requirement recall does not depend on the summary's reasoning.
- **Verify:** the inventory long-horizon task, which compacts 3 to 11 times. Compare total
  compaction time and the recall probe (currently 12/12) against the current arm, with at least
  two trials each; the synthesis shows one trial cannot separate close arms.
- **Expected:** most of the 63 s decode, if the summary itself is on the order of 1,000 tokens.
  Unmeasured until then.

### 1b. Summary requests get no prefix-cache reuse

| Request type | n | Median reuse | Cold (<10%) |
| --- | --- | --- | --- |
| Agent turns | 5,692 | 98% | 578 |
| Summary-shaped | 128 | 0% | 93 |

Compaction re-serializes the history into a new prompt, so the engine prefills it from scratch.
Making the summary a turn of the loop, appended to the prompt the session already sent, would make
nearly all of it a cache hit.

- **Ceiling:** prefill is 43% of compaction time, and reused prompts decode against the whole
  session context, which is slower. Expect well under 40%. Do 1a first and measure again.
- **Cost:** turn assembly lives inside the prompt loop (frozen system prefix, message cache,
  reminders), so this restructures code that changed heavily in the last ten days.

## 2. The benchmark does not cover the regime that failed

All 14 long-horizon runs used a 64k window. The session that died was at 162k on the full 262k
window, where cold prefill, compaction inputs and memory pressure are all larger.

- **Change:** add a long-horizon arm at the full window. The test LLM server can now refuse a
  request the way the engine does (`llm.refuse`), so the recovery path can be exercised in the
  benchmark rather than waiting for real memory pressure.

## 3. Memory headroom on the engine

KV costs about 26 KB per token with `--kv-quant off` (bf16), measured from the engine's own cache
billing (149,140 tokens = 4,929 MB including 1,010 MB of SSM state). 8-bit KV would roughly halve
that and double the context the machine can admit.

- **Change:** only with a quality measurement. KV quantization changes the model's numerics, and
  every result in the harness study assumes the current setting. Recorded, not recommended blind.

## 4. From the synthesis, ranked there by impact on unattended runs

- **False "done".** The same arm passed in 73 minutes and failed in 88; hidden checks missed;
  verification claimed and not matching output. Unattended, nobody feeds the failure back.
- **No progress or completion detection.** `webcore` ran to the 900 s limit twice without
  terminating; stale-observation loops in computer use; the executive that would catch this is
  advisory and unmeasured.
- **Cold prefill on restart.** 235 to 324 s at 90k. A restarted engine or session re-pays it.

## Engine notes that are not Zero Code changes

- The SSD prefix-cache tier is the time-to-first-token tail (p90 1.1 s, worst 36 s). Keeping the
  active session's prefix resident, and not letting auxiliary requests with other prefixes evict
  it, is the lever.
- The admission check evicted a stale 4.3 GB snapshot to admit the 162k request, yet available
  memory rose only from 11.8 to 12.0 GB. Freed buffers did not come back before the re-check.
  Engine behaviour worth reporting upstream.
