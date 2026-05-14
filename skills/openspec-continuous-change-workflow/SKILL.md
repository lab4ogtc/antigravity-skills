---
name: openspec-continuous-change-workflow
description: "Use when the user wants an automated, continuous OpenSpec change lifecycle where the main session is only explore/brainstorm/controller, workflow control state stays in an uncommitted .workflow-style directory, multiple queued changes may plan in parallel, apply remains serialized by queue order, and each apply is driven by one apply owner that may dispatch scoped worker subagents before verification, archive, clean commit, context compression, reload, and summary return."
---

# OpenSpec Continuous Change Workflow

## Purpose

Run OpenSpec changes continuously while keeping the main conversation clean. The main session owns only exploration, brainstorming, reusable decision capture, role-subagent launch/control, Controller-relayed planning review, queue control, final summary intake, and the next exploration window. Multiple queued changes may run their planning phases in parallel, but apply remains serialized: only the queue head may enter implementation, archive, and clean commit. Each individual change uses separate planning and apply roles: the planning subagent is launched before `new change`, returns a reviewed planning lock and apply handoff, is recycled, and a fresh apply owner subagent then handles apply/TDD through archive, clean commit, compression, reload, and summary return. The apply owner may dispatch scoped worker subagents for independent implementation, test, or verification subtasks, but the apply owner remains responsible for integration, final verification, archive, and commit. The main session does not execute the change lifecycle when an independent subagent/session runtime is available.

The intended loop is:

```text
main: classify state and load project constraints
-> main: explore next candidate
-> main: brainstorm and batch decision-worthy questions
-> main: record reusable answers as project constraints
-> main: create uncommitted .workflow control state for each accepted queued candidate/change
-> main: start independent planning subagents for multiple queued candidates when their scopes are safely separable
-> each planning subagent: create or resume its assigned OpenSpec change
-> each planning subagent: generate planning documents and return planning-review request
-> main: run agent-review-dialogue on each planning request as Controller capacity allows
-> main: capture durable A/B result or blocker packet outside review coordination dir
-> main: filter A/B review context to durable packet, current plan, decisions, blockers, and freshness evidence
-> main: deliver filtered packet to the same planning subagent
-> planning subagent: create planning lock, cleanup-ready evidence, and durable apply handoff, then stop
-> main: enqueue reviewed apply handoff as ready-for-apply without starting implementation
-> main: when the prior queue head is archived, clean committed, summarized, and recycled, start the next apply owner
-> apply owner: apply with TDD, optionally dispatching scoped worker subagents for independent subtasks
-> apply owner: integrate worker results, verify, archive, and clean commit
-> apply owner: compress context and reload workflow instructions
-> apply owner: return durable completion summary
-> main: recycle apply owner and any workers, ingest summary, then advance the apply queue
```

This skill is an automation variant, not a shortcut around correctness. Do not skip stage classification, stale-evidence checks, planning review, verification, archive inspection, unrelated-change protection, or repository history safety checks.

## Non-Negotiable Model

- Use separate dedicated planning and apply roles per OpenSpec change. Never reuse a planning subagent for apply, never reuse an apply owner for planning, and never reuse planning, apply owner, or apply worker subagents for another change.
- When multiple decision-ready candidates are queued and their scopes are separable, the main session may launch multiple planning subagents in parallel. Parallel planning is only for OpenSpec planning and Controller-relayed review; it is not permission for parallel apply.
- Apply is serialized by queue order. Only one change may be in apply, verification, archive, clean commit, compression, reload, or summary-return at a time. A later ready-for-apply change must wait until the current queue head is archived, clean committed, summarized, and recycled.
- Treat planning subagent, apply owner, and apply worker timeouts as soft signals. Use the extended timeout policy below before declaring a recorded role subagent stalled, missing, or unrecoverable.
- Start each planning subagent before `openspec new change`, `openspec-ff-change`, or the CLI planning-generation fallback for its assigned change.
- Do not run planning generation, internal/manual planning review, apply, verification, archive, or clean commit in the main session when an independent subagent/session runtime is available. The only planning-review work the main session may perform is Controller-orchestrated `agent-review-dialogue` in response to a recorded planning subagent request.
- Do not reuse one role subagent across lifecycle roles or changes. Close, retire, freeze, or discard each planning subagent after its apply handoff is durably captured and queued; close, retire, freeze, or discard the apply owner and all apply workers after archive, clean commit, compression, reload, and final summary.
- If a run is interrupted after launch, continue the recorded role subagent/session for its current gate only when durable workflow control state and role identity evidence both exist and agree; otherwise return a recovery blocker instead of taking over the lifecycle in the main session or launching a replacement subagent.
- Persist planning, apply owner, apply worker, queue position, and lifecycle state for each queued or active change; timeout recovery, resume decisions, role handoff, apply ordering, and next-change authorization must be based on that durable state plus repository evidence.
- Store workflow control state only under a dedicated repo-local control directory such as `.workflow/openspec-continuous/<change-id-or-candidate>/`. Do not put workflow control state under `openspec/` or `.openspec/`, do not list it as an OpenSpec planning artifact, and never stage, commit, archive, or push it.
- Keep main-session context out of implementation. Pass only explicit artifacts selected by the Controller: exploration result, brainstorm decisions, project constraints, relevant archived specs, relevant post-archive summaries, and current repository state needed to start safely.
- After OpenSpec planning documents exist, treat those documents and explicit decision notes as the implementation contract. Do not rely on broad conversation history, hidden assumptions, terminal scrollback, or subagent private memory.
- The planning subagent must not launch `agent-review-dialogue` itself. After planning documents exist, it returns a planning-review request to the main Controller; the Controller launches `agent-review-dialogue`, captures and filters the durable result or blocker packet, and only then resumes the same planning subagent with that filtered packet.
- After Controller-run A/B review, filter the main session's active context before any role-subagent resume, handoff, planning lock, apply, archive, clean commit, summary, or next-change exploration. Keep only the durable result or blocker packet, current planning files, recorded decisions, blockers, and freshness evidence; exclude raw A/B dialogue, round-by-round critique, temporary coordination notes, and review-agent private reasoning from later prompts, handoffs, summaries, and exploration inputs.
- Automatically create a scoped clean commit after archive and post-archive validation when the diff contains only the completed change.
- Never run destructive git operations, delete unrelated files, accept failed required verification, or archive unexpected diffs without explicit user authorization.

## Role Subagent Timeout Policy

Planning subagents, apply owners, and apply workers often perform slow OpenSpec planning, Controller-relayed review intake, TDD, builds, validation, archive, clean commit, context compression, and workflow reload. The main session and apply owner must therefore use longer waits than ordinary exploration subagent calls and must not treat one or more `wait_agent` timeouts as completion failure.

