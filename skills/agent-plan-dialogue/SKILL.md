---
name: agent-plan-dialogue
description: Use when the user requests an A/B planner-reviewer dialogue, adversarial plan review, multi-agent planning loop, interrupted planning recovery, opencode/Codex-compatible planning orchestration, or human-in-the-loop planning before implementation.
---

# Agent Plan Dialogue

## Overview

Use this skill to orchestrate a persistent Agent A / Agent B planning dialogue before implementation. Agent A drafts and revises an executable plan; Agent B reviews adversarially for ambiguity, unsafe assumptions, missing verification, and scope drift.

Shared files are the source of truth. The controller must not advance phases based on terminal scrollback, partial agent output, or assumed success. Every transition is gated by validating `state.json` plus the expected `plan.md` or `review.md` header.

Stop only when B approves, A or B identifies a genuine user-blocking decision, recovery limits are exhausted, or the configured maximum review rounds is reached.

## When To Use

Use this skill when the user wants a plan refined through separate planner and reviewer roles before implementation. It is useful for ambiguous requirements, high-risk edits, interrupted planning sessions, or workflows that need auditable recovery state.

Do not use it for normal single-agent implementation unless the user asks for a planner-reviewer loop or the task requires explicit human-in-the-loop plan approval.

## Roles

- **Controller:** current session. Owns orchestration, phase transitions, validation, shared files, and user communication. It may summarize agent outputs but must not invent agent decisions or pretend an agent ran if no tool/session was actually executed. It records user answers in `decisions.md`.
- **Agent A / Planner:** writes complete replacement versions of `plan.md`. It must surface unknowns instead of assuming them, include concrete verification and acceptance criteria, record decisions, and mark `blocked-on-user` only when the answer changes plan correctness.
- **Agent B / Reviewer:** reviews only. It must not execute implementation or rewrite the plan. It distinguishes blocking findings from optional suggestions and must approve when no blocking issues remain.

## Shared State

Create a working directory unless the user specifies one:

```text
.agent-dialogue/<task-slug>/<session-key>/
  plan.md
  review.md
  decisions.md
  state.json
  session-ids.md
  transcript.md
```

`session-key` must be unique per controller session, for example `<UTC timestamp>-<short-random>` or a native session id. Do not reuse `.agent-dialogue/<task-slug>/` directly; concurrent runs with the same task slug would overwrite each other's coordination files.

After every round:

- `plan.md` contains A's latest complete plan.
- `review.md` contains B's latest review and status.
- `decisions.md` records user answers and accepted assumptions.
- `state.json` records machine-readable recovery state.
- `session-ids.md` records A/B session ids, titles, commands, replacements, and runtime limitations.
- `transcript.md` records concise round summaries, not full raw logs unless needed.

`state.json` must be updated before and after every agent call:

```json
{
  "task": "<task-slug>",
  "session_key": "<session-key>",
  "shared_dir": ".agent-dialogue/<task-slug>/<session-key>",
  "round": 1,
  "attempt": 1,
  "request_id": "<task-slug>-r001-a001-A-planner-draft",
  "phase": "planner-draft | user-input | reviewer-review | planner-revision | approved | paused | failed",
  "planner_session_id": "",
  "reviewer_session_id": "",
  "planner_title": "A Planner - <task-slug> - <timestamp>",
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

## State Contract

- `session_key`: unique controller-session identifier used to isolate coordination files for concurrent runs.
- `shared_dir`: exact path to this run's shared state directory. All A/B prompts must reference this path, not just the task slug.
- `round`: review cycle number. Start at 1. Increment only before A starts a revision after B returns `needs-revision`, or when the user explicitly requests another cycle after a pause.
- `attempt`: outbound agent call number within the current round. Increment before every A or B call, including retries and replacement sessions.
- `request_id`: freshness token. Regenerate before every outbound call as `<task-slug>-rNNN-aNNN-<A|B>-<phase>`.
- `phase`: current controller state. It describes the next expected operation, not the previous completed operation.
- Fresh output: the expected file starts with a header whose `Round:`, `Attempt:`, and `Request-ID:` exactly match `state.json`, and whose `Updated:` is later than the pre-call checkpoint.
- Valid status: A may write only `blocked-on-user` or `ready-for-review`; B may write only `needs-revision` or `approved`.

## Markdown Safety

- The controller must create the unique `shared_dir` before A writes anything and must never point two active runs at the same `shared_dir`.
- Agent output files must be complete Markdown documents, not streamed fragments.
- The required metadata header must be plain text at the very top of `plan.md` and `review.md`, before any Markdown heading.
- If an output includes code fences, nested prompt examples must use a longer outer fence such as four backticks when they contain triple-backtick examples.
- Do not leave an unclosed code fence in `plan.md`, `review.md`, `decisions.md`, or `transcript.md`.
- Do not use placeholders such as `TBD`, `TODO`, `same as above`, or `fill later`.
- Paths, commands, statuses, and request ids must be wrapped in backticks when mentioned in prose.

## Session Setup

### Runtime Selection

Prefer native subagent/session tools if available. If using Codex native subagents, record their identifiers in `session-ids.md`. If true independent subagents are unavailable, emulate role separation by running explicit A and B phases against the shared files; do not claim an agent reviewed or revised anything until that phase has been executed and the expected file passes validation.

When using Codex without opencode session ids, set the session id field to a stable descriptive value such as `native-subagent-A`, `native-subagent-B`, or `not-available`, and explain the runtime limitation in `session-ids.md`.

### opencode CLI Sessions

If using the opencode CLI, create named sessions with unique titles and continue them by session id. Create A first. Create B only after A has produced a `ready-for-review` plan, unless the runtime supports creating an idle session without running the reviewer prompt.

```bash
TASK_SLUG="<task-slug>"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
A_TITLE="A Planner - ${TASK_SLUG} - ${STAMP}"

