---
name: openspec-continuous-change-workflow
description: "Use when the user wants an automated, continuous OpenSpec change lifecycle where the main session is only explore/brainstorm/controller, each OpenSpec change is delegated before new change creation to a dedicated workflow subagent, and planning review is relayed through the Controller before the same subagent continues apply/TDD, verification, archive, clean commit, context compression, reload, and summary return."
---

# OpenSpec Continuous Change Workflow

## Purpose

Run OpenSpec changes continuously while keeping the main conversation clean. The main session owns only exploration, brainstorming, reusable decision capture, workflow-subagent launch/control, Controller-relayed planning review, final summary intake, and the next exploration window. Each individual change runs in its own dedicated workflow subagent, launched before `new change`, and the main session does not execute the change lifecycle when an independent subagent/session runtime is available.

The intended loop is:

```text
main: classify state and load project constraints
-> main: explore next candidate
-> main: brainstorm and batch decision-worthy questions
-> main: record reusable answers as project constraints
-> main: start one workflow subagent before new change
-> subagent: create new change or resume the assigned OpenSpec change
-> subagent: generate planning documents and return planning-review request
-> main: run agent-review-dialogue on generated planning documents
-> main: capture durable A/B result or blocker packet outside review coordination dir
-> main: filter A/B review context to durable packet, current plan, decisions, blockers, and freshness evidence
-> main: deliver filtered packet to the same workflow subagent
-> subagent: apply with TDD
-> subagent: verify, archive, and clean commit
-> subagent: compress context and reload workflow instructions
-> subagent: return durable completion summary
-> main: ingest summary, then explore the next candidate
```

This skill is an automation variant, not a shortcut around correctness. Do not skip stage classification, stale-evidence checks, planning review, verification, archive inspection, unrelated-change protection, or repository history safety checks.

## Non-Negotiable Model

- Use exactly one dedicated workflow subagent per active OpenSpec change.
- Treat workflow subagent timeouts as soft signals. Use the extended timeout policy below before declaring a workflow subagent stalled, missing, or unrecoverable.
- Start that workflow subagent before `openspec new change`, `openspec-ff-change`, or the CLI planning-generation fallback.
- Do not run planning generation, internal/manual planning review, apply, verification, archive, or clean commit in the main session when an independent subagent/session runtime is available. The only planning-review work the main session may perform is Controller-orchestrated `agent-review-dialogue` in response to a recorded workflow subagent request.
- Do not reuse one workflow subagent across multiple changes. End, freeze, or discard it after its change is archived, clean committed, compressed, and summarized.
- If a run is interrupted after launch, continue the recorded workflow subagent/session for that change only when durable workflow state and identity evidence both exist and agree; otherwise return a recovery blocker instead of taking over the lifecycle in the main session or launching a replacement subagent.
- Persist workflow subagent identity and lifecycle state for the active change; timeout recovery, resume decisions, and next-change authorization must be based on that durable state plus repository evidence.
- Keep main-session context out of implementation. Pass only explicit artifacts selected by the Controller: exploration result, brainstorm decisions, project constraints, relevant archived specs, relevant post-archive summaries, and current repository state needed to start safely.
- After OpenSpec planning documents exist, treat those documents and explicit decision notes as the implementation contract. Do not rely on broad conversation history, hidden assumptions, terminal scrollback, or subagent private memory.
- The workflow subagent must not launch `agent-review-dialogue` itself. After planning documents exist, it returns a planning-review request to the main Controller; the Controller launches `agent-review-dialogue`, captures and filters the durable result or blocker packet, and only then resumes the same workflow subagent with that filtered packet.
- After Controller-run A/B review, filter the main session's active context before any workflow-subagent resume, handoff, planning lock, apply, archive, clean commit, summary, or next-change exploration. Keep only the durable result or blocker packet, current planning files, recorded decisions, blockers, and freshness evidence; exclude raw A/B dialogue, round-by-round critique, temporary coordination notes, and review-agent private reasoning from later prompts, handoffs, summaries, and exploration inputs.
- Automatically create a scoped clean commit after archive and post-archive validation when the diff contains only the completed change.
- Never run destructive git operations, delete unrelated files, accept failed required verification, or archive unexpected diffs without explicit user authorization.

## Workflow Subagent Timeout Policy