- Default to long waits for role subagents: use at least 10 minutes per `wait_agent` call when waiting for planning/review/apply/verify/archive/commit/compression/reload/summary results.
- For implementation, builds, full test runs, archive, clean commit, context compression, or workflow reload, prefer 20-30 minute waits when the tool allows it.
- A `wait_agent` timeout means "no final message returned in that window"; it does not mean the subagent is dead, failed, or safe to replace.
- After a timeout, inspect durable evidence before drawing conclusions: `git status`, `.workflow/openspec-continuous/queue.md`, `.workflow/openspec-continuous/<change>/` control state, the active change directory, recorded role subagent/session identity evidence, OpenSpec status, project constraint artifacts, and recent planning/review/verification/archive/commit/compression/reload artifacts. Prefer evidence written after the role subagent launch or latest main-session or apply-owner handoff over stale terminal output.
- If durable workflow control state is missing, conflicts with identity evidence, is stored in an OpenSpec directory, or cannot be reconciled with repository artifacts for the active change, return a recovery blocker that freezes the active change. Identity evidence or artifact progress alone is not enough to continue, replace, take over in the main session, or explore the next change when the durable state anchor is unsafe.
- If the durable workflow control state names the active change, current gate, and recorded role subagent/session, and any OpenSpec change artifact, review artifact, verification artifact, repository diff, or state transition shows current or plausible progress for that gate, continue or resume that same recorded role subagent instead of launching a replacement.
- Before declaring a recorded role subagent stalled, record one concise status request in durable workflow control state, send it to that same recorded role subagent/session, and wait again with the extended timeout. Do not send repeated status pings that could distract from long-running work. If no reliable same-role continuation target exists, return a recovery blocker instead of guessing a status target. For apply workers, the apply owner records and sends the status request, then reports the result to the Controller.
- Return a recovery blocker when no gate result or final summary has returned and safe same-role continuation cannot be established from durable workflow control state, identity evidence, artifact progress, and the single follow-up status request when a reliable status target existed. The blocker is a safety stop, not authorization for replacement, main-session takeover, cross-role reuse, or next-change exploration; those actions remain prohibited while they could risk duplicate planning, duplicate implementation, mixed commits, skipped gates, or acting while the current subagent may still be running. A silent follow-up wait is still only missing evidence, not proof that replacement is safe.
- When reporting a possible stall or returning a recovery blocker, explicitly say that tool-level timeout is a soft signal and name the evidence checked. Avoid saying the subagent has not created artifacts unless the filesystem check was run after the latest possible subagent notification.

## Workflow Control State And Identity Evidence

Every active change needs a durable workflow control state record that survives main-session interruption and tool timeouts. Store it in a dedicated repo-local control directory, not in OpenSpec content. Prefer:

```text
.workflow/openspec-continuous/queue.md
.workflow/openspec-continuous/<change-id-or-candidate>/
  state.md
  subagents.md
  planning-handoff.md
  review-result-or-blocker.md
  apply-handoff.md
  apply-workers.md
  recovery.md
```

These files are workflow control state, not OpenSpec artifacts. They must not live under `openspec/` or `.openspec/`, must not be referenced as OpenSpec proposal/design/spec/task files, must not be included in OpenSpec archive output, and must never be staged, committed, or pushed. If `.workflow/` is not ignored, leave it untracked and enforce the exclusion during staging; do not modify `.gitignore` or project policy files just to hide it unless the user or repository policy explicitly asks for that.

The state record is for lifecycle control, not for broad implementation context. It must stay small and include:

- Active change id or pre-change candidate id.
- Queue status: `planning`, `planning-review`, `ready-for-apply`, `applying`, `blocked`, `done`, or `retired`; queue position; dependency or conflict notes; and whether the change may be planned in parallel.
- Planning subagent/session identity: runtime, title or handle, session id when available, launch timestamp, latest continuation target, and terminal status such as `closed`, `retired`, `frozen`, or `not-closeable-retired`.
- Apply owner subagent/session identity: runtime, title or handle, session id when available, launch timestamp, latest continuation target, and terminal status such as `closed`, `retired`, `frozen`, or `not-closeable-retired`.
- Apply worker subagent/session identities when used: owner-assigned task, allowed write scope, launch timestamp, latest continuation target, result status, integration status, verification evidence, and terminal status.
- Current lifecycle gate, latest completed gate, next expected evidence, and paths to planning/review/verification/archive/commit/compression/reload artifacts.
- Planning review request, Controller A/B result or blocker packet path, Controller review context filter evidence, and reviewed-file freshness evidence.
- Planning handoff path, apply handoff path, planning-subagent recycle evidence, apply-owner recycle evidence, apply-worker recycle evidence, and any cross-role blocker.
- Planning lock coverage and cleanup-ready evidence written by the planning subagent; Controller cleanup validation result and cleaned coordination directory path when cleanup has happened.
- Project-level constraint update status: Controller-written constraint entry path, or a planning-subagent `constraint update required` note path for Controller action.
- Git safety evidence for archive and clean commit gates: latest working-tree summary, staged-file summary, explicit `.workflow/` and `.agent-review-dialogue/` exclusion check, and unrelated-change assessment.
- Main-session handoff timestamp and the timestamp/content summary of the single timeout status request, if one was sent.
- Terminal state: planning handoff returned and planning subagent recycled; apply final summary returned, blocker returned, or apply owner plus all workers recycled after archive, clean commit, context compression, workflow reload, and summary intake.

When a timeout status request is sent, record the request timestamp, request text summary, target continuation/session id, and follow-up wait result. The follow-up result should say whether the same recorded role subagent returned its gate result or final summary, returned a blocker, showed durable progress, remained silent through the extended wait, or could not be contacted through the recorded continuation target.

The active planning subagent or apply owner must update durable workflow control state whenever it enters or completes its lifecycle gate, starts or finishes a long verification/build/test run, returns a planning or apply handoff, and before and after archive, clean commit, context compression, workflow reload, or final summary return. Apply workers update only their owner-assigned worker evidence through the apply owner; they do not update queue state, archive state, commit state, or final completion state directly. The apply owner must update `apply-workers.md` before launching workers, after each worker result, after integration, and before final verification. The main session must update queue state when it launches, resumes, closes, retires, or freezes a role subagent, sends the single timeout status request, completes the follow-up extended wait, ingests the final summary, advances the apply queue, or records a recovery blocker. Stale state is not a valid anchor for replacement, cross-role reuse, main-session takeover, queue advancement, or next-change exploration.

The main session may use this state only to decide launch, resume, close/retire/freeze, status-request, recovery-blocker, and next-exploration actions. It must not start a replacement subagent, take over lifecycle work, reuse a subagent across roles, or explore the next change while the state is non-terminal or while a recorded subagent may still be running.

## Parallel Planning Queue

Use a queue when the exploration and brainstorming window identifies more than one decision-ready candidate. The queue increases throughput by planning future work early, but it must not parallelize implementation.

