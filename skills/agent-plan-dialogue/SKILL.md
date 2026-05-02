---
name: agent-plan-dialogue
description: Use when the user requests an A/B planner-reviewer dialogue, adversarial plan review, multi-agent planning loop, interrupted planning recovery, opencode/Codex-compatible planning orchestration, or human-in-the-loop planning before implementation.
---

# Agent Plan Dialogue

## Overview

Use this skill to orchestrate a persistent Agent A / Agent B planning dialogue before implementation. Agent A drafts and revises an executable plan; Agent B reviews adversarially for ambiguity, unsafe assumptions, missing verification, and scope drift.

Shared files are the source of truth. The controller must not advance phases based on terminal scrollback, partial agent output, or assumed success. Every transition is gated by validating `state.json` plus the expected `plan.md`, `review.md`, or `verification.md` header.

Stop only when B approves, A's follow-up double-check reports no further plan updates, and Controller verification passes; or when A or B identifies a genuine user-blocking decision, recovery limits are exhausted, or the configured maximum B review/revision cycle limit is reached before approval or before another B review can be opened.

## When To Use

Use this skill when the user wants a plan refined through separate planner and reviewer roles before implementation. It is useful for ambiguous requirements, high-risk edits, interrupted planning sessions, or workflows that need auditable recovery state.

Do not use it for normal single-agent implementation unless the user asks for a planner-reviewer loop or the task requires explicit human-in-the-loop plan approval.

### Self-Application

When this skill is itself the planning target, follow the same planning protocol. The Controller must freeze the active run's scope, state, role prompts, request id, and allowed writes before A writes `plan.md`. A may only describe proposed changes to `skills/agent-plan-dialogue/SKILL.md` inside `plan.md`; it must not edit the skill file during this planning workflow.

Self-optimization must preserve the role split. A may plan clarifications to the skill text, examples, validation rules, and prompt templates, but those changes are only proposals until a later implementation/editor-review workflow runs or the user explicitly asks the current Controller to implement them outside this planning protocol. B reviews the proposed plan for changing the skill, not an actual target-file diff, and may not rewrite `plan.md` or the skill file. If self-application reveals that the active run needs a different target scope, cleanup decision, acceptance criterion, or verification command, A or B must surface that through its artifact instead of changing `state.json` directly.

## Roles

- **Controller:** current session. Owns orchestration, phase transitions, validation, shared files, and user communication. It may summarize agent outputs but must not invent agent decisions or pretend an agent ran if no tool/session was actually executed. It records user answers in `decisions.md`. It is the only role that updates `state.json`, `session-ids.md`, `transcript.md`, `decisions.md`, or `verification.md`.
- **Agent A / Planner:** writes complete replacement versions of `plan.md`. It treats all other coordination files as read-only inputs, must surface unknowns instead of assuming them, include concrete verification and acceptance criteria, record decisions in the plan body, and mark `blocked-on-user` only when the answer changes plan correctness. After B approves, A must run one double-check pass before Controller verification and write `no-change` only when rereading the current plan, review, decisions, and scope reveals no further updates.
- **Agent B / Reviewer:** reviews only. It must not execute implementation or rewrite the plan. It reads `plan.md`, `decisions.md`, prior review/verification context when present, and any user-specified scope; first reviews the concrete plan for ambiguity and executability, then performs a broader adversarial pass over the full proposed scope for missed requirements, unsafe assumptions, stale context, and verification gaps. It treats `review.md` as its only writable file and must approve only after both passes find no blocking issue.

Role boundaries are strict even when one human or model emulates multiple roles in one session. A role may read shared artifacts needed for its phase, but it may write only its assigned artifact. The Controller is responsible for advancing state between phases; A and B must copy current state values into artifacts and must not invent new rounds, attempts, phases, request ids, cleanup decisions, or verification outcomes.

## Shared Directory

Create one unique run directory unless the user specifies an existing one:

```text
.agent-dialogue/<task-slug>/<session-key>/
  plan.md
  review.md
  verification.md
  decisions.md
  state.json
  session-ids.md
  transcript.md
```