Workflow subagents often perform slow OpenSpec planning, Controller-relayed review intake, TDD, builds, validation, archive, clean commit, context compression, and workflow reload. The main session must therefore use longer waits than ordinary exploration subagent calls and must not treat one or more `wait_agent` timeouts as completion failure.

- Default to long waits for workflow subagents: use at least 10 minutes per `wait_agent` call when waiting for planning/review/apply/verify/archive/commit/compression/reload/summary results.
- For implementation, builds, full test runs, archive, clean commit, context compression, or workflow reload, prefer 20-30 minute waits when the tool allows it.
- A `wait_agent` timeout means "no final message returned in that window"; it does not mean the subagent is dead, failed, or safe to replace.
- After a timeout, inspect durable evidence before drawing conclusions: `git status`, the active change directory, recorded workflow state and subagent/session identity evidence, OpenSpec status, project constraint artifacts, and recent planning/review/verification/archive/commit/compression/reload artifacts. Prefer evidence written after the subagent launch or latest main-session handoff over stale terminal output.
- If durable workflow state is missing, conflicts with identity evidence, or cannot be reconciled with repository artifacts for the active change, return a recovery blocker that freezes the active change. Identity evidence or artifact progress alone is not enough to continue, replace, take over in the main session, or explore the next change when the durable state anchor is unsafe.
- If the durable workflow state names the active change and recorded subagent/session, and any OpenSpec change artifact, review artifact, verification artifact, repository diff, or state transition shows current or plausible progress for that change, continue or resume that same recorded subagent instead of launching a replacement.
- Before declaring a recorded workflow subagent stalled, record one concise status request in durable workflow state, send it to that same recorded subagent/session, and wait again with the extended timeout. Do not send repeated status pings that could distract from long-running work. If no reliable same-subagent continuation target exists, return a recovery blocker instead of guessing a status target.
- Return a recovery blocker when no final summary has returned and safe same-subagent continuation cannot be established from durable workflow state, identity evidence, artifact progress, and the single follow-up status request when a reliable status target existed. The blocker is a safety stop, not authorization for replacement, main-session takeover, or next-change exploration; those actions remain prohibited while they could risk duplicate planning, duplicate implementation, mixed commits, skipped gates, or acting while the current subagent may still be running.
- When reporting a possible stall or returning a recovery blocker, explicitly say that tool-level timeout is a soft signal and name the evidence checked. Avoid saying the subagent has not created artifacts unless the filesystem check was run after the latest possible subagent notification.

## Durable Workflow State And Identity Evidence

Every active change needs a durable workflow state record that survives main-session interruption and tool timeouts. Prefer a repository-established state artifact when one exists; otherwise use a change-local `workflow-state.md` under the active OpenSpec change directory once the change id exists. If the workflow subagent is launched before the change directory exists, record the same fields in the explicit handoff or controller session record and require the workflow subagent to copy them into the change-local state file as soon as it creates or resumes the change.

The state record is for lifecycle control, not for broad implementation context. It must stay small and include:

- Active change id or pre-change candidate id.
- Workflow subagent/session identity: runtime, title or handle, session id when available, launch timestamp, and latest continuation target.
- Current lifecycle gate, latest completed gate, next expected evidence, and paths to planning/review/verification/archive/commit/compression/reload artifacts.
- Planning review request, Controller A/B result or blocker packet path, Controller review context filter evidence, and reviewed-file freshness evidence.
- Main-session handoff timestamp and the timestamp/content summary of the single timeout status request, if one was sent.
- Terminal state: final summary returned, blocker returned, or subagent closed after archive, clean commit, context compression, workflow reload, and summary intake.

When a timeout status request is sent, record the request timestamp, request text summary, target continuation/session id, and follow-up wait result. The follow-up result should say whether the same subagent returned a final summary, returned a blocker, showed durable progress, remained silent through the extended wait, or could not be contacted through the recorded continuation target.

The workflow subagent must update durable workflow state whenever it enters or completes a lifecycle gate, starts or finishes a long verification/build/test run, and before and after archive, clean commit, context compression, workflow reload, or final summary return. The main session must update the same state when it launches or resumes the subagent, sends the single timeout status request, completes the follow-up extended wait, ingests the final summary, or records a recovery blocker. Stale state is not a valid anchor for replacement, main-session takeover, or next-change exploration.

