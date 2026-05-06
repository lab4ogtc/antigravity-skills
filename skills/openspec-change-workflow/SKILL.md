---
name: openspec-change-workflow
description: Use when the user wants to run, resume, audit, or continue an OpenSpec change lifecycle at any stage, including planning document review, apply/TDD, verification, archive, commit handoff, or next-change discovery.
---

# OpenSpec Change Workflow

## Purpose

Use this workflow to take one OpenSpec change from an idea to archived completion, or to resume an already-started change from its current stage. The main session is the Controller: it classifies state, explores, brainstorms, asks required human-confirmation questions, launches or resumes one per-change workflow subagent, and ingests the final summary. The workflow subagent owns the change lifecycle from before `new change` through planning, review, apply, verification, archive, commit handling, compression, and summary return.

The full intended path is:

```text
main: superpowers:brainstorming and required decision capture
-> main: launch or resume one workflow subagent before openspec-ff-change/new change
-> subagent: openspec-ff-change, or CLI-supported fast-forward fallback
-> subagent: agent-review-dialogue on generated planning documents
-> subagent: pre-apply self-check and confirmation packet to main
-> subagent: A/B review artifact cleanup after durable capture
-> subagent: openspec-apply-change + superpowers:test-driven-development
-> subagent: openspec-verify-change + OpenSpec validation + human review packet
-> subagent: openspec archive "<change-id>"
-> subagent: user-confirmed clean commit handling
-> subagent: context compression
-> subagent: reload openspec-change-workflow skill
-> subagent: durable summary return
-> main: ingest summary, then openspec-explore
-> repeat or stop
```

This skill is not a shortcut around review. Fast generation creates planning documents quickly; it does not approve their content.

The frontmatter `description` is only an activation trigger. It is not permission to skip current-stage classification, workflow-subagent isolation, human confirmation gates, review, verification, archive checks, or clean commit handling.

Do not assume the workflow always starts at step 1. When invoked midstream, first identify the current stage, verify that prior gates have evidence, then continue from the earliest incomplete required gate.

## Non-Negotiable Rules

- Start by detecting the current lifecycle stage from repository state, OpenSpec files, review artifacts, git status, and the user's latest instruction.
- Do not restart from brainstorming or regenerate OpenSpec documents when valid later-stage artifacts already exist, unless the user explicitly asks to restart.
- When resuming midstream, run only the missing prerequisites for the current or next stage; preserve completed, validated stages.
- The Controller uses `superpowers:brainstorming` before any workflow subagent creates or changes OpenSpec planning documents.
- For lifecycle actions after workflow-subagent launch, the workflow subagent prefers the corresponding OpenSpec workflow skill when available: `openspec-ff-change`, `openspec-apply-change`, and `openspec-verify-change`. The Controller uses `openspec-explore` only after the completed subagent summary is ingested.
- Do not call unsupported top-level OpenSpec CLI commands. In OpenSpec CLI 1.3.1, `openspec apply`, `openspec verify`, `openspec explore`, and `openspec new change --ff` are not valid top-level invocations.
- After the Controller confirms the goal, scope, and acceptance criteria, launch or resume the workflow subagent; the workflow subagent uses `openspec-ff-change` to generate the initial planning documents. If that skill is not available, the workflow subagent uses the CLI-supported fast-forward fallback described below.
- The workflow subagent uses `agent-review-dialogue` only after the planning documents exist, with those generated OpenSpec files as the reviewed targets.
- Do not use `agent-plan-dialogue` for this workflow. OpenSpec owns planning generation; `agent-review-dialogue` owns adversarial review and refinement of the generated documents.
- The workflow subagent must not start OpenSpec apply work until the reviewed planning documents pass a final self-check and the Controller records explicit user approval to apply them.
- Before the workflow subagent starts apply work, it cleans the session-specific A/B review coordination artifacts for the planning review only after final B approval, A double-check, Controller verification, and mandatory durable result capture are complete.
- Use exactly one dedicated workflow subagent per active OpenSpec change when an independent subagent/session runtime is available.
- Treat workflow subagent timeouts as soft signals. Use the extended timeout policy below before declaring a workflow subagent stalled, missing, or unrecoverable.
- Start the workflow subagent before `openspec new change`, `openspec-ff-change`, or the CLI planning-generation fallback. The main session must not generate planning documents, run planning review, apply, verify, archive, or perform commit handling for a new change after the subagent runtime is available.
- Do not reuse one workflow subagent across multiple changes. End, freeze, or discard it after the change is archived, commit handling is resolved, context is compressed, the workflow is reloaded, and the summary is returned.
- If a run is interrupted after launch, continue the recorded workflow subagent/session for that change only when identity evidence exists. If identity evidence is missing, stop for a recovery decision instead of silently taking over the lifecycle in the main session or launching an unrecorded replacement.
- Cross-change learning must come from explicit artifacts selected by the Controller, not from a workflow subagent's implicit memory. Valid inputs include archived specs, post-archive context compression summaries, commit hashes, changed-file lists, compatibility notes, and current OpenSpec documents.
- Keep `agent-review-dialogue` A/B agents isolated from workflow subagents. A/B review must receive only the exact target manifest, current planning documents, and explicitly recorded decisions; it must not inherit or rely on workflow subagent memory.
- Nested `agent-review-dialogue` A/B sessions may run inside the workflow subagent's planning-review stage. They are review roles, not replacement lifecycle subagents, and their approval does not replace workflow-subagent verification, human-confirmation gates, or Controller summary intake.
- The workflow subagent implements after apply with `superpowers:test-driven-development`.
- The workflow subagent must not archive until `openspec-verify-change` or its manual fallback has no blocking findings, `openspec validate`, relevant tests/builds, and human review all pass. Failed verification cannot be waived into an archive.
- Do not commit or push unless the user explicitly confirms. Preserve unrelated user changes and stage only files that belong to the completed change.
- This ordinary workflow keeps every human-confirmation gate. Do not auto-continue through apply, archive, commit, push, or next-change start merely because a subagent has produced a packet or a previous continuous workflow variant would proceed.
- After `openspec archive "<change-id>"` completes and before the next OpenSpec exploration, the workflow subagent proactively performs context compression with a durable structured summary of the completed change, then reloads this workflow skill from disk or the current runtime skill source.
- After archive, user-confirmed clean commit handling, context compression, workflow reload, and durable summary intake are complete, the Controller uses `openspec-explore` to discover the next candidate change, then asks before starting the next loop. If the commit is deferred, pause or stop instead of exploring.