`session-key` must be unique per controller session, for example `<UTC timestamp>-<short-random>` or a native session id. Do not reuse `.agent-dialogue/<task-slug>/` directly; concurrent runs with the same task slug would overwrite each other's coordination files.

Shared files record A's latest complete plan (`plan.md`), B's latest review (`review.md`), Controller verification evidence (`verification.md`), user answers and accepted constraints (`decisions.md`), machine-readable recovery state (`state.json`), runtime/session details (`session-ids.md`), and concise checkpoints (`transcript.md`).

Controller initialization must happen before any A/B prompt is sent:

1. Create the exact `shared_dir` path.
2. Create `plan.md`, `review.md`, `verification.md`, `decisions.md`, `state.json`, `session-ids.md`, and `transcript.md`.
3. Record the user request, known constraints, planned runtime, and any explicit exclusions in `decisions.md` or `transcript.md`.
4. Write `state.json` with `phase` set to the next expected operation.
5. Reference the exact `shared_dir` in every A/B prompt and recovery prompt.

`state.json` must exist before A writes anything and must be updated before and after every A, B, or Controller verification phase:

```json
{
  "task": "<task-slug>",
  "session_key": "<session-key>",
  "shared_dir": ".agent-dialogue/<task-slug>/<session-key>",
  "round": 1,
  "attempt": 1,
  "request_id": "<task-slug>-r001-a001-A-planner-draft",
  "phase": "planner-draft | user-input | reviewer-review | planner-revision | planner-double-check | controller-verify | approved | paused | failed",
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
- `round`: planning phase sequence number. Start at 1. Increment only before A starts a revision after B returns `needs-revision`, before A runs a double-check after B approval, before A starts a revision after Controller verification returns `verification-failed`, or when the user explicitly requests another cycle after a pause. Every increment changes the `rNNN` segment used in later `request_id` values.
- Maximum review rounds: the configured cap applies to B review/revision cycles, not to the mandatory post-approval `planner-double-check` and `controller-verify` completion gates. If B approves on the final allowed review cycle, Controller must still run one `planner-double-check` and Controller verification. If that double-check changes `plan.md` and would require another B review beyond the cap, pause and ask the user whether to extend the limit. Verification-failure revisions count as new review/revision cycles because they must return to B after A revises.
- `attempt`: outbound phase number within the current round. Increment before every A call, B call, retry, replacement session, and Controller verification write. The Controller increments it; A and B copy the current values into their artifact headers and do not self-advance state.
- `request_id`: freshness token. Regenerate before every outbound phase as `<task-slug>-rNNN-aNNN-<A|B|controller>-<phase>`.
- `phase`: current controller state. It describes the next expected operation, not the previous completed operation.
- Fresh output: the expected file starts with a header whose `Round:`, `Attempt:`, and `Request-ID:` exactly match `state.json`, and whose `Updated:` is later than the Controller's pre-call checkpoint.
- Valid status: A may write `blocked-on-user` or `ready-for-review` during `planner-draft` and `planner-revision`; in `planner-double-check`, A may write only `ready-for-review` or `no-change`; B may write only `needs-revision` or `approved`; Controller verification may write only `verification-passed` or `verification-failed`.
- `last_successful_step` records the most recent validated phase, not the most recent attempted command.

## Scope And Safety

- Create `shared_dir`, `state.json`, and all coordination files before A writes anything.
- A may modify only `plan.md`. B may modify only `review.md`. Controller may modify only coordination files in the current `shared_dir`, unless the user explicitly expands scope.
- If planning requires repository inspection, A and B may read relevant files but must not edit implementation targets during planning. If another writable artifact or repository file becomes necessary, A writes `Status: blocked-on-user` in `plan.md` and explains the required scope expansion.
- Never run `git commit`, `git push`, destructive checkout/reset, or implementation commands unless the user explicitly requests them.
- Preserve unrelated user changes. When Git is available, use `git diff -- <target>` or equivalent file comparison before recommending implementation over changed files, and use `git diff --name-only` or equivalent after each role phase to confirm only allowed files changed.
- For C/C++ plans, A and B must consider ownership, RAII, exception/error safety, undefined behavior, resource lifetimes, build/test commands, and concrete tests.

## Markdown Safety

- The controller must create the unique `shared_dir` before A writes anything and must never point two active runs at the same `shared_dir`.
- Agent output files must be complete Markdown documents, not streamed fragments.
- The required metadata header must be plain text at the very top of `plan.md`, `review.md`, and `verification.md`, before any Markdown heading.
- If an output includes code fences, nested prompt examples must use a longer outer fence such as four backticks when they contain triple-backtick examples.
- Do not leave an unclosed code fence in `plan.md`, `review.md`, `verification.md`, `decisions.md`, or `transcript.md`.
- Do not use unresolved placeholders such as `TBD`, `TODO`, `same as above`, or `fill later`. The literal value `pending` is allowed only in templates or runtime bookkeeping fields such as `session-ids.md` while a session or title has not been created yet; it must not remain as unresolved final content in `plan.md`, `review.md`, `verification.md`, or the final user-facing artifact.
- Paths, commands, statuses, and request ids must be wrapped in backticks when mentioned in prose.

Validation checklist:

- The expected artifact exists, is complete, and starts at byte one with the five-line header.
- `Round:`, `Attempt:`, and `Request-ID:` exactly match current `state.json`.
- `Updated:` is UTC ISO-8601 and later than the Controller's pre-call checkpoint.
- `Status:` is valid for the role and matches the next workflow branch.
- Markdown fences are balanced, no unresolved merge conflict markers exist, and no unresolved placeholder text such as `pending`, `TODO later`, or `to be filled` remains. Treat `pending` as valid only when it appears in a documented template/runtime field that is expected to be unresolved at that phase.
- Scope checks confirm only allowed coordination files changed for the current role.

Validation must inspect files on disk after the role finishes. A role's terminal summary, exit status, or claim that a file was written is not enough. If any required artifact is missing, stale, malformed, or written outside the role's authority, the Controller must keep the same phase and run recovery instead of proceeding.

## Session Setup

### Runtime Selection

Prefer native subagent/session tools if available. If using Codex native subagents, record their identifiers in `session-ids.md`. If true independent subagents are unavailable, emulate role separation by running explicit A and B phases against the shared files; do not claim an agent reviewed or revised anything until that phase has been executed and the expected file passes validation.

When using Codex without opencode session ids, set the session id field to a stable descriptive value such as `native-subagent-A`, `native-subagent-B`, or `not-available`, and explain the runtime limitation in `session-ids.md`.

### Environment Agent Check

Before creating or replacing any A/B subagent/session, inspect the current environment for predefined agents. Use the platform's native agent discovery/listing mechanism when available.

- Run discovery before the first A session and again before creating B or any replacement session, because available predefined agents may differ by runtime or after a handoff.
- Prefer explicit platform metadata over name inference. In Codex, use native tool/plugin metadata or `tool_search` when available to discover predefined agents. In opencode, run the supported agent-listing command for the installed version, such as `opencode agent list --format json`; if the command is unavailable or returns non-JSON output, record discovery as unavailable and continue normally.
- Treat discovery as read-only. Do not install plugins, mutate runtime configuration, or change repository files as part of this check.
- If a predefined agent named exactly `dialogue-designer` exists, create Agent A / Planner with that predefined agent.
- If a predefined agent named exactly `dialogue-reviewer` exists, create Agent B / Reviewer with that predefined agent.
- If only one predefined agent exists, use it only for its matching role and create the other role normally.
- If neither predefined agent exists, if agent discovery is unavailable, or if the name match is ambiguous, create A/B normally.
- Do not use a similarly named predefined agent. Only exact names are valid.
- Record the discovery method, command or tool used, raw match names or summary, planned predefined-agent choice for each role, actual session creation mechanism used for each role, fallback reason, and timestamp in `session-ids.md`. Keep discovery results separate from successful use: finding `dialogue-designer` or `dialogue-reviewer` is not enough to claim the role used that predefined agent unless the created or continued session was actually launched with it.

Predefined agents do not replace the skill protocol. Always pass the same shared directory, freshness token, role prompt, planning scope, and artifact requirements to A/B, and validate their outputs exactly as usual.

Binding rules:

- If the runtime supports choosing a predefined agent for a native subagent/session, bind A to `dialogue-designer` and B to `dialogue-reviewer` using that runtime's explicit selector.
- If using opencode and it supports an agent selector, put the selector in the A/B command at the `<A-agent-binding-args>` or `<B-agent-binding-args>` placeholder shown below.
- If the runtime has predefined agents but no supported way to bind them to the actual A/B call, continue with normal A/B sessions and record the unsupported binding as a fallback.
- `session-ids.md` must record, for each role, the discovered predefined agent name, whether it was bound, the actual invocation method or binding arguments, the resulting session id when available, and the fallback reason when not bound.

### opencode CLI Sessions

If using the opencode CLI, create named sessions with unique titles and continue them by session id. Create A first. Create B only after A has produced a `ready-for-review` plan, unless the runtime supports creating an idle session without running the reviewer prompt.

```bash
TASK_SLUG="<task-slug>"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
A_TITLE="A Planner - ${TASK_SLUG} - ${STAMP}"

