---
name: openspec-continuous-change-workflow
description: "Use when the user wants an automated, continuous OpenSpec change lifecycle where the main session is only explore/brainstorm/controller, and each OpenSpec change is delegated before new change creation to a dedicated workflow subagent that owns planning review, apply/TDD, verification, archive, clean commit, context compression, reload, and summary return."
---

# OpenSpec Continuous Change Workflow

## Purpose

Run OpenSpec changes continuously while keeping the main conversation clean. The main session owns only exploration, brainstorming, reusable decision capture, workflow-subagent launch/control, final summary intake, and the next exploration window. Each individual change runs in its own dedicated workflow subagent, launched before `new change`, and the main session does not execute the change lifecycle when an independent subagent/session runtime is available.

The intended loop is:

```text
main: classify state and load project constraints
-> main: explore next candidate
-> main: brainstorm and batch decision-worthy questions
-> main: record reusable answers as project constraints
-> main: start one workflow subagent before new change
-> subagent: create new change or resume the assigned OpenSpec change
-> subagent: generate/review planning documents
-> subagent: apply with TDD
-> subagent: verify, archive, and clean commit
-> subagent: compress context and reload workflow instructions
-> subagent: return durable completion summary
-> main: ingest summary, then explore the next candidate
```

This skill is an automation variant, not a shortcut around correctness. Do not skip stage classification, stale-evidence checks, planning review, verification, archive inspection, unrelated-change protection, or repository history safety checks.

## Non-Negotiable Model

- Use exactly one dedicated workflow subagent per active OpenSpec change.
- Start that workflow subagent before `openspec new change`, `openspec-ff-change`, or the CLI planning-generation fallback.
- Do not run planning generation, planning review, apply, verification, archive, or clean commit in the main session when an independent subagent/session runtime is available.
- Do not reuse one workflow subagent across multiple changes. End, freeze, or discard it after its change is archived, clean committed, compressed, and summarized.
- If a run is interrupted after launch, continue the recorded workflow subagent/session for that change only when identity evidence exists; otherwise return a recovery blocker instead of taking over the lifecycle in the main session or launching a replacement subagent.
- Keep main-session context out of implementation. Pass only explicit artifacts selected by the Controller: exploration result, brainstorm decisions, project constraints, relevant archived specs, relevant post-archive summaries, and current repository state needed to start safely.
- After OpenSpec planning documents exist, treat those documents and explicit decision notes as the implementation contract. Do not rely on broad conversation history, hidden assumptions, terminal scrollback, or subagent private memory.
- Nested `agent-review-dialogue` planning review may run inside the workflow subagent. Its A/B agents remain isolated review roles; their approval does not replace workflow-subagent verification or Controller summary intake.
- Automatically create a scoped clean commit after archive and post-archive validation when the diff contains only the completed change.
- Never run destructive git operations, delete unrelated files, accept failed required verification, or archive unexpected diffs without explicit user authorization.

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
- Ingest the workflow subagent's final durable summary after completion.
- Resume exploration only after the subagent reports archive, clean commit, context compression, and workflow reload evidence.

The main session must not carry detailed implementation context forward. Its durable memory between changes should be project constraints plus compressed summaries.

## Workflow Subagent Responsibilities

The workflow subagent owns the full lifecycle for one change:

- Reclassify current state for the assigned change from repository and OpenSpec artifacts.
- Create or resume the assigned OpenSpec change, starting with `new change` when the change does not yet exist.
- Generate proposal, design, spec deltas, tasks, and decision notes using the available OpenSpec skill or supported CLI fallback.
- Run `agent-review-dialogue` on generated planning documents when planning review evidence is missing or stale.
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
5. If the next safe action belongs to a change lifecycle, launch the dedicated workflow subagent before `new change`. When resuming an interrupted change, continue the same recorded workflow subagent/session only when identity evidence exists; if identity evidence is missing, return a recovery blocker to the main session instead of executing the lifecycle in the main session or launching a replacement subagent.

If multiple active changes exist and the active change cannot be determined safely, ask once before acting.

## Stage Router

