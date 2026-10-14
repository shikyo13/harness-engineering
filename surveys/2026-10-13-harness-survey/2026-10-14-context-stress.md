# Zero Code capability pass: mlx/qwen3.8-flash (ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit)

Collected 2026-10-14 on branch `long-horizon`, commit `6fae42319960`, working tree clean (Apple M4 Max, 128 GB, macOS 26.6.2). Engine: mlx-serve 26.10.2
mlx 0.32.3
mlx-c 56b2d39fc831
nax off (requires M5-class GPU)
ggml 0.26.0 (d81235049)
llama.cpp b11429
gguf 3
ds4 unknown at http://127.0.0.1:11234/v1. Runner: `bun run --cwd packages/zero-eval eval`.

## Speed

_not run_

## Reasoning

_not run_

## Coding

_not run_

## Agentic (long run, subagent, browser)

| task | status | wall | turns | compactions | recalled | goal iterations | tool calls | failed | guard refusals | read before edit | tok/s | prefix hit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| long-kv-cli | pass | 267 s | 1 | 0 |  | 0 | 10 (bash 4, write 4, todowrite 2) | 0 | 0 | 1/4 | 59.1 | 90% |

1/1 passed.

## Notes

- Power at start: AC power
- long-kv-cli: activity below expectation: expected at least 12 tool calls, saw 10; correctness is scored independently
