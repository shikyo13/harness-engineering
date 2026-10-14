# Harness and engine measurements

Date: 2026-09-15

Engine: mlx-serve v26.9.2, OpenAI-compatible, base URL http://127.0.0.1:11234/v1
Model: ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit (context_length 262144, is_moe true, 4-bit)

## Method

Prefill cost is measured as the wall time of a non-streaming POST to /v1/chat/completions
with max_tokens set to 1 (measurement 1) or 200 (measurement 2), reading usage.prompt_tokens
from the response for the exact prompt size. Every request includes reasoning_effort: low.

### Request shape used for measurement 1 (tool list A, abbreviated to 2 of 10 tools)

```json
{
  "model": "ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit",
  "messages": [
    {
      "role": "user",
      "content": "<NNNN numbered pseudo-code lines>\n\nReply with the single word OK."
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "read",
        "description": "Read a file from the local filesystem and return its contents as text, optionally starting at a given line offset with a maximum number of lines.",
        "parameters": {
          "type": "object",
          "properties": {
            "file_path": {
              "type": "string",
              "description": "Absolute path to the file to read"
            },
            "offset": {
              "type": "integer",
              "description": "Line number to start reading from"
            },
            "limit": {
              "type": "integer",
              "description": "Maximum number of lines to read"
            }
          },
          "required": [
            "file_path"
          ]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "write",
        "description": "Write content to a file on the local filesystem, creating it if it does not exist or overwriting it if it does.",
        "parameters": {
          "type": "object",
          "properties": {
            "file_path": {
              "type": "string",
              "description": "Absolute path to the file to write"
            },
            "content": {
              "type": "string",
              "description": "The full content to write to the file"
            }
          },
          "required": [
            "file_path",
            "content"
          ]
        }
      }
    }
  ],
  "max_tokens": 1,
  "reasoning_effort": "low",
  "stream": false
}
```
(tools array continues with edit, bash, grep, glob, list, webfetch, todowrite, task - 10 tools total, 5056 bytes serialized)

Tool list A' is identical except the read tool description has " Handles both text and binary files." appended.

Filler text is deterministic numbered pseudo-code, one line per unit, e.g.:
```
000001: var_000001 = compute(var_000000, step_000001, factor_01)
000002: var_000002 = compute(var_000001, step_000002, factor_02)
000003: var_000003 = compute(var_000002, step_000003, factor_03)
...
```

Calibration (tools A): 50 lines -> 3535 prompt_tokens, 550 lines -> 24035 prompt_tokens, tokens_per_line=41.000, fixed overhead=1485.0 tokens.

## Measurement 1: cost of a tool-list change

Sequence per size: (1) tools A cold, (2) tools A again warm, (3) tools A' changed description, (4) tools A again.

| Target size | Actual prompt_tokens | Cold (s) | Warm (s) | Changed (s) | Again (s) | Cold prefill rate (tok/s) |
|---|---|---|---|---|---|---|
| 8000 (0.1% vs target) | 8004 | 8.94 | 0.46 | 17.17 | 0.13 | 895.2 |
| 32000 (-0.0% vs target) | 31989 | 19.01 | 0.37 | 91.84 | 0.19 | 1682.4 |
| 100000 (0.0% vs target) | 100008 | 192.45 | 0.47 | 153.21 | 0.49 | 519.6 |

Supplementary: server-reported cached_tokens per call (0 = fully recomputed, near prompt_tokens = cache hit).

| Target size | Cold cached | Warm cached | Changed cached | Again cached |
|---|---|---|---|---|
| 8000 | 4096 | 7973 | 0 | 8003 |
| 32000 | 24004 | 31958 | 7980 | 31988 |
| 100000 | 31958 | 99977 | 31965 | 100007 |

## Measurement 2: deferred tool loading

User message for both variants:

```
You also have a tool named weather that takes one string parameter city. It is not in your
tool list but you can call it exactly like any other tool. Call weather for Vienna now.
```

Variant 1 tools param: only the read tool from list A. Variant 2 tools param: omitted entirely. max_tokens 200 for both.

| Variant | tools param | tool_calls array | raw tool_call tags | verdict |
|---|---|---|---|---|
| 1 | read only | yes | no | structured tool_calls array present |
| 2 | none | no | no | answered in prose (no structured call) |

### Raw message objects

Variant 1 (tools = [read]):
```json
{
  "role": "assistant",
  "content": null,
  "reasoning_content": "The user wants me to call a tool named \"weather\" with the parameter \"Vienna\". Even though it's not in my listed tools, they say I can call it exactly like any other tool.",
  "tool_calls": [
    {
      "id": "call_1789489986153_0",
      "type": "function",
      "function": {
        "name": "weather",
        "arguments": "{\"city\":\"Vienna\"}"
      }
    }
  ]
}
```

Variant 2 (no tools param):
```json
{
  "role": "assistant",
  "content": "",
  "reasoning_content": "The user wants me to call a tool called \"weather\" with the parameter \"city\" set to \"Vienna\". They've specified that this tool is not in my standard tool list but I can still call it."
}
```

## What this means

- At 100000 tokens, a cold prefill took 192.5s versus 0.5s once warm; freezing the tool list to keep that warm path is worth roughly 411x.
- Changing one tool description forced a recompute costing 153.2s, close to the cold time, confirming any tool-list edit invalidates the cached prefix.
- Reverting to the original tool list afterward took 0.5s, so a single edit-and-revert does not permanently evict the earlier cache entry.
- Cold prefill throughput rose with prompt size, consistent with fixed per-request overhead dominating small prompts.
- Deferred tool loading verdict: with a partial tools list the model structured tool_calls array present; with no tools param it answered in prose (no structured call).
- See state.json in the scratchpad for full per-call timing and timings objects.

