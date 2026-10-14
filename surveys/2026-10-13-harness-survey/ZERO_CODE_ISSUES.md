# Zero Code issues hit during research

Bugs, limitations, and friction encountered while working inside Zero Code on this assignment. Research-only run: no
code changes are made here.

- Nothing blocked this research session itself. Recorded as the master limitation surfaced by the baseline audit:
  the shipping execution path is the legacy V1 loop (`SessionPrompt.runLoop`) while the durable V2 core is
  flag-gated off and its only HTTP entry (`promptV2`) blocks the request for the entire drain
  (`packages/zero/src/server/routes/instance/httpapi/handlers/session.ts:335-355`), so V2 cannot be exercised
  end-to-end through the normal UI during this research without behavior differing from production.
- Subagent (task tool) runs of 30–60 min on the shared local engine completed fine but expose no intermediate
  progress to the parent beyond completion; long audits must simply be awaited.
- Eval-boot bug found while producing 06 (2026-10-14, `long-horizon` at 44714a1abb): `zero-eval` against the local
  engine dies at boot — `client.session.create` (sdk/v2) hangs ~60 s then returns an error that serializes to `{}`;
  the spawned `zero serve` logs nothing beyond "engine running" and exits clean. Direct `POST /session` without auth
  gets 401, so the eval path's auth/instance-bootstrap is implicated, not the engine. Not diagnosed further
  (research-only run). Artifacts: `2026-10-13-context-stress.md/.json`, serve logs under `$TMPDIR/zero-eval/`.
  **Resolved by the owner in `dc49a6f04a`**: a zero-eval launched *inside* a Zero Code session inherited the desktop
  server's password, so the spawned server answered 401 to the credential-less client (the 401 seen in the direct
  curl repro was the same shape). Evals now start in ~2 s; the E3 context-stress rerun is unblocked.
