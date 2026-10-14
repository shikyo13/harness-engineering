# Harness engineering

What makes a coding-agent harness effective, collected from source-level audits, subsystem studies, controlled
evaluations and the literature, for use in any project that builds or tunes an agent harness. It started with Zero
Code (a local-first harness running a local Qwen model on an M4 Max), so many measurements come from that setup;
the audits, studies and methods are general.

Start with [`INDEX.md`](INDEX.md): it finds material by harness, by subsystem and by question.

## Layout

| folder | what it holds |
|---|---|
| `surveys/2026-10-13-harness-survey/` | The October 2026 survey: source audits of 18 harnesses (`02-COMPETITOR_AUDITS/`), Zero Code's own baseline (`01`), the Antigravity vs Gemini CLI case study (`03`), the capability matrix (`04`), engineering findings (`05`), experiment design with results (`06`), synthesis opportunities (`07`), and Zero Code's target architecture, roadmap and executive assessment (`08`-`10`). The files cite each other by number, so they keep their original names. `GOAL.md` is the brief, `PROGRESS.md` the work log with a 44-claim verification table, `SOURCES.md` the repository pins. |
| `surveys/2026-09-13-harness-comparison.md` | An earlier file-by-file comparison of OpenCode, pi, oh-my-pi and Codex. |
| `studies/2026-10-14-subsystems/` | Full-source studies per subsystem (agent kernel, tools and editing, context engine, providers and model tables, autonomy, multi-agent, extensions, observability, protocol), each comparing the strongest implementations with file:line references and proposing a design. Drafts until marked final. |
| `literature/` | Research outside the audited harnesses: memory architectures (2026-09-15), self-improving harnesses and inference engines (2026-09-15), and `2026-10-14-proven-advances.md`, a verified survey of the latest proven advances. |
| `evaluation/methods/` | How to compare harnesses fairly: the same-model comparison protocol, the all-around matrix and leaderboard design, and the Qwen harness study's experiment design, decision log, OpenBench audit and harness roster. |
| `evaluation/results/` | What has been measured: the 2026-09-29 synthesis of every measured win and null result, the cross-harness scorecard, tool-search and tool-profile A/Bs, long-horizon results, and the Qwen vs Sonnet ladder. |
| `PROVENANCE.md` | Every imported file with its source repository, path and commit. |

## Standards

- **Evidence is labelled.** Claims say whether they are measured (with numbers and setup), read from source
  (with `file:line` and a pinned commit) or only documented or claimed. Keep that labelling in anything added.
- **Primary sources.** Source code, official engineering posts, papers and our own runs; never secondary summaries
  without saying so.
- **Corrections are recorded, not silently overwritten.** Later documents correct earlier ones in place with a
  dated note (see the survey's correction pass and the studies' "beyond or correcting the audits" sections).
- **Only build on sources with a usable license.** Re-implement AGPL designs such
  as OpenWolf clean-room only.

## What is not here

- The cloned harness repositories (several GB) live at
  `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/`, pinned in the survey's `SOURCES.md`.
- Raw transcripts and vendored tools of the Qwen harness study stay in `~/Documents/Dev/qwen-harness-study`
  (about 300 MB). Its README also lists a `HARNESS_ARCHITECTURE_MATRIX.md`, which does not exist in that repository.
- Zero Code's project-specific material (its behaviour specification, progress logs and architecture for the
  rebuild) stays in the `zero-code-next` repository; finished general results are copied here.

## Keeping it current

Active programs work in their own repositories. At each milestone, finished general documents are copied here
with a `PROVENANCE.md` row, and drafts are replaced by final versions. Add new harness audits under the latest
survey or a new dated survey, new measurements under `evaluation/results/`, and update `INDEX.md`.