## OpenSpec Invocation Compatibility

Before running any OpenSpec action, check whether the corresponding OpenSpec workflow skill is callable in the current agent environment. A template bundled inside the OpenSpec package is not enough; it must be exposed as a native skill, slash command, or equivalent current-runtime action. If it is callable, use the skill instead of inventing a top-level CLI command.

Keep discovery inside the current agent runtime's visible skill or command list. Do not scan another tool's configuration directories to find an OpenSpec workflow skill.

| Workflow action | Preferred OpenSpec skill | CLI-supported fallback when the skill is unavailable |
| --- | --- | --- |
| Fast-forward planning artifact creation | `openspec-ff-change` | `openspec new change "<change-id>"`, then loop through `openspec status --change "<change-id>" --json` and `openspec instructions <artifact-id> --change "<change-id>" --json` until all `applyRequires` artifacts are complete |
| Apply implementation tasks | `openspec-apply-change` | `openspec instructions apply --change "<change-id>" --json`, read returned `contextFiles`, implement pending tasks with TDD, and update task checkboxes |
| Verify implementation against artifacts | `openspec-verify-change` | `openspec validate "<change-id>" --type change --strict --no-interactive`, plus manual artifact-to-implementation review and project tests |
| Archive completed change | No separate skill required unless installed locally | `openspec archive "<change-id>"`; use `--yes` only with explicit user approval |
| Explore next change | `openspec-explore` | Use `openspec list --json`, `openspec show`, repository inspection, and discussion to identify candidates; do not call `openspec explore` unless local help proves it exists |

If the local OpenSpec version differs, inspect `openspec --help` and the relevant subcommand help before acting. If neither the matching skill nor a supported CLI fallback exists, stop and ask the user rather than calling a guessed command.

## Workflow Subagent Timeout Policy

Workflow subagents often perform slow OpenSpec planning, A/B review, TDD, builds, validation, archive, commit handling, and context compression. The Controller must therefore use longer waits than ordinary exploratory subagent calls and must not treat one or more `wait_agent` timeouts as completion failure.

- Default to long waits for workflow subagents: use at least 10 minutes per `wait_agent` call when waiting for planning/review/apply/verify/archive/commit/compression results, unless the user explicitly asks for a shorter check-in cadence.
- For implementation, builds, full test runs, archive, commit handling, or compression, prefer 20-30 minute waits when the tool allows it.
- A `wait_agent` timeout means "no final message returned in that window"; it does not mean the subagent is dead, failed, or safe to replace.
- After a timeout, inspect durable evidence before drawing conclusions: `git status`, the active change directory, bootstrap or change-local `workflow-state.md`, OpenSpec status, and recent planning/review/verification artifacts. Prefer artifacts written after the subagent launch or after the last Controller handoff over stale terminal output.
- Treat current `workflow-state.md` identity as the recovery anchor. If it names the active change and recorded workflow subagent/session, and its current or immediately preceding gate matches durable artifact progress, continue or resume that same recorded subagent instead of launching a replacement.
- Before declaring a workflow subagent stalled, send one concise status request to that same recorded subagent and wait again with the extended timeout. Do not send repeated status pings that could distract from long-running work.
- Enter recovery only when all are true: no final summary has returned, no current workflow-state or artifact progress exists for the active change, the status request to the same recorded subagent has not produced a response after an extended wait, and replacing or taking over would not risk duplicate planning, duplicate implementation, mixed commits, skipped gates, or violation of the single-subagent-per-change boundary.
- Recovery still requires the ordinary workflow's human confirmation gates. If replacement binding, main-session takeover, or manual continuation is considered, stop with the evidence checked and ask for an explicit recovery decision before doing any lifecycle work.
- When reporting a possible stall to the user, explicitly say that tool-level timeout is a soft signal and name the evidence checked. Avoid saying the subagent has not created artifacts unless the filesystem check was run after the latest possible subagent notification.

