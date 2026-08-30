---
description: Phase 0 + Phase 1 of ASDD — orient in the relevant repo, then write a feature spec for approval.
argument-hint: Feature description
---

# /feature-spec $ARGUMENTS

Follow the `asdd` skill's phase structure. This command covers **Phase 0 (Orient)**
and **Phase 1 (Feature Spec)** only. Do not proceed to planning or implementation.

## 1. Determine repo scope

Read the requirement: "$ARGUMENTS"

Decide whether this is `api`-only, `ui`-only, or cross-cutting, per the "Step 0 —
Determine repo scope" section of the `asdd` skill. If ambiguous, ask the user
before continuing.

## 2. Orient (Phase 0)

Read `CLAUDE.md` at the top level, plus the target repo's existing structure and
conventions (routes, services, or components/screens relevant to this feature) —
don't assume patterns from memory; confirm them by reading the actual files.

## 3. Write the feature spec (Phase 1)

Write the spec to:
- `fengshui-shifu-api/ai/feature-specs/<name>.md` if api-scoped
- `fengshui-shifu-ui/ai/feature-specs/<name>.md` if ui-scoped
- **Both**, driver-repo pattern, if cross-cutting (see the skill for details) —
  the companion spec should explicitly reference the driver spec's filename.

`<name>` should be a short kebab-case slug derived from the requirement.

Use this template:

```markdown
# Feature Spec: <name>

## Repo(s) Touched
<api | ui | api + ui (cross-cutting)>

## Summary
<1-3 sentences>

## Requirements
- ...

## Non-Goals
- ...

## Edge Cases
- ...

## Acceptance Criteria
- ...

## Open Questions
- <anything you need the user to clarify before this can be planned>
```

If you have genuine ambiguity about requirements, put it in **Open Questions**
rather than guessing — do not invent acceptance criteria to fill the section.

Tell the user they can answer inline in the spec file, directly under each
question, or in chat. The spec file is preferred — it's the durable record and
survives a lost session. `/feature-plan` re-reads it before planning, so answers
left there will be picked up; mentioning it in chat is still helpful if you want
them acted on sooner.

## 4. Stop

After writing the file(s), stop. Tell the user the path(s) and summarize the
Open Questions (if any). Do not run `/feature-plan` automatically — wait for
explicit approval.