opencode run <A-agent-binding-args> --title "$A_TITLE" --dir "$PWD" --format json "<planner prompt>"
opencode session list --format json --max-count 20

# Run this only when entering reviewer-review.
B_TITLE="B Reviewer - ${TASK_SLUG} - ${STAMP}"
opencode run <B-agent-binding-args> --title "$B_TITLE" --dir "$PWD" --format json "<reviewer prompt>"
opencode session list --format json --max-count 20

opencode run --session "<session-id>" --dir "$PWD" --format json "<next message>"
```

Extract session ids by matching the unique title in `opencode session list --format json --max-count 20`. The match must be exact and unique. If zero or multiple sessions match, do not guess; create a new unique title and record the ambiguous lookup in `session-ids.md`.

Use empty binding placeholders only after recording that opencode has no supported selector or no exact predefined-agent match. Do not silently drop a discovered predefined agent from the invocation.

`session-ids.md` must use this format:

```markdown
# Session IDs

- A title: `<exact title>`
- A session id: `<session id or pending>`
- A predefined agent: `<dialogue-designer | none | unavailable>`
- B title: `<exact title or pending>`
- B session id: `<session id or pending>`
- B predefined agent: `<dialogue-reviewer | none | unavailable>`
- Agent discovery method: `<native mechanism, command, or unavailable>`
- Agent discovery result: `<exact matches, no exact matches, ambiguous, or unavailable>`
- A creation mechanism: `<predefined-agent | normal-session | emulated-phase | unavailable>`
- B creation mechanism: `<predefined-agent | normal-session | emulated-phase | unavailable>`
- Agent discovery fallback reason: `<none, unavailable, ambiguous, not found, or role mismatch>`
- Last lookup: `<UTC ISO-8601 timestamp>`
- Notes:
  - `<command, replacement reason, ambiguity, or runtime limitation>`
