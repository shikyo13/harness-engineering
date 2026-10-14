# All-around harness leaderboard protocol

The leaderboard has two levels:

1. The **coding composite** ranks the five harnesses on the identical pinned
   OpenBench task cells that already have runtime evidence.
2. The **all-around composite** ranks the same candidates across six shared
   dimensions once every candidate has a valid row for every dimension.

The all-around score is a weighted mean of normalized scores:

| dimension | weight | runtime proof |
|---|---:|---|
| Coding correctness and completion | 35% | OpenBench external checker, clean process exit |
| Speed | 15% | provider TTFT, decode throughput, wall time |
| Context efficiency and retention | 15% | fixed long-prefix probes, cache split, context-rotation verifier |
| Long-running completion | 15% | multi-file project task, hidden verifier, fixed wall budget |
| Tool use | 10% | shared local browser fixture, tool errors, artifact verifier |
| Recovery | 10% | controlled interruption, same-session resume, duplicate/lost-edit verifier |

The machine-readable contract is
`docs/evals/2026-09-17-all-around-matrix.json`. The scorecard generator reads
that matrix and emits `coverage_incomplete` until each candidate has an
equivalent runtime row in all six dimensions. It never treats a missing row as
zero and never mixes desktop-only features into the cross-harness score.

## Desktop capability lane

Zero Code also has a separate desktop lane for features that the CLI harnesses
cannot exercise on this host: desktop status visibility, in-app browser
transport, computer use, and Minecraft control. Those results remain visible
in the scorecard as runtime evidence and are not silently folded into the
cross-harness composite. If another harness receives the same desktop adapter,
that dimension can be added to the shared matrix with the same verifier.

## Current state

The coding composite and the first identical three-trial speed probe are
populated. The speed row ran through Zero Code's packaged desktop app and each
native comparison interface; raw evidence is recorded in
`docs/evals/2026-09-17-native-speed-probe.json`. A repeated 16k-word context
retention lane is also complete for Pi, Aider, Qwen Code, and OpenCode; its
raw rows and observed cache telemetry are in
`docs/evals/2026-09-17-context-probe.json`. That lane is deliberately not
promoted into the all-around matrix until the 64k, near-limit, cold-cache, and
context-rotation fixtures are run. Shared long-running, tool-use, and recovery
rows still need runtime collection. Zero's desktop Minecraft lane is currently
a clean runtime pass after explicit MCP schemas and native argument
normalization were added.

The 64k lane has one controlled runtime row for every candidate in
`docs/evals/2026-09-17-context-64k-probe.json`: each returned the exact marker
after a roughly 91k–99k-token prompt. Zero Code's packaged desktop row showed
`37% used`, 96,761 input tokens, and a 3m56s visible completion. An earlier Pi
observation was inconclusive because the client was terminated before its
bound; it remains preserved and excluded from scoring. Three-trial repeats and
the cold-cache lane are still required before this becomes a scored dimension.

The shared `taskflow` long-running fixture also has one fresh hidden-verifier
row per candidate in `docs/evals/2026-09-16-openbench-taskflow.json`: Zero,
Pi, OpenCode, and Qwen Code scored 1.00; Aider scored 0.28 (7/25 tests). The
scorecard exposes these rows as `single_trial_provisional` and leaves the
long-running matrix dimension empty until repeated cells are collected.
