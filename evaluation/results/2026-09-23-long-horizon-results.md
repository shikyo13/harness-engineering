# Long-horizon benchmark results (2026-09-22/23)

Three long tasks from `packages/zero-eval/fixtures`, one trial per arm, run back to back on the
local engine (mlx-serve, `mlx/qwen3.8-flash`, Apple M4 Max) with the context window capped at
64k (`--context 65536`, about 57k usable). The user's 64 MCP tools were connected in every run,
about 25k tokens of schema, so roughly half of the usable window was spoken for before any
conversation. Each task has a 90 minute limit, a hidden verifier, and a recall probe asked after
verification ("list every requirement from my first message").

| Arm | Code |
|---|---|
| baseline | `8c1fb77e16`, before the long-horizon work |
| after | `c1049454ec`; from the after live-bugs run on, plus the new-message replay fix (`632e803b5d`) |
| tool search | after, with `ZERO_EXPERIMENTAL_TOOL_SEARCH=true` |
| repo map | after, with `ZERO_EXPERIMENTAL_REPO_MAP=true` |

The two inventory arms that ran before the replay fix were run again with it ("rerun" rows).
Recall is scored from the model's own reply to the probe (see "Scoring fix" below).

## Results

| task | arm | result | wall | compactions | compaction time | tool calls | reads (repeats) | edits | recall |
|---|---|---|---|---|---|---|---|---|---|
| inventory | baseline | timeout | 90 min | 9 | ~27 min | 130 | 100 (75) | 22 | not reached |
| inventory | after | pass | 73 min | 7 | 12 min | 138 | 50 (20) | 57 | 12/12 |
| inventory | after, rerun | fail (2 checks) | 88 min | 10 | 21 min | 162 | 61 (29) | 68 | 12/12 |
| inventory | tool search | pass | 65 min | 4 | 9 min | 148 | 46 (8) | 66 | 12/12 |
| inventory | tool search, rerun | pass | 48 min | 3 | 7 min | 120 | 41 (17) | 56 | 12/12 |
| inventory | repo map | timeout (1 check) | 90 min | 11 | 20 min | 168 | 32 (10) | 85 | not reached |
| live-bugs | baseline | pass | 47 min | 3 | | 63 | 9 (1) | 14 | 9/11 |
| live-bugs | after | pass | 20 min | 1 | 1 min | 36 | 10 (1) | 7 | 11/11 |
| live-bugs | tool search | pass | 24 min | 1 | 1 min | 48 | 15 (1) | 16 | 11/11 |
| live-bugs | repo map | pass | 45 min | 3 | 4 min | 70 | 14 (2) | 10 | 11/11 |
| qa-spec | baseline | fail (1 check) | 33 min | 4 | | 114 | 31 (8) | 20 | 8/10 |
| qa-spec | after | pass | 32 min | 3 | 4 min | 65 | 15 (1) | 22 | 10/10 |
| qa-spec | tool search | pass | 14 min | 1 | 1 min | 58 | 26 (3) | 20 | 10/10 |
| qa-spec | repo map | pass | 19 min | 2 | 2 min | 43 | 2 (0) | 18 | 10/10 |

Baseline compaction time comes from the session log of its inventory run (136 to 252 s each); the
baseline predates per-run data isolation, so the other two baseline runs have no separate record.
Desktop app builds ran during the baseline live-bugs run and slowed it; its wall time is not a
fair comparison.

## Reading

- **Baseline to after.** Baseline failed two of three tasks (inventory timed out still on its first
  request, qa-spec missed a hidden sort check) and recalled 9/11 and 8/10 requirements where it
  got to the probe. After passed 3 of 4 runs and recalled every requirement in every run,
  including one where 9 compactions came before the probe. On inventory the baseline read files 100 times (75
  repeats) against 50 (20); the read, compact, re-read loop the context-pressure note targeted is
  mostly gone. Each compaction takes 1 to 2 minutes instead of 2 to 4 (lightest reasoning variant
  for the summary).
- **Variance is large.** The same after arm passed inventory in 73 minutes and failed it in 88,
  with nothing in between that touches the failing feature (reservation expiry). Single runs are
  not a verdict on a small difference; the baseline-to-after gap (every task, recall on every run)
  is large enough to stand, the arm-to-arm comparisons below are weaker.
- **Tool search** passed every run (4 of 4) and needed the fewest compactions wherever the context
  filled repeatedly: inventory 7 and 10 to 4 and 3, qa-spec 3 to 1. On live-bugs, which compacted
  once, it made no difference. With the MCP schemas taking half the window, this is the expected
  mechanism. It is now automatic (`43e1546883`): on unless set, once the catalog would take 5% of the
  context budget. The first cut was a tenth, which an 18k-token catalog on a 262k window missed.
- **Repo map** is mixed: fastest reads on qa-spec (the map replaced nearly all file reads), but a
  timeout on inventory and a slower live-bugs run with more compactions. It stays off.
- **Background shell** worked when used (repo-map live-bugs started the server in the background,
  read it with `shell_output`, stopped it with `shell_kill`), but no run showed a time saving from it.
- **Browser.** Baseline drove the qa-spec site through the Playwright MCP tools (39 calls); no after
  arm opened a browser for it and all passed with shell tests. The task does not require one, so
  these runs say nothing about browser skill either way.

## Harness bugs the benchmark found

- **A message sent to a full session was lost** (`632e803b5d`). When the last turn left the context
  over the limit, the next message triggered a compaction before its turn; the message was folded
  into the summary and only a "continue if you have next steps" nudge followed it.
- **A finished answer was followed by unrequested work** (`7a25304f26`). When the reply that ended a
  turn left the context over the limit, the loop compacted at once and posted the nudge; the model
  took another turn (re-ran the tests, wrote a status report) and the answer ended up above it.
  Compaction now waits for the next message, which the fix above keeps.

## Scoring fix

The recall probe was scored on the session's last assistant message, which after the second bug
was the unrequested extra turn: both after inventory runs first scored 0/12 and 1/12 although their
replies recalled 12/12. The runner now scores the reply to the probe itself (`5a7e81ebc5`), and
every run above was re-scored that way from its database.

## Artifacts

Per-run reports are in `long-horizon-2026-09-23/` (`lh-<arm>-<task>.md/.json`; `-prefix` marks the
inventory runs before the replay fix, the unmarked after and tool-search inventory files are the
reruns). Their recall fields come from the old scorer; the table above is the re-scored value.
Run databases and server logs stayed in the session scratchpad and are not kept.