opencode run --title "$A_TITLE" --dir "$PWD" --format json "<planner prompt>"
opencode session list --format json --max-count 20

# Run this only when entering reviewer-review.
B_TITLE="B Reviewer - ${TASK_SLUG} - ${STAMP}"
opencode run --title "$B_TITLE" --dir "$PWD" --format json "<reviewer prompt>"
opencode session list --format json --max-count 20

opencode run --session "<session-id>" --dir "$PWD" --format json "<next message>"
```

Extract session ids by matching the unique title in `opencode session list --format json --max-count 20`. The match must be exact and unique. If zero or multiple sessions match, do not guess; create a new unique title and record the ambiguous lookup in `session-ids.md`.

`session-ids.md` must use this format:

```markdown
# Session IDs

- A title: `<exact title>`
- A session id: `<session id or pending>`
- B title: `<exact title or pending>`
- B session id: `<session id or pending>`
- Last lookup: `<UTC ISO-8601 timestamp>`
- Notes:
  - `<command, replacement reason, ambiguity, or runtime limitation>`
```

If session ids are unavailable, each agent call must include the shared directory path plus the latest `state.json`, `plan.md`, `review.md`, `decisions.md`, and `transcript.md` content or concise summaries. State persistence through files remains mandatory.

## Agent Call Protocol

### Before Every Agent Call

1. Increment `attempt`.
2. Generate a new `request_id`.
3. Write `state.json` with the next `phase`, target agent, round number, attempt number, `request_id`, and current retry counters.
4. Append a checkpoint to `transcript.md`: message summary, expected output file, expected status, and success condition.
5. Ensure `plan.md`, `review.md`, `decisions.md`, `state.json`, `session-ids.md`, and `transcript.md` exist.

### Required Output Header

Every `plan.md` and `review.md` produced by A or B must start with this exact header order:

```markdown
Round: <integer matching state.json.round>
Attempt: <integer matching state.json.attempt>
Request-ID: <exact state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <role-specific status>
```

### After Every Agent Call

1. Verify the expected file exists and is non-empty.
2. Verify the first five lines are the required header in the required order.
3. Verify `Round:`, `Attempt:`, and `Request-ID:` exactly match `state.json`.
4. Verify `Updated:` is later than the checkpoint written before the call.
5. Verify status is role-valid: A uses `blocked-on-user` or `ready-for-review`; B uses `needs-revision` or `approved`.
6. Verify there are no unresolved merge conflict markers.
7. Verify there is no unclosed code fence if the file contains code blocks.
8. Update `state.json` with `last_successful_step`, clear `last_error`, and reset all retry counters to 0.
9. Append the result summary to `transcript.md`.

If any validation check fails, do not continue to the next phase. Treat it as a failed agent call and run the recovery flow for the same role and phase.

## Failure Handling and Recovery

Treat network errors, model provider errors, process crashes, timeouts, malformed output, missing file writes, invalid headers, or a closed terminal as recoverable unless repeated recovery proves otherwise.

Before each recovery attempt:

1. Increment `attempt`.
2. Generate a new `request_id`.
3. Update `state.json.last_error` with the concrete failure.
4. Increment the corresponding retry counter.
5. Set `phase` to the phase being recovered.
6. Append the failure and recovery attempt to `transcript.md`.

Recovery order:

1. Retry the same command once for clearly transient failures such as network reset, timeout, rate limit, provider unavailable, or empty stream. Increment `same_command_retries`.
2. For malformed or partial output, first ask the same role to repair its target file using the same session. Do not start a replacement session until repair fails or limits are reached.
3. Continue the same session by id. Increment `same_session_retries` before each attempt.

```bash
opencode run --session "<session-id>" --dir "$PWD" --format json "<recovery prompt>"
```

4. If the last session id is unknown, inspect sessions and identify the titled A/B session by exact title.

```bash
opencode session list --format json --max-count 20
```

If the title lookup is not exact and unique, skip same-session recovery and start a replacement session.

5. If the original session cannot continue, start a replacement session for the failed role. Use file attachments so the new session has recovery state:

```bash
ROLE_TITLE="<A Planner | B Reviewer> - ${TASK_SLUG} - replacement - $(date -u +%Y%m%dT%H%M%SZ)"
opencode run \
  --title "$ROLE_TITLE" \
  --dir "$PWD" \
  --format json \
  -f "<shared-dir>/state.json" \
  -f "<shared-dir>/plan.md" \
  -f "<shared-dir>/review.md" \
  -f "<shared-dir>/decisions.md" \
  -f "<shared-dir>/transcript.md" \
  "<recovery prompt>"
