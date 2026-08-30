---
description: Phase 2 of ASDD — research and write an implementation plan for an approved feature spec.
argument-hint: feature-spec md file
---

# /feature-plan $ARGUMENTS

Follow the `asdd` skill's phase structure. This command covers **Phase 2 (Plan)**
only. Do not begin implementation.

## 1. Locate the approved spec

`$ARGUMENTS` is a feature name or note identifying which spec to plan for. Search
`fengshui-shifu-api/ai/feature-specs/` and `fengshui-shifu-ui/ai/feature-specs/`
for a matching file. If it's a cross-cutting feature, read both the driver spec
and its companion.

If no matching approved spec exists, stop and tell the user — do not plan from
an unapproved or invented spec.

Before planning, re-read the spec's **Open Questions** — both specs, for a
cross-cutting feature. If any are unanswered — in the file and in this
conversation — stop and ask. Do not plan on assumed answers.

Once answered, promote the section to **Resolved Decisions**: restate each as
the decision made, keeping the rationale, and confirm none remain open. Editing
the spec is the only write this command makes outside `ai/plans/`. If a spec has
no Open Questions section, or it is already resolved, proceed.

## 2. Research

Explore the target repo's actual code relevant to this feature (don't rely on
memory of the codebase). Confirm assumptions against `CLAUDE.md`, especially:
- the API's stateless architecture (no DB layer exists unless this feature adds one)
- the UI's lack of a test runner/linter — the test plan needs to account for this
- whether this feature changes the API/UI response contract, which requires a
  matching update to `fengshui-shifu-ui/src/services/api.ts`

## 3. Write the plan

Write to the same repo(s) as the spec:
- `fengshui-shifu-api/ai/plans/<name>.md`
- `fengshui-shifu-ui/ai/plans/<name>.md`
- Both, for cross-cutting features — see below for checkpoint sequencing.

Use this template:

```markdown
# Plan: <name>

## Confirmed Decisions
<the spec's resolved questions, restated as decisions — or "none raised">

## Approach
<how this will be implemented, key design decisions>

## Files Touched
- <repo>/<path> — <what changes>

## Checkpoints
1. <checkpoint> — <repo> it lands in; the user commits it
2. ...

## Test Plan
- api: <specific RSpec files/cases to add or run>
- ui: <no test runner exists — describe what `npm run build:web` will and won't
  catch, and what needs manual verification instead>

## Risks / Rollback
- ...
```

**For cross-cutting features:** order checkpoints so API changes (and their
tests) land and pass *before* any UI checkpoint that consumes them. Name each
checkpoint's target repo explicitly — never leave a checkpoint that spans
pending changes in both repos simultaneously, since each repo is committed
separately. The plan describes commits the **user** will make; never run
`git commit` and never create a branch.

## 4. Stop

After writing the file(s), stop and wait for approval. Do not begin
implementation until the user explicitly approves the plan.