The main session may use this state only to decide launch, resume, status-request, recovery-blocker, and next-exploration actions. It must not start a replacement subagent, take over lifecycle work, or explore the next change while the state is non-terminal or while a recorded subagent may still be running.

## Controller-Relayed Planning Review

Use this relay whenever a recorded workflow subagent reaches planning-document review. Do not run A/B review inside the workflow subagent, and do not let the workflow subagent replace A/B review with an internal self-review.

The workflow subagent must stop after planning generation, update durable workflow state, and return a planning-review request to the main Controller. The request must include:

- Active change id, workflow subagent/session identity, current gate, and planning files with paths plus freshness evidence such as timestamps, hashes, or latest diff summary.
- Exact review target manifest, allowed edit scope, durable decision paths, and explicit exclusions.
- Relevant brainstorm decisions, project constraints, change-local decisions, and selected cross-change artifacts, not broad transcript context.
- Existing review artifacts to reuse or invalidate, if any.
- Blockers or decisions needed before A/B review can continue.

The Controller must validate the request against durable workflow state, then launch `agent-review-dialogue` in the main session with only the requested manifest, planning files, durable decisions, and explicit exclusions. If A/B raises product, API, compatibility, migration, dependency, scope, acceptance, verification, or repository-history decisions not already covered by durable constraints, resolve them from project constraints or return a blocker; do not let the workflow subagent guess.

After `agent-review-dialogue` reaches approval, the Controller must capture a durable result packet outside any review coordination directory that may later be deleted. The packet must include active change id, requesting workflow subagent/session identity, planning-review request id or equivalent correlation evidence, B approval, A double-check result, Controller verification result, review round count, reviewed file freshness evidence, unresolved assumptions, decisions, and the active session-specific A/B coordination directory path for later cleanup.

If `agent-review-dialogue` blocks, fails artifact validation, falls back to internal review, or finds stale reviewed files, the Controller must capture a durable blocker packet outside the review coordination directory before resuming the workflow subagent. The blocker packet must include active change id, requesting workflow subagent/session identity, planning-review request id or equivalent correlation evidence, blocker status, failing validation or freshness evidence, affected files, required decision or recovery action, and next gate.

Immediately after producing an approved result packet or blocker packet, the Controller must filter its active review context. The post-review Controller context used for resuming the workflow subagent or future exploration is limited to:

- The durable result or blocker packet path and concise contents.
- The current reviewed planning files and freshness evidence.
- Decisions, unresolved assumptions, blockers, and exact next gate.
- The coordination directory path only for later cleanup.

Exclude A/B conversation transcripts, intermediate comments that were resolved, raw review-agent outputs, private coordination files, and speculative reasoning from subsequent prompts, handoffs, summaries, and next-change exploration. If the runtime supports context compaction, filtering, or thread summarization, run it immediately around the result or blocker packet before resuming the workflow subagent. If not, explicitly mark the raw A/B review as excluded and never paste it into the workflow subagent resume prompt.

Record the filter evidence in durable workflow state: result or blocker packet path, kept categories, excluded categories, filter timestamp, and next gate. This evidence is required before the Controller resumes the workflow subagent.

The Controller then sends only the filtered result or blocker packet to the same workflow subagent and waits for it to verify freshness before creating the planning lock. If A/B blocks, fails validation, falls back to internal review, or reviews stale files, the workflow subagent records the blocker intake in durable workflow state and does not advance to planning lock or apply.

## Main Session Responsibilities

The main session is an explorer and controller, not the change executor.

It must:

- Classify whether there is an active in-progress change or whether the next action is exploration.
- Load durable project-level constraints before asking questions.
- Run `openspec-explore` or the supported exploration fallback when the next candidate is unknown.
- Use `superpowers:brainstorming` after exploration and before any planning document is created.
- Ask only high-value questions during the exploration-to-brainstorming window.
- Record reusable answers in a project-level constraint artifact.
- Record change-local answers in a handoff note that the workflow subagent must materialize into OpenSpec planning documents or decision notes.
- Start one workflow subagent with a minimal, explicit handoff.
- Run `agent-review-dialogue` only when the recorded workflow subagent returns a planning-review request, then filter the review context before resuming that same subagent.
- Record the workflow subagent identity and lifecycle state before making timeout, resume, recovery, or next-change decisions.
- Ingest the workflow subagent's final durable summary after completion.
- Resume exploration only after the subagent reports archive, clean commit, context compression, and workflow reload evidence.