```

If session ids are unavailable, each agent call must include the shared directory path plus the latest `state.json`, `plan.md`, `review.md`, `verification.md`, `decisions.md`, and `transcript.md` content or concise summaries. State persistence through files remains mandatory.

## Phase Protocol

### Before A, B, Or Verification

1. Controller increments `attempt`.
2. Controller generates a new `request_id`.
3. Controller writes `state.json` with the next `phase`, target role, round number, attempt number, `request_id`, and current retry counters.
4. Controller appends a UTC checkpoint to `transcript.md`: message summary, expected output file, expected status, and success condition.
5. Controller ensures `plan.md`, `review.md`, `verification.md`, `decisions.md`, `state.json`, `session-ids.md`, and `transcript.md` exist.
6. A/B prompts must include the exact `shared_dir`, current `request_id`, allowed writes, required reads, required artifact header, and any user decisions from `decisions.md`.

### Required Output Header

Every `plan.md`, `review.md`, and `verification.md` must start with this exact header order:

```markdown
Round: <integer matching state.json.round>
Attempt: <integer matching state.json.attempt>
Request-ID: <exact state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <role-specific status>
```

### After A, B, Or Verification

1. Verify the expected file exists and is non-empty.
2. Verify the first five lines are the required header in the required order.
3. Verify `Round:`, `Attempt:`, and `Request-ID:` exactly match `state.json`.
4. Verify `Updated:` is later than the checkpoint written before the call.
5. Verify status is role-valid: A uses `blocked-on-user` or `ready-for-review` in `planner-draft` and `planner-revision`; A uses `ready-for-review` or `no-change` in `planner-double-check`; B uses `needs-revision` or `approved`; Controller verification uses `verification-passed` or `verification-failed`.
6. Verify there are no unresolved merge conflict markers.
7. Verify there is no unclosed code fence if the file contains code blocks.
8. Verify the changed files are allowed for the current role.
9. Update `state.json` with `last_successful_step`, clear `last_error`, and reset all retry counters to 0.
10. Append the result summary to `transcript.md`.

If any validation check fails, do not continue to the next phase. Treat it as a failed phase and run recovery for the same role and phase.

## Controller Verification

Controller verification is a planning-quality gate, not implementation. It must not execute the planned implementation steps unless the user explicitly requested implementation.

When B writes `Status: approved`, Controller must increment `round` and prepare `planner-double-check` before verification. A must reread the latest `plan.md`, B's approved `review.md`, `decisions.md`, and known scope, then either revise `plan.md` with `Status: ready-for-review` or confirm no more plan changes with `Status: no-change`.

When A's double-check writes `Status: ready-for-review`, Controller must validate `plan.md` and return to `reviewer-review`; B must review the new plan and full proposed scope again. When A's double-check writes `Status: no-change`, Controller must prepare `controller-verify`, write a fresh `state.json`, and complete `verification.md` with the required header and one of these statuses:

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <verification-passed | verification-failed>
```

