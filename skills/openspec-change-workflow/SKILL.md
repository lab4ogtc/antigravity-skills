---
name: openspec-change-workflow
description: Use when the user wants to run, resume, audit, or continue an OpenSpec change lifecycle at any stage, including planning document review, apply/TDD, verification, archive, commit handoff, or next-change discovery.
---

# OpenSpec Change Workflow

## Purpose

Use this workflow to take one OpenSpec change from an idea to archived completion, or to resume an already-started change from its current stage. The full intended path is:

```text
superpowers:brainstorming
-> openspec-ff-change, or CLI-supported fast-forward fallback
-> agent-review-dialogue on generated planning documents
-> pre-apply self-check
-> A/B review artifact cleanup
-> openspec-apply-change + superpowers:test-driven-development
-> openspec-verify-change + OpenSpec validation + human review loop
-> openspec archive "<change-id>"
-> user-confirmed clean commit
-> context compression
-> reload openspec-change-workflow skill
-> openspec-explore
-> repeat or stop
```

This skill is not a shortcut around review. Fast generation creates planning documents quickly; it does not approve their content.

The frontmatter `description` is only an activation trigger. It is not permission to skip current-stage classification, human confirmation gates, review, verification, archive checks, or clean commit handling.

Do not assume the workflow always starts at step 1. When invoked midstream, first identify the current stage, verify that prior gates have evidence, then continue from the earliest incomplete required gate.

## Non-Negotiable Rules

- Start by detecting the current lifecycle stage from repository state, OpenSpec files, review artifacts, git status, and the user's latest instruction.
- Do not restart from brainstorming or regenerate OpenSpec documents when valid later-stage artifacts already exist, unless the user explicitly asks to restart.
- When resuming midstream, run only the missing prerequisites for the current or next stage; preserve completed, validated stages.
- Use `superpowers:brainstorming` before creating or changing OpenSpec planning documents.
- Prefer the corresponding OpenSpec workflow skill for lifecycle actions when available: `openspec-ff-change`, `openspec-apply-change`, `openspec-verify-change`, and `openspec-explore`.
- Do not call unsupported top-level OpenSpec CLI commands. In OpenSpec CLI 1.3.1, `openspec apply`, `openspec verify`, `openspec explore`, and `openspec new change --ff` are not valid top-level invocations.
- Use `openspec-ff-change` to generate the initial planning documents once the user has confirmed the goal, scope, and acceptance criteria. If that skill is not available, use the CLI-supported fast-forward fallback described below.
- Use `agent-review-dialogue` only after the planning documents exist, with those generated OpenSpec files as the reviewed targets.
- Do not use `agent-plan-dialogue` for this workflow. OpenSpec owns planning generation; `agent-review-dialogue` owns adversarial review and refinement of the generated documents.
- Do not start OpenSpec apply work until the reviewed planning documents pass a final self-check and the user explicitly approves applying them.
- Before starting apply work, clean the session-specific A/B review coordination artifacts for the planning review only after final B approval, A double-check, Controller verification, and mandatory durable result capture are complete.
- Implement after apply with `superpowers:test-driven-development`.
- Do not archive until `openspec-verify-change` or its manual fallback has no blocking findings, `openspec validate`, relevant tests/builds, and human review all pass. Failed verification cannot be waived into an archive.
- Do not commit or push unless the user explicitly confirms. Preserve unrelated user changes and stage only files that belong to the completed change.
- After `openspec archive "<change-id>"` completes and before the next OpenSpec exploration, proactively perform context compression with a durable structured summary of the completed change, then reload this workflow skill from disk or the current runtime skill source.
- After archive and the user-confirmed clean commit are complete, use `openspec-explore` to discover the next candidate change, then ask before starting the next loop. If the commit is deferred, pause or stop instead of exploring.

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

## Entry Protocol

Before choosing any numbered workflow step, classify the current state:

1. Read the user's latest request for an explicit stage, such as "review the plan", "apply", "verify", "archive", "commit", or "explore next".
2. Inspect repository state with non-destructive commands such as `git status`, OpenSpec directories, existing change files, review artifacts, and test results when available.
3. Build a current-stage classification that names the active change id when known, the requested entry stage, the latest completed gate with evidence, and the earliest required gate that lacks evidence.
4. Continue from that gate, not from the beginning.
5. If the current state is ambiguous and choosing the wrong stage could overwrite planning documents, skip review, archive failing work, or mix commits, ask the user before acting.

