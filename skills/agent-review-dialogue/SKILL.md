---
name: agent-review-dialogue
description: Use when the user requests an A/B editor-reviewer dialogue, iterative document optimization, code improvement with adversarial review, interrupted edit recovery, opencode/Codex-compatible review orchestration, or human-in-the-loop refinement of specified files.
---

# Agent Review Dialogue

## Purpose

Use this skill to improve specified documents, source files, tests, configuration, or skills through a persistent Agent A / Agent B loop.

Agent A edits only the approved target scope. Agent B reviews the actual changes. The Controller owns state, validation, recovery, user communication, and final verification.

Shared files are the source of truth. Do not advance the workflow from terminal scrollback, partial output, assumptions, or an agent's verbal claim. A phase is complete only after `state.json` and the expected artifact header validate.

Stop only when B approves and Controller verification passes, a user-blocking decision appears, recovery limits are exhausted, or the configured maximum review rounds is reached.

## Use This When

Use this skill when the user asks for A/B editor-reviewer dialogue, iterative document or code optimization, adversarial review before accepting changes, interrupted-run recovery, or human-in-the-loop refinement of specified files.

Do not use it for pure planning before implementation; use `agent-plan-dialogue` instead. Do not use it for broad repository refactors unless the user provides an explicit target scope.

## Roles

- **Controller:** current session. Creates scope, writes and validates shared state, starts or emulates A/B phases, records user answers in `decisions.md`, protects unrelated changes, runs final verification, and reports results. It must not claim A edited or B reviewed until the corresponding artifact validates.
- **Agent A / Editor:** edits only files listed in `target-manifest.md` and writes `change-log.md`. It preserves user constraints, avoids unrelated refactors, runs practical verification, and uses `Status: blocked-on-user` only when a missing answer changes correctness, scope, or acceptance.
- **Agent B / Reviewer:** reviews only. It reads the actual target files, diff, `target-manifest.md`, `change-log.md`, and `decisions.md`; writes `review.md`; separates blocking findings from suggestions; and approves when no blocking issue remains. It must not edit target files.

## Shared Directory

Create one unique run directory unless the user provides an existing one:

```text
.agent-review-dialogue/<task-slug>/<session-key>/
  target-manifest.md
  change-log.md
  review.md
  decisions.md
  verification.md
  state.json
  session-ids.md
  transcript.md
```

`session-key` must be unique per Controller run, such as `<UTC timestamp>-<short-random>` or a native session id. Never reuse `.agent-review-dialogue/<task-slug>/` directly, because concurrent runs with the same slug would overwrite each other.

Shared files record exact scope and verification commands (`target-manifest.md`), A's edits and risks (`change-log.md`), B's findings (`review.md`), user decisions (`decisions.md`), Controller evidence (`verification.md`), machine state (`state.json`), runtime/session details (`session-ids.md`), and concise checkpoints (`transcript.md`).

## State Machine

`state.json` must exist before A edits anything and must be updated before and after every A, B, or Controller verification phase.

```json
{
  "task": "<task-slug>",
  "session_key": "<session-key>",
  "shared_dir": ".agent-review-dialogue/<task-slug>/<session-key>",
  "round": 1,
  "attempt": 1,
  "request_id": "<task-slug>-r001-a001-A-editor-change",
  "phase": "scope-check | editor-change | user-input | reviewer-review | editor-revision | controller-verify | approved | paused | failed",
  "editor_session_id": "",
  "reviewer_session_id": "",
  "editor_title": "A Editor - <task-slug> - <timestamp>",
  "reviewer_title": "B Reviewer - <task-slug> - <timestamp>",
  "last_successful_step": "",
  "last_agent": "A | B | controller",
  "last_error": "",
  "same_command_retries": 0,
  "same_session_retries": 0,
  "replacement_session_retries": 0,
  "updated_at": "YYYY-MM-DDTHH:MM:SSZ"
}
```

State invariants:

- `shared_dir` is the exact run directory and must appear in every A/B prompt.
- `round` starts at `1`; increment it only before A revises after `needs-revision`, or when the user requests another cycle after a pause.
- `attempt` increments before every A/B call, retry, replacement session, and Controller verification write.
- `request_id` is the freshness token: `<task-slug>-rNNN-aNNN-<A|B|controller>-<phase>`.
- `phase` describes the next expected operation.
- Fresh output requires matching `Round:`, `Attempt:`, `Request-ID:`, and an `Updated:` timestamp later than the pre-call checkpoint.
- Status values are role-specific: A uses `blocked-on-user` or `ready-for-review`; B uses `needs-revision` or `approved`; Controller verification uses `verification-passed` or `verification-failed`.

## Scope And Safety

- Create `shared_dir`, `state.json`, and `target-manifest.md` before A edits anything.
- `target-manifest.md` must list exact targets, explicit exclusions, baseline status, and known verification commands.
- A may modify only in-scope target files. If another file is required, A writes `Status: blocked-on-user` in `change-log.md` and explains the required scope change.
- Never run `git commit`, `git push`, destructive checkout/reset, or broad formatting commands unless the user explicitly requests them.
- Preserve unrelated user changes. When Git is available, use `git diff -- <target>` or equivalent file comparison to verify the changed scope.
- For code, A and B must consider tests, build, error handling, compatibility, and regressions. For C/C++, also consider ownership, RAII, exception/error safety, undefined behavior, resource lifetimes, and concrete tests.

## Markdown Artifact Safety

Coordination artifacts must be complete Markdown files, not streamed fragments.

Required artifact header, always at the top of `change-log.md`, `review.md`, and `verification.md` before any heading:

```markdown
Round: <integer matching state.json.round>
Attempt: <integer matching state.json.attempt>
Request-ID: <exact state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <role-specific status>
```

Artifact rules: keep the first five lines in the exact order above; use longer outer fences for nested fence examples; leave no unclosed fences or deferred-content wording; wrap paths, commands, statuses, and request ids in backticks; do not add coordination headers to target files unless their format explicitly requires them.

## Runtime Selection

Prefer native independent subagent/session tools and record identifiers in `session-ids.md`. If true independent subagents are unavailable, emulate separation with explicit A and B phases against shared files; do not claim review happened until B executes and `review.md` validates.

When Codex has no session ids, use stable descriptive ids such as `native-subagent-A`, `native-subagent-B`, or `not-available`, and document the limitation in `session-ids.md`. When opencode is available, create named sessions and continue them by exact session id.

### opencode Pattern

Create A first. Create B only after A produces validated `change-log.md` with `Status: ready-for-review`.

```bash
TASK_SLUG="<task-slug>"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
A_TITLE="A Editor - ${TASK_SLUG} - ${STAMP}"

opencode run --title "$A_TITLE" --dir "$PWD" --format json "<editor prompt>"
opencode session list --format json --max-count 20

B_TITLE="B Reviewer - ${TASK_SLUG} - ${STAMP}"
opencode run --title "$B_TITLE" --dir "$PWD" --format json "<reviewer prompt>"
opencode session list --format json --max-count 20

opencode run --session "<session-id>" --dir "$PWD" --format json "<next message>"
```

Resolve session ids by exact unique title match from `opencode session list --format json --max-count 20`. If the match is missing or ambiguous, do not guess; create a new unique title and record the lookup problem in `session-ids.md`.

If file attachments or session ids are unavailable, include the shared directory path and the latest relevant shared-state contents or concise summaries in the prompt.

## Phase Protocol

### Before A, B, Or Verification

Increment `attempt`, generate a fresh `request_id`, write `state.json` with the next `phase` and retry counters, append a checkpoint to `transcript.md`, and ensure every shared file exists.

### After A, B, Or Verification

Validate before moving to the next phase: expected artifact exists and is non-empty; the first five lines match the required header order; `Round:`, `Attempt:`, and `Request-ID:` match `state.json`; `Updated:` is later than the checkpoint; `Status:` is role-valid; no unresolved merge conflict markers exist; code fences are balanced; A changed only in-scope targets; `state.json` and `transcript.md` record the validated result.

