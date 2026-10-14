# mlx-serve lifecycle for Zero Code

Research date 2026-09-15. Engine: mlx-serve v26.9.2 at `~/.mlx-serve/engine/v26.9.2`, wrapper `~/.local/bin/mlx-serve`, port 11234, serving a 125B MoE (~75 GB resident). The engine stayed stopped throughout. Sources: `mlx-serve serve --help`, the install dir (LICENSE/NOTICE only, no changelog), `~/.mlx-serve/logs/serve.log`, and github.com/ddalcu/mlx-serve (NOTICE's copyright holder, David Dalcu, matches the repo owner).

## 1. Unload, idle TTL, endpoints, memory return

`--idle-evict-secs <n>` ("Evict `.ready` entries with refcount==0 if idle for this many seconds", default off) is the idle-TTL/auto-unload knob. `--max-resident-models` (default 3) and `--max-resident-mem <n>|auto` (default 80% of the wired limit, 86.0 GB observed here) drive LRU eviction when a new on-demand load would exceed them. There is no manual unload flag and no `/v1/unload-model`; `--help` documents `/v1/load-model` only for `--model-dir` siblings. The startup route banner lists `GET /`, `GET /health`, `GET /props`, `GET /v1/models`, `POST /v1/chat/completions` (not exhaustive, `/metrics` is enabled but absent from it). `/health` is exempt from `--api-key`, the right health-check target. `/v1/models` returns a per-model `loaded` boolean and `state`, already parsed by `packages/zero-plugins/engine/src/probe.ts`.

Memory return isn't documented locally; the repo confirms it. PR #398 (github.com/ddalcu/mlx-serve/pull/398) made `--idle-evict-secs` actually fire (previously "accepted... but never executed") and fixed a ~196 MB/cycle leak in `unloadResident`; testing quoted there shows RSS flat at 457-459 MB across repeated load/unload cycles versus 1634 MB before the fix. So weights return to the OS on eviction in this build (v26.9.2, released 2026-09-09). A separate PR, #428, warns a status poll can itself reset the idle clock and suppress eviction, relevant to the health-check design below. Separately, MLX keeps its own small reusable buffer pool (`[mem] MLX buffer-pool cap 8192 MB`, tuned to 4096 MB via `MLX_SERVE_CACHE_LIMIT` in `serve.sh`), unrelated to the ~75 GB of weights.

## 2. Startup and first-load time

mlx-serve does not timestamp its own log lines (checked `serve.log`, `mlx-serve-11234.log`, `serve.out`, `serve-stdout.log`); the only `[2026-09-...]` strings present are pasted content inside chat requests, not server timestamps. No duration can be quoted. The last run's log sequence, in order: `mlx-serve 26.9.2 (headless, models load on demand)`, `Server listening on http://127.0.0.1:11234`, tokenizer load, 101 `Loading model-XXXXX.safetensors...` lines, `Loaded 3167 weights from 101 file(s)`, `[qwen4] MTP head loaded`, then on the first chat request: `[registry] default model -> ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit (first chat-capable load on a headless server)` and `[registry] model id=... ready (70.13 GB resident)`. Since the binary won't provide timing, Zero Code's own lifecycle manager should timestamp spawn, first `/health` 200, and first `loaded:true` itself.

## 3. How the last run stopped

The log ends at `Shutting down gracefully...` right after the last streamed response, with nothing after: no signal name, no exit code, no crash trace. `serve.log`'s mtime is 2026-09-15 08:22:53. Reads as an intentional stop, but the log can't say who sent the signal.

## 4. Zero Code today

`packages/zero/src/cli/cmd/serve.ts` starts Zero's own Effect-based `Server.listen()` and blocks on `Effect.never`; no engine awareness. Desktop (`packages/desktop/src/main/server.ts` + `sidecar.ts`) already has the pattern this design needs, for Zero's own server: `spawnLocalServer()` forks a `utilityProcess`, posts `{type:"start"}`, races `ready` against a 60s stall timeout, then polls `checkHealth()` every 100ms; `stop()` posts `{type:"stop"}`, races exit against a 6s timeout, then `child.kill()`. `packages/zero-plugins/engine` is probe-only: checks `isLoopback`, GETs `/v1/models` (15s cache), syncs model metadata, and on failure tells the user to run `serve.sh`/`pull-model.sh` by hand; it never spawns or stops anything. `zero.engine` config today has only `enabled` and `probe_timeout_ms` (`config.ts:171-179`).

## Lifecycle design

**Start**: on Zero Code startup, GET `/health`; if unreachable, check whether the port is listening at all. If not, spawn `scripts/mlx-serve/serve.sh` in its own process group (own pgid, not a child of the CLI/Electron tree) and poll `/health` with backoff up to a generous timeout (minutes; unmeasured, but 101 shards plus warmup is not fast). If something already listens on 11234, adopt it: verify via `/health` and `/v1/models`, skip spawning a second copy, and mark it "not ours" so shutdown leaves it running.

**Status**: surface reachable/loading/ready/unreachable in the app using `/health` plus `/v1/models`' `loaded`/`state` fields, exactly what `probe.ts` already parses. Keep polling infrequent or route it around whatever resets mlx-serve's idle clock (per PR #428) so status checks don't fight idle-evict.

**Restart**: on health-check failure or unexpected exit, restart with exponential backoff and a max-attempts cap; surface failure to the user rather than loop forever.

**Stop**: only send a stop signal if Zero Code owns the process (recorded at adopt/spawn time). Signal the process group: SIGTERM, wait a grace period (mirror the desktop sidecar's 6s, but longer given a 75 GB unload), then SIGKILL. Never touch an adopted, externally-started engine.

**Idle unload**: pass `--idle-evict-secs` through a config key for memory back during idle without a full process stop; cheaper than restart but still pays the reload cost next use.

**Recommended `zero.engine` keys**: `enabled`, `probe_timeout_ms` (existing); add `manage_lifecycle` (default true), `serve_script` (default `scripts/mlx-serve/serve.sh`), `port` (default 11234), `startup_timeout_ms`, `restart_backoff_ms`/`max_restarts`, `idle_evict_secs` (default off), `stop_grace_ms`.

**Risks**: CLI and desktop both running can race to spawn or stop the same engine; needs a lock (PID file, owner tag) so only one manager acts. Dev and prod builds sharing port 11234 will collide or reuse the wrong build's engine; consider a build-id check via `/props` or a distinct dev port. A full restart drops the RAM-resident hot prefix cache (the SSD tier under `~/.mlx-serve/kv-cache` survives restarts, so cost is a slower first turn, not a cold cache). Reload cost for 75 GB is real and unmeasured; confirm it before tuning idle-evict, since mlx-serve's log carries no timestamps to show that cost today.
