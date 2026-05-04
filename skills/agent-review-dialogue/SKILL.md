---
name: agent-review-dialogue
description: Use when the user requests an A/B editor-reviewer dialogue, iterative document optimization, code improvement with adversarial review, interrupted edit recovery, opencode/Codex-compatible review orchestration, or human-in-the-loop refinement of specified files.
---

# Agent Review Dialogue

## Purpose

Use this skill to improve specified documents, source files, tests, configuration, or skills through a persistent Agent A / Agent B loop.

Agent A edits only the approved target scope. Agent B reviews the actual changes. After B approves, A must run one more double-check pass before Controller verification. The Controller owns state, validation, recovery, user communication, and final verification.

Shared files are the source of truth. Do not advance the workflow from terminal scrollback, partial output, assumptions, or an agent's verbal claim. A phase is complete only after `state.json` and the expected artifact header validate.

Stop only when B reports no review issues, A's follow-up double-check reports no further updates, and Controller verification passes; or when a user-blocking decision appears, recovery limits are exhausted, or the configured maximum review rounds is reached.

## Use This When

Use this skill when the user asks for A/B editor-reviewer dialogue, iterative document or code optimization, adversarial review before accepting changes, interrupted-run recovery, or human-in-the-loop refinement of specified files.

Do not use it for pure planning before implementation; use `agent-plan-dialogue` instead. Do not use it for broad repository refactors unless the user provides an explicit target scope.

### Self-Application

When this skill is itself the target, follow the same protocol. The Controller must freeze the active run's scope, state, and role prompts before A edits `skills/agent-review-dialogue/SKILL.md`; edits made during the run improve future invocations and B's document review target, but they do not retroactively change the current `state.json`, request id, allowed writes, or artifact requirements unless the Controller explicitly starts a new attempt with those updated instructions.

Self-optimization must preserve the role split. A may clarify the skill text, examples, validation rules, and prompt templates, but must not edit Controller-owned coordination files except `change-log.md`; B must review the changed skill as documentation and may not rewrite it. If self-application reveals that the current run needs a different target scope, cleanup decision, or acceptance criterion, A or B must surface that through its artifact rather than changing `state.json` directly.

## Roles

- **Controller:** current session. Creates scope, writes and validates shared state, starts or emulates A/B phases, records user answers in `decisions.md`, protects unrelated changes, runs final verification, and reports results. It is the only role that updates `state.json`, `session-ids.md`, `transcript.md`, or `verification.md`. It must not claim A edited or B reviewed until the corresponding artifact validates.
- **Agent A / Editor:** edits only files listed in `target-manifest.md` and writes `change-log.md`. It treats all other coordination files as read-only inputs, preserves user constraints, avoids unrelated refactors, runs practical verification, and uses `Status: blocked-on-user` only when a missing answer changes correctness, scope, or acceptance.
- **Agent B / Reviewer:** reviews only. It reads the actual target files, diff, `target-manifest.md`, `change-log.md`, and `decisions.md`; writes `review.md`; first reviews the concrete diff, then performs a broader adversarial pass over the full in-scope files for missed improvement opportunities, stale assumptions, and regressions beyond A's stated optimization; categorizes findings by section; and approves only after both passes find no review issue of any severity. It treats `review.md` as its only writable file and must not edit target files.

Role boundaries are strict even when one human or model emulates multiple roles in one session. A role may read shared artifacts needed for its phase, but it may write only its assigned artifact and allowed targets. The Controller is responsible for advancing state between phases; A and B must copy current state values into artifacts and must not invent new rounds, attempts, phases, request ids, or cleanup decisions.

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
  "phase": "scope-check | editor-change | user-input | reviewer-review | editor-revision | editor-double-check | controller-verify | approved | paused | failed",
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
- `round` starts at `1`; increment it only before A revises after `needs-revision`, before A runs a double-check after B approval, or when the user requests another cycle after a pause.
- `attempt` increments before every A/B call, retry, replacement session, and Controller verification write. The Controller increments it; A and B copy the current values into their artifact headers and do not self-advance state.
- `request_id` is the freshness token: `<task-slug>-rNNN-aNNN-<A|B|controller>-<phase>`.
- `phase` describes the next expected operation, not the role that most recently returned output.
- Fresh output requires matching `Round:`, `Attempt:`, `Request-ID:`, and an `Updated:` timestamp later than the Controller's pre-call checkpoint.
- Status values are role-specific: A uses `blocked-on-user`, `ready-for-review`, or `no-change`; B uses `needs-revision` or `approved`; Controller verification uses `verification-passed` or `verification-failed`.
- `last_successful_step` records the most recent validated phase, not the most recent attempted command.