The ordered gates are: scope confirmation, planning document generation, planning document review, pre-apply self-check and approval, A/B review artifact cleanup, apply/TDD implementation, fresh verification and human review, archive, clean user-confirmed commit, post-archive context compression, workflow skill reload, then next-change exploration. When resuming, never treat a later user request as proof that earlier gates passed; route to the earliest missing gate in that ordered list.

### Stage Router

Use this table to choose the correct resume point:

| Observed state | Resume at | Required check before continuing |
| --- | --- | --- |
| No OpenSpec change exists for the requested work | Brainstorm and confirm scope | Goal, scope, and acceptance criteria are explicit |
| Goal/scope are approved but planning files do not exist | Generate planning documents | Change id, capability, and scope choices are known |
| Planning files exist but no approved `agent-review-dialogue` result exists | Review generated planning documents | Manifest scope covers only generated OpenSpec planning files |
| Planning review is approved but apply has not run | Pre-apply self-check | Final reviewed plan is still current; user approves apply; final B approval, A double-check, and Controller verification evidence are captured; session-specific A/B review artifacts are cleaned before apply starts |
| Apply has run and implementation is incomplete | Apply and implement with TDD | Approved plan still matches intended implementation scope |
| Implementation appears complete but verification is missing or failed | Verify and human-review | Required commands are known; failures return to TDD |
| Verification and human review passed but change is not archived | Archive | Archive command form is known and required checks pass |
| Archive is complete but no clean user-confirmed commit exists | Prepare clean commit | Diff contains only this completed change and user confirms commit |
| Clean commit is complete but no post-archive compression exists | Context compression | Archive result, commit state, changed files, verification, and remaining assumptions are captured |
| Post-archive compression is complete but this workflow skill has not been reloaded | Reload workflow skill | Current `openspec-change-workflow` instructions are loaded after compression |
| Clean commit, post-archive compression, and workflow skill reload are complete, and user wants the next item | Explore next change | Working tree state is suitable for discovery, compressed context is current, and workflow instructions are current |

If multiple changes are present, identify the active change id before continuing. If that cannot be determined safely, ask the user.

Recognize these middle-entry intents explicitly:

- `review`, `planning review`, or `review the plan`: route to planning document review unless an approved, current review already exists.
- `pre-apply`, `ready to apply`, or `apply check`: route to the pre-apply self-check unless planning review is missing or stale.
- `apply`, `implement`, or `TDD`: route to apply/TDD only after user-approved pre-apply and completed A/B review artifact cleanup; otherwise stop at the missing gate.
- `verify`, `test`, or `human review`: route to fresh verification; failed or missing verification returns to TDD.
- `archive`: route to archive only after fresh verification and human review pass.
- `commit`: route to clean commit handling only after archive succeeds and the diff is scoped to the completed change.
- `compress context`, `context compression`, or `summarize before next`: route to post-archive context compression after archive and clean commit handling are complete, then reload this workflow skill before explore; otherwise stop at the missing earlier gate.
- `reload workflow`, `reload skill`, or `refresh workflow`: route to workflow skill reload only after post-archive context compression is current, or stop at the missing earlier gate.
- `explore`, `next change`, or `what next`: route to next-change exploration only after archive, a clean user-confirmed commit, post-archive context compression, and workflow skill reload are complete.

### Evidence Freshness Rules