## Workflow Subagent Context Boundaries

Use a workflow subagent as the executor for one active OpenSpec change, starting before planning generation for new changes. The Controller owns stage classification, exploration, brainstorming, human-confirmation dialogue, workflow-subagent launch/resume decisions, durable handoff preparation, final summary intake, and next-change discovery. The Controller does not execute the change lifecycle while an independent workflow subagent/session runtime is available.

A per-change workflow subagent may benefit from previous changes only through an explicit context handoff prepared by the Controller. That handoff may include:

- Active change id or naming constraints, if known.
- Exploration result, brainstorm decisions, acceptance criteria, and explicit human confirmations.
- Current OpenSpec planning documents when resuming an existing change.
- Relevant archived specs from previous changes.
- The latest post-archive context compression summaries that directly affect this change.
- Commit hashes, changed-file lists, compatibility or migration notes, unresolved assumptions, and follow-up items selected as relevant.
- Current repository state needed to avoid mixing unrelated changes.

The handoff must not include a broad transcript dump, unrelated prior-change discussion, stale terminal output, private A/B coordination artifacts, hidden assumptions from a previous workflow session, or subagent private memory. If the previous context is useful but not captured in an artifact, first write a durable summary or ask the user before using it.

Do not reuse one long-lived workflow subagent across multiple changes as a way to preserve context. End, freeze, or discard the workflow subagent after the change is verified, archived, commit handling is resolved, context is compressed, the workflow is reloaded, and the final summary is returned. For the next change, create a fresh workflow subagent from the newly reloaded workflow skill, current OpenSpec documents, and explicit artifacts only.

If a change predates this boundary and no recorded workflow subagent/session identity exists, do not silently continue the lifecycle in the main session. Ask whether to bind a new recovery workflow subagent using only the current OpenSpec artifacts and durable decisions, or pause for manual handling.

`agent-review-dialogue` remains separate from workflow execution:

- Agent A/B sessions are created per review run and follow `agent-review-dialogue` state, manifest, artifact, and cleanup rules.
- A/B agents may read previous-change artifacts only when the Controller explicitly lists them in `target-manifest.md` or `decisions.md` for that review.
- A/B agents must not read or rely on a workflow subagent's private memory, session transcript, unstated conclusions, or implementation-only scratchpad.
- Workflow subagent output is not review approval. Planning review approval still requires B approval, A double-check, and Controller verification from `agent-review-dialogue`.

## Workflow State Artifact

Record workflow subagent identity in a durable artifact, not only in the chat transcript or terminal output.

Use these locations:

1. Primary location after the OpenSpec change directory exists: `<openspec-change-dir>/workflow-state.md`.
2. Bootstrap location before the change directory exists: `<openspec-root>/workflow-state/pending-<change-id-or-timestamp>.md`.

Create or update the workflow state before any OpenSpec lifecycle command runs in the workflow subagent. If the bootstrap location is used, migrate or copy the state into `<openspec-change-dir>/workflow-state.md` immediately after the change directory exists, then continue from the change-local artifact.

The workflow state must include:

- Change id, capability/spec area, and OpenSpec root/change directory paths.
- Controller session id or identifying note, workflow subagent/session id, runtime, launch timestamp, and last update timestamp.
- Handoff summary, selected durable input artifacts, and explicit exclusions.
- Current gate, completed gates, and next expected gate.
- Human confirmations recorded so far, with date, scope, and source.
- Planning review evidence paths and freshness notes.
- Verification, archive, commit, compression, reload, and summary-return evidence as each gate completes.
- Blockers, recovery decisions, assumptions, and unrelated-change warnings.

Use stable headings or fields so recovery can parse the artifact consistently:

```markdown
# Workflow State

- Change ID:
- Capability / Spec Area:
- OpenSpec Root:
- Change Directory:
- Controller Identity:
- Workflow Subagent Identity:
- Runtime:
- Launched At:
- Last Updated At:
- Current Gate:
- Completed Gates:
- Next Expected Gate:
- Durable Inputs:
- Explicit Exclusions:
- Human Confirmations:
- Planning Review Evidence:
- Verification Evidence:
- Archive Evidence:
- Commit / Push Handling:
- Context Compression:
- Workflow Reload:
- Summary Return:
- Blockers / Recovery Decisions:
- Unrelated-Change Warnings:
```

Update the artifact after launch, after planning generation, after planning review, before and after every human-confirmation packet, after apply/TDD milestones, after verification, after archive, after commit handling, after context compression, after workflow reload, and before final summary return.

Identity evidence is current only when this artifact exists, names the active change, names the workflow subagent/session, and records the current or immediately preceding gate. If the artifact is missing, stale, or points to another change, stop for recovery instead of continuing from private memory.

## Entry Protocol

Before choosing any numbered workflow step, classify the current state:

1. Read the user's latest request for an explicit stage, such as "review the plan", "apply", "verify", "archive", "commit", or "explore next".
2. Inspect repository state with non-destructive commands such as `git status`, OpenSpec directories, existing change files, review artifacts, workflow-state artifacts, and test results when available.
3. Build a current-stage classification that names the active change id when known, the requested entry stage, the recorded workflow subagent/session from `workflow-state.md` when one exists, the latest completed gate with evidence, and the earliest required gate that lacks evidence.
4. Continue from that gate, not from the beginning.
5. If the current state is ambiguous and choosing the wrong stage could overwrite planning documents, skip review, archive failing work, mix commits, or replace a workflow subagent, ask the user before acting.

The ordered gates are: scope confirmation, workflow subagent launch or recorded resume, planning document generation, planning document review, pre-apply self-check and approval, A/B review artifact cleanup, apply/TDD implementation, fresh verification and human review, archive, clean user-confirmed commit, post-archive context compression, workflow skill reload, durable summary return, then next-change exploration. When resuming, never treat a later user request as proof that earlier gates passed; route to the earliest missing gate in that ordered list.

### Stage Router

Use this table to choose the correct resume point:

| Observed state | Resume at | Required check before continuing |
| --- | --- | --- |
| No OpenSpec change exists for the requested work | Brainstorm and confirm scope | Goal, scope, and acceptance criteria are explicit |
| Goal/scope are approved but planning files do not exist | Launch workflow subagent and generate planning documents | Change id, capability, scope choices, handoff, and subagent/session identity recording are ready |
| Planning files exist but no approved `agent-review-dialogue` result exists | Continue recorded workflow subagent for planning review | Manifest scope covers only generated OpenSpec planning files; subagent identity exists or a legacy recovery decision is made |
| Planning review is approved but apply has not run | Continue recorded workflow subagent for pre-apply self-check | Final reviewed plan is still current; user approves apply; final B approval, A double-check, and Controller verification evidence are captured; session-specific A/B review artifacts are cleaned before apply starts |
| Apply has run and implementation is incomplete | Continue recorded workflow subagent for apply/TDD | Approved plan still matches intended implementation scope; subagent identity and handoff are current |
| Implementation appears complete but verification is missing or failed | Continue recorded workflow subagent for verification and human review | Required commands are known; failures return to TDD |
| Verification and human review passed but change is not archived | Continue recorded workflow subagent for archive | Archive command form is known and required checks pass |
| Archive is complete but no clean user-confirmed commit exists | Continue recorded workflow subagent for clean commit handling | Diff contains only this completed change and user confirms commit |
| Clean commit is complete but no post-archive compression exists | Continue recorded workflow subagent for context compression | Archive result, commit state, changed files, verification, and remaining assumptions are captured |
| Post-archive compression is complete but this workflow skill has not been reloaded | Continue recorded workflow subagent for workflow reload | Current `openspec-change-workflow` instructions are loaded after compression |
| Workflow reload is complete but no durable summary has returned | Continue recorded workflow subagent for summary return | Summary includes archive, verification, commit state, constraints, risks, and next action |
| Durable summary is ingested and the user wants the next item | Explore next change | Working tree state is suitable for discovery, compressed context is current, workflow instructions are current, and the previous subagent is not reused |

If multiple changes are present, identify the active change id before continuing. If that cannot be determined safely, ask the user.

Recognize these middle-entry intents explicitly:

- `review`, `planning review`, or `review the plan`: route to planning document review unless an approved, current review already exists.
- `pre-apply`, `ready to apply`, or `apply check`: route to the pre-apply self-check unless planning review is missing or stale.
- `apply`, `implement`, or `TDD`: route to apply/TDD only after user-approved pre-apply, completed A/B review artifact cleanup, and current workflow subagent identity/handoff; otherwise stop at the missing gate.
- `verify`, `test`, or `human review`: route to fresh verification; failed or missing verification returns to TDD.
- `archive`: route to archive only after fresh verification and human review pass.
- `commit`: route to clean commit handling only after archive succeeds and the diff is scoped to the completed change.
- `compress context`, `context compression`, or `summarize before next`: route to post-archive context compression after archive and clean commit handling are complete, then reload this workflow skill before explore; otherwise stop at the missing earlier gate.
- `reload workflow`, `reload skill`, or `refresh workflow`: route to workflow skill reload only after post-archive context compression is current, or stop at the missing earlier gate.
- `explore`, `next change`, or `what next`: route to next-change exploration only after archive, a clean user-confirmed commit, post-archive context compression, workflow skill reload, and durable summary intake are complete.

### Evidence Freshness Rules