opencode session list --format json --max-count 20
```

If file attachments are unavailable, inline the complete `state.json` and concise summaries of `plan.md`, `review.md`, `decisions.md`, and `transcript.md` in the recovery prompt.

6. Increment `replacement_session_retries` before each replacement attempt. Update `session-ids.md` with the replacement title, session id, and reason.
7. Validate the repaired or replacement output with the same `After Every Agent Call` checks.
8. Continue the loop from the persisted `phase`.

Retry limits:

- `same_command_retries`: at most 1 per failure.
- `same_session_retries`: at most 2 per phase.
- `replacement_session_retries`: at most 2 per role per round.
- After limits are exhausted, set `state.json.phase` to `paused`, summarize the blocker for the user, and wait for instruction.

Recovery prompt template:

````markdown
继续之前中断的 Agent Plan Dialogue。

角色：Agent <A Planner | B Reviewer>
共享目录：`<shared-dir>`
恢复依据：
- 读取 `<shared-dir>/state.json`
- 读取 `<shared-dir>/plan.md`
- 读取 `<shared-dir>/review.md`
- 读取 `<shared-dir>/decisions.md`
- 读取 `<shared-dir>/transcript.md`

要求：
- 不要从头重做已完成步骤。
- 从 `state.json.last_successful_step` 之后继续。
- 如果你的上一次输出可能只完成了一半，先检查目标文件并修复为完整格式。
- 完成后写入对应文件，并给出合法 `Status:`。
- 输出文件顶部必须写入与 `state.json` 完全一致的 `Round:`、`Attempt:`、`Request-ID:`，并写入新的 `Updated:`。
````

## Initial Prompts

Planner prompt requirements:

````markdown
你是 Agent A / Planner。目标：根据用户需求编写可执行计划，并在信息不足时提出必须确认的问题。

必须读取：
- `<shared-dir>/state.json`
- `<shared-dir>/decisions.md`
- 如果是修订阶段，读取 `<shared-dir>/review.md`

输出目标：`<shared-dir>/plan.md`

约束：
- 不要默默假设关键需求、环境、范围或验收标准。
- `plan.md` 必须是完整替换文件，不要盲目追加重复内容。
- `plan.md` 顶部必须严格使用如下 header，并匹配 `state.json`：

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <blocked-on-user | ready-for-review>
```

- 如果存在真正阻塞计划正确性的用户问题，在 `Status:` 写 `blocked-on-user`，并列出最多 5 个具体问题。
- 如果可以继续，在 `Status:` 写 `ready-for-review`。
- 计划必须包含具体文件、步骤、验证命令、预期结果、验收标准，以及相关风险或回滚说明。
- 如果这是对 B 的修订响应，添加 `Review Disposition` 并逐项回应 B 的 blocking findings。
````