- `verification-passed`: B is approved, A double-check returned `no-change`, freshness checks pass, no blocking placeholders or malformed Markdown remain, the plan is executable, verification commands and expected results are concrete, user decisions are reflected, and no unresolved recovery state is required.
- `verification-failed`: any freshness, syntax, state, scope, user-decision, or plan completeness check fails.

`verification.md` body must include:

- `Checks Run`: exact commands or manual checks performed by Controller.
- `Results`: pass/fail result for each check.
- `Blocking Issues`: issues that require A revision; write `None` only when `Status: verification-passed`.
- `Non-Blocking Notes`: residual assumptions or follow-up notes.

If verification fails, Controller must not report completion. Before preparing the next `planner-revision`, increment `round`, regenerate `request_id` with the new `rNNN` value, send `verification.md` plus the latest `review.md` to A, and require `Verification Disposition` to address the verification failures. Because the revised plan must return to B, this path consumes another B review/revision cycle and is subject to the configured cap before that next B review opens.

If verification passes, Controller must validate `verification.md`, update `state.json.phase` to `approved`, set `last_successful_step` to `controller-verify`, append the result to `transcript.md`, and only then report completion or apply cleanup.

## Failure Handling and Recovery

Treat network errors, model provider errors, process crashes, timeouts, malformed output, missing file writes, invalid headers, failed verification, or a closed terminal as recoverable unless repeated recovery proves otherwise.

Before each recovery attempt:

1. Increment `attempt`.
2. Generate a new `request_id`.
3. Update `state.json.last_error` with the concrete failure.
4. Increment the corresponding retry counter.
5. Set `phase` to the phase being recovered.
6. Append the failure and recovery attempt to `transcript.md`.

Recovery routing:

- If A's artifact is missing, malformed, stale, or edits disallowed files, recover A for `planner-draft` or `planner-revision`.
- If B's artifact is missing, malformed, stale, or edits disallowed files, recover B for `reviewer-review`.
- If A's double-check artifact is missing, malformed, stale, or edits disallowed files, recover A for `planner-double-check`.
- If Controller verification fails after B approved and A double-check returned `no-change`, route back to A with `verification.md` as revision input; do not ask B to reinterpret a failed verification as approval.
- If a required scope, acceptance, or planning decision is discovered, set `phase` to `user-input`, record the blocker, and ask the user through Controller.