## Scope And Safety

- Create `shared_dir`, `state.json`, and `target-manifest.md` before A edits anything.
- `target-manifest.md` must list exact targets, explicit exclusions, baseline status, and known verification commands.
- A may modify only in-scope target files plus its required `change-log.md` artifact. B may modify only `review.md`. Controller may modify only coordination files in the current `shared_dir`, unless the user explicitly expands scope.
- If another repository file is required, A writes `Status: blocked-on-user` in `change-log.md` and explains the required scope change.
- Never run `git commit`, `git push`, destructive checkout/reset, or broad formatting commands unless the user explicitly requests them.
- Preserve unrelated user changes. When Git is available, read `git diff -- <target>` before editing any target with existing uncommitted changes and treat that diff as protected baseline context; after editing, use `git diff --name-only` or equivalent to confirm only allowed files changed.
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

Validation checklist:

- The expected artifact exists, is complete, and starts at byte one with the five-line header.
- `Round:`, `Attempt:`, and `Request-ID:` exactly match current `state.json`.
- `Updated:` is UTC ISO-8601 and later than the Controller's pre-call checkpoint.
- `Status:` is valid for the role and matches the next workflow branch.
- Markdown fences are balanced, no unresolved merge conflict markers exist, and no placeholder text such as "pending", "TODO later", or "to be filled" remains.
- Scope checks confirm only allowed target files and the current role's coordination artifact changed.

Validation must inspect files on disk after the role finishes. A role's terminal summary, exit status, or claim that a file was written is not enough. If any required artifact is missing, stale, malformed, or written outside the role's authority, the Controller must keep the same phase and run recovery instead of proceeding.

## Runtime Selection

Prefer native independent subagent/session tools and record identifiers in `session-ids.md`. If true independent subagents are unavailable, emulate separation with explicit A and B phases against shared files; do not claim review happened until B executes and `review.md` validates.

When Codex has no session ids, use stable descriptive ids such as `native-subagent-A`, `native-subagent-B`, or `not-available`, and document the limitation in `session-ids.md`. When opencode is available, create named sessions and continue them by exact session id.

### Environment Agent Check

Before creating or replacing any A/B subagent/session, inspect the current environment for predefined agents. Use the active agent tool's native discovery/listing mechanism when available.

The active agent tool is the runtime tool that will actually create or continue the A/B sessions, such as `claude code`, `codex`, or `opencode`. Predefined-agent discovery must stay inside that tool's own configuration hierarchy and namespace:

- If the current run launches A/B with `opencode`, search only opencode-managed agent locations or opencode's native agent list.
- If the current run launches A/B with `codex`, search only Codex-managed agent locations or Codex's native agent list.
- If the current run launches A/B with `claude code`, search only Claude Code-managed agent locations or Claude Code's native agent list.
- Do not scan sibling, parent, plugin-cache, global, or repository directories that belong to another agent tool just to find `dialogue-designer` or `dialogue-reviewer`.
- If a native discovery mechanism aggregates agents from multiple tools, filter the result to the active tool's hierarchy before matching names. If the hierarchy cannot be distinguished, treat the match as ambiguous and create A/B normally.

- If a predefined agent named exactly `dialogue-designer` exists, create Agent A / Editor with that predefined agent.
- If a predefined agent named exactly `dialogue-reviewer` exists, create Agent B / Reviewer with that predefined agent.
- If only one predefined agent exists, use it only for its matching role and create the other role normally.
- If neither predefined agent exists, if agent discovery is unavailable, or if the name match is ambiguous, create A/B normally.
- Do not use a similarly named predefined agent. Only exact names are valid.
- Record the active agent tool, discovery method, discovery root or native listing source, excluded cross-tool roots if relevant, match result, chosen predefined agent names, fallback reason, and timestamp in `session-ids.md`.

Predefined agents do not replace the skill protocol. Always pass the same shared directory, freshness token, role prompt, target scope, and artifact requirements to A/B, and validate their outputs exactly as usual.

Binding rules:

- If the runtime supports choosing a predefined agent for a native subagent/session, bind A to `dialogue-designer` and B to `dialogue-reviewer` using that runtime's explicit selector.
- If using opencode and it supports an agent selector, put the selector in the A/B command at the `<A-agent-binding-args>` or `<B-agent-binding-args>` placeholder shown below.
- If the runtime has predefined agents but no supported way to bind them to the actual A/B call, continue with normal A/B sessions and record the unsupported binding as a fallback.
- `session-ids.md` must record, for each role, the discovered predefined agent name, whether it was bound, the actual invocation method or binding arguments, the resulting session id when available, and the fallback reason when not bound.