- Existing OpenSpec planning documents must be preserved. Do not overwrite or regenerate them merely because the user invoked the skill again; inspect and resume from review or a later missing gate.
- An `agent-review-dialogue` approval is current only when its target scope and timestamps correspond to the current planning files. If any reviewed planning file changed after approval, rerun review.
- Workflow subagent identity is current only when the active workflow-state artifact belongs to the active change, names the recorded subagent/session, and shows it has not completed, been discarded, or been superseded by an explicit recovery decision.
- Workflow subagent handoff is current only when it is assembled after the latest relevant exploration, brainstorm decisions, project constraints, planning review, A/B cleanup, and user approvals for the stage being resumed. If planning documents, decisions, or prior-change artifacts change afterward, refresh the handoff before the workflow subagent continues.
- A/B review evidence is not current if it depends on a workflow subagent's implicit memory. Any cross-change input used by A/B must appear in the review manifest or decisions.
- OpenSpec apply work must not be repeated blindly. If implementation appears to have started, inspect the working tree, OpenSpec status, apply instructions, and task progress; continue implementation from the next missing or failing test.
- Verification evidence is current only when it was produced after the latest relevant implementation, planning, archive, or spec change. Old terminal output without a durable artifact is not sufficient.
- Archive success must be verified from command output and resulting file changes. If archive modifies specs, generated state, or metadata, rerun `openspec validate --all --strict --no-interactive` before commit handling.
- Failed verification cannot be waived into an archive. The only valid paths are fix within scope, rescope with review and approval, or pause.
- Clean commit handling, post-archive context compression, workflow skill reload, and durable summary intake must finish before next-change exploration. If the completed change remains uncommitted, partially staged, mixed with unrelated changes, awaiting user confirmation, not yet summarized after archive, missing a returned subagent summary, or the workflow skill has not been reloaded after compression, do not run `openspec-explore` or the exploration fallback.
- A/B review cleanup is current only after the final `agent-review-dialogue` result has been durably captured outside the coordination directory that will be deleted, including B approval, A double-check no-change, Controller verification passed, review round count, unresolved assumptions, active change id, and user decisions that affect implementation. Clean only the active session-specific review coordination directory; do not delete the parent `.agent-review-dialogue/` tree, sibling review runs, OpenSpec planning documents, decisions that were copied into the plan, implementation files, or unrelated review runs.
- Post-archive context compression is current only when it is produced after the latest archive, validation rerun, and clean commit decision. If files, archive state, or commit state change afterward, refresh the compressed context before workflow skill reload, `openspec-explore`, or the exploration fallback.
- Workflow skill reload is current only when it happens after the latest post-archive context compression. If context is compressed again, or if this skill file changes, reload the workflow skill again before `openspec-explore` or the exploration fallback.
- Durable summary intake is current only when the recorded workflow subagent returns it after archive, clean commit handling, context compression, workflow reload, and workflow-state update. Do not use a prior subagent's summary for a new change.

## Workflow

### 1. Brainstorm And Confirm Scope

Use `superpowers:brainstorming` in the main session to clarify the change before touching OpenSpec documents. Keep the dialogue short when the request is already concrete, but still confirm the decisions that affect correctness. Skip this step only when resuming an existing change whose goal, scope, and acceptance criteria are already captured in validated OpenSpec documents or explicit user decisions.

Do not create or edit OpenSpec planning documents in the main session during this step. After scope is confirmed, the next lifecycle action is launching or resuming the workflow subagent.

Human confirmation required before `openspec-ff-change` or the CLI-supported fast-forward fallback:

- Change goal and user-visible outcome.
- In-scope and out-of-scope boundaries.
- Acceptance criteria and verification expectations.
- Compatibility, migration, rollback, or public API constraints when relevant.

If any of these are missing and would affect the generated plan, ask the user instead of choosing defaults.

### 2. Launch Workflow Subagent And Generate Planning Documents

Use this step only when planning documents are missing or the user explicitly asked to regenerate them. If planning documents already exist, inspect them and route to review or the earliest incomplete later gate instead of overwriting them.

Before `openspec-ff-change`, `openspec new change`, or the CLI-supported planning-generation fallback, launch one dedicated workflow subagent for this change when an independent subagent/session runtime is available. Record identity evidence in the workflow state artifact before the workflow subagent runs any OpenSpec lifecycle command.

The workflow subagent handoff and workflow-state artifact must include only:

- Active change id or naming constraints, if known.
- Confirmed goal, scope, acceptance criteria, and verification expectations.
- Affected capability/spec area and compatibility or migration decisions.
- Relevant archived specs, post-archive summaries, commit/file references, and project constraints selected by the Controller.
- Current repository state needed to avoid mixing unrelated changes.
- Human-confirmation gates and stop conditions that must return to the Controller.

Exclude broad transcript history, raw terminal scrollback, private A/B coordination state, unrelated prior-change context, and hidden assumptions. The workflow subagent must materialize handoff decisions into OpenSpec planning documents or durable decision notes before apply.

If no independent subagent/session runtime is available, stop and ask before running the change lifecycle in the main session. Do not silently collapse the workflow back into the Controller session.

The workflow subagent uses `openspec-ff-change` when available. If it is not available, the workflow subagent uses the CLI-supported fallback:

```bash
openspec new change "<change-id>"
openspec status --change "<change-id>" --json
openspec instructions <artifact-id> --change "<change-id>" --json
```

