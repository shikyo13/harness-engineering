# Qwen 3.8 Flash Next in Zero Code against Sonnet 5.5: a difficulty ladder and the harness changes it led to (2026-10-09/10)

Host: Apple M4 Max, 128 GB. Local model: `ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit` on mlx-serve 26.10.1
(decode-end build), driven by Zero Code through `packages/zero-eval`. Reference: Sonnet 5.5 as Claude Code subagents on
cleanroom copies of the same fixtures (`packages/zero-eval/src/cleanroom.ts`). Raw runs, blind reviews, screenshots and
the run ledger: `~/DevData/Scratch/Codex/2026-10-09/ladder/` (`LEDGER.md`, `review/*-SUMMARY.md`).

## How runs were judged

Hidden tests decide pass or fail. Quality is judged separately: an Opus reviewer compares two builds labelled A and B in
random order, scores them 1-5 on design, states, phone layout, function (including the saved data), accessibility, dark
theme, code and tests, and every headline claim is re-checked by hand before it is reported. One reviewer's scores of
the same build differed by up to 4 of 40 points, so single comparisons near that margin are not conclusive.

## Findings

1. Hidden tests tied on every task (tiers 2 to 4, Mealbook), yet the blind review preferred Sonnet on every task, and
   the gap grew with size: tier 3 34 vs 20 of 35, tier 4 38 vs 20 of 40, Mealbook 40 vs 27 of 40.
2. Qwen follows explicit specs well. On `long-csv-sum`, whose README spells out every edge case, all six Qwen builds were
   perfect. Its bugs are unstated pitfalls: a damaged data file replaced by an empty one, user keys matching inherited
   names (`constructor`, `__proto__`), `process.exit()` cutting off piped output, local and UTC date methods mixed
   (a week skipped at the daylight-saving change), data dropped when a follow-up added state (a recipe delete that wiped
   the pantry), and layout choices that bury the main view.
3. Time goes to generation: 80-86% of a run is decode, of which reasoning is about 70% on tier 3 and 45% on Mealbook.
   Zero Code's own overhead is under 1%, and Qwen rarely rewrites whole files (0-6% of file output). On battery the
   Mac throttles and decode halves (45-80 to 21-29 tok/s).
4. Low reasoning effort is not a speed lever here: on Mealbook it was slower (87.6 vs 69.2 min) and worse (26 vs 30).

## Harness changes and their measured effect

| change | commit | effect |
|---|---|---|
| warm up the configured model; keep a replacement for a hand-started engine running | `b8743ccc35` | removed a 27B load per warm-up; no engine crash in about 4 h of runs since |
| design review: a fresh read-only subagent uses the app in a browser before a UI turn ends | `41020a765f`, `69a036af6f`, `8f51db298e`, `d1a608f481` | tier 4: blind 29 vs 22 against the run without it; Mealbook: +3 and +6 in two samples; about 2x the time (69 vs 32 min) |
| a blank subagent reply is reported as such | `af16ceb8bf` | a failed review no longer reads as a clean one |
| a subagent's own variant applies; hidden subagents are not offered to the model | `7bdb7f15a8`, `5be551e04d` | one automatic review per request instead of three |
| robustness instructions (eval-only so far, `--config instructions`) | not in the default prompt | kv-cli: 14/14 + 4/4 in 3 of 3 runs against 13-14/14 + 2-3/4, about 2.5x the time; no gain on the explicit CSV spec, 1.6x the time |
| pitfall hints after JS/TS edits | `9223351065` | flags the four recurring patterns at no model cost; effect not yet measured |
| eval: `--config`, power source in the report, held-out fixtures `ladder-t4-meals`, `long-csv-sum` | `b800256c41`, `7debaf6c31`, `38e0a55f0c`, `58c300cea6` | A/B runs and comparable timings |

## Open

- Measure the pitfall hints (kv-cli x3, tier 3) and the robustness instructions on tier 3 on AC power.
- Decide whether the robustness instructions belong in the default prompt given their time cost.
- Repeat trials per tier; mobile-app and game-mod tasks for the design track.
