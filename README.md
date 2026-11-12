# Hermes + DeepSeek Harness (DSH) Coding Architecture

A two-level coding orchestration system: an AI planner/checker agent (**Hermes**) decomposes requirements into blocks with verifiable acceptance criteria, and **DSH (DeepSeek Harness) headless** implements each block — review runs read-only, implement runs execute inside an isolated git worktree whose changes are applied back to the main project only by a **trusted file-copying applier**. The orchestrator never trusts the executor's sandbox, its exit code, or its self-report.

> Shares the executor-agnostic governance layer with `hermes-opencode-coding-architecture`. This repo focuses on the DSH adapter: capability probing, event-stream contract, worktree isolation, and session-log model verification.

## Why DSH needs its own adapter

DSH headless (>= 0.2.2) provides `--json` machine-readable run events, permission presets (`read-only` / `workspace-write`), `--approval-policy never`, session resume, and stdin task input — enough to build a controlled executor. But it has sharp edges the adapter must handle:

- Policy denials and tool errors still exit 0 → success is judged from events, never the exit code
- The stdout event stream carries **no model identity** → the adapter verifies the actual model by parsing the persisted session log (`zstd`)
- No shell command whitelist; the FS permission layer is not a kernel security boundary (symlink TOCTOU accepted upstream) → implement runs are isolated in a git worktree and only `allowed_files` are copied back by the controller

## What's in this repo

```
coding_control.py      # single-file controller with the DSH adapter:
                       # capability probe (version + headless flags),
                       # review path (read-only direct run),
                       # implement path (worktree + trusted applier,
                       #   out-of-scope/baseline-drift/cancel protections),
                       # event parsing (usage aggregation, policy denials),
                       # session-log model verification (zstd, async-safe)
tests/                 # contract suite: fake DSH CLI, fake zstd,
                       # worktree fixtures, cancel/timeout semantics
skills/                # executor-facing skills (TDD, debugging, gates, ...)
tools/git_gate.py      # commit gate
hooks/install.py       # git hook installer
EXECUTOR.md            # executor contract
docs/deployment-guide.md  # from-zero DSH deployment guide
UPSTREAM-LICENSE       # MIT license of upstream-derived components
```

## Quick start

1. **Prerequisites**: Python 3.11+, git, Node.js >= 20, `zstd` on PATH.
2. **Install DSH**: `npm install -g @deepseek-ai/dsh` (verify `dsh --version`). **Require >= 0.2.2** — earlier headless versions lack the required flags.
3. **Dedicated instance**: create an empty `DSH_HOME` directory and a `settings.yaml` with one provider (key via `apiKeyEnv`, never on disk) — see `docs/deployment-guide.md` §2.
4. **Capability check**: the controller probes `dsh --version` and `dsh --profile headless --help`; missing required flags (`--json`, `--permission-mode`, `--approval-policy`, `--session-id`) blocks the run.
5. **Run a review task**, then an **implement task in a scratch git repo** — the DSH process only ever touches a temporary worktree.

## Core contract (short version)

Same candidate protocol as the OpenCode variant (final line `CODING_CONTROL_CANDIDATE {json}`), plus DSH-specific judgment rules: policy denials (`FS_SANDBOX_DENIED` tool errors) fail the run; usage is aggregated from `step_end` events; the model is verified against the session log; cancel means "process group gone + no filesystem change", never the exit code.

## Safety properties

- Executor confined to a `git worktree` (workspace-write); main project never touched by the DSH process
- Trusted applier copies only `allowed_files` back; out-of-scope changes, unauthorized deletions, and baseline drift are recorded and fail the run; timeout/cancel applies nothing
- Read-only review runs cannot write; escalation is auto-rejected under `never`
- Minimal environment: PATH/HOME allowlist + `DSH_HOME` + one API-key variable

## License

MIT — see `LICENSE`. `UPSTREAM-LICENSE` preserves attribution for MIT-licensed upstream components.
