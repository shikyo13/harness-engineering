# Tool search A/B (2026-09-17)

`fix-failing-test` × 3 trials per arm, packaged binary rebuilt with the tool-search
change, same loaded engine, filler MCP server (`scripts/openbench-zero/mcp-filler.ts`,
24 tools, 13.5 KB raw schema) added to the bench config so both arms carry a real
MCP catalog. Baseline declares the catalog directly; treatment enables
`ZERO_EXPERIMENTAL_TOOL_SEARCH=true`.

| metric (median wall) | off | on |
| --- | --- | --- |
| wall s | 26.2 (20.8–43.2) | 27.2 (25.5–38.0) |
| checker | 3/3 | 3/3 |

| engine metrics (sum over run) | off | on |
| --- | --- | --- |
| tool schema bytes per request | 35,084 | 22,417 |
| prompt tokens, first turn | 11,055 | 8,050 |
| prompt tokens, max turn | 12,112 | 9,210 |
| prompt tokens, total | 174,771 | 104,210 |
| cache hit ratio | 0.943 | 0.959 |

## Reading

- Mechanically proven: per-request schema payload −36% at this catalog size,
  first-turn prefill −3,005 prompt tokens; savings scale with catalog size
  (the desktop profile's real peekaboo+playwright+minecraft catalog is ~5× larger).
- No timing claim from these numbers: n=3 medians with overlapping ranges;
  this task barely generates long context, where the per-turn schema cost is
  mostly cache reads. Wall time is indistinguishable, and that is the honest result.
- The search round-trip cost is not measured here: the task never needs an MCP
  tool, so the treatment arm never called `tool_search`. A task that must use
  MCP tools is the follow-up if we ever need to bound that overhead.
- Accuracy neutral on this cell (6/6 pass).

Per the promotion rule ("single-task medians are single-task evidence"), tool
search stays opt-in; nothing is promoted to default on this data. The defensible
claims are: fewer request bytes, cheaper cold prefill and no re-prefill when an
MCP server connects or changes late, at no observed accuracy cost.

Artifacts: /tmp/ob-off.jsonl, /tmp/ob-on.jsonl, /tmp/ob-off.mlx.json, /tmp/ob-on.mlx.json.
