---
name: openspec-continuous-change-workflow
description: "Use when the user wants an automated, continuous OpenSpec change lifecycle that mirrors openspec-change-workflow while minimizing repeated human confirmation: explore-time brainstorming decision batching, durable project-level constraints, automatic execution after planning or tasks begin, planning review, apply/TDD, verification, archive, clean commit, context compression, reload, and next-change exploration."
---

# OpenSpec Continuous Change Workflow

## Purpose

Run the same lifecycle as `openspec-change-workflow`, but optimize for continuous execution. Keep human interruptions rare, concentrated in the brainstorming window immediately after exploration and before planning, and justified by clear value. Convert reusable human decisions into durable project-level constraints so later changes do not ask the same question again.

The intended loop is:

```text
classify current stage
-> discover project constraints and reusable decisions
-> explore next change
-> brainstorm and batch all decision-worthy questions
-> create or resume OpenSpec planning
-> review planning documents
-> apply with TDD, preferably via a per-change development subagent
-> verify with OpenSpec validation and project checks
-> archive when all required checks pass
-> automatically create a scoped clean commit
-> compress context and update constraints
-> reload workflow
-> explore next change
-> repeat while the next action is authorized and safe
```

This skill is an automation variant, not a bypass for correctness. Do not skip current-stage classification, stale-evidence checks, verification, archive inspection, unrelated-change protection, or repository history safety checks.

## Automation Contract

- Continue automatically through any gate whose decision is already covered by current OpenSpec documents, repository conventions, or durable project-level constraints.
- Ask the user only during the explore-time brainstorming window, before OpenSpec planning documents or tasks are created, when the answer has high decision value and cannot be safely inferred from explicit artifacts.
- Once proposal, design, spec deltas, or tasks start being written, continue automatically through review, apply, verification, archive, clean commit, compression, reload, and the next exploration unless a stop condition makes automation unsafe.
- Batch all decision-worthy human questions in the brainstorming window after exploration and before planning. Do not introduce a separate pre-apply confirmation gate.
- Record reusable answers in the project-level constraints artifact before continuing, then consult that artifact in later cycles.
- Record one-off answers in the active change's decision notes instead of polluting project-level constraints.
- Prefer development subagents in apply steps when the change is implementation-heavy, but keep the Controller responsible for stage classification, verification, archive, git handling, compression, and next-change discovery.
- Automatically make a scoped clean commit after archive and post-archive validation when the diff contains only the completed change. Never run destructive git operations, delete unrelated files, accept failed required verification, or archive unexpected diffs without explicit user authorization.

## Automation Boundary

The planned human decision window closes as soon as the workflow creates or modifies the first OpenSpec proposal, design, spec delta, or task file for the current change.

After that point, do not ask for routine approval to review, apply, verify, archive, commit, compress, reload, or explore the next candidate. Continue until the current change reaches a clean commit and the workflow reaches the next change's exploration-to-brainstorming window, unless a stop condition applies.

If a high-value question appears after planning starts, first try to resolve it from project constraints, OpenSpec artifacts, tests, or repository conventions. Stop only when the missing answer would make continued automation unsafe or invalid.

## Project-Level Constraints

Before planning or applying a change, discover a durable project-level constraint artifact. Prefer an existing canonical location in this order:

1. Repository instruction files that already govern agents, such as `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, or `.codex/instructions.md`.
2. Existing OpenSpec project files that document conventions, such as `openspec/project.md`, `.openspec/project.md`, `openspec/CONSTRAINTS.md`, or `.openspec/CONSTRAINTS.md`.
3. Existing project decision logs, architecture records, or policy files that the repository clearly uses.

If no project-level constraint artifact exists, ask once in the explore-time brainstorming window where durable workflow constraints should live. After the user chooses, record that location and reuse it for future cycles. Do not invent a hidden constraints file when the repository has no convention.

When writing constraints:

- Use concise, dated entries with the decision, scope, rationale, source, and reuse rule.
- Separate project-wide constraints from change-local decisions.
- Mark constraints as `active`, `superseded`, or `experimental` when the repository already uses such states; otherwise keep plain text and append newer decisions below older ones.
- Do not overwrite user-authored policy. Append or make the smallest targeted edit.
- Do not treat a one-time exception as a project rule.

Recommended entry shape when no local format exists:

```markdown
## OpenSpec workflow constraints

- Date: YYYY-MM-DD
  Decision: ...
  Scope: project-wide | capability | change-local
  Reuse rule: apply automatically when ...
  Source: user confirmation in explore-time brainstorming for <change-id>