- Store queue control in `.workflow/openspec-continuous/queue.md` plus one control directory per queued candidate or change. These files are workflow control state and must never be staged, committed, archived, or pushed. The queue must record queue order, one optional active apply item, per-item status, per-item control directory, dependency notes, and the evidence required for the next transition.
- A queued candidate may start planning in parallel only when the main session has enough decisions to define its scope, expected OpenSpec change id or naming constraints, acceptance criteria, verification expectations, and obvious dependency or conflict boundaries.
- Parallel planning subagents must have distinct control directories, distinct runtime identities, and distinct OpenSpec change ids. They must not edit another queued item's planning documents or control state.
- A planning subagent may create planning documents, run Controller-relayed review, create a planning lock, write cleanup-ready evidence, and return an apply handoff. It must not start apply, spawn apply workers, modify implementation files, archive, commit, or mark itself as the active apply item.
- The main session may mark an item `ready-for-apply` only after the reviewed apply handoff, Controller-owned A/B cleanup evidence, planning-subagent recycle evidence, required constraint-update evidence, and queue-position evidence all exist. If a later plan finishes first, it waits in the queue and does not start apply until every earlier unblocked queue item is terminal or explicitly reordered by the user.
- The apply queue has at most one `applying` item. Starting an apply owner for a second item before the current applying item is archived, clean committed, summarized, and recycled is a stop condition.
- If parallel planning reveals overlapping implementation scope, conflicting specs, incompatible migration assumptions, shared public API changes, or ordering dependencies, freeze or block the later queue item and record the conflict. Do not let two plans race to define the same contract.
- Controller-owned project constraint updates are serialized. If multiple planning subagents request reusable constraint updates, the main session resolves them in queue order or records a blocker when they conflict.

## Controller-Relayed Planning Review

Use this relay whenever a recorded planning subagent reaches planning-document review. Do not run A/B review inside the planning subagent, and do not let the planning subagent replace A/B review with an internal self-review.

The planning subagent must stop after planning generation, update durable workflow control state, and return a planning-review request to the main Controller. The request must include:

- Active change id, planning subagent/session identity, current gate, and planning files with paths plus freshness evidence such as timestamps, hashes, or latest diff summary.
- Exact review target manifest, allowed edit scope, durable decision paths, and explicit exclusions.
- Relevant brainstorm decisions, project constraints, change-local decisions, and selected cross-change artifacts, not broad transcript context.
- Existing review artifacts to reuse or invalidate, if any.
- Blockers or decisions needed before A/B review can continue.

The Controller must validate the request against durable workflow control state, then launch `agent-review-dialogue` in the main session with only the requested manifest, planning files, durable decisions, and explicit exclusions. Multiple planning reviews may be in progress only when their target manifests, coordination directories, and planning files are disjoint; otherwise the Controller serializes them. If A/B raises product, API, compatibility, migration, dependency, scope, acceptance, verification, or repository-history decisions not already covered by durable constraints, resolve them from project constraints or return a blocker; do not let the planning subagent, apply owner, or apply workers guess.

After `agent-review-dialogue` reaches approval, the Controller must capture a durable result packet outside any review coordination directory that may later be deleted and preferably inside the `.workflow/openspec-continuous/<change>/` control directory. The packet must include active change id, requesting planning subagent/session identity, planning-review request id or equivalent correlation evidence, B approval, A double-check result, Controller verification result, review round count, reviewed file freshness evidence, unresolved assumptions, decisions, and the active session-specific A/B coordination directory path for later cleanup.

If `agent-review-dialogue` blocks, fails artifact validation, falls back to internal review, or finds stale reviewed files, the Controller must capture a durable blocker packet outside the review coordination directory before resuming the planning subagent. The blocker packet must include active change id, requesting planning subagent/session identity, planning-review request id or equivalent correlation evidence, blocker status, failing validation or freshness evidence, affected files, required decision or recovery action, and next gate.

Immediately after producing an approved result packet or blocker packet, the Controller must filter its active review context. The post-review Controller context used for resuming the planning subagent or future exploration is limited to:

- The durable result or blocker packet path and concise contents.
- The current reviewed planning files and freshness evidence.
- Decisions, unresolved assumptions, blockers, and exact next gate.
- The coordination directory path only for later cleanup.

Exclude A/B conversation transcripts, intermediate comments that were resolved, raw review-agent outputs, private coordination files, and speculative reasoning from subsequent prompts, handoffs, summaries, and next-change exploration. Do not pass `change-log.md`, `review.md`, `verification.md`, `transcript.md`, or raw `.agent-review-dialogue/` contents to planning subagents, apply owners, or apply workers except as concise facts captured in the durable result or blocker packet. If the runtime supports context compaction, filtering, or thread summarization, run it immediately around the result or blocker packet before resuming the planning subagent. If not, explicitly mark the raw A/B review as excluded and never paste it into planning, apply owner, or apply worker prompts.

Record the filter evidence in durable workflow control state: result or blocker packet path, kept categories, excluded categories, filter timestamp, and next gate. This evidence is required before the Controller resumes the planning subagent.

The Controller then sends only the filtered result or blocker packet to the same planning subagent and waits for it to verify freshness before creating the planning lock. If A/B blocks, fails validation, falls back to internal review, or reviews stale files, the planning subagent records the blocker intake in durable workflow control state and does not advance to planning lock or apply handoff.

The active `.agent-review-dialogue/<task>/<session>/` coordination directory is Controller-owned from creation through cleanup. Planning subagents, apply owners, and apply workers must not delete, move, truncate, rewrite, or otherwise clean any `.agent-review-dialogue/` files. The planning subagent may only write cleanup-ready evidence into `.workflow/openspec-continuous/<change>/`, naming the active coordination directory and proving durable result capture, Controller context filter evidence, same-planning-subagent filtered-result intake, reviewed-file freshness, and planning lock coverage. Only the Controller may trigger, validate, and execute deletion or cleanup of the active session-specific coordination directory, and only after that cleanup-ready evidence plus the Controller's own durable result packet and filter evidence are present.

## Main Session Responsibilities

The main session is an explorer and controller, not the change executor.

It must:

- Classify whether there is an active in-progress change or whether the next action is exploration.
- Load durable project-level constraints before asking questions.
- Run `openspec-explore` or the supported exploration fallback when the next candidate is unknown.
- Use `superpowers:brainstorming` after exploration and before any planning document is created.
- Ask only high-value questions during the exploration-to-brainstorming window.
- Record reusable answers in a project-level constraint artifact.
- Record change-local answers in a handoff note that the planning subagent must materialize into OpenSpec planning documents or decision notes.
- Maintain `.workflow/openspec-continuous/queue.md`; launch multiple planning subagents only for queued candidates with separable scopes, distinct control directories, and enough durable decisions to plan safely.
- Keep apply serialized by queue order. A ready-for-apply item may wait while another item applies, but it must not start implementation until the current apply owner finishes archive, clean commit, context compression, workflow reload, summary return, and recycle.
- Process any planning-subagent `constraint update required` note before cleanup, apply handoff acceptance, or apply launch; only the main session or Controller may update project-level constraint artifacts.
- Start one planning subagent per queued candidate with a minimal, explicit handoff.
- Run `agent-review-dialogue` only when the recorded planning subagent returns a planning-review request, then filter the review context before resuming that same planning subagent.
- Keep review coordination artifacts Controller-owned; later role subagents receive only the durable packet and freshness evidence, not raw A/B files or dialogue.
- Validate planning-subagent cleanup-ready evidence and clean only the active session-specific A/B coordination directory when the cleanup gate is satisfied; do not delegate `.agent-review-dialogue/` cleanup to any role subagent.
- Recycle the planning subagent after it returns a durable apply handoff; do not let it run apply.
- Start one fresh apply owner subagent for the queue head with the planning lock, apply handoff, and selected artifacts only.
- Record planning, apply owner, apply worker, queue position, lifecycle state, and recycle evidence before making timeout, resume, recovery, role-handoff, queue-advance, or next-change decisions.
- Ingest the apply owner's final durable summary after completion.
- Recycle the apply owner and every apply worker after summary intake.
- Resume exploration or advance the next apply item only after the current apply owner reports archive, clean commit, context compression, and workflow reload evidence and the owner plus workers are recycled or explicitly marked retired.

The main session must not carry detailed implementation context forward. Its durable memory between changes should be project constraints plus compressed summaries.

## Planning And Apply Subagent Responsibilities

The planning subagent owns planning only:

- Reclassify current state for the assigned change from repository and OpenSpec artifacts.
- Create or resume the assigned OpenSpec change, starting with `new change` when the change does not yet exist.
- Generate proposal, design, spec deltas, tasks, and decision notes using the available OpenSpec skill or supported CLI fallback.
- Modify only planning artifacts, decision notes, and the assigned `.workflow/openspec-continuous/<change>/` control files needed for planning; inspect implementation files only to understand constraints.
- Return a planning-review request when generated planning documents need review or review evidence is stale; do not launch `agent-review-dialogue` directly.
- Ingest only the Controller-filtered review result or blocker packet and verify freshness before planning lock.
- Verify reusable constraints have already been written by the main session or Controller. If a required project-level constraint is missing, record `constraint update required` in the planning lock or assigned `.workflow/openspec-continuous/<change>/` control files and return a blocker or cleanup-ready note for Controller action; do not edit project-level constraint artifacts directly.
- Create a planning lock after review approval, durable decision capture, and required Controller-owned constraint updates, then write cleanup-ready evidence in `.workflow/openspec-continuous/<change>/` for Controller validation.
- Return a durable apply handoff and terminal planning summary, then stop. It must not run `openspec-apply-change`, implement code, verify implementation, archive, commit, compress, or explore another change.

The apply owner owns apply and completion only:

- Start only after the Controller has recycled the planning subagent or marked it retired and recorded the apply handoff.
- Read the planning lock, apply handoff, current OpenSpec planning documents, project constraints, and current repository state.
- Refuse to regenerate planning documents or rerun planning review. If planning is stale or insufficient, return a blocker to the Controller instead of planning inside the apply phase.
- Apply and implement with TDD.
- Dispatch apply worker subagents only for independent implementation, test, documentation, migration, or verification subtasks with explicit file/module ownership, expected outputs, and conflict boundaries.
- Record each worker in `.workflow/openspec-continuous/<change>/apply-workers.md` before launch. Workers are not alone in the codebase; they must not revert or overwrite others' work, and they must adapt to already-integrated changes.
- Integrate and review every worker result before final verification. Worker success is not sufficient evidence for archive or commit.
- Modify only approved implementation files, tests, generated apply/archive outputs, task-status evidence, and the assigned `.workflow/openspec-continuous/<change>/` control files. It must not edit proposal, design, spec-delta, decision, or raw A/B coordination files to bypass planning review.
- Verify with OpenSpec validation and required project checks.
- Archive only after required verification passes.
- Create a scoped clean commit after archive and post-archive validation.
- Compress the change context and reload the workflow instructions before returning.
- Return a durable completion summary to the main session.

Apply workers are subordinate to the apply owner:

- They may modify only files or modules explicitly assigned by the apply owner and recorded in `apply-workers.md`.
- They must not edit OpenSpec planning documents, project-level constraints, `.agent-review-dialogue/`, queue state, archive artifacts, git staging, commits, or push state.
- They must return concise implementation or verification evidence to the apply owner. If the apply owner is unavailable and recovery instructions explicitly request worker status, a worker may return only a limited status or recovery evidence packet to the main session. That packet must not report final completion, authorize archive or commit, stand in for the apply owner's durable completion summary, or allow queue advancement.
- They must not launch additional subagents unless the apply owner explicitly records a nested worker plan and distinct write scopes; prefer one worker layer.

The active role subagent must stop and return a blocker to the main session when the change requires scope expansion, policy decisions not covered by the brainstorm window, unclear OpenSpec command semantics, failed verification that cannot be fixed within scope, archive surprises, unrelated diff contamination, unsafe git history operations, role boundary violations, or reuse of a retired subagent.

## Project-Level Constraints

Before brainstorming or launching a role subagent, discover a durable project-level constraint artifact. Prefer an existing canonical location in this order:

1. Repository instruction files that already govern agents, such as `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, or `.codex/instructions.md`.
2. Existing OpenSpec project files that document conventions, such as `openspec/project.md`, `.openspec/project.md`, `openspec/CONSTRAINTS.md`, or `.openspec/CONSTRAINTS.md`.
3. Existing project decision logs, architecture records, or policy files that the repository clearly uses.

If no project-level constraint artifact exists, ask once during the exploration-to-brainstorming window where durable workflow constraints should live. After the user chooses, record that location and reuse it for future cycles. Do not invent a hidden constraints file when the repository has no convention.

Only the main session or Controller writes project-level constraints. Planning subagents, apply owners, and apply workers may read the project-level constraint artifact and may record proposed or required constraint updates in planning artifacts, decision notes, or assigned `.workflow/openspec-continuous/<change>/` control files, but they must not edit repository-level constraint or policy files such as `AGENTS.md`, `.codex/instructions.md`, OpenSpec project files, decision logs, or architecture records.

When the main session or Controller writes constraints:

- Use concise, dated entries with the decision, scope, rationale, source, and reuse rule.
- Separate project-wide constraints from change-local decisions.
- Append or make the smallest targeted edit; do not overwrite user-authored policy.
- Do not treat a one-time exception as a project rule.

Recommended entry shape when no local format exists:

```markdown
## OpenSpec workflow constraints

- Date: YYYY-MM-DD
  Decision: ...
  Scope: project-wide | capability | change-local
  Reuse rule: apply automatically when ...
  Source: user confirmation in exploration-to-brainstorming for <change-id>