Reviewer prompt requirements:

````markdown
你是 Agent B / Reviewer。目标：审查 `<shared-dir>/plan.md`，找出会导致实现偏差、返工或无法验收的问题。

必须读取：
- `<shared-dir>/state.json`
- `<shared-dir>/plan.md`
- `<shared-dir>/decisions.md`

只做审查，不执行实现，不重写计划。

输出目标：`<shared-dir>/review.md`

`review.md` 顶部必须严格使用如下 header，并匹配 `state.json`：

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <needs-revision | approved>
```

审查正文使用这些小节：
- `Blocking Findings`: 必须修复的问题；可能需要用户输入的问题也写在这里，由 Controller 转述。
- `Assumptions`: 计划中未经确认的假设。
- `Gaps`: 缺失的任务、测试、验收标准、回滚方案或边界条件。
- `Suggestions`: 非阻塞改进。
- `Approval Rationale`: 若 `approved`，说明为什么没有剩余阻塞问题。

如果没有 blocking findings，必须写 `Status: approved`，不要为了风格偏好要求重写。
````

## Loop

1. Controller prepares `planner-draft` or `planner-revision`.
2. A writes `plan.md`.
3. Controller validates A output.
4. If A is `blocked-on-user`, Controller asks the user and records answers in `decisions.md`; then repeats A phase with a new `attempt` and `request_id`.
5. If A is `ready-for-review`, Controller prepares `reviewer-review`.
6. B writes `review.md`.
7. Controller validates B output.
8. If B is `needs-revision`, Controller increments `round`, forwards B's review verbatim to A, and requires `Review Disposition`.
9. If B is `approved`, Controller reports completion.
10. Controller runs cleanup according to `Cleanup Policy`.
11. If any call fails, run the recovery flow and resume from `state.json.phase`.
12. Stop after 5 review rounds unless the user explicitly asks to continue.

## Review Standards

B must reject plans that contain:

- hidden assumptions about requirements, repository structure, user intent, dependencies, credentials, dates, or deployment targets;
- a header that does not match `state.json`;
- placeholders such as `TBD`, `TODO`, `handle edge cases`, `add tests`, or `similar to above`;
- missing exact file paths for changes;
- missing verification commands or expected results;
- implementation steps that cannot be executed independently;
- proposed changes outside the user's scope;
- implementation work when the user requested planning only;
- Markdown syntax that can break downstream patching;
- assumptions about unavailable tools, credentials, package managers, or network access;
- contradictions with local `AGENTS.md`, user constraints, or explicit decisions;
- C/C++ plans without concrete tests, resource ownership strategy, error handling, and safety considerations when relevant.

A must not resolve review feedback by deleting requirements, weakening acceptance criteria, or moving uncertainty into vague wording. If a review item is invalid, A must explain why using evidence from the user request, codebase, or `decisions.md`.

## User Interaction Rules

- The controller talks to the user; A and B do not ask the user directly.
- Ask the user only when plan correctness depends on an unknown decision.
- Show the source of the question: `A 需要确认...` or `B 发现一个阻塞问题...`.
- If the user declines to answer, record the constraint and ask A to produce a plan with explicit alternatives or risk notes.

## Cleanup Policy

The shared directory is intermediate state. Clean it up after successful completion unless the user explicitly asks to keep the audit trail.

Cleanup rules:

- Clean only the current run's unique `shared_dir`, never the parent `.agent-dialogue/` directory.
- Clean only after B has `approved`, the controller has reported the final plan/review request ids, and no recovery state is needed.
- Do not clean when `state.json.phase` is `paused` or `failed`; recovery requires the shared files.
- If cleanup is requested or default cleanup applies, delete the current `shared_dir` after the final response data has been captured.
- If keeping audit files, report the exact `shared_dir` path and remind that it is session-specific.

## Completion Output

When B approves, report:

- final `plan.md` path and request id;
- final `review.md` path and request id;
- number of review rounds;
- whether the final plan passed freshness validation;
- any interruptions and how they were recovered;
- remaining non-blocking assumptions, if any;
- whether the session-specific shared directory was cleaned or retained;
- recommended next action as an option, such as executing the plan or saving it for later.