The main session must not carry detailed implementation context forward. Its durable memory between changes should be project constraints plus compressed summaries.

## Workflow Subagent Responsibilities

The workflow subagent owns the full lifecycle for one change:

- Reclassify current state for the assigned change from repository and OpenSpec artifacts.
- Create or resume the assigned OpenSpec change, starting with `new change` when the change does not yet exist.
- Generate proposal, design, spec deltas, tasks, and decision notes using the available OpenSpec skill or supported CLI fallback.
- Return a planning-review request when generated planning documents need review or review evidence is stale; do not launch `agent-review-dialogue` directly.
- Ingest only the Controller-filtered review result or blocker packet and verify freshness before planning lock.
- Create a planning lock after review approval and durable decision capture.
- Apply and implement with TDD.
- Verify with OpenSpec validation and required project checks.
- Archive only after required verification passes.
- Create a scoped clean commit after archive and post-archive validation.
- Compress the change context and reload the workflow instructions before returning.
- Return a durable completion summary to the main session.

The workflow subagent must stop and return a blocker to the main session when the change requires scope expansion, policy decisions not covered by the brainstorm window, unclear OpenSpec command semantics, failed verification that cannot be fixed within scope, archive surprises, unrelated diff contamination, or unsafe git history operations.

## Project-Level Constraints

Before brainstorming or launching a workflow subagent, discover a durable project-level constraint artifact. Prefer an existing canonical location in this order:

1. Repository instruction files that already govern agents, such as `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, or `.codex/instructions.md`.
2. Existing OpenSpec project files that document conventions, such as `openspec/project.md`, `.openspec/project.md`, `openspec/CONSTRAINTS.md`, or `.openspec/CONSTRAINTS.md`.
3. Existing project decision logs, architecture records, or policy files that the repository clearly uses.

If no project-level constraint artifact exists, ask once during the exploration-to-brainstorming window where durable workflow constraints should live. After the user chooses, record that location and reuse it for future cycles. Do not invent a hidden constraints file when the repository has no convention.

When writing constraints:

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
| `blocker` | Wrong answer may invalidate the change, damage history, or violate policy | Ask before launching the workflow subagent, or return a blocker if discovered later |
| `reusable-policy` | Answer should guide future changes | Ask once, then record in project constraints |
| `change-local` | Answer affects only this change | Ask before launch and pass as subagent handoff input |
| `low-value` | Answer is inferable, testable, or immaterial | Do not ask; proceed and document the assumption only if useful |

## Entry Protocol

Start every invocation by classifying the current lifecycle state:

1. Read the user's latest request for a stage intent: explore, brainstorm, create, review, apply, verify, archive, commit, compress, reload, continue, or run continuously.
2. Inspect repository state with non-destructive commands: `git status`, OpenSpec directories, current change files, review artifacts, constraint artifacts, and durable verification results when present.
3. Identify the active change id, requested stage, latest completed gate with evidence, stale or missing evidence, and next safe action.
4. Load project-level constraints before deciding whether to ask anything.
5. If the next safe action belongs to a change lifecycle, launch the dedicated workflow subagent before `new change`. When resuming an interrupted change, continue the same recorded workflow subagent/session only when durable workflow state and identity evidence exist; if either is missing, return a recovery blocker to the main session instead of executing the lifecycle in the main session or launching a replacement subagent.

If multiple active changes exist and the active change cannot be determined safely, ask once before acting.

## Stage Router

| Observed state | Main-session action | Workflow-subagent action |
| --- | --- | --- |
| No next candidate is known | Explore | None yet |
| Candidate is known but scope decisions are missing | Brainstorm decision window | None yet |
| Brainstorm decisions are complete and no OpenSpec change exists | Launch per-change workflow subagent before `new change` | Create change and planning documents |
| Planning files exist but no approved review exists | Continue the recorded workflow subagent until it emits a planning-review request, then Controller runs A/B review; otherwise return a recovery blocker | Ingest filtered review result or blocker packet |
| Planning review is current but implementation has not started | Continue the recorded workflow subagent/session only with durable workflow state, identity evidence, durable Controller result capture, Controller context filter evidence, and same-subagent filtered-result intake; otherwise return a recovery blocker | Create planning lock, clean review artifacts, apply with TDD |
| Implementation is incomplete | Continue the recorded workflow subagent/session only with durable workflow state and identity evidence; otherwise return a recovery blocker | Continue from failing or missing tests |
| Implementation appears complete | Continue the recorded workflow subagent/session only with durable workflow state and identity evidence; otherwise return a recovery blocker | Run fresh verification |
| Verification passes and archive is pending | Continue the recorded workflow subagent/session only with durable workflow state and identity evidence; otherwise return a recovery blocker | Archive and post-archive validation |
| Archive completed but no clean commit exists | Continue the recorded workflow subagent/session only with durable workflow state and identity evidence; otherwise return a recovery blocker | Stage scoped files and commit |
| Clean commit exists but no returned summary exists | Continue the recorded workflow subagent/session only with durable workflow state and identity evidence; otherwise return a recovery blocker | Compress context, reload workflow, and return summary |
| Workflow subagent returned durable summary | Ingest summary and update durable context | End this subagent; do not reuse it |
| Summary is ingested and continuity is authorized | Explore next | None; previous subagent is closed |

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

- An `agent-review-dialogue` approval is current only when it was launched by the Controller from the recorded workflow subagent's planning-review request, and its target scope plus freshness evidence correspond to the current planning files. If any reviewed planning file changed after approval, rerun review through the Controller relay.
- A/B review evidence is not current if it depends on workflow subagent implicit memory, was produced by an internal-review fallback, was not returned to the same workflow subagent as a durable Controller result packet, or lacks evidence that the Controller filtered A/B context after review.
- A/B review cleanup is current only after the final Controller-orchestrated `agent-review-dialogue` result has been durably captured outside the coordination directory that will be deleted, Controller context filtering is complete, the same workflow subagent has ingested the filtered result packet, and the planning lock confirms required decisions are covered by brainstorm answers, durable constraints, or change-local decision notes. Required durable evidence includes active change id, requesting workflow subagent/session identity, planning-review request id or equivalent correlation evidence, B approval, A double-check no-change, Controller verification passed, review round count, reviewed file freshness evidence, Controller context filter evidence, unresolved assumptions, and decisions that affect implementation.
- Clean only the active session-specific A/B coordination directory. Do not delete the parent `.agent-review-dialogue/` tree, sibling review runs, OpenSpec planning documents, decision notes, implementation files, or project constraints.
- If cleanup already happened, continue only when durable evidence proves the same active change, requesting workflow subagent/session identity, planning-review correlation, B approval, A double-check no-change, Controller verification, reviewed file freshness, Controller context filter evidence, same-subagent filtered-result intake, and planning lock coverage. Otherwise return a recovery blocker instead of recreating or guessing deleted review context.

## Workflow

### 1. Explore And Brainstorm In Main Session

Use `openspec-explore` or the supported exploration fallback first when the next change is not already known. Then use `superpowers:brainstorming` before launching the workflow subagent.

During this window, decide the change goal, user-visible outcome, in-scope and out-of-scope boundaries, acceptance criteria, verification expectations, compatibility/migration policy, and reusable constraints worth recording.

Do not create or edit OpenSpec planning documents in the main session during this step. The workflow subagent creates or resumes them after launch.

### 2. Prepare The Per-Change Handoff

Create a minimal handoff for the workflow subagent. Include only:

- Active change id or naming constraints, if known.
- Exploration result and chosen candidate.
- Brainstorm decisions and acceptance criteria.
- Project-level constraint artifact path and relevant entries.
- Change-local assumptions and verification expectations.
- Relevant archived specs or compressed summaries selected by the Controller.
- Durable workflow state location or pre-change state fields that the subagent must materialize once the change directory exists.
- Current repository state needed to avoid mixing unrelated changes.
- Stop conditions and requirement to return blockers to the main session.

Exclude broad transcript history, raw terminal scrollback, private reasoning, unrelated prior-change context, and stale assumptions. The subagent must turn this handoff into OpenSpec planning documents or decision notes before apply.

### 3. Launch One Workflow Subagent

Launch one dedicated workflow subagent for the change. Tell it:

- It owns this single change from before `new change` through clean commit, context compression, workflow reload, and summary return.
- It must not start or explore another change.
- It must preserve or create the durable workflow state record for its assigned change, including its own identity and current gate.
- It must use OpenSpec planning documents and explicit decision notes as the implementation contract.
- It must return a planning-review request when current review evidence is missing or stale, then ingest only the Controller-filtered result or blocker packet before apply.
- It must report blockers to the main session instead of expanding scope or weakening verification.
- It must return a durable final summary with evidence.

Record launch identity evidence in durable workflow state or the explicit handoff before the first wait. If the runtime returns the final session id only after launch, update the state from the launch result or native session listing before treating any timeout as a stall.

If no independent subagent/session runtime is available, stop and ask before running the change lifecycle in the main session. Do not silently collapse the whole workflow back into the main session.

### 4. Subagent Creates Or Resumes Planning

The workflow subagent generates planning documents only when they are missing or the user explicitly requested regeneration. If documents exist, it inspects them and resumes from the earliest missing gate.

Use `openspec-ff-change` when available. Otherwise use the supported CLI loop:

```bash
openspec new change "<change-id>"
openspec status --change "<change-id>" --json
openspec instructions <artifact-id> --change "<change-id>" --json
```

After planning documents exist, unresolved assumptions must live in change-local decision notes. Do not rely on the main conversation as the source of truth.

### 5. Subagent Reviews Planning Documents

The workflow subagent must not call `agent-review-dialogue` directly. After concrete OpenSpec planning files exist, the workflow subagent updates durable workflow state and returns a planning-review request to the main Controller.

The Controller validates that the request belongs to the active recorded workflow subagent, then runs `agent-review-dialogue` in the main session. Scope Agent A edits to generated OpenSpec planning documents and its `change-log.md`; Agent B reviews and writes `review.md`.

Review for ambiguous requirements, unverifiable tasks, missing failure/migration/rollback/security considerations, contradictions across artifacts, scope creep, and whether implementation can proceed with TDD.

Proceed only when the Controller-run review loop reaches its own approval standard: B `approved`, A double-check, and Controller verification. Agent B's `review.md` may report only `needs-revision` or `approved`. If the review loop exposes a blocker through Agent A's `change-log.md` `Status: blocked-on-user` or the Controller's user-input phase, resolve it from project constraints or return a blocker to the main session.

After approval, the Controller captures the durable result packet described in the relay protocol, filters its active A/B review context to that packet plus current planning files and recorded decisions, records filter evidence in durable workflow state, then sends the filtered packet to the same workflow subagent. The workflow subagent verifies that the reviewed file freshness evidence still matches the current planning files before creating the planning lock. If the review is blocked, stale, invalid, or fell back to internal review, the Controller captures and filters a blocker packet, records filter evidence in durable workflow state, returns only that blocker packet to the workflow subagent, and the workflow does not advance.

### 6. Subagent Creates Planning Lock

Before apply, reread final planning documents, the filtered Controller review result packet, project constraints, and brainstorm decisions.

Do not ask for ceremonial apply approval. Instead, create a planning lock:

- Confirm that human-facing decisions for this change were handled during exploration/brainstorming or are covered by durable constraints.
- Write reusable answers to the project-level constraints artifact.
- Write change-local answers to OpenSpec decision notes or planning documents.
- Mark low-value or deferred non-blockers as assumptions with verification expectations.
- Clean only the active session-specific A/B coordination directory after durable Controller result capture, Controller context filtering, same workflow subagent filtered-result intake, and planning lock coverage are complete.
- Before deleting the coordination directory, durably capture active change id, requesting workflow subagent/session identity, planning-review request id or equivalent correlation evidence, B approval, A double-check no-change, Controller verification passed, review round count, reviewed file freshness evidence, Controller context filter evidence, unresolved assumptions, and decisions that affect implementation in a location that will not be deleted by cleanup.

After the planning lock exists, continue directly to apply.

### 7. Subagent Applies And Implements With TDD

Run apply only after current Controller-relayed review evidence, Controller context filter evidence, same workflow subagent filtered-result intake, durable decisions, planning lock, completed A/B cleanup, and required constraint updates exist.

Use `openspec-apply-change` when available. Otherwise use:

```bash
openspec instructions apply --change "<change-id>" --json
```

Then implement with `superpowers:test-driven-development`:

1. Add or expose a failing test for the approved behavior.
2. Implement the smallest correct change.
3. Refactor while tests stay green.
4. Update OpenSpec task checkboxes only when evidence exists.

For C/C++ changes, also check RAII, ownership, exception/error safety, resource lifetime, undefined behavior, integer and bounds safety, concurrency assumptions, and concrete unit coverage.

### 8. Subagent Verifies, Archives, Commits, And Compresses

Run fresh verification after implementation changes:

```bash
openspec validate "<change-id>" --type change --strict --no-interactive
```

Also run required build, test, lint, static analysis, or project-specific checks. If verification fails, return to TDD and rerun verification. Do not archive a failed required check.

Archive automatically only when verification is current and passing, archive command form is known, and the working tree diff is scoped to the active change. Rerun `openspec validate --all --strict --no-interactive` if archive changed specs, generated state, or metadata.

Create a clean commit automatically when all are true:

- Archive result is verified.
- Required validation and project checks pass.
- Staged diff contains only the completed OpenSpec change and implementation.
- No unrelated user changes are staged.

Inspect `git status`, `git diff`, and `git diff --cached` before staging. Stage only files belonging to the completed change.

Use a Chinese commit message in `类型: 简短描述` format, for example `feat: 完成 <change-id> 变更`.

Preserve unrelated user changes. If unrelated staged files cannot be separated non-destructively, return a blocker to the main session.

After commit, compress context and reload this workflow skill before returning to the main session.

### 9. Subagent Returns Durable Summary

The workflow subagent's final summary must include:

- Change id, capability/spec area, and archive result.
- Planning document paths and review evidence.
- Controller review result or blocker packet path and context filter evidence.
- Verification commands and final results.
- Files changed.
- Commit hash and commit message.
- Project constraints added or reused.
- Change-local decisions, assumptions, residual risks, and follow-up work.
- Confirmation that the subagent must not be reused for the next change.
- Confirmation that durable workflow state is terminal for this change.
- Safe next action for the main session.

The main session should ingest this summary, update its durable context if needed, then continue with `openspec-explore` for the next candidate.

## Stop Conditions

Stop and ask instead of guessing when:

- Current lifecycle stage or active change id cannot be identified safely.
- No independent subagent/session runtime is available for a new change lifecycle.
- An interrupted lifecycle lacks durable workflow state or identity evidence for the original workflow subagent/session.
- A workflow subagent appears timed out, but the extended timeout policy has not yet checked durable evidence, recorded and sent one status request to the same subagent, and completed the follow-up extended wait.
- Durable workflow state is non-terminal or conflicting, so launching a replacement subagent, taking over lifecycle work, or exploring the next change could overlap with a still-running subagent.
- No project-level constraint location exists and a reusable decision must be recorded outside the brainstorming window.
- A missing decision discovered after planning starts affects correctness, compatibility, migration, verification, repository history, destructive operations, or scope and cannot be safely handled as an assumption.
- OpenSpec skill availability, CLI fallback, generated file layout, or archive semantics are unclear.
- Planning review fails artifact validation, falls back to internal review, or requires a decision outside the current scope.
- Controller A/B review completed but the active context has not been filtered to the durable result or blocker packet.
- A/B cleanup cannot identify the active session-specific coordination directory safely.
- A/B cleanup lacks durable evidence for the active change, requesting workflow subagent/session identity, planning-review correlation, Controller context filter evidence, same-subagent filtered-result intake, or planning lock coverage.
- The workflow subagent would rely on implicit prior memory instead of explicit artifacts.
- Required verification fails and cannot be fixed within approved scope.
- Archive or diff includes unrelated files.
- A clean commit would require including unrelated changes or altering user changes.
- Any step would destructively alter git history, delete unrelated files, broadly reformat, or revert user changes.

## Relationship To Manual Workflow

Use `openspec-change-workflow` when the user wants explicit human approval at each major gate. Use this skill when the user wants the main session to handle only explore/brainstorm/controller duties while a dedicated per-change workflow subagent executes the full OpenSpec lifecycle from before `new change` through clean commit, context compression, reload, and summary return.