```

## Decision Value Assessment

Evaluate every possible question before asking it.

Ask during exploration-to-brainstorming only when all are true:

- The decision affects product behavior, compatibility, migration, public API, security, dependency policy, verification obligations, repository history, destructive operations, or scope.
- The answer is not already explicit in OpenSpec documents, project constraints, tests, code conventions, or prior durable decisions.
- Choosing wrong would cause meaningful rework, unsafe behavior, invalid verification, mixed commits, or policy violation.

Do not ask when:

- The answer is a local implementation detail that TDD can validate.
- The repository has a clear convention.
- The question is about wording, formatting, or naming with no compatibility impact.
- A conservative default is already required by project constraints.
- The issue can be documented as a change-local assumption and verified later.

Classify queued questions:

| Class | Meaning | Action |
| --- | --- | --- |
| `blocker` | Wrong answer may invalidate the change, damage history, or violate policy | Ask before launching the planning subagent, or return a blocker if discovered later |
| `reusable-policy` | Answer should guide future changes | Ask once, then record in project constraints |
| `change-local` | Answer affects only this change | Ask before launch and pass as subagent handoff input |
| `low-value` | Answer is inferable, testable, or immaterial | Do not ask; proceed and document the assumption only if useful |

## Entry Protocol

Start every invocation by classifying the current lifecycle state:

1. Read the user's latest request for a stage intent: explore, brainstorm, create, review, apply, verify, archive, commit, compress, reload, continue, or run continuously.
2. Inspect repository state with non-destructive commands: `git status`, `.workflow/openspec-continuous/queue.md`, per-change `.workflow/openspec-continuous/` control state, OpenSpec directories, current change files, review artifacts, constraint artifacts, and durable verification results when present.
3. Identify queued change ids, active apply change id if any, requested stage, latest completed gate with evidence, stale or missing evidence, and next safe action.
4. Load project-level constraints before deciding whether to ask anything.
5. If the next safe action belongs to planning, launch one or more dedicated planning subagents before `new change` only for queued candidates with separable scopes and distinct control directories. If the next safe action belongs to apply or later implementation, launch or continue the dedicated apply owner only for the queue head after a current planning lock, apply handoff, Controller cleanup, planning-subagent recycle evidence, required constraint-update evidence, queue-head authorization, and no other active apply exist. When resuming an interrupted change, continue the same recorded role subagent/session only when durable workflow control state and matching role identity evidence exist; if either is missing, return a recovery blocker to the main session instead of executing the lifecycle in the main session or launching a replacement subagent.

If multiple active changes exist and the active change cannot be determined safely, ask once before acting.

## Stage Router

| Observed state | Main-session action | Role-subagent action |
| --- | --- | --- |
| No next candidate is known | Explore | None yet |
| Candidate is known but scope decisions are missing | Brainstorm decision window | None yet |
| Multiple decision-ready candidates are queued | Create or update `.workflow/openspec-continuous/queue.md`; launch parallel planning subagents only for separable queued candidates | Each planning subagent creates or resumes only its assigned change and planning documents |
| Brainstorm decisions are complete and no OpenSpec change exists | Create `.workflow/openspec-continuous/<candidate>/` control state and launch planning subagent before `new change` | Create change and planning documents |
| Planning files exist but no approved review exists | Continue the recorded planning subagent until it emits a planning-review request, then Controller runs A/B review; otherwise return a recovery blocker | Return review request; after Controller approval, ingest only the filtered result or blocker packet |
| Planning review is current but no apply handoff exists | Continue the recorded planning subagent/session only with durable workflow control state, identity evidence, durable Controller result capture, Controller context filter evidence, same-planning-subagent filtered-result intake, and Controller-owned reusable constraint updates; otherwise return a recovery blocker | Create planning lock and cleanup-ready evidence; return durable apply handoff only when required Controller-owned constraint updates are complete, otherwise return a blocker or `constraint update required` note |
| Apply handoff exists and A/B cleanup is pending | Controller validates cleanup-ready evidence and cleans only the active session-specific A/B coordination directory; otherwise return a recovery blocker | None; role subagents must not clean `.agent-review-dialogue/` |
| Apply handoff exists and planning subagent is not recycled | Recycle, close, freeze, or mark the planning subagent retired before apply | None; planning role is terminal |
| Apply handoff exists for a non-head queue item | Mark or keep the item `ready-for-apply`; do not start implementation | None; wait for queue head |
| Queue head apply handoff exists and no other apply is active | Launch a fresh apply owner only after Controller cleanup, planning-subagent recycle, required constraint-update evidence, and queue-head authorization are complete; never reuse the planning subagent | Apply owner starts TDD and may dispatch scoped workers |
| Implementation is incomplete | Continue the recorded apply owner only with durable workflow control state and apply identity evidence; otherwise return a recovery blocker | Continue from failing or missing tests; integrate worker results |
| Apply workers are running | Wait through the apply owner, not the main session, unless recovery requires limited worker status; inspect `apply-workers.md` before acting | Workers complete assigned scopes and report to owner; recovery status packets are evidence only and cannot authorize archive, commit, summary, or queue advancement |
| Implementation appears complete | Continue the recorded apply owner only with durable workflow control state and apply identity evidence; otherwise return a recovery blocker | Run integrated fresh verification |
| Verification passes and archive is pending | Continue the recorded apply owner only with durable workflow control state and apply identity evidence; otherwise return a recovery blocker | Archive and post-archive validation |
| Archive completed but no clean commit exists | Continue the recorded apply owner only with durable workflow control state and apply identity evidence; otherwise return a recovery blocker | Stage scoped files excluding `.workflow/`, `.agent-review-dialogue/`, and controller-only artifacts, then commit |
| Clean commit exists but no returned summary exists | Continue the recorded apply owner only with durable workflow control state and apply identity evidence; otherwise return a recovery blocker | Compress context, reload workflow, and return summary |
| Apply owner returned durable summary | Ingest summary, update durable context, recycle apply owner and workers, then advance the queue | End this apply owner and workers; do not reuse them |
| Summary is ingested and next ready-for-apply item exists | Start the next queue-head apply owner only after previous apply terminal evidence and recycle evidence exist | Next apply owner begins |
| Summary is ingested and no ready-for-apply item exists | Explore next or continue parallel planning queue | None; previous planning, apply owner, and workers are closed or retired |

## OpenSpec Invocation Compatibility

Before each OpenSpec lifecycle action, use the corresponding available skill or command exposed in the current runtime. Do not scan other runtimes or external tool directories for hidden OpenSpec workflows.

| Action | Preferred skill | CLI-supported fallback |
| --- | --- | --- |
| Planning generation | `openspec-ff-change` | `openspec new change "<change-id>"`, then loop through `openspec status --change "<change-id>" --json` and `openspec instructions <artifact-id> --change "<change-id>" --json` |
| Apply | `openspec-apply-change` | `openspec instructions apply --change "<change-id>" --json` |
| Verify | `openspec-verify-change` | Manual artifact-to-implementation review plus `openspec validate "<change-id>" --type change --strict --no-interactive` |
| Archive | local archive command | Inspect `openspec archive --help`, then run the supported `openspec archive "<change-id>"` form |
| Explore next | `openspec-explore` | `openspec list --json`, `openspec show`, repository inspection, and concise candidate summary |

Do not call unsupported guessed top-level commands such as `openspec apply`, `openspec verify`, `openspec explore`, or `openspec new change --ff` unless local help proves they exist.

## Evidence Freshness Rules

- An `agent-review-dialogue` approval is current only when it was launched by the Controller from the recorded planning subagent's planning-review request, and its target scope plus freshness evidence correspond to the current planning files. If any reviewed planning file changed after approval, rerun review through the Controller relay.
- A/B review evidence is not current if it depends on planning subagent implicit memory, was produced by an internal-review fallback, was not returned to the same planning subagent as a durable Controller result packet, or lacks evidence that the Controller filtered A/B context after review.
- A/B review cleanup is current only after the final Controller-orchestrated `agent-review-dialogue` result has been durably captured outside the coordination directory that will be deleted, Controller context filtering is complete, the same planning subagent has ingested the filtered result packet, the planning lock confirms required decisions are covered by brainstorm answers, durable constraints, or change-local decision notes, and the planning subagent has written cleanup-ready evidence in `.workflow/openspec-continuous/<change>/`. Required durable evidence includes active change id, requesting planning subagent/session identity, planning-review request id or equivalent correlation evidence, B approval, A double-check no-change, Controller verification passed, review round count, reviewed file freshness evidence, Controller context filter evidence, same-planning-subagent filtered-result intake, planning lock coverage, unresolved assumptions, and decisions that affect implementation.
- Clean only the active session-specific A/B coordination directory, and only from the Controller after it validates durable result capture, filter evidence, same-planning-subagent intake, planning lock coverage, and cleanup-ready evidence. Do not delete the parent `.agent-review-dialogue/` tree, sibling review runs, OpenSpec planning documents, decision notes, implementation files, or project constraints.
- If cleanup already happened, continue only when Controller-written durable evidence proves the same active change, requesting planning subagent/session identity, planning-review correlation, B approval, A double-check no-change, Controller verification, reviewed file freshness, Controller context filter evidence, same-planning-subagent filtered-result intake, planning lock coverage, cleanup-ready evidence, cleanup executor, cleaned path, and cleanup timestamp. Otherwise return a recovery blocker instead of recreating or guessing deleted review context.
- Apply evidence is current only when it was produced by a separate apply owner launched for the queue head after a current apply handoff, Controller cleanup, planning-subagent recycle evidence, required constraint-update evidence, queue-head authorization, and proof that no other apply owner was active at launch. If the same subagent identity appears in planning, apply owner, or apply worker roles, stop with a recovery blocker.
- Apply worker evidence is current only when the apply owner assigned explicit write scope before launch, the worker result was returned to the owner, the owner integrated or rejected it, and the owner reran required integrated verification. A worker recovery status packet sent to the main session is limited recovery evidence only; it is not final completion evidence and cannot authorize archive, commit, summary, or queue advancement. Worker-local tests alone are not enough for archive or commit.
- Queue evidence is current only when `.workflow/openspec-continuous/queue.md` and the per-change control directory agree on queue status, queue position, active apply item, required transition evidence, and terminal state of prior items including archive, clean commit, summary intake, and recycle evidence. If they conflict, return a recovery blocker instead of applying a later item.
- Workflow control evidence is not current if it is stored under `openspec/` or `.openspec/`, listed as an OpenSpec artifact, staged for commit, included in archive output, or missing from the `.workflow/openspec-continuous/<change>/` control record.

## Workflow

### 1. Explore And Brainstorm In Main Session

Use `openspec-explore` or the supported exploration fallback first when the next change is not already known. Then use `superpowers:brainstorming` before launching planning subagents.

During this window, decide the change goal, user-visible outcome, in-scope and out-of-scope boundaries, acceptance criteria, verification expectations, compatibility/migration policy, and reusable constraints worth recording.

When several candidates are ready, classify each candidate as `parallel-plannable`, `blocked`, or `must-follow`. A candidate is `parallel-plannable` only when it has clear scope, no known implementation overlap with earlier queue items, no dependency on unarchived implementation details from another candidate, and no unresolved decision that would affect another candidate's contract.

Do not create or edit OpenSpec planning documents in the main session during this step. Planning subagents create or resume them after launch.

### 2. Prepare Workflow Control State And Planning Handoff

Create or update the uncommitted queue and control directory before launching any role subagent:

```text
.workflow/openspec-continuous/queue.md
.workflow/openspec-continuous/<change-id-or-candidate>/
```

Record the queue position and control state location in each handoff. Keep this directory out of OpenSpec and git staging: do not place it under `openspec/` or `.openspec/`, do not include it in OpenSpec target manifests, and do not stage or commit it.

Create a minimal handoff for the planning subagent. Include only:

- Active change id or naming constraints, if known.
- Queue position, queue status, known dependencies, and whether parallel planning is authorized for this candidate.
- Exploration result and chosen candidate.
- Brainstorm decisions and acceptance criteria.
- Project-level constraint artifact path and relevant entries.
- Change-local assumptions and verification expectations.
- Relevant archived specs or compressed summaries selected by the Controller.
- `.workflow/openspec-continuous/<change-id-or-candidate>/` control state location and fields the planning subagent must update.
- Current repository state needed to avoid mixing unrelated changes.
- Stop conditions and requirement to return blockers to the main session.

Exclude broad transcript history, raw terminal scrollback, private reasoning, unrelated prior-change context, and stale assumptions. The subagent must turn this handoff into OpenSpec planning documents or decision notes before apply.

### 3. Launch Planning Subagents

Launch one dedicated planning subagent for each authorized queued candidate. Launch several planning subagents in parallel only when their scopes are separable and each has distinct control state. Tell each planning subagent:

- It owns planning for this single change from before `new change` through reviewed planning lock and durable apply handoff only.
- It must not start or explore another change.
- It may run in parallel with other planning subagents, but it must not assume apply order or start implementation.
- It must stop after returning a durable apply handoff; the Controller queues that handoff as `ready-for-apply`.
- It must preserve or create the durable workflow control state record for its assigned change, including its own planning identity and current gate.
- It must use OpenSpec planning documents and explicit decision notes as the implementation contract.
- It must return a planning-review request when current review evidence is missing or stale, then ingest only the Controller-filtered result or blocker packet before apply.
- It must return a durable apply handoff after planning lock.
- It may write cleanup-ready evidence under `.workflow/openspec-continuous/<change>/`, but must not clean, delete, or modify `.agent-review-dialogue/` coordination files.
- It must not run `openspec-apply-change`, implement, verify implementation, archive, commit, compress, or return final completion summary.
- It must report blockers to the main session instead of expanding scope or weakening verification.
- It must return a terminal planning summary with evidence.

Record launch identity evidence in durable workflow control state and queue state before the first wait. If the runtime returns the final session id only after launch, update the state from the launch result or native session listing before treating any timeout as a stall.

If no independent subagent/session runtime is available for a planning subagent now and a separate apply owner later, stop and ask before running the change lifecycle in the main session. Do not silently collapse the whole workflow back into the main session or a single reused subagent.

### 4. Planning Subagent Creates Or Resumes Planning

The planning subagent generates planning documents only when they are missing or the user explicitly requested regeneration. If documents exist, it inspects them and resumes from the earliest missing planning gate.

Use `openspec-ff-change` when available. Otherwise use the supported CLI loop:

```bash
openspec new change "<change-id>"
openspec status --change "<change-id>" --json
openspec instructions <artifact-id> --change "<change-id>" --json
```

After planning documents exist, unresolved assumptions must live in change-local decision notes. Do not rely on the main conversation as the source of truth.

### 5. Planning Subagent Reviews Planning Documents

The planning subagent must not call `agent-review-dialogue` directly. After concrete OpenSpec planning files exist, the planning subagent updates durable workflow control state and returns a planning-review request to the main Controller.

The Controller validates that the request belongs to the active recorded planning subagent, then runs `agent-review-dialogue` in the main session. Scope Agent A edits to generated OpenSpec planning documents and its `change-log.md`; Agent B reviews and writes `review.md`.

Review for ambiguous requirements, unverifiable tasks, missing failure/migration/rollback/security considerations, contradictions across artifacts, scope creep, and whether implementation can proceed with TDD.

Proceed only when the Controller-run review loop reaches its own approval standard: B `approved`, A double-check, and Controller verification. Agent B's `review.md` may report only `needs-revision` or `approved`. If the review loop exposes a blocker through Agent A's `change-log.md` `Status: blocked-on-user` or the Controller's user-input phase, resolve it from project constraints or return a blocker to the main session.

After approval, the Controller captures the durable result packet described in the relay protocol, filters its active A/B review context to that packet plus current planning files and recorded decisions, records filter evidence in durable workflow control state, then sends the filtered packet to the same planning subagent. The planning subagent verifies that the reviewed file freshness evidence still matches the current planning files before creating the planning lock. If the review is blocked, stale, invalid, or fell back to internal review, the Controller captures and filters a blocker packet, records filter evidence in durable workflow control state, returns only that blocker packet to the planning subagent, and the workflow does not advance.

### 6. Planning Subagent Creates Planning Lock And Apply Handoff

Before apply, reread final planning documents, the filtered Controller review result packet, project constraints, and brainstorm decisions.

Do not ask for ceremonial apply approval. Instead, create a planning lock:

- Confirm that human-facing decisions for this change were handled during exploration/brainstorming or are covered by durable constraints.
- Confirm reusable answers have been recorded in the project-level constraints artifact by the main session or Controller. If any required reusable constraint is missing, write `constraint update required` in the planning lock or assigned `.workflow/openspec-continuous/<change>/` control files and return a blocker or cleanup-ready note for Controller action; do not write to the project-level constraint artifact.
- Write change-local answers to OpenSpec decision notes or planning documents.
- Mark low-value or deferred non-blockers as assumptions with verification expectations.
- Write cleanup-ready evidence in `.workflow/openspec-continuous/<change>/` after durable Controller result capture, Controller context filtering, same planning subagent filtered-result intake, reviewed-file freshness verification, and planning lock coverage are complete.
- The cleanup-ready evidence must name the active session-specific A/B coordination directory and durably capture active change id, requesting planning subagent/session identity, planning-review request id or equivalent correlation evidence, B approval, A double-check no-change, Controller verification passed, review round count, reviewed file freshness evidence, Controller context filter evidence, same-planning-subagent filtered-result intake, planning lock coverage, unresolved assumptions, and decisions that affect implementation.
- Do not clean, delete, move, truncate, or rewrite the coordination directory or any `.agent-review-dialogue/` file. Coordination cleanup is Controller-owned and happens only after the Controller validates the cleanup-ready evidence.

After the planning lock exists and required Controller-owned constraint updates are complete, write a durable apply handoff in `.workflow/openspec-continuous/<change>/apply-handoff.md`. It must identify the active change id, queue position, planning lock, approved planning files and freshness evidence, constraints, verification expectations, allowed implementation scope, excluded files, potential worker-splittable areas, and exact next gate. If a required reusable constraint update is still missing, do not create an apply-ready handoff; instead return a blocker or cleanup-ready note with the `constraint update required` path for Controller action. Then the planning subagent returns a terminal planning summary and stops.

The Controller must validate cleanup-ready evidence, perform active A/B coordination directory cleanup itself, recycle the planning subagent, confirm required Controller-owned constraint updates, and only then mark the item `ready-for-apply` before implementation. Prefer closing the planning subagent with the runtime tool. If the runtime cannot close it, mark it `retired` or `not-closeable-retired` in workflow control state and never send it more work. Do not launch the apply owner until Controller cleanup is complete, planning-subagent recycle evidence is recorded, the item is the queue head, queue state and per-change state agree, and no other apply owner is active.

### 7. Launch Apply Owner And Implement With TDD

Launch one fresh apply owner for the queue head. It must have a different runtime identity or session handle from the planning subagent and from any apply worker. Tell it to read only the apply handoff, current OpenSpec planning documents, project constraints, and current repository state needed for implementation. Do not pass planning-subagent private memory or raw A/B dialogue.

Run apply only after current Controller-relayed review evidence, Controller context filter evidence, same planning subagent filtered-result intake, durable decisions, planning lock, cleanup-ready evidence, completed Controller-owned A/B cleanup, planning-subagent recycle evidence, queue-head authorization, apply-owner identity evidence, no other active apply owner, and required Controller-owned constraint updates exist.

Use `openspec-apply-change` when available. Otherwise use:

```bash
openspec instructions apply --change "<change-id>" --json
```

Then the apply owner implements with `superpowers:test-driven-development`:

1. Add or expose a failing test for the approved behavior.
2. Implement the smallest correct change.
3. Refactor while tests stay green.
4. Update OpenSpec task checkboxes only when evidence exists.

The apply owner may use worker subagents to accelerate independent subtasks:

- Split only along clear file/module or test-suite boundaries that avoid overlapping writes.
- Record each worker's assignment, allowed write scope, expected tests, and integration checkpoint in `apply-workers.md` before launch.
- Tell workers they are not alone in the codebase, must not revert others' edits, and must report conflicts instead of forcing changes.
- Continue meaningful owner work while workers run, but do not archive or commit until every worker is integrated, rejected, or retired with evidence.
- Rerun integrated verification after worker results are merged. Worker-local verification is supporting evidence only.

For C/C++ changes, also check RAII, ownership, exception/error safety, resource lifetime, undefined behavior, integer and bounds safety, concurrency assumptions, and concrete unit coverage.

If apply discovers that planning is stale, incomplete, or outside approved scope, the apply owner must return a blocker to the Controller. It must not revise OpenSpec planning documents, run planning review, or keep working from implicit assumptions.

### 8. Apply Owner Verifies, Archives, Commits, And Compresses

Run fresh verification after implementation changes:

```bash
openspec validate "<change-id>" --type change --strict --no-interactive
```

Also run required build, test, lint, static analysis, or project-specific checks. If verification fails, return to TDD and rerun verification. Do not archive a failed required check.

Archive automatically only when verification is current and passing, archive command form is known, and the working tree diff is scoped to the active change. Rerun `openspec validate --all --strict --no-interactive` if archive changed specs, generated OpenSpec files, or metadata.

Create a clean commit automatically when all are true:

- Archive result is verified.
- Required validation and project checks pass.
- Staged diff contains only the completed OpenSpec change and implementation.
- Staged diff does not contain `.workflow/`, workflow control state, `.agent-review-dialogue/`, role subagent coordination logs, or other controller-only artifacts.
- No unrelated user changes are staged.

Inspect `git status`, `git diff`, and `git diff --cached` before staging. Stage only files belonging to the completed OpenSpec change and implementation, using explicit pathspecs instead of broad `git add -A` or `git add .` unless the pathspecs exclude every controller-only artifact. Never stage `.workflow/openspec-continuous/<change>/` control files or `.agent-review-dialogue/` coordination files, even if they are untracked or modified. If any workflow control or review coordination file is already staged, unstage it only when it is clearly generated by this workflow and can be separated non-destructively; otherwise return a blocker to the main session.

Use a Chinese commit message in `类型: 简短描述` format, for example `feat: 完成 <change-id> 变更`.

Preserve unrelated user changes. If unrelated staged files cannot be separated non-destructively, return a blocker to the main session.

After commit, compress context and reload this workflow skill before returning to the main session. Workflow control state remains local and uncommitted; it may be cleaned or retained according to local policy after final summary intake, but it must not be pushed.

### 9. Apply Owner Returns Durable Summary

The apply owner's final summary must include:

- Change id, capability/spec area, and archive result.
- Planning document paths and review evidence.
- Controller review result or blocker packet path and context filter evidence.
- Planning subagent id plus recycle status, apply owner id, apply worker ids plus integration status, queue position, and `.workflow/openspec-continuous/<change>/` control state status.
- Verification commands and final results.
- Files changed.
- Commit hash and commit message.
- Project constraints added or reused.
- Change-local decisions, assumptions, residual risks, and follow-up work.
- Confirmation that neither the planning subagent, apply owner, nor apply workers must be reused for another role or the next change.
- Confirmation that durable workflow control state is terminal for this change and was not committed.
- Safe next action for the main session.

The main session should ingest this summary, update its durable context if needed, recycle the apply owner and all workers, mark the queue item terminal, then start the next ready-for-apply queue item or continue with `openspec-explore` for more candidates.

## Stop Conditions

Stop and ask instead of guessing when:

- Current lifecycle stage or active change id cannot be identified safely.
- Queue state and per-change control state disagree about queue order, active apply item, terminal status, or whether a candidate is ready for apply.
- No independent subagent/session runtime is available for separate planning and apply owner roles in a new change lifecycle.
- An interrupted lifecycle lacks durable workflow control state or identity evidence for the original role subagent/session.
- A planning subagent, apply owner, or apply worker appears timed out, but the extended timeout policy has not yet checked durable evidence, recorded and sent one status request to the same role subagent, and completed the follow-up extended wait.
- Durable workflow control state is non-terminal or conflicting, so launching a replacement subagent, taking over lifecycle work, reusing a role subagent, advancing the queue, or exploring the next change could overlap with a still-running subagent.
- Workflow control state is under `openspec/` or `.openspec/`, is listed as an OpenSpec artifact, or would be staged, archived, committed, or pushed.
- Two parallel planning subagents would edit the same OpenSpec change id, control directory, planning files, or overlapping contract without an explicit dependency order.
- A ready-for-apply item is not the queue head, but apply would start anyway.
- Another item is already in apply, verification, archive, clean commit, compression, reload, or summary return.
- The planning subagent, apply owner, or apply worker have the same runtime identity/session, or the workflow would reuse a retired subagent across roles or changes.
- Planning-subagent recycle evidence is missing before apply-owner launch, or apply-owner/worker recycle evidence is missing before queue advancement or next-change exploration.
- No project-level constraint location exists and a reusable decision must be recorded outside the brainstorming window.
- A planning subagent, apply owner, or apply worker would write directly to a project-level constraint artifact or repository policy file instead of returning a `constraint update required` note for Controller action.
- A required reusable constraint update is missing before cleanup, apply handoff acceptance, or apply launch.
- A missing decision discovered after planning starts affects correctness, compatibility, migration, verification, repository history, destructive operations, or scope and cannot be safely handled as an assumption.
- OpenSpec skill availability, CLI fallback, generated file layout, or archive semantics are unclear.
- Planning review fails artifact validation, falls back to internal review, or requires a decision outside the current scope.
- Controller A/B review completed but the active context has not been filtered to the durable result or blocker packet.
- A/B cleanup cannot identify the active session-specific coordination directory safely.
- A/B cleanup would be triggered or executed by a planning subagent, apply owner, or apply worker instead of the Controller.
- A/B cleanup lacks durable evidence for the active change, requesting planning subagent/session identity, planning-review correlation, Controller context filter evidence, same-planning-subagent filtered-result intake, planning lock coverage, or cleanup-ready evidence.
- A role subagent would rely on implicit prior memory instead of explicit artifacts.
- The apply owner would need to alter proposal, design, spec deltas, decision notes, or raw A/B coordination artifacts instead of returning a planning-stale blocker.
- An apply worker would need to modify files outside its recorded write scope, stage files, commit, archive, update queue state, report final task completion directly to the main session, or present a recovery status packet as authorization for archive, commit, summary, or queue advancement.
- Apply owner cannot reconcile worker results, worker write scopes overlap, or integrated verification has not run after worker changes.
- Required verification fails and cannot be fixed within approved scope.
- Archive or diff includes unrelated files.
- A clean commit would require including unrelated changes, workflow control state, `.agent-review-dialogue/` files, controller-only artifacts, or altering user changes.
- Any step would destructively alter git history, delete unrelated files, broadly reformat, or revert user changes.

## Relationship To Manual Workflow

Use `openspec-change-workflow` when the user wants explicit human approval at each major gate. Use this skill when the user wants the main session to handle only explore/brainstorm/controller duties while queued changes plan in parallel, apply remains serialized by queue order, and each queue-head apply is driven by one apply owner that may dispatch scoped workers, with uncommitted `.workflow` control state, explicit subagent recycling, clean commit, context compression, reload, and summary return.