If any check fails, do not continue. Run recovery for the same role and phase.

## Loop

1. Controller creates `shared_dir`, `state.json`, `target-manifest.md`, `decisions.md`, `session-ids.md`, and `transcript.md`.
2. Controller prepares `editor-change`.
3. A edits in-scope targets and writes `change-log.md`.
4. Controller validates A output and scope.
5. If A returns `blocked-on-user`, Controller asks the user, records the answer in `decisions.md`, and repeats A with a new attempt and request id.
6. If A returns `ready-for-review`, Controller prepares `reviewer-review`.
7. B reviews actual target files and diff, then writes `review.md`.
8. Controller validates B output.
9. If B returns `needs-revision`, Controller increments `round`, forwards B's blocking findings to A, and requires `Review Disposition`.
10. If B returns `approved`, Controller prepares `controller-verify`, runs verification commands, and writes `verification.md`.
11. If verification fails, Controller sends `verification.md` and B's latest blocking context back to A as a revision input.
12. If verification passes, Controller reports completion and applies the cleanup policy.
13. Stop after five review rounds unless the user explicitly asks to continue.

## Initial Prompts

### Agent A / Editor

````markdown
你是 Agent A / Editor。目标：优化用户指定的文档或代码文件，并在信息不足时提出必须确认的问题。

必须读取：
- `<shared-dir>/state.json`
- `<shared-dir>/target-manifest.md`
- `<shared-dir>/decisions.md`
- 修订阶段还必须读取 `<shared-dir>/review.md` 和 `<shared-dir>/verification.md`

允许修改：只修改 `target-manifest.md` 列出的 in-scope target files。
输出目标：完整替换 `<shared-dir>/change-log.md`。

约束：
- 不要默默假设关键需求、环境、范围或验收标准。
- 不要修改 scope 之外的文件；如必须修改，写 `Status: blocked-on-user` 并说明原因。
- 不要执行 `git commit`、`git push`、destructive reset/checkout。
- `change-log.md` 顶部必须严格使用如下 header，并匹配 `state.json`：

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <blocked-on-user | ready-for-review>
```

正文必须列出：
- `Modified Files`
- `Summary`
- `Verification Run by A`
- `Known Risks`
- `Open Questions`

修订阶段还必须添加 `Review Disposition`，逐项回应 B 的 blocking findings。
````

### Agent B / Reviewer

````markdown
你是 Agent B / Reviewer。目标：审查 Agent A 对指定文档或代码做出的实际变更。

必须读取：
- `<shared-dir>/state.json`
- `<shared-dir>/target-manifest.md`
- `<shared-dir>/change-log.md`
- `<shared-dir>/decisions.md`
- 当前目标文件内容和可用 diff

只做审查，不执行实现，不修改目标文件。
输出目标：完整替换 `<shared-dir>/review.md`。

`review.md` 顶部必须严格使用如下 header，并匹配 `state.json`：

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <needs-revision | approved>
```

审查正文使用这些小节：
- `Blocking Findings`: 必须修复的问题；需要用户输入的问题也写在这里，由 Controller 转述。
- `Scope Check`: 是否只修改了允许范围。
- `Quality Gaps`: 正确性、清晰度、安全性、测试、文档或兼容性缺口。
- `Suggestions`: 非阻塞改进。
- `Approval Rationale`: 若 `approved`，说明为什么没有剩余阻塞问题。

如果没有 blocking findings，必须写 `Status: approved`，不要为了风格偏好要求重写。
````

## Recovery

Treat network errors, provider errors, process crashes, timeouts, malformed output, missing file writes, invalid headers, failed verification, and closed terminals as recoverable until retry limits are reached.

Before each recovery attempt, increment `attempt`, generate a fresh `request_id`, keep `phase` on the phase being recovered, record the concrete failure in `state.json.last_error`, increment the relevant retry counter, and append the recovery plan to `transcript.md`.

Recovery order:

1. Retry the same command once only for clearly transient failures such as network reset, timeout, rate limit, provider unavailable, or empty stream. Increment `same_command_retries`.
2. For malformed or partial coordination output, ask the same role in the same session to repair only its expected coordination file before replacing the session.
3. For failed Controller verification, send `verification.md` and B's latest blocking context to A for revision. Do not ask B to approve failed verification.
4. Continue the same session by id when the id is known:

```bash
opencode run --session "<session-id>" --dir "$PWD" --format json "<recovery prompt>"
```

5. If the session id is unknown, inspect sessions and match the exact unique title:

```bash
opencode session list --format json --max-count 20
```

6. If same-session recovery is impossible, start a replacement session for the failed role and provide shared-state files:

```bash
ROLE_TITLE="<A Editor | B Reviewer> - ${TASK_SLUG} - replacement - $(date -u +%Y%m%dT%H%M%SZ)"
opencode run \
  --title "$ROLE_TITLE" \
  --dir "$PWD" \
  --format json \
  -f "<shared-dir>/state.json" \
  -f "<shared-dir>/target-manifest.md" \
  -f "<shared-dir>/change-log.md" \
  -f "<shared-dir>/review.md" \
  -f "<shared-dir>/decisions.md" \
  -f "<shared-dir>/verification.md" \
  -f "<shared-dir>/transcript.md" \
  "<recovery prompt>"
opencode session list --format json --max-count 20
```

If attachments are unavailable, inline the complete `state.json` and concise summaries of the shared files.

Retry limits:

- `same_command_retries`: at most 1 per failure.
- `same_session_retries`: at most 2 per phase.
- `replacement_session_retries`: at most 2 per role per round.
- After limits are exhausted, set `state.json.phase` to `paused`, summarize the blocker for the user, and wait for instruction.

## Review Standards

B must reject changes that contain:

- edits outside `target-manifest.md` scope;
- hidden assumptions about requirements, repository structure, user intent, dependencies, credentials, dates, or deployment targets;
- coordination headers that do not match `state.json`;
- placeholder or deferred-content wording;
- missing or inadequate verification;
- code changes without relevant tests or build checks when tests/build are available;
- document changes that reduce clarity, remove necessary constraints, or conflict with user decisions;
- unsafe code patterns, data-loss risk, destructive commands, or unrequested behavior changes;
- Markdown syntax that can break downstream patching or rendering;
- contradictions with local `AGENTS.md`, user constraints, or explicit decisions;
- C/C++ changes with unclear ownership, unchecked resource lifetimes, undefined behavior risk, missing error handling, or no concrete tests when relevant.

A must not resolve review feedback by deleting requirements, weakening acceptance criteria, or moving uncertainty into vague wording. If a review item is invalid, A must explain why with evidence from the user request, target files, or `decisions.md`.

## User Interaction

- Controller is the only role that talks to the user.
- Ask the user only when correctness, scope, or acceptance depends on an unknown decision.
- Prefix questions with the source, such as `A 需要确认...` or `B 发现一个阻塞问题...`.
- If the user declines to answer, record the constraint in `decisions.md` and ask A to proceed with explicit alternatives or risk notes.

## Cleanup Policy

The shared directory is intermediate state. Clean it up after successful completion unless the user explicitly asks to keep the audit trail.

Cleanup rules:

- Clean only the current run's unique `shared_dir`, never the parent `.agent-review-dialogue/` directory.
- Clean only after B has `approved`, Controller verification has passed, and the final response has captured target files, review result, verification evidence, and remaining assumptions.
- Do not clean when `state.json.phase` is `paused` or `failed`; recovery requires the shared files.
- If cleanup applies, delete only the current `shared_dir` after final response data is captured.
- If keeping audit files, report the exact `shared_dir` path and note that it is session-specific.

## Completion Output

When B approves and verification passes, report:

- target files changed;
- final `change-log.md` path and request id;
- final `review.md` path and request id;
- final `verification.md` path and request id;
- number of review rounds;
- verification commands and results;
- interruptions and recovery actions, if any;
- remaining non-blocking assumptions, if any;
- whether the session-specific shared directory was cleaned or retained;
- recommended next action, such as reviewing the diff or running broader tests.
