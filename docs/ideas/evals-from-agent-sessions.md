# Idea: evals from your own agent sessions

Status: notes only, no implementation. Sources checked 2026-10-19.

## Summary

Ibragim Badertdinov ([@ibragim_bad](https://x.com/ibragim_bad)) works on coding agents, evals and training datasets at Nebius [3]. He turned his own coding-agent history into a small private benchmark. His X post says he "had 1,270 agent sessions sitting in ~/.claude, ~/.codex and ~/.pi", and that "my corrections to the agent are the test cases" [1]. From about three months of agent chats he removed duplicates and picked 10 tasks, each one something that would usually take him under an hour. He turned them into Harbor tasks with automated checks [3]. Each configuration runs 3 times per task, which is 30 attempts per configuration [3][4].

The headline result is from his leaderboard, suite `core10-v1.3`, updated 2026-10-17 [4]. Pi with GPT-6.1 Sol (medium effort) passed every run at $0.0704 per task. Codex with the same model and effort also passed every run, at $0.1375 per task. That is the "2× cheaper setup" in the article's title [1][4]. He uses the benchmark to choose models and to test changes to the agent, its prompts and its plugins [3].

His own caveats: "10 tasks × 3 runs" is "enough to pick my daily setup, not a general claim" [2b]. Several configurations pass every check, so cost and speed are what tell them apart [3][2c]. The same pass rate doesn't mean the same code quality [3].

## Pipeline

1. **Session logs.** Copy the raw JSONL transcripts and work only on the copy. Scan the copy for secrets with gitleaks [5].
2. **Task extraction.** List candidate "episodes": one user request plus the work that satisfied it. Keep an episode only if the result can be checked, the work takes a human about 1 hour or less, it's not trivial, and it's real recurring work. Prefer episodes with a decisive user correction [5]. Each task is pinned to a frozen repo snapshot at the parent commit. The session only explains where the task came from [2d]. Once a task is accepted it's never edited, and any change becomes a new version [2e][5].
3. **Verifier.** A hidden, offline, deterministic `tests/` script writes a 1/0 reward [5][6]. A correction doesn't become an exact-diff check. It tells you which edge cases the tests need, for example "cached tokens counted twice" [2f]. The verifier itself gets validated: a do-nothing run fails, the oracle passes 5/5, near-miss mutants fail, an alternative correct solution passes (so it doesn't grade style), and a leak probe runs [5][2g].
4. **Repeated Harbor runs.** Run each agent/model/effort configuration k=3 times per task. For example, `harbor run -p tasks/ ... -k 3` [5]. Every run starts fresh, so nothing carries over between runs, though the provider's prompt cache still works within a run [2h].
5. **Scoring.** For each configuration, record pass@1, pass@3 and pass-all-3, plus $/task, time/task, steps, tokens/task and cache hit rate [4].

The loop repeats: a new setup produces new sessions, and new failure modes become new tasks [2i].

## How it could map onto forgeeval

- **Phase 0 (`phase0_baseline.py`, currently empty).** Pick the baseline: one agent/model configuration run against a handful of hand-made Harbor tasks. Write down the result schema now, using the scoring fields above, so later phases can be compared with it.
- **Phase 1: session inventory.** A read-only step that copies Rajiv's session logs, scans them for secrets, and produces a candidate list for human approval. The skill uses the same checkpoint [5].
- **Phase 2: task authoring.** Turn approved episodes into versioned Harbor task directories (instruction, environment, tests, solution, provenance), validated as in pipeline step 3. Either reuse the published skill directly or treat it as the spec.
- **Phase 3: comparison runs.** Run a matrix of configurations × tasks × 3 runs through Harbor. Store the per-trial results.
- **Phase 4: report.** Rank configurations by pass rate, then by cost and time. Add a "re-run on new model or plugin" routine and a holdout set of newer tasks (see open questions).

## Inputs and prerequisites

| Agent | Session location | Format | Source |
|---|---|---|---|
| Claude Code | `~/.claude/projects/<project>/<session-id>.jsonl` | JSONL | Official docs [7]. Deleted after 30 days by default (`cleanupPeriodDays`) [7][8] |
| Codex CLI | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` | JSONL | From the skill [5]. **Not verified in OpenAI's docs** |
| Pi | `~/.pi/agent/sessions/--<path>--/<timestamp>_<session-id>.jsonl` | JSONL, entries form a tree via `id`/`parentId` | Official docs [9] |
| Gemini CLI | `~/.gemini/tmp/<hash>/chats/` | not verified | From the skill [5]. **Not verified** |

Other requirements: Docker and git, Python 3.11+, Harbor (`uv tool install harbor`), gitleaks 8, and credentials for at least one agent/model [5][6]. Harbor is the harness from the Terminal-Bench team. It evaluates agents such as Claude Code and Codex CLI, either in Docker or through cloud sandbox providers [6]. His leaderboard ran on Harbor 0.21.0. The skill's examples use 0.24 syntax [4][5].

## Open questions

- **Privacy and secret scrubbing.** Transcripts can contain API keys and private code. Private-mode tasks include a repo snapshot and must stay private [5]. forgeeval is a public repo, so only re-authored "shareable" tasks could ever be committed here.
- **Verifier flakiness.** What's the false-negative rate? Should a verifier be rejected after N flaky runs? The skill allows at most two version bumps after the first smoke run [5].
- **Is 3 runs enough?** With 10 tasks × 3 runs, many configurations hit 100% [4], so pass rate stops separating them [2c]. Do we need more tasks, harder tasks, or confidence intervals on cost?
- **Measuring cost.** Do we use API list prices or what was actually billed? How do we count cache hits? Cache hit rates range from 0% to 97.9% across configurations [4].
- **Contamination and overfitting.** The tasks only cover the old setup's failure modes, so a cheaper setup can ace them and still fail in new ways [2j]. One suggestion: hold out next week's tasks and choose a setup before running them [2k]. The skill blocks future git history and keeps tests hidden [5].
- **Picking tasks.** Corrections that changed the result versus style preferences [2g]. "Discovery" versus "routine" tasks [5]. How many per category?

## References

Link checks were run on 2026-10-19 (HTTP status, following redirects).

1. X post, @ibragim_bad, 2026-10-16: https://x.com/ibragim_bad/status/2111102467924496721 (200; text read through the X API). The linked X Article "Turn your agent sessions into evals. Mine found a 2× cheaper setup", https://x.com/i/article/2110460904642949123, **returned 404/403 and its body was not read: link not verified**.
2. @ibragim_bad's replies in the same thread, read through the X API: (b) https://x.com/ibragim_bad/status/2111111498521358465 · (c) https://x.com/ibragim_bad/status/2111146014426206447 · (d) https://x.com/ibragim_bad/status/2111121129876107382 · (e) https://x.com/ibragim_bad/status/2111151633753358826 · (f) https://x.com/ibragim_bad/status/2111128000779743382 · (g) https://x.com/ibragim_bad/status/2111111579584676102 · (h) https://x.com/ibragim_bad/status/2111140059093020754 · (i) https://x.com/ibragim_bad/status/2111124322077090224 · (j) reader reply from @kartikb753, which he agreed with: https://x.com/kartikb753/status/2111121274814828544 · (k) reader suggestion from @MisterSnowCrash ("good call, will try that"): https://x.com/MisterSnowCrash/status/2111125720785420657. All reply URLs returned 200.
3. "My mini-benchmark for coding agents", 2026-09-26: https://ibragim.dev/writing/mini-benchmark-for-coding-agents/ (200)
4. Personal Coding Evals leaderboard: https://ibragim.dev/leaderboard/ (200)
5. `evals-from-sessions` skill, v1.0.4, MIT: https://github.com/ibragim-bad/evals-from-sessions (200). The owner's GitHub profile lists ibragim.dev, and the README links his leaderboard.
6. Harbor: https://github.com/harbor-framework/harbor (200; https://github.com/laude-institute/harbor redirects here) · docs: https://harborframework.com/docs (200)
7. Claude Code, Manage sessions: https://code.claude.com/docs/en/sessions (200)
8. Claude Code settings reference (`cleanupPeriodDays`): https://code.claude.com/docs/en/settings-reference (200)
9. Pi session file format: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/session-format.md (200)