Recovery order:

1. Retry the same command once for clearly transient failures such as network reset, timeout, rate limit, provider unavailable, or empty stream. Increment `same_command_retries`.
2. For malformed or partial output, first ask the same role to repair only its expected coordination file using the same session. Do not start a replacement session until repair fails or limits are reached. The repair prompt must forbid implementation edits.
3. For failed Controller verification, send `verification.md` and the latest `review.md` to A for revision. Require A to add `Verification Disposition` to `plan.md`, with one entry per failed check and the updated plan or blocker. Do not ask B to approve a plan that failed Controller verification.
4. Continue the same session by id. Increment `same_session_retries` before each attempt.

```bash
opencode run --session "<session-id>" --dir "$PWD" --format json "<recovery prompt>"
```

5. If the last session id is unknown, inspect sessions and identify the titled A/B session by exact title.

```bash
opencode session list --format json --max-count 20
```

If the title lookup is not exact and unique, skip same-session recovery and start a replacement session.

6. If the original session cannot continue, rerun the environment agent check for the failed role, record the selection or fallback in `session-ids.md`, then start a replacement session with the matching predefined agent when available and provide shared-state files:

```bash
ROLE_TITLE="<A Planner | B Reviewer> - ${TASK_SLUG} - replacement - $(date -u +%Y%m%dT%H%M%SZ)"
opencode run \
  --title "$ROLE_TITLE" \
  --dir "$PWD" \
  --format json \
  -f "<shared-dir>/state.json" \
  -f "<shared-dir>/plan.md" \
  -f "<shared-dir>/review.md" \
  -f "<shared-dir>/verification.md" \
  -f "<shared-dir>/decisions.md" \
  -f "<shared-dir>/transcript.md" \
  "<recovery prompt>"
opencode session list --format json --max-count 20
```

If file attachments are unavailable, inline the complete `state.json` and concise summaries of `plan.md`, `review.md`, `verification.md`, `decisions.md`, and `transcript.md` in the recovery prompt.

7. Increment `replacement_session_retries` before each replacement attempt. Update `session-ids.md` with the replacement title, session id, and reason.
8. Validate the repaired or replacement output with the same `After A, B, Or Verification` checks.
9. Continue the loop from the persisted `phase`.

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
- 读取 `<shared-dir>/verification.md`
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

共享目录：`<shared-dir>`
当前 request_id：`<state.json.request_id>`

必须读取：
- `<shared-dir>/state.json`
- `<shared-dir>/decisions.md`
- `<shared-dir>/transcript.md`
- 如果 `<shared-dir>/plan.md` 已存在且非空，读取它以避免重复或丢失已确认内容。
- 如果是修订阶段，读取 `<shared-dir>/review.md` 和 `<shared-dir>/verification.md`（如果存在）。

输出目标：`<shared-dir>/plan.md`
允许修改：只完整替换 `<shared-dir>/plan.md`；不要修改 `state.json`、`review.md`、`verification.md`、`decisions.md`、`session-ids.md` 或 `transcript.md`。

约束：
- 不要默默假设关键需求、环境、范围或验收标准。
- `plan.md` 必须是完整替换文件，不要盲目追加重复内容。
- 不要执行实现步骤，不要修改计划目标文件，不要运行会改变仓库、依赖、生成物、外部服务或用户数据的命令。
- 不要执行 `git commit`、`git push`、destructive reset/checkout。
- 如果规划对象就是本 skill，仍以 Controller 提供的本轮 `state.json`、request id、允许写入范围和 artifact 要求为准；只能在 `plan.md` 中提出对 skill 的修改计划，不要编辑 skill 文件，也不要把计划中的新规则当作本轮状态变更。
- `plan.md` 顶部必须严格使用如下 header，并匹配 `state.json`：