The fallback is a loop, not a single `--ff` flag: create the change directory, read status JSON, generate each ready artifact from `openspec instructions`, then repeat until every `applyRequires` artifact is complete. Expected outputs commonly include proposal, tasks, design notes, spec deltas, and verification guidance, but trust the local OpenSpec project structure over generic filenames.

Human confirmation required before choosing any missing:

- Change id or naming convention.
- Affected capability or spec area.
- Compatibility policy.
- Migration or rollout strategy.
- Scope split when the generated change would be too large for one cycle.

If the local OpenSpec skill, CLI flags, or project layout differ from expectation, stop and ask or inspect local OpenSpec help before inventing a workflow.

### 3. Review Generated Planning Documents

The recorded workflow subagent uses `agent-review-dialogue` after `openspec-ff-change` or the CLI-supported fallback has produced concrete files. Configure the review scope so Agent A may edit only the generated OpenSpec planning documents and its `change-log.md`; Agent B reviews only and writes `review.md`.

The review must check:

- Ambiguous requirements, implicit assumptions, and missing acceptance criteria.
- Tasks that cannot be tested or verified.
- Missing failure, migration, rollback, compatibility, or security considerations.
- Scope creep beyond the user-approved goal.
- Contradictions between proposal, tasks, design, spec deltas, and verification plan.
- Whether implementation can proceed with TDD.

Do not continue from Agent A's summary alone. Proceed only after the `agent-review-dialogue` loop reaches approval according to its own artifact validation: B approves, A double-checks, and controller verification passes.

Human confirmation required during review when A or B raises any decision that changes product behavior, compatibility, migration, public API, acceptance criteria, verification obligations, dependency policy, or scope. The workflow subagent must return those questions to the Controller with the relevant artifact paths and risk. The Controller asks the user, records the decision durably, then resumes the same workflow subagent.

When resuming from this stage, reuse existing valid `agent-review-dialogue` artifacts only if they correspond to the current planning files. If the planning files changed after approval, rerun review in the recorded workflow subagent.

### 4. Run Pre-Apply Self-Check

Before applying the reviewed plan, the workflow subagent rereads the final OpenSpec documents and checks for:

- Unresolved placeholders or stale assumptions.
- Vague tasks or unverifiable acceptance criteria.
- Missing tests for changed behavior.
- Missing migration, rollback, compatibility, or documentation work.
- Inconsistency between the reviewed plan and intended implementation scope.
- Any implementation work outside the reviewed OpenSpec change.

Human confirmation required before apply work: the workflow subagent prepares a concise confirmation packet that summarizes the final reviewed planning state, calls out residual risks, and names the exact apply command or fallback. The Controller asks the user to approve applying the change, records the answer durably, and resumes the same workflow subagent.

After the user approves apply and before the workflow subagent starts `openspec-apply-change` or the CLI-supported apply fallback, clean the A/B review intermediate artifacts created for the planning-document review. This cleanup must:

- Run only after final B approval, A's double-check reports no further changes, and Controller verification has passed.
- Before deleting the coordination directory, durably capture B approval, A double-check no-change, Controller verification passed, review round count, unresolved assumptions, active change id, and user decisions that affect implementation in a location that will not be deleted by the cleanup.
- Delete only the session-specific coordination directory that is active for the approved planning review, such as `.agent-review-dialogue/<change-id>/<session-key>/`.
- Never delete `.agent-review-dialogue/` itself, sibling session directories, unrelated review runs, reviewed OpenSpec planning documents, implementation files, or user-approved decisions that affect implementation.
- Stop and ask if multiple review runs exist, the active session-specific directory cannot be identified safely, or the available evidence does not prove which review run approved the current planning files.
- Treat cleanup as the final pre-apply gate. After cleanup, the same workflow subagent starts apply work only from the preserved OpenSpec planning documents and captured decisions, not from deleted coordination artifacts.

When resuming here, verify that the approved review still matches the current planning files. If the approval is stale, return to the review stage. If the review coordination directory is already absent because it was cleaned, continue only when durable evidence proves B approval, A double-check no-change, Controller verification passed, review round count, unresolved assumptions, active change id, and user decisions that affect implementation; otherwise stop and ask instead of recreating or guessing the deleted review context.

### 5. Apply And Implement With TDD

The recorded workflow subagent runs `openspec-apply-change` only after user-approved pre-apply self-check, final result durable capture, session-specific A/B coordination cleanup, and current workflow-subagent handoff are all complete.

The workflow subagent owns apply and implementation for this change. Do not hand the change to a different worker or subagent after planning review unless the user explicitly authorizes a recovery path; that would break the single-subagent-per-change boundary. The workflow subagent must not explore another change, expand OpenSpec scope, or treat prior-change context as approval. It must report blockers to the Controller when implementation requires scope expansion, dependency changes, weaker tests, or behavior not captured in the reviewed plan.

If `openspec-apply-change` is not available, run the CLI-supported apply fallback:

```bash
openspec instructions apply --change "<change-id>" --json
```

Read every returned `contextFiles` path, implement pending tasks, and update task checkboxes. Then use `superpowers:test-driven-development` for implementation:

1. Write or expose a failing test for the approved behavior.
2. Implement the smallest correct change.
3. Refactor while keeping tests green.
4. Repeat until every approved task is complete.

For C/C++ changes, also check ownership, RAII, exception/error safety, resource lifetime, undefined behavior, integer and bounds safety, concurrency assumptions, and concrete unit coverage.

Human confirmation required if implementation requires any of the following:

- Expanding beyond the reviewed OpenSpec scope.
- Changing external behavior not captured in the documents.
- Adding or replacing a dependency.
- Changing build, packaging, test infrastructure, CI, or repository policy.
- Accepting a weaker test strategy than the reviewed plan required.

When resuming after apply work starts, do not re-apply blindly. Inspect the working tree, OpenSpec status, apply instructions, and task progress to determine which approved tasks remain incomplete, then continue TDD from the next failing or missing test.

When resuming a workflow subagent, reuse it only for the same active change and only if its identity evidence and context handoff are still current. If the workflow has moved to a new change, start a new workflow subagent from explicit artifacts instead of continuing the old subagent.

### 6. Verify And Human-Review

The workflow subagent uses `openspec-verify-change` when available. If it is unavailable, perform the manual artifact-to-implementation review described in the compatibility fallback. Always run OpenSpec structural validation as well:

```bash
openspec validate "<change-id>" --type change --strict --no-interactive
```

Also run the relevant build, test, lint, or project-specific checks implied by the OpenSpec documents and repository norms. If verification fails, return to the TDD loop, fix the cause, and rerun the checks.

Human confirmation required before archive. The workflow subagent returns a human-review packet to the Controller with:

- `openspec-verify-change` or its manual fallback, `openspec validate`, and all required project checks pass.
- The final behavior and diff that the user needs to review.
- Remaining risks, skipped tests, and follow-up work are visible and do not represent failed required checks.

The Controller asks the user for archive approval, records the answer durably, and resumes the same workflow subagent only after approval. If any required command does not pass, do not archive. Return to implementation, adjust the approved scope, or pause with the risk recorded; a human acceptance of a failing command is not permission to run `openspec archive "<change-id>"`.

When resuming here, do not rely on old terminal output unless it is captured in a durable artifact. Re-run the required verification commands or clearly report that fresh verification is still needed.

### 7. Archive

The workflow subagent runs `openspec archive "<change-id>"` only after automated verification and human review are complete. Before running it, determine the exact local command form. If archive requires a different change id, flag, path, or project-specific option, inspect local help such as `openspec archive --help` or existing OpenSpec project conventions instead of assuming a bare command is correct.

Inspect the archive output and resulting diff.

After archiving:

- Re-run `openspec validate --all --strict --no-interactive` if archive changed specs, generated state, or project metadata.
- Confirm no unrelated files were modified.
- Confirm any generated archive or spec updates match the completed change.

Human confirmation required if archive output changes unexpected files, exposes unresolved follow-up work, or requires a policy decision.

When resuming here, confirm the verification evidence is current for the implementation being archived. If files changed after verification, return to verification.

### 8. Prepare Clean Commit

The workflow subagent reviews `git status`, `git diff`, and `git diff --cached` before staging or committing. Account for untracked files and any pre-existing staged changes. Stage only files related to the completed OpenSpec change and its implementation.

If unrelated files are already staged, unstaged, or untracked, pause and ask the user how to handle them. Do not unstage, revert, delete, or include unrelated changes without explicit user direction.

Do not run `git commit` or `git push` without explicit user confirmation. The workflow subagent prepares the clean-commit packet, the Controller asks the user, records the answer durably, and the same workflow subagent performs only the confirmed git action. A commit confirmation is not a push confirmation; ask again before any push. If the repository defines commit-message rules, follow them. In this repository, commit messages must be Chinese and use `类型: 简短描述`, for example `fix: 修正内存泄漏问题`.

Human confirmation required before every commit and every push, even when all checks pass.

When resuming here, check whether a commit already exists for the archived change. If it does, do not create a duplicate; report the existing commit and continue only if the user asks for more git handling.

### 9. Compress Context And Reload Workflow Before Next Explore

After `openspec archive "<change-id>"` and clean commit handling are complete, the workflow subagent proactively compresses context before the Controller runs `openspec-explore` or the exploration fallback. Use `context-compression` principles: preserve the artifact trail and decisions needed to continue without rereading the whole completed change.

The compressed context must capture:

- Active change id, completed capability/spec area, and archive result.
- Files created, modified, archived, or intentionally left unchanged.
- Verification commands and final results, including any post-archive `openspec validate --all --strict --no-interactive` rerun.
- Final commit state: commit hash if committed, or the explicit user decision if commit/push handling was deferred and the workflow is stopping.
- User-approved decisions, compatibility/migration notes, unresolved assumptions, and follow-up work.
- Workflow subagent state: recorded identity evidence, completed gates, remaining blockers, and confirmation that it must not be reused for the next change.
- Workflow skill reload status: pending before reload, then completed after reload with evidence.
- The safe next action and whether workflow skill reload and `openspec-explore` or the exploration fallback are allowed.