| Observed state | Main-session action | Workflow-subagent action |
| --- | --- | --- |
| No next candidate is known | Explore | None yet |
| Candidate is known but scope decisions are missing | Brainstorm decision window | None yet |
| Brainstorm decisions are complete and no OpenSpec change exists | Launch per-change workflow subagent before `new change` | Create change and planning documents |
| Planning files exist but no approved review exists | Continue the recorded workflow subagent/session only with identity evidence; otherwise return a recovery blocker | Run planning review |
| Planning review is current but implementation has not started | Continue the recorded workflow subagent/session only with identity evidence; otherwise return a recovery blocker | Create planning lock, apply with TDD |
| Implementation is incomplete | Continue the recorded workflow subagent/session only with identity evidence; otherwise return a recovery blocker | Continue from failing or missing tests |
| Implementation appears complete | Continue the recorded workflow subagent/session only with identity evidence; otherwise return a recovery blocker | Run fresh verification |
| Verification passes and archive is pending | Continue the recorded workflow subagent/session only with identity evidence; otherwise return a recovery blocker | Archive and post-archive validation |
| Archive completed but no clean commit exists | Continue the recorded workflow subagent/session only with identity evidence; otherwise return a recovery blocker | Stage scoped files and commit |
| Clean commit exists but no returned summary exists | Continue the recorded workflow subagent/session only with identity evidence; otherwise return a recovery blocker | Compress context, reload workflow, and return summary |
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
- Current repository state needed to avoid mixing unrelated changes.
- Stop conditions and requirement to return blockers to the main session.

Exclude broad transcript history, raw terminal scrollback, private reasoning, unrelated prior-change context, and stale assumptions. The subagent must turn this handoff into OpenSpec planning documents or decision notes before apply.

### 3. Launch One Workflow Subagent

Launch one dedicated workflow subagent for the change. Tell it:

- It owns this single change from before `new change` through clean commit, context compression, workflow reload, and summary return.
- It must not start or explore another change.
- It must use OpenSpec planning documents and explicit decision notes as the implementation contract.
- It must run planning review before apply when current review evidence is missing or stale.
- It must report blockers to the main session instead of expanding scope or weakening verification.
- It must return a durable final summary with evidence.

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

Use `agent-review-dialogue` after concrete OpenSpec planning files exist. Scope Agent A edits to generated OpenSpec planning documents and its `change-log.md`; Agent B reviews and writes `review.md`.

Review for ambiguous requirements, unverifiable tasks, missing failure/migration/rollback/security considerations, contradictions across artifacts, scope creep, and whether implementation can proceed with TDD.

Proceed only when the review loop reaches its own approval standard: B `approved`, A double-check, and Controller verification. Agent B's `review.md` may report only `needs-revision` or `approved`. If the nested review loop exposes a blocker through Agent A's `change-log.md` `Status: blocked-on-user` or the Controller's user-input phase, resolve it from project constraints or return a blocker to the main session.

### 6. Subagent Creates Planning Lock

Before apply, reread final planning documents, review results, project constraints, and brainstorm decisions.

Do not ask for ceremonial apply approval. Instead, create a planning lock:

- Confirm that human-facing decisions for this change were handled during exploration/brainstorming or are covered by durable constraints.
- Write reusable answers to the project-level constraints artifact.
- Write change-local answers to OpenSpec decision notes or planning documents.
- Mark low-value or deferred non-blockers as assumptions with verification expectations.
- Clean only the active session-specific A/B coordination directory after durable review evidence is captured.

After the planning lock exists, continue directly to apply.

### 7. Subagent Applies And Implements With TDD

Run apply only after current review evidence, durable decisions, and required constraint updates exist.

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
- Verification commands and final results.
- Files changed.
- Commit hash and commit message.
- Project constraints added or reused.
- Change-local decisions, assumptions, residual risks, and follow-up work.
- Confirmation that the subagent must not be reused for the next change.
- Safe next action for the main session.

The main session should ingest this summary, update its durable context if needed, then continue with `openspec-explore` for the next candidate.

## Stop Conditions

Stop and ask instead of guessing when:

- Current lifecycle stage or active change id cannot be identified safely.
- No independent subagent/session runtime is available for a new change lifecycle.
- An interrupted lifecycle lacks identity evidence for the original workflow subagent/session.
- No project-level constraint location exists and a reusable decision must be recorded outside the brainstorming window.
- A missing decision discovered after planning starts affects correctness, compatibility, migration, verification, repository history, destructive operations, or scope and cannot be safely handled as an assumption.
- OpenSpec skill availability, CLI fallback, generated file layout, or archive semantics are unclear.
- Planning review fails artifact validation or requires a decision outside the current scope.
- A/B cleanup cannot identify the active session-specific coordination directory safely.
- The workflow subagent would rely on implicit prior memory instead of explicit artifacts.
- Required verification fails and cannot be fixed within approved scope.
- Archive or diff includes unrelated files.
- A clean commit would require including unrelated changes or altering user changes.
- Any step would destructively alter git history, delete unrelated files, broadly reformat, or revert user changes.

## Relationship To Manual Workflow

Use `openspec-change-workflow` when the user wants explicit human approval at each major gate. Use this skill when the user wants the main session to handle only explore/brainstorm/controller duties while a dedicated per-change workflow subagent executes the full OpenSpec lifecycle from before `new change` through clean commit, context compression, reload, and summary return.