For `planner-draft` and `planner-revision`, use:

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <blocked-on-user | ready-for-review>
```

Only in `planner-double-check`, use:

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <ready-for-review | no-change>
```

- 如果存在真正阻塞计划正确性的用户问题，在 `Status:` 写 `blocked-on-user`，并列出最多 5 个具体问题。
- 如果可以继续审查，在 `Status:` 写 `ready-for-review`。
- 仅在 `planner-double-check` 中，如果复查后没有更多计划更新，写 `Status: no-change`，并说明已完成复查。
- 计划必须包含具体文件、执行步骤、验证命令、每个验证命令的预期结果、验收标准、风险、回滚或恢复说明。
- 如果计划依赖外部工具、凭据、网络、包管理器、平台版本或未确认的用户偏好，必须明确列为阻塞问题、前置条件或可替代路径，不能写成隐含假设。
- 如果这是对 B 的修订响应，添加 `Review Disposition` 并逐项回应 B 的 blocking findings。
- 如果这是 Controller verification 失败后的修订响应，添加 `Verification Disposition` 并逐项回应 `verification.md` 中的 blocking issues。
- 如果这是 `planner-double-check`，添加 `Double-Check Result`：如果还有任何更新，修改计划并写 `Status: ready-for-review`；如果没有更多更新，写 `Status: no-change`。
````

Reviewer prompt requirements:

````markdown
你是 Agent B / Reviewer。目标：审查 `<shared-dir>/plan.md`，找出会导致实现偏差、返工或无法验收的问题。

共享目录：`<shared-dir>`
当前 request_id：`<state.json.request_id>`

必须读取：
- `<shared-dir>/state.json`
- `<shared-dir>/plan.md`
- `<shared-dir>/decisions.md`
- `<shared-dir>/transcript.md`
- 如果是修订阶段或正在审查 revised plan，还必须读取最新的 `<shared-dir>/review.md` 和 `<shared-dir>/verification.md`（如果存在），并核对 A 的 `Review Disposition` 是否覆盖了 B 的 blocking findings 以及 Controller verification 的 blocking issues。

只做审查，不执行实现，不重写计划。可以读取相关仓库文件来验证计划前提，但不要修改任何计划目标文件、协调文件之外的文件、生成物、依赖或外部服务状态。
输出目标：`<shared-dir>/review.md`
允许修改：只完整替换 `<shared-dir>/review.md`；不要修改 `plan.md`、`state.json`、`verification.md`、`decisions.md`、`session-ids.md` 或 `transcript.md`。

审查顺序：
1. 先重点审查 `plan.md` 的具体方案，确认它是否准确覆盖用户需求、已记录决策、文件范围、执行步骤、验收标准和验证命令。
2. 再对完整拟实施范围做对抗性复查，主动寻找计划未提及的遗漏需求、陈旧上下文、隐含假设、范围漂移、回滚缺口、测试缺口和验收歧义。
3. 最后才决定 `Status`。不得在完成整体复查前提前给出 `approved` 结论。
4. 如果目标文件或 repository context 不足以判断计划正确性，必须在 `Blocking Findings` 或 `Gaps` 中说明需要 Controller 获取的信息。
5. 如果计划包含实现命令、验证命令或清理命令，只审查其安全性和可执行性；不要替 A 执行这些命令。

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
- `Plan Review`: 针对 `plan.md` 具体方案的审查结论。
- `Adversarial Full-Pass Review`: 针对完整拟实施范围的继续对抗性审查结论。
- `Assumptions`: 计划中未经确认的假设。
- `Gaps`: 缺失的任务、测试、验收标准、回滚方案或边界条件。
- `Suggestions`: 非阻塞改进。
- `Approval Rationale`: 若 `approved`，说明为什么没有剩余阻塞问题。

如果没有 blocking findings，必须写 `Status: approved`，不要为了风格偏好要求重写。
````

## Loop