Store the summary in a durable location or message that will survive context compaction. Do not rely on raw terminal scrollback. If the compression cannot preserve the archive result, commit state, workflow-subagent identity/state, and next-action constraints, stop and ask before exploring.

Immediately after context compression, reload `openspec-change-workflow` from disk or the current runtime skill source before any next-change discovery. The first action after compaction must be to re-read this skill so the next loop uses the full workflow instead of a compressed memory of it.

Record that reload in the durable summary or current response with the skill name, source path or runtime source, and timestamp. If the workflow skill cannot be reloaded, stop and ask before exploring.

After reload, the workflow subagent returns a durable completion summary to the Controller. The Controller ingests that summary before any next-change exploration.

When resuming here, check whether context compression was produced after the latest archive and clean commit handling, whether `openspec-change-workflow` was reloaded after that compression, and whether the workflow subagent returned a durable summary after reload. If any item is missing or stale, refresh compression, reload the workflow skill, or resume the recorded workflow subagent before `openspec-explore` or the exploration fallback.

### 10. Explore Next Change

After archive, clean commit handling, post-archive context compression, workflow skill reload, and durable workflow-subagent summary intake are complete, the Controller uses `openspec-explore` when available. If it is unavailable, use the CLI-supported exploration fallback:

```bash
openspec list --json
openspec show "<item-name>" --json
```

Summarize any next actionable change. Ask the user whether to start it before generating new OpenSpec documents. If the user approves, return to workflow-subagent launch and planning document generation with a fresh subagent. Do not reuse the previous change's workflow subagent.

If the user defers or declines the clean commit, do not run `openspec-explore` or the exploration fallback. Pause or stop the current lifecycle with the uncommitted state clearly reported, after compressing the completed archive context if the archive has already finished.

When resuming here, first ensure the working tree is not carrying uncommitted files from the completed change, the compressed context is current, this workflow skill was reloaded after compression, and the previous workflow subagent returned a durable summary. If any condition fails, return to clean commit handling, context compression, workflow skill reload, or summary intake instead of discovering a new change.

## Human Confirmation Gates

Always pause for explicit user approval at these gates:

| Gate | Required before |
| --- | --- |
| Current-stage classification | Any action that could overwrite, apply, archive, commit, or start another change |
| Goal, scope, and acceptance criteria approval | `openspec-ff-change` or CLI-supported fast-forward fallback |
| Workflow subagent launch or recovery binding | `openspec-ff-change`, `openspec new change`, or any lifecycle continuation without recorded subagent identity |
| Missing change id, capability, compatibility, migration, rollout, or scope split decision | Choosing defaults |
| A/B review surfaces product, API, compatibility, migration, dependency, scope, or verification decisions | Continuing the review loop |
| Final reviewed plan approval after pre-apply self-check | `openspec-apply-change` or CLI-supported apply fallback |
| A/B review intermediate cleanup | Starting apply work |
| Scope expansion, dependency changes, infrastructure changes, or weaker tests | Implementation beyond reviewed plan |
| Automated verification and human diff/behavior review | `openspec archive "<change-id>"` |
| Failed required verification | Any archive attempt; fix, rescope, or pause instead |
| Unexpected archive output | Accepting archive result |
| Clean commit | `git commit` |
| Push | `git push` |
| Post-archive context compression, workflow skill reload, and durable summary intake | `openspec-explore` or exploration fallback |
| Next-change approval | Next `openspec-ff-change` or CLI-supported fast-forward fallback |

## Stop Conditions

Stop and ask the user instead of guessing when:

- A requirement affects correctness, scope, compatibility, migration, verification, or repository history and is not explicit.
- The current lifecycle stage cannot be identified safely from user input and repository artifacts.
- A new change would begin without a dedicated workflow subagent even though an independent subagent/session runtime is available.
- An interrupted change lacks recorded workflow subagent/session identity, or the only available identity belongs to another change.
- The OpenSpec skill, CLI fallback, generated file layout, or archive semantics are unclear.
- The generated change is too broad for one implementation cycle.
- `agent-review-dialogue` reports `blocked-on-user` or fails artifact validation.
- The pre-apply self-check finds unresolved planning issues.
- A/B review artifacts must be cleaned before apply but the active session-specific directory cannot be identified safely.
- A workflow subagent would rely on implicit prior-change memory instead of explicit archived specs, compressed summaries, or Controller-selected artifacts.
- A/B review would depend on workflow subagent memory or artifacts not listed in the review manifest or decisions.
- `openspec-verify-change`, `openspec validate`, or required tests fail and cannot be fixed within the approved scope.
- Archive or diff includes unrelated files.
- A clean user-confirmed commit is not complete; do not proceed to next-change discovery.
- Post-archive context compression is missing or stale before `openspec-explore` or the exploration fallback.
- Workflow skill reload is missing or stale after context compression and before `openspec-explore` or the exploration fallback.
- Durable workflow-subagent summary intake is missing or stale before next-change exploration.
- Any step would require committing, pushing, destructive git operations, broad formatting, or reverting user changes.