### opencode Pattern

Create A first. Create B only after A produces validated `change-log.md` with `Status: ready-for-review`.

```bash
TASK_SLUG="<task-slug>"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
A_TITLE="A Editor - ${TASK_SLUG} - ${STAMP}"

opencode run <A-agent-binding-args> --title "$A_TITLE" --dir "$PWD" --format json "<editor prompt>"
opencode session list --format json --max-count 20

B_TITLE="B Reviewer - ${TASK_SLUG} - ${STAMP}"
opencode run <B-agent-binding-args> --title "$B_TITLE" --dir "$PWD" --format json "<reviewer prompt>"
opencode session list --format json --max-count 20

opencode run --session "<session-id>" --dir "$PWD" --format json "<next message>"
```

Use empty binding placeholders only after recording that opencode has no supported selector or no exact predefined-agent match. Do not silently drop a discovered predefined agent from the invocation.

Resolve session ids by exact unique title match from `opencode session list --format json --max-count 20`. If the match is missing or ambiguous, do not guess; create a new unique title and record the lookup problem in `session-ids.md`.

If file attachments or session ids are unavailable, include the shared directory path and the latest relevant shared-state contents or concise summaries in the prompt.

## Phase Protocol

### Before A, B, Or Verification

Controller increments `attempt`, generates a fresh `request_id`, writes `state.json` with the next `phase` and retry counters, appends a UTC checkpoint to `transcript.md`, and ensures every shared file exists. A/B prompts must include the exact `shared_dir`, current `request_id`, allowed writes, required reads, required artifact header, and any user decisions from `decisions.md`.

### After A, B, Or Verification

Controller validates before moving to the next phase: expected artifact exists and is non-empty; the first five lines match the required header order; `Round:`, `Attempt:`, and `Request-ID:` match `state.json`; `Updated:` is later than the checkpoint; `Status:` is role-valid; no unresolved merge conflict markers exist; code fences are balanced; the changed files are allowed for the role; `state.json` and `transcript.md` record the validated result.

If any check fails, do not continue. Run recovery for the same role and phase.

## Loop

1. Controller creates `shared_dir`, `state.json`, `target-manifest.md`, `decisions.md`, `session-ids.md`, and `transcript.md`.
2. Controller runs the environment agent check, records the result in `session-ids.md`, and prepares `editor-change`.
3. A edits in-scope targets and writes `change-log.md`.
4. Controller validates A output, artifact freshness, and changed-file scope.
5. If A returns `blocked-on-user`, Controller asks the user, records the answer in `decisions.md`, and repeats A with a new attempt and request id.
6. If A returns `ready-for-review`, Controller reuses or refreshes the environment agent check as needed, records the B selection in `session-ids.md`, and prepares `reviewer-review`.
7. B reviews actual target files and diff in two passes: first the concrete diff, then the full in-scope files adversarially for issues outside the current optimization thread; then writes `review.md`.
8. Controller validates B output and confirms B did not modify target files.
9. If B returns `needs-revision`, Controller increments `round`, forwards all B review issues to A, and requires `Review Disposition`.
10. If B returns `approved`, Controller increments `round` and prepares `editor-double-check`; A must reread the current target files, B's approved `review.md`, and decisions before deciding whether more improvements remain.
11. If A's double-check returns `ready-for-review`, Controller validates A output and returns to `reviewer-review`; B must review the new diff and full in-scope files again.
12. If A's double-check returns `no-change`, Controller prepares `controller-verify`, runs verification commands, and writes `verification.md`.
13. If verification fails, Controller sends `verification.md` and B's latest review context back to A as a revision input; A must add `Verification Disposition` to `change-log.md` and address every failed command or check.
14. If verification passes, Controller reports completion and applies the cleanup policy.
15. Stop after five review rounds unless the user explicitly asks to continue.

`Status: no-change` is valid only for A's `editor-double-check` phase. In normal `editor-change` or `editor-revision`, A must use `ready-for-review` after target edits are complete or `blocked-on-user` when a required decision is missing. A double-check that changes any target must use `ready-for-review`, because B must review the new diff before Controller verification.

## Initial Prompts

### Agent A / Editor

````markdown
你是 Agent A / Editor。目标：优化用户指定的文档或代码文件，并在信息不足时提出必须确认的问题。