1. Controller creates the unique `shared_dir`, initializes all shared files, and writes `state.json`.
2. Controller runs the environment agent check, records the result in `session-ids.md`, and prepares `planner-draft` or `planner-revision`.
3. A writes `plan.md`.
4. Controller validates A output, artifact freshness, and changed-file scope.
5. If A is `blocked-on-user`, Controller asks the user and records answers in `decisions.md`; then repeats A phase with a new `attempt` and `request_id`.
6. If A is `ready-for-review`, Controller reuses or refreshes the environment agent check as needed, records the B selection in `session-ids.md`, and prepares `reviewer-review`.
7. B reviews `plan.md` in two passes: first the concrete plan, then the full proposed implementation scope adversarially for issues outside A's stated approach; then writes `review.md`.
8. Controller validates B output and confirms B did not modify `plan.md` or other coordination files.
9. If B is `needs-revision`, Controller increments `round`, forwards B's blocking findings to A, and requires `Review Disposition`.
10. If B is `approved`, Controller increments `round` and prepares `planner-double-check`; A rereads the current plan, B's approved review, decisions, and scope before deciding whether more updates remain.
11. If A's double-check is `ready-for-review`, Controller validates A output and returns to `reviewer-review`; B must review the updated plan and full proposed scope again.
12. If A's double-check is `no-change`, Controller prepares `controller-verify`, runs planning-quality checks, and writes `verification.md`.
13. If Controller verification is `verification-failed`, Controller increments `round` before the next A revision, forwards `verification.md` and the latest `review.md` to A, and requires `Verification Disposition`.
14. If Controller verification is `verification-passed`, Controller validates `verification.md`, updates `state.json.phase` to `approved`, records the validated result in `transcript.md`, then reports completion and applies `Cleanup Policy`.
15. If any phase fails, run the recovery flow and resume from `state.json.phase`.
16. Stop after 5 B review/revision cycles unless the user explicitly asks to continue. This cap does not block the mandatory `planner-double-check` and `controller-verify` after B approves; however, if the post-approval double-check changes `plan.md` and would require another B review beyond the cap, pause and ask the user whether to extend the limit.

`Status: no-change` is valid only for A's `planner-double-check` phase. In normal `planner-draft` or `planner-revision`, A must use `ready-for-review` after the plan is complete or `blocked-on-user` when a required decision is missing. A double-check that changes `plan.md` must use `ready-for-review`, because B must review the updated plan before Controller verification.

## Review Standards

B must use two review passes every round:

1. Plan-first review: inspect the actual `plan.md` against `decisions.md`, prior feedback, user constraints, and any known repository context.
2. Adversarial full-pass review: reread the full proposed implementation scope and actively look for remaining issues that A did not mention, including problems outside the immediate planning approach.

B must not mark `Status: approved` until both passes are complete and no blocking issue remains. If the concrete plan is coherent but the full-pass review finds a blocking gap, B must use `Status: needs-revision`.

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
- Clean only after B has `approved`, A's double-check has `Status: no-change`, Controller verification has `verification-passed`, `state.json.phase` is `approved`, the final response has captured the final plan/review/double-check/verification request ids, and no recovery state is needed.
- Do not clean when `state.json.phase` is `paused` or `failed`; recovery requires the shared files.
- If cleanup is requested or default cleanup applies, delete the current `shared_dir` after the final response data has been captured.
- After cleanup, report final artifact paths as last validated artifact locations only, not as paths the user can still read. If the user needs readable audit artifacts, keep the shared directory and report that it was retained.
- If keeping audit files, report the exact `shared_dir` path and remind that it is session-specific.

## Completion Output

When B approves, A double-check returns `no-change`, and Controller verification passes, report:

- final `plan.md` path and request id;
- final `review.md` path and request id;
- final `verification.md` path and request id;
- A double-check status and request id;
- number of B review/revision cycles plus any post-approval double-check or verification rounds;
- verification checks and results;
- any interruptions and how they were recovered;
- remaining non-blocking assumptions, if any;
- whether the session-specific shared directory was cleaned or retained; if cleaned, state that the reported artifact paths identify the validated run artifacts before deletion and are no longer readable;
- recommended next action as an option, such as executing the plan or saving it for later.