```

## Decision Value Assessment

Evaluate every possible question before asking it.

Ask during explore-time brainstorming only when all are true:

- The decision affects product behavior, compatibility, migration, public API, security, dependency policy, verification obligations, repository history, or destructive operations.
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
| `blocker` | Wrong answer may invalidate the change, damage history, or violate policy | Ask in explore-time brainstorming before planning, or stop only if discovered later and automation would be unsafe |
| `reusable-policy` | Answer should guide future changes | Ask once in explore-time brainstorming, then record in project constraints |
| `change-local` | Answer affects only this change | Ask in explore-time brainstorming and record in change decisions |
| `low-value` | Answer is inferable, testable, or immaterial | Do not ask; proceed and document the assumption only if useful |

## Entry Protocol

Start every invocation by classifying the current lifecycle state:

1. Read the user's latest request for a stage intent: explore, brainstorm, create, review, apply, verify, archive, commit, compress, reload, continue, or run continuously.
2. Inspect repository state with non-destructive commands: `git status`, OpenSpec directories, current change files, review artifacts, constraint artifacts, and durable verification results when present.
3. Identify the active change id, requested stage, latest completed gate with evidence, stale or missing evidence, and next safe action.
4. Load project-level constraints before deciding whether to ask anything.
5. Resume from the earliest incomplete required gate. Do not restart planning or overwrite documents when valid later-stage artifacts exist.

If multiple active changes exist and the active change cannot be determined safely, ask once before acting.

## Stage Router

| Observed state | Resume at | Automation behavior |
| --- | --- | --- |
| No change exists for the requested work | Explore then brainstorm | Infer from user request and constraints; batch only blocker/reusable/change-local questions before planning |
| Next candidate is known but planning files are missing | Brainstorm decision window | Resolve reusable and change-local decisions now; after this point do not stop for routine confirmation |
| Scope is sufficient but planning files are missing | Generate planning documents | Use OpenSpec skill or supported CLI fallback |
| Planning files exist without current approval | Planning review | Run review and fix planning issues automatically within brainstorm-approved scope |
| Review is current but apply has not started | Apply/TDD | Start apply automatically, preferably with a per-change development subagent |
| Implementation is incomplete | Continue TDD | Resume from failing or missing tests; avoid re-applying blindly |
| Implementation appears complete | Verify | Run fresh OpenSpec validation and project checks |
| Verification passes and archive is pending | Archive | Archive automatically unless archive semantics or diff are ambiguous |
| Archive completed | Post-archive validation and clean commit | Validate all affected specs; stage only scoped files and commit automatically |
| Clean commit is complete | Compress and reload | Produce durable summary, update constraints, reload this workflow |
| Compression/reload are current and user requested continuity | Explore next | Discover next candidate and continue only if starting it is authorized |

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

### 1. Explore And Brainstorm Before Planning

Use `openspec-explore` or the supported exploration fallback first when the next change is not already known. Then use `superpowers:brainstorming` before creating or changing OpenSpec planning documents unless the scope is already captured in current OpenSpec files or durable user decisions.

Treat the exploration-to-brainstorming window as the only planned human confirmation point for the change. During this window, decide change goal, user-visible outcome, in-scope and out-of-scope boundaries, acceptance criteria, verification expectations, compatibility/migration policy, and any reusable constraints worth recording.

Treat scope as sufficient when these decisions are explicit or inferable from project constraints. Ask only for blocker, reusable-policy, or change-local questions whose answers cannot be inferred. Once planning documents or tasks start being written, do not add a later pre-apply confirmation gate.

### 2. Generate Or Resume Planning Documents

Generate planning documents only when they are missing or the user explicitly requests regeneration. If documents exist, inspect them and route forward.

Use `openspec-ff-change` when available. Otherwise use the supported CLI loop:

```bash
openspec new change "<change-id>"
openspec status --change "<change-id>" --json
openspec instructions <artifact-id> --change "<change-id>" --json
```

After generation, record unresolved assumptions in the change-local decision notes. Do not queue new human questions unless a stop condition appears and automation would be unsafe.

### 3. Review Planning Documents

Use `agent-review-dialogue` after concrete OpenSpec planning files exist. Scope Agent A edits to generated OpenSpec planning documents and its `change-log.md`; Agent B reviews and writes `review.md`.

Review for ambiguous requirements, unverifiable tasks, missing failure/migration/rollback/security considerations, contradictions across artifacts, scope creep, and whether implementation can proceed with TDD.

Unlike the manual workflow, do not stop for every review comment. Route review findings as follows:

- Apply document fixes directly when they do not change behavior or policy.
- Resolve anything covered by the brainstorm decisions or project constraints.
- If review uncovers a blocker that should have been handled during brainstorming, stop only when continuing would be unsafe; otherwise document the assumption and verify it later.
- Drop `low-value` questions after recording the assumption if it helps verification.

Proceed only when the review loop reaches its own approval standard: B approval, A double-check, and Controller verification. If review reports `blocked-on-user`, either resolve it from existing constraints or stop with a concise explanation of why it could not be safely deferred.

### 4. Planning Lock And Constraint Update

Before apply, reread final planning documents, review results, project constraints, and brainstorm decisions.

Do not ask for ceremonial apply approval. Instead, create a planning lock:

- Confirm in the artifact trail that all human-facing decisions for this change were handled during exploration/brainstorming or are covered by durable constraints.
- Write reusable answers to the project-level constraints artifact.
- Write change-local answers to the active OpenSpec change decision notes or planning documents.
- Mark low-value or deferred non-blockers as assumptions with verification expectations.
- Clean only the active session-specific A/B coordination directory after durable review evidence is captured.

After the planning lock exists, continue directly to apply.

### 5. Apply And Implement With TDD

Run apply only after current review evidence, durable decisions, and any required constraint updates exist.

Prefer a per-change development subagent for implementation. The handoff must include:

- Active change id and exact implementation ownership.
- Current planning documents, apply instructions, task list, explicit user decisions, project constraints, relevant archived specs, and selected prior summaries.
- Boundaries: no next-change exploration, no scope expansion, no dependency or infrastructure changes without reporting a blocker. Commit and archive remain Controller responsibilities, not subagent responsibilities.

If no development subagent is used, implement directly with `superpowers:test-driven-development`:

1. Add or expose a failing test for the approved behavior.
2. Implement the smallest correct change.
3. Refactor while tests stay green.
4. Update OpenSpec task checkboxes only when evidence exists.

For C/C++ changes, also check RAII, ownership, exception/error safety, resource lifetime, undefined behavior, integer and bounds safety, concurrency assumptions, and concrete unit coverage.

Stop during apply only for blockers: scope expansion, external behavior not captured in documents, new/replaced dependency, build/test/CI policy change, weaker test strategy, or destructive file/history operations.

### 6. Verify

Run fresh verification after implementation changes:

```bash
openspec validate "<change-id>" --type change --strict --no-interactive
```

Also run the build, test, lint, static analysis, or project checks required by OpenSpec documents and repository norms. Use `openspec-verify-change` when available; otherwise perform manual artifact-to-implementation review.

If verification fails, return to TDD and rerun verification. Do not archive a failed required check, even with user pressure. If the approved scope cannot satisfy verification, pause with a scoped blocker and propose rescope or rollback.

Human diff/behavior review is optional in this automation workflow unless project constraints require it or verification exposes a decision-worthy risk.

### 7. Archive

Archive automatically only when verification is current and passing, the archive command form is known, and the working tree diff is scoped to the active change.

Inspect local help before assuming command details when needed:

```bash
openspec archive --help
```

After archive:

- Inspect command output and resulting diff.
- Run `openspec validate --all --strict --no-interactive` if archive changed specs, generated state, or metadata.
- Stop if archive output changes unexpected files, exposes unresolved follow-up work, or mixes unrelated changes.

### 8. Automatic Clean Commit

Review `git status`, `git diff`, and `git diff --cached`. Stage only files belonging to the completed change.

Run `git commit` automatically after archive and post-archive validation when all are true:

- The archive result is verified.
- Required validation and project checks pass.
- The staged diff contains only the completed OpenSpec change and implementation.
- No unrelated user changes are staged.

Use a Chinese commit message in `类型: 简短描述` format, for example `feat: 完成 <change-id> 变更`. Preserve unrelated user changes.

If unrelated files are staged, unstaged, or untracked, do not include them. If unrelated staged files prevent a clean commit and cannot be separated non-destructively, stop and ask how to proceed.

### 9. Compress Context, Update Constraints, Reload

After archive and automatic clean commit are complete, produce durable context compression before next-change exploration. Capture:

- Change id, capability/spec area, archive result, and files changed.
- Verification commands and final results.
- Commit state: commit hash and commit message.
- Project constraints added or reused.
- Change-local decisions, assumptions, residual risks, and follow-up work.
- Development subagent scope and confirmation that it must not be reused for the next change.
- Next safe action and whether exploration is allowed.

Reload this skill from disk or current runtime source after compression. Record the reload source and timestamp in the durable summary or response. If the skill cannot be reloaded, stop before exploring next.

### 10. Explore Next And Continue

Explore next only when archive, automatic clean commit, compression, and workflow reload are current.

Use `openspec-explore` when available. Otherwise use supported inspection commands and repository review. Summarize the next candidate, then run the next brainstorming decision window before creating any planning documents. Continue automatically only when the user's instruction authorizes continuous execution. If authorization is unclear, stop before creating the next change.

## Stop Conditions

Stop and ask instead of guessing when:

- Current lifecycle stage or active change id cannot be identified safely.
- No project-level constraint location exists and a reusable decision must be recorded outside the brainstorming window.
- A missing decision discovered after planning starts affects correctness, compatibility, migration, verification, repository history, destructive operations, or scope and cannot be safely handled as an assumption.
- OpenSpec skill availability, CLI fallback, generated file layout, or archive semantics are unclear.
- Planning review fails artifact validation or requires a decision outside the current scope.
- A/B cleanup cannot identify the active session-specific coordination directory safely.
- Development would rely on implicit prior memory instead of explicit artifacts.
- Required verification fails and cannot be fixed within approved scope.
- Archive or diff includes unrelated files.
- A clean commit would require including unrelated changes or altering user changes.
- Any step would destructively alter git history, delete unrelated files, broadly reformat, or revert user changes.

## Relationship To Manual Workflow

Use `openspec-change-workflow` when the user wants explicit human approval at each major gate. Use this skill when the user wants the same lifecycle to run continuously after explore-time brainstorming, with reusable answers preserved as project constraints and automatic clean commits before the next change.
