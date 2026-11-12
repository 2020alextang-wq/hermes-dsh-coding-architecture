# Hermes + DSH (DeepSeek Harness) Coding System — Deployment Guide

- **Audience**: any third-party machine or AI orchestration agent ("Hermes") that wants to run DSH headless as a controlled coding executor
- **Self-contained**: no prior installation assumed; all paths/models/keys are placeholders
- **Origin**: distilled from the internal mechanism guide; contains no revision history or run evidence
- **Prerequisite reading**: the governance model (four roles, two trust boundaries, two-level planning, P0–P6 pipeline, block acceptance) is identical to the Hermes + OpenCode deployment guide and is not repeated here; this document covers only the DSH-specific differences

## 1. DSH executor in one sentence

DSH is a second executor. What transfers is the governance method (task contract, least privilege, independent acceptance, evidence loop); what does **not** transfer is CLI flags, sandbox policy, events, and plugin assembly — hence a dedicated adapter contract (§4–§6).

## 2. Install DSH from zero

1. Prerequisites: Node.js >= 20, git; also install the `zstd` command (used for model verification, §6).
2. Install: `npm install -g @deepseek-ai/dsh` (or run ad hoc via `npx -y @deepseek-ai/dsh`).
3. Verify: `dsh --version` prints a version.
4. **Version hard line**: the critical headless flags (`--json`, `--permission-mode`, `--approval-policy`, `--session-id`, stdin `-`) do not exist in early versions; **require >= 0.2.2**. A flag appearing in help does not prove its semantics — verify behavior item by item before adoption (§3.2).
5. Dedicated instance: create an empty home directory for coding use and point `DSH_HOME` at it; never share configuration with any existing DSH web instance.
6. Model configuration — `<DSH_HOME>/settings.yaml` with one provider (placeholder example):

```yaml
llm-pi-ai:
  providers:
    my-provider:
      displayName: My Coding Provider
      apiKeyEnv: MY_MODEL_API_KEY      # key via environment variable only
      api: openai-completions           # or openai-responses / anthropic-messages
      baseURL: https://<your-gateway-or-official-api>/v1
      models:
        - id: <MODEL_ID>
          contextWindow: 1000000        # field name is contextWindow
```

7. Smoke test: `MY_MODEL_API_KEY=<key> DSH_HOME=<DSH_HOME> dsh --profile headless "Reply OK"` returns.

## 3. Headless capability verification (mandatory before adoption)

### 3.1 Probe commands

```
dsh --version
dsh --profile headless --help
```

Required capability set: `--json`, `--permission-mode`, `--approval-policy`, `--session-id`. Any missing → refuse adoption (no silent downgrade).

### 3.2 Behavior checklist (help presence != correct semantics)

| # | Item | Method | Expected |
|---|---|---|---|
| C1 | Event stream | run a minimal task with `--json` | stdout is jsonl: session/status/usage/text/tool_call/tool_result/final |
| C2 | Usage events | same | step_end carries usage{inputTokens,outputTokens,...} |
| C3 | read-only blocking | `--permission-mode read-only --approval-policy never`, ask it to write a file | write denied with `tool_result.error.code` like `FS_SANDBOX_DENIED`; no file created |
| C4 | workspace-write | switch preset | target file created correctly |
| C5 | Resume | `--session-id <id>` in a new process, ask about the previous turn | context genuinely preserved |
| C6 | Cancel | start a long task in its own process group, SIGTERM the group | whole group dead; no late writes (cancel criteria in §6.3) |

## 4. DSH adapter contract (dispatch layer)

### 4.1 Shared rules

- **The governance layer is fully shared with the OpenCode variant**: task-package JSON, allowed_files, allow_delete, command_groups, candidate protocol, snapshot-diff acceptance, behavioral matrix — all identical.
- **Runner settings** must use `provider/model` form: `{"kind":"dsh","binary":"<dsh path>","model":"<PROVIDER>/<MODEL>","dsh_home":"<DSH_HOME>","api_key_env":"<KEY_ENV_NAME>"}`.

### 4.2 Review (read-only) path

```
argv: <dsh> --profile headless --json --permission-mode read-only \
      --approval-policy never [--session-id <ID>] -
prompt: via stdin ("-"), never in argv (length and process-list exposure)
cwd: <PROJECT_DIR>
env: host-variable allowlist + DSH_HOME + exactly one API-key variable
     (value taken from the host environment, never written to disk)
```

### 4.3 Implement (write) path — worktree isolation + trusted applier

The DSH sandbox strength is not trusted (see §7); implement always:

1. The project must be a git repo; the controller creates a temporary worktree (`git worktree add <tmp>/<run_id> HEAD`).
2. DSH executes inside the worktree (`--permission-mode workspace-write`); the main project directory is never touched by the DSH process.
3. After the run, the controller parses the worktree's changes vs HEAD (`git status --porcelain` + untracked):
   - the **trusted applier copies files** — only `allowed_files` changes go back to the main repo; it never executes anything produced by DSH and never `git apply`s arbitrary patch text
   - authorized deletions (in `allow_delete`) land in the main repo; unauthorized deletions are recorded as `worktree_unauthorized_delete` and FAIL
   - changes/new files outside allowed_files are **not applied**, recorded as `worktree_out_of_scope`, and FAIL
   - before copying, the main repo's baseline hash is checked; external modification during the run → do not overwrite, FAIL (baseline drift)
   - bytecode artifacts (`__pycache__` components / `.pyc` / `.pyo` / `.pytest_cache`) do not count as out-of-scope; they are recorded fully in `worktree_ignored_artifacts`
   - timeout/cancel: the worktree is cleaned up and **nothing is applied**
4. Cleanup in `finally` via `git worktree remove --force`.
5. After application, the shared acceptance chain runs on the main repo (snapshot diff, content scan, deleted-assertion detection, independent final-gate execution).

## 5. Event-stream contract

`--json` stdout emits one JSON object per line (observed structure):

```
{"type":"session","sessionId":"...","cwd":"..."}
{"type":"status","phase":"turn_start","turn":1}
{"type":"status","phase":"step_start","turn":1,"step":1}
{"type":"thinking","text":"..."}
{"type":"text","text":"..."}
{"type":"tool_call","callId":"...","tool":"bash|write|edit|read","input":{...}}
{"type":"tool_result","callId":"...","status":"result|error","result":"...","error":{"name":"...","code":"FS_SANDBOX_DENIED"}}
{"type":"status","phase":"step_end","turn":1,"step":1,"usage":{"inputTokens":N,"outputTokens":N,"cacheReadTokens":N,"cacheWriteTokens":N,"totalTokens":N}}
{"type":"status","phase":"turn_end","turn":1,"reason":{"kind":"completed"}}
{"type":"final","sessionId":"...","reason":{...},"text":"<final text>"}
```

The controller must:
- persist every line as the run log; record the sessionId from the `session` event (mismatch with `final` → FAILED)
- aggregate usage across all step_end events into the ledger (missing → usage_incomplete, READY demoted to FAILED)
- treat `tool_result.status=error` with a code starting `FS_` or containing `PERMISSION` as a policy denial → record `policy_denials`, fail the run (`policy-denial: <code>`)
- parse the candidate protocol from the last non-empty line of `final.text` (same format as the OpenCode variant)

## 6. Four judgment rules (DSH-specific traps)

1. **exit 0 is not success**: policy denials and tool errors still end `completed` with exit 0. Judge by events + candidate protocol + independent tests — never the exit code.
2. **The event stream has no model identity**: verify the model by parsing the persisted session log `<DSH_HOME>/sessions/<slug>/<sessionId>/session.vN.jsonl.zstd` (`zstd -dc`, pick the largest vN): extract `provider` + `model`, compare with settings; mismatch → FAILED; log missing or unparseable → record `model_verified=false` with the reason — do not fail, do not fabricate. Note the session log may be flushed asynchronously after process exit — poll for it (up to ~15 s) before giving up.
3. **Cancel criteria**: run DSH in its own process group (e.g. Python `start_new_session=True`); SIGTERM the group first (SIGKILL on a group with dying members can raise EPERM → kill pids individually); the post-kill exit code is meaningless — the criteria are "process group gone + no filesystem change".
4. **`never` is not auto-approve**: `--approval-policy never` means "reject actions that need approval"; escalation attempts are auto-rejected — a permission denial is a failure signal, not something to route around.

## 7. Trust limitations (state honestly; never advertise as an absolute sandbox)

- The DSH FS path checks are **not a kernel security boundary**; symlink TOCTOU windows are accepted by its model
- read-only does not restrict sensitive reads or network access; there is no shell command whitelist (under workspace-write, bash can run arbitrary commands)
- therefore implement security comes from **outside**: git worktree isolation + a trusted applier that only applies authorized files + the main-repo snapshot/scan/test acceptance chain — not from the DSH sandbox itself

## 8. Usage discipline

- **No default coding tool**: DSH runs only when explicitly requested; "hermes+dsh analysis" (read-only deep reading) and controlled coding are separate paths
- DSH self-reviews are never an independent Checker; they may serve as read-only leads, with Hermes verifying key judgments at the source
- Any DSH version change requires re-running the §3 behavior checklist

## 9. Minimal acceptance checklist

- [ ] Install verified: `dsh --version`; headless smoke returns
- [ ] All six §3.2 behavior checks pass with evidence
- [ ] review: read-only write blocking, candidate protocol, usage aggregation, policy-denial→FAILED each triggered once
- [ ] implement: worktree isolation effective (main repo untouched by DSH), trusted applier copies only allowed_files, and negative tests for out-of-scope / drift / cancel each once
- [ ] model verification: both the success path (log parsed) and the fallback path (log missing) exercised
- [ ] the full behavioral matrix (same as the OpenCode variant) passes