- Existing OpenSpec planning documents must be preserved. Do not overwrite or regenerate them merely because the user invoked the skill again; inspect and resume from review or a later missing gate.
- An `agent-review-dialogue` approval is current only when its target scope and timestamps correspond to the current planning files. If any reviewed planning file changed after approval, rerun review.
- OpenSpec apply work must not be repeated blindly. If implementation appears to have started, inspect the working tree, OpenSpec status, apply instructions, and task progress; continue implementation from the next missing or failing test.
- Verification evidence is current only when it was produced after the latest relevant implementation, planning, archive, or spec change. Old terminal output without a durable artifact is not sufficient.
- Archive success must be verified from command output and resulting file changes. If archive modifies specs, generated state, or metadata, rerun `openspec validate --all --strict --no-interactive` before commit handling.
- Failed verification cannot be waived into an archive. The only valid paths are fix within scope, rescope with review and approval, or pause.
- Clean commit handling, post-archive context compression, and workflow skill reload must finish before next-change exploration. If the completed change remains uncommitted, partially staged, mixed with unrelated changes, awaiting user confirmation, not yet summarized after archive, or the workflow skill has not been reloaded after compression, do not run `openspec-explore` or the exploration fallback.
- A/B review cleanup is current only after the final `agent-review-dialogue` result has been durably captured outside the coordination directory that will be deleted, including B approval, A double-check no-change, Controller verification passed, review round count, unresolved assumptions, active change id, and user decisions that affect implementation. Clean only the active session-specific review coordination directory; do not delete the parent `.agent-review-dialogue/` tree, sibling review runs, OpenSpec planning documents, decisions that were copied into the plan, implementation files, or unrelated review runs.
- Post-archive context compression is current only when it is produced after the latest archive, validation rerun, and clean commit decision. If files, archive state, or commit state change afterward, refresh the compressed context before workflow skill reload, `openspec-explore`, or the exploration fallback.
- Workflow skill reload is current only when it happens after the latest post-archive context compression. If context is compressed again, or if this skill file changes, reload the workflow skill again before `openspec-explore` or the exploration fallback.

## Workflow

### 1. Brainstorm And Confirm Scope

Use `superpowers:brainstorming` to clarify the change before touching OpenSpec documents. Keep the dialogue short when the request is already concrete, but still confirm the decisions that affect correctness. Skip this step only when resuming an existing change whose goal, scope, and acceptance criteria are already captured in validated OpenSpec documents or explicit user decisions.

Human confirmation required before `openspec-ff-change` or the CLI-supported fast-forward fallback:

- Change goal and user-visible outcome.
- In-scope and out-of-scope boundaries.
- Acceptance criteria and verification expectations.
- Compatibility, migration, rollback, or public API constraints when relevant.

If any of these are missing and would affect the generated plan, ask the user instead of choosing defaults.

### 2. Generate Planning Documents

Use this step only when planning documents are missing or the user explicitly asked to regenerate them. If planning documents already exist, inspect them and route to review or the earliest incomplete later gate instead of overwriting them.

Use `openspec-ff-change` when available. If it is not available, use the CLI-supported fallback:

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

Use `agent-review-dialogue` after `openspec-ff-change` or the CLI-supported fallback has produced concrete files. Configure the review scope so Agent A may edit only the generated OpenSpec planning documents and its `change-log.md`; Agent B reviews only and writes `review.md`.

The review must check:

- Ambiguous requirements, implicit assumptions, and missing acceptance criteria.
- Tasks that cannot be tested or verified.
- Missing failure, migration, rollback, compatibility, or security considerations.
- Scope creep beyond the user-approved goal.
- Contradictions between proposal, tasks, design, spec deltas, and verification plan.
- Whether implementation can proceed with TDD.

Do not continue from Agent A's summary alone. Proceed only after the `agent-review-dialogue` loop reaches approval according to its own artifact validation: B approves, A double-checks, and controller verification passes.

Human confirmation required during review when A or B raises any decision that changes product behavior, compatibility, migration, public API, acceptance criteria, verification obligations, dependency policy, or scope.

When resuming from this stage, reuse existing valid `agent-review-dialogue` artifacts only if they correspond to the current planning files. If the planning files changed after approval, rerun review.

### 4. Run Pre-Apply Self-Check

Before applying the reviewed plan, reread the final OpenSpec documents and check for:

- Unresolved placeholders or stale assumptions.
- Vague tasks or unverifiable acceptance criteria.
- Missing tests for changed behavior.
- Missing migration, rollback, compatibility, or documentation work.
- Inconsistency between the reviewed plan and intended implementation scope.
- Any implementation work outside the reviewed OpenSpec change.

Human confirmation required before apply work: summarize the final reviewed planning state, call out residual risks, and ask the user to approve applying the change.