必须读取：
- `<shared-dir>/state.json`
- `<shared-dir>/target-manifest.md`
- `<shared-dir>/decisions.md`
- 修订阶段和 double-check 阶段还必须读取 `<shared-dir>/review.md`；Controller verification 失败后还必须读取 `<shared-dir>/verification.md`

允许修改：只修改 `target-manifest.md` 列出的 in-scope target files，并完整替换 `<shared-dir>/change-log.md`。不要修改其他协调文件。
输出目标：完整替换 `<shared-dir>/change-log.md`。

约束：
- 不要默默假设关键需求、环境、范围或验收标准。
- 不要修改 scope 之外的文件；如必须修改，写 `Status: blocked-on-user` 并说明原因。
- 不要执行 `git commit`、`git push`、destructive reset/checkout。
- 不要修改 `state.json`、`session-ids.md`、`transcript.md`、`review.md` 或 `verification.md`；这些只作为输入读取。
- 如果目标文件就是本 skill，仍以 Controller 提供的本轮 `state.json`、request id、允许写入范围和 artifact 要求为准；不要把自己刚写入的新规则当作本轮状态变更。
- `change-log.md` 顶部必须严格使用如下 header，并匹配 `state.json`：

For `editor-change` and `editor-revision`, use:

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <blocked-on-user | ready-for-review>
```

Only in `editor-double-check`, use:

```markdown
Round: <state.json.round>
Attempt: <state.json.attempt>
Request-ID: <state.json.request_id>
Updated: <UTC ISO-8601 timestamp>
Status: <ready-for-review | no-change>
```

正文必须列出：
- `Modified Files`
- `Summary`
- `Verification Run by A`
- `Known Risks`
- `Open Questions`

修订阶段还必须添加 `Review Disposition`，逐项回应 B 的 review issues。
Controller verification 失败后的修订阶段还必须添加 `Verification Disposition`，逐项回应 `verification.md` 中每个失败命令或检查，说明根因、已做修复、重新运行结果，或说明仍需用户决定的阻塞原因。
double-check 阶段必须添加 `Double-Check Result`：如果还有任何更新，修改目标文件并写 `Status: ready-for-review`；如果没有更多更新，写 `Status: no-change` 并说明 A 已完成复查且没有进一步改动。
````

### Agent B / Reviewer

````markdown
你是 Agent B / Reviewer。目标：审查 Agent A 对指定文档或代码做出的实际变更，并继续对完整 in-scope 内容做对抗性复查。

必须读取：
- `<shared-dir>/state.json`
- `<shared-dir>/target-manifest.md`
- `<shared-dir>/change-log.md`
- `<shared-dir>/decisions.md`
- 当前目标文件内容和可用 diff

只做审查，不执行实现，不修改目标文件。
输出目标：完整替换 `<shared-dir>/review.md`。
允许修改：只完整替换 `<shared-dir>/review.md`；不要修改目标文件、`state.json`、`change-log.md`、`decisions.md`、`session-ids.md`、`transcript.md` 或 `verification.md`。

审查顺序：
1. 先重点 review 可用 diff，确认 A 的实际改动是否正确、完整、在 scope 内，并且没有引入回归。
2. 再整体阅读所有 in-scope 目标文件，继续对抗性查找改进点、遗漏风险、陈旧假设、内部矛盾、测试或文档缺口；不要只围绕 A 当前声明的优化点讨论。
3. 最后才决定 `Status`。不得在完成整体复查前提前给出 `approved` 结论；只要发现任何当前 scope 内需要处理的审查问题，无论是否阻塞，都必须写 `Status: needs-revision`。
4. 如果 diff 不可用，必须在 `Scope Check` 中说明缺口，并基于完整文件、`change-log.md` 和 manifest 做保守审查。
5. 如果目标文件是本 skill，额外检查自我应用是否会造成角色越权、状态回写、artifact 校验绕过、double-check 跳过、清理时机错误或最终输出不一致。

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
- `Diff Review`: 针对可用 diff 的重点审查结论。
- `Adversarial Full-Pass Review`: 针对完整 in-scope 内容的继续对抗性审查结论。
- `Quality Gaps`: 正确性、清晰度、安全性、测试、文档或兼容性缺口；如有任何条目，`Status` 必须是 `needs-revision`。
- `Suggestions`: 当前 scope 内仍应改进的问题；如有任何条目，`Status` 必须是 `needs-revision`。不要把纯个人风格偏好列为 suggestion。
- `Approval Rationale`: 若 `approved`，说明为什么没有剩余审查问题。

只有在没有任何 finding、quality gap、suggestion、遗漏风险或其他审查问题时，才能写 `Status: approved`。不要为了纯风格偏好要求重写；但一旦把某项列为当前审查问题，就必须返回 `needs-revision`。
````

## Recovery

Treat network errors, provider errors, process crashes, timeouts, malformed output, missing file writes, invalid headers, failed verification, and closed terminals as recoverable until retry limits are reached.

Before each recovery attempt, increment `attempt`, generate a fresh `request_id`, keep `phase` on the phase being recovered, record the concrete failure in `state.json.last_error`, increment the relevant retry counter, and append the recovery plan to `transcript.md`.

Recovery routing:

- If A's artifact is missing, malformed, stale, or outside scope, recover A for `editor-change` or `editor-revision`.
- If B's artifact is missing, malformed, stale, or edits files, recover B for `reviewer-review`.
- If Controller verification fails after B approved and A double-check returned `no-change`, route back to A with `verification.md` as revision input; do not ask B to reinterpret a failed verification as approval.
- If a required scope expansion or acceptance decision is discovered, set `phase` to `user-input`, record the blocker, and ask the user through Controller.
- If a coordination file owned by Controller is corrupt or stale, Controller repairs or reconstructs it from validated artifacts and `transcript.md`; do not ask A or B to edit Controller-owned files.

Recovery order:

1. Retry the same command once only for clearly transient failures such as network reset, timeout, rate limit, provider unavailable, or empty stream. Increment `same_command_retries`.
2. For malformed or partial coordination output, ask the same role in the same session to repair only its expected coordination file before replacing the session. The repair prompt must forbid additional target edits unless the failed phase is an A implementation phase and the target edit is needed to fix a validated review issue.
3. For failed Controller verification, send `verification.md` and B's latest review context to A for revision. Require A to add `Verification Disposition` to `change-log.md`, with one entry per failed command or check and the rerun result after the fix. Do not ask B to approve failed verification.
4. Continue the same session by id when the id is known:

```bash
opencode run --session "<session-id>" --dir "$PWD" --format json "<recovery prompt>"
```

5. If the session id is unknown, inspect sessions and match the exact unique title:

```bash
opencode session list --format json --max-count 20
```

6. If same-session recovery is impossible, rerun the environment agent check for the failed role, record the selection or fallback in `session-ids.md`, then start a replacement session with the matching predefined agent when available and provide shared-state files:

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

B must use two review passes every round:

1. Diff-first review: inspect the actual diff against `change-log.md`, `target-manifest.md`, `decisions.md`, and user constraints.
2. Adversarial full-pass review: reread all in-scope target files and actively look for remaining issues that A did not mention, including problems outside the immediate optimization topic.

B must not mark `Status: approved` until both passes are complete and no review issue of any severity remains. If the diff is clean but the full-pass review finds any quality gap, missing improvement, suggestion that should be handled in the current scope, ambiguity, or blocking issue, B must use `Status: needs-revision`.

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
- Prefix questions with the source, such as `A 需要确认...` or `B 发现一个审查问题...`.
- If the user declines to answer, record the constraint in `decisions.md` and ask A to proceed with explicit alternatives or risk notes.

## Cleanup Policy

The shared directory is intermediate state. Clean it up after successful completion unless the user explicitly asks to keep the audit trail.

Cleanup rules:

- Clean only the current run's unique `shared_dir`, never the parent `.agent-review-dialogue/` directory.
- Clean only after B has `approved`, A's double-check has `Status: no-change`, Controller verification has passed, and the final response has captured target files, review result, double-check result, verification evidence, and remaining assumptions.
- Do not clean when `state.json.phase` is `paused` or `failed`; recovery requires the shared files.
- If cleanup applies, delete only the current `shared_dir` after final response data is captured.
- If the user asks to keep the audit trail, or if local policy requires retaining artifacts, do not clean; report the exact `shared_dir` path and note that it is session-specific.
- If cleanup is performed, the final response must still include the captured request ids, statuses, and verification evidence, and must say that the session-specific shared directory was removed.

## Completion Output

When B approves, A double-check returns `no-change`, and verification passes, report:

- target files changed;
- final A `change-log.md` path and request id, including the double-check result;
- final `review.md` path and request id;
- final `verification.md` path and request id;
- number of review rounds;
- verification commands and results;
- interruptions and recovery actions, if any;
- remaining non-blocking assumptions, if any;
- whether the session-specific shared directory was cleaned or retained;
- recommended next action, such as reviewing the diff or running broader tests.