After the user approves apply and before starting `openspec-apply-change` or the CLI-supported apply fallback, clean the A/B review intermediate artifacts created for the planning-document review. This cleanup must:

- Run only after final B approval, A's double-check reports no further changes, and Controller verification has passed.
- Before deleting the coordination directory, durably capture B approval, A double-check no-change, Controller verification passed, review round count, unresolved assumptions, active change id, and user decisions that affect implementation in a location that will not be deleted by the cleanup.
- Delete only the session-specific coordination directory that is active for the approved planning review, such as `.agent-review-dialogue/<change-id>/<session-key>/`.
- Never delete `.agent-review-dialogue/` itself, sibling session directories, unrelated review runs, reviewed OpenSpec planning documents, implementation files, or user-approved decisions that affect implementation.
- Stop and ask if multiple review runs exist, the active session-specific directory cannot be identified safely, or the available evidence does not prove which review run approved the current planning files.
- Treat cleanup as the final pre-apply gate. After cleanup, start apply work only from the preserved OpenSpec planning documents and captured decisions, not from deleted coordination artifacts.

When resuming here, verify that the approved review still matches the current planning files. If the approval is stale, return to the review stage. If the review coordination directory is already absent because it was cleaned, continue only when durable evidence proves B approval, A double-check no-change, Controller verification passed, review round count, unresolved assumptions, active change id, and user decisions that affect implementation; otherwise stop and ask instead of recreating or guessing the deleted review context.

### 5. Apply And Implement With TDD

Run `openspec-apply-change` only after user-approved pre-apply self-check, final result durable capture, and session-specific A/B coordination cleanup are all complete. If that skill is not available, run the CLI-supported apply fallback:

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

### 6. Verify And Human-Review

Use `openspec-verify-change` when available. If it is unavailable, perform the manual artifact-to-implementation review described in the compatibility fallback. Always run OpenSpec structural validation as well:

```bash
openspec validate "<change-id>" --type change --strict --no-interactive
```

Also run the relevant build, test, lint, or project-specific checks implied by the OpenSpec documents and repository norms. If verification fails, return to the TDD loop, fix the cause, and rerun the checks.

Human confirmation required before archive:

- `openspec-verify-change` or its manual fallback, `openspec validate`, and all required project checks pass.
- The user has reviewed the final behavior and diff.
- Remaining risks, skipped tests, and follow-up work are visible and do not represent failed required checks.

If any required command does not pass, do not archive. Return to implementation, adjust the approved scope, or pause with the risk recorded; a human acceptance of a failing command is not permission to run `openspec archive "<change-id>"`.

When resuming here, do not rely on old terminal output unless it is captured in a durable artifact. Re-run the required verification commands or clearly report that fresh verification is still needed.

### 7. Archive

Run `openspec archive "<change-id>"` only after automated verification and human review are complete. Before running it, determine the exact local command form. If archive requires a different change id, flag, path, or project-specific option, inspect local help such as `openspec archive --help` or existing OpenSpec project conventions instead of assuming a bare command is correct.

Inspect the archive output and resulting diff.

After archiving:

- Re-run `openspec validate --all --strict --no-interactive` if archive changed specs, generated state, or project metadata.
- Confirm no unrelated files were modified.
- Confirm any generated archive or spec updates match the completed change.

Human confirmation required if archive output changes unexpected files, exposes unresolved follow-up work, or requires a policy decision.

When resuming here, confirm the verification evidence is current for the implementation being archived. If files changed after verification, return to verification.

### 8. Prepare Clean Commit

Review `git status`, `git diff`, and `git diff --cached` before staging or committing. Account for untracked files and any pre-existing staged changes. Stage only files related to the completed OpenSpec change and its implementation.

If unrelated files are already staged, unstaged, or untracked, pause and ask the user how to handle them. Do not unstage, revert, delete, or include unrelated changes without explicit user direction.

Do not run `git commit` or `git push` without explicit user confirmation. If the repository defines commit-message rules, follow them. In this repository, commit messages must be Chinese and use `类型: 简短描述`, for example `fix: 修正内存泄漏问题`.

Human confirmation required before every commit and every push, even when all checks pass.

When resuming here, check whether a commit already exists for the archived change. If it does, do not create a duplicate; report the existing commit and continue only if the user asks for more git handling.

### 9. Compress Context And Reload Workflow Before Next Explore

After `openspec archive "<change-id>"` and clean commit handling are complete, proactively compress context before running `openspec-explore` or the exploration fallback. Use `context-compression` principles: preserve the artifact trail and decisions needed to continue without rereading the whole completed change.

The compressed context must capture:

- Active change id, completed capability/spec area, and archive result.
- Files created, modified, archived, or intentionally left unchanged.
- Verification commands and final results, including any post-archive `openspec validate --all --strict --no-interactive` rerun.
- Final commit state: commit hash if committed, or the explicit user decision if commit/push handling was deferred and the workflow is stopping.
- User-approved decisions, compatibility/migration notes, unresolved assumptions, and follow-up work.
- Workflow skill reload status: pending before reload, then completed after reload with evidence.
- The safe next action and whether workflow skill reload and `openspec-explore` or the exploration fallback are allowed.

Store the summary in a durable location or message that will survive context compaction. Do not rely on raw terminal scrollback. If the compression cannot preserve the archive result, commit state, and next-action constraints, stop and ask before exploring.

Immediately after context compression, reload `openspec-change-workflow` from disk or the current runtime skill source before any next-change discovery. The first action after compaction must be to re-read this skill so the next loop uses the full workflow instead of a compressed memory of it.

Record that reload in the durable summary or current response with the skill name, source path or runtime source, and timestamp. If the workflow skill cannot be reloaded, stop and ask before exploring.

When resuming here, check whether context compression was produced after the latest archive and clean commit handling, and whether `openspec-change-workflow` was reloaded after that compression. If either is missing or stale, refresh compression or reload the workflow skill before `openspec-explore` or the exploration fallback.

### 10. Explore Next Change

After archive, clean commit handling, post-archive context compression, and workflow skill reload are complete, use `openspec-explore` when available. If it is unavailable, use the CLI-supported exploration fallback:

```bash
openspec list --json
openspec show "<item-name>" --json
```

Summarize any next actionable change. Ask the user whether to start it before generating new OpenSpec documents. If the user approves, return to the planning document generation step and run the loop again.

If the user defers or declines the clean commit, do not run `openspec-explore` or the exploration fallback. Pause or stop the current lifecycle with the uncommitted state clearly reported, after compressing the completed archive context if the archive has already finished.

When resuming here, first ensure the working tree is not carrying uncommitted files from the completed change, the compressed context is current, and this workflow skill was reloaded after compression. If any condition fails, return to clean commit handling, context compression, or workflow skill reload instead of discovering a new change.

## Human Confirmation Gates

Always pause for explicit user approval at these gates:

| Gate | Required before |
| --- | --- |
| Current-stage classification | Any action that could overwrite, apply, archive, commit, or start another change |
| Goal, scope, and acceptance criteria approval | `openspec-ff-change` or CLI-supported fast-forward fallback |
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
| Post-archive context compression and workflow skill reload | `openspec-explore` or exploration fallback |
| Next-change approval | Next `openspec-ff-change` or CLI-supported fast-forward fallback |

## Stop Conditions

Stop and ask the user instead of guessing when:

- A requirement affects correctness, scope, compatibility, migration, verification, or repository history and is not explicit.
- The current lifecycle stage cannot be identified safely from user input and repository artifacts.
- The OpenSpec skill, CLI fallback, generated file layout, or archive semantics are unclear.
- The generated change is too broad for one implementation cycle.
- `agent-review-dialogue` reports `blocked-on-user` or fails artifact validation.
- The pre-apply self-check finds unresolved planning issues.
- A/B review artifacts must be cleaned before apply but the active session-specific directory cannot be identified safely.
- `openspec-verify-change`, `openspec validate`, or required tests fail and cannot be fixed within the approved scope.
- Archive or diff includes unrelated files.
- A clean user-confirmed commit is not complete; do not proceed to next-change discovery.
- Post-archive context compression is missing or stale before `openspec-explore` or the exploration fallback.
- Workflow skill reload is missing or stale after context compression and before `openspec-explore` or the exploration fallback.
- Any step would require committing, pushing, destructive git operations, broad formatting, or reverting user changes.
