---
name: asdd
description: AI Spec Driven Development workflow for the fengshui-shifu project. Use whenever the user invokes /feature-spec, /feature-plan, /feature-implement, or /commit-message, or asks to build a feature "the spec-driven way." Governs the phase structure (Orient, Feature Spec, Plan, Implement, Self-Verify, Docs & Changelog, Manual Test, Summary & Handoff) and how work is split across the two independent repos, fengshui-shifu-api and fengshui-shifu-ui.
user-invocable: false
---

# ASDD — AI Spec Driven Development (fengshui-shifu)

This repo layout is **not** a single git repo. `fengshui-shifu/` is a plain folder
containing two independent git repos:

- `fengshui-shifu-api/` — Rails 8 API (Ruby, RSpec, Rubocop)
- `fengshui-shifu-ui/` — Expo / React Native Web (TypeScript, **no test runner or
  linter configured**)

Claude Code is launched from the top level (`fengshui-shifu/`), so `.claude/commands`
and `.claude/skills` live there once and are shared. But `ai/feature-specs/` and
`ai/plans/` live **inside each sub-repo**, because specs/plans are scoped per-repo
and each repo has its own git history to commit them into.

## Step 0 — Determine repo scope, every time

Before doing anything else in `/feature-spec` or `/feature-plan`, decide whether the work is:

- **`api`-only** — backend logic, routes, `bazi_calculator_service.rb`, etc.
- **`ui`-only** — screens, components, styling, `App.tsx`, etc.
- **Cross-cutting** — touches the API/UI contract (`api.ts` interfaces mirroring
  `build_calculation_result`), e.g. adding a new field to the calculation response
  that the UI needs to render.

Infer this from the requirement text where obvious (e.g. "rate limit the login
endpoint" → api; "add a settings screen" → ui). If genuinely ambiguous, ask the
user directly rather than guessing — do not silently default to one repo.

**For cross-cutting features**, use the driver-repo pattern:
- Write the primary spec in whichever repo's change is the root cause (usually
  `fengshui-shifu-api`, since the contract shape drives what the UI can do).
- Write a short companion spec in the other repo that references the driver
  spec by name and describes only that repo's half of the work.
- Do not create a third, top-level spec location — always attribute the spec to
  a specific repo's `ai/feature-specs/`.

## Phase structure

0. **Orient** — read the relevant repo's `CLAUDE.md` and existing conventions
   before writing anything.
1. **Feature Spec** → `<repo>/ai/feature-specs/<name>.md`. Stop for approval.
2. **Plan** (via `/feature-plan`) → `<repo>/ai/plans/<name>.md`. Stop for approval.
3. **Implement** (via `/feature-implement`) in checkpoints, **one at a time** —
   verify a checkpoint, report it with its commit message, then stop and wait
   for the user to commit before starting the next. Running checkpoints
   back-to-back leaves both sets of changes in one working tree and destroys the
   split. For cross-cutting features, sequence API checkpoints before UI
   checkpoints that depend on them, and keep each checkpoint's changes within a
   single repo. Do not commit; see Principles.
4. **Self-Verify** — must be green before moving on:
   - `fengshui-shifu-api`: `bundle exec rspec` and `bundle exec rubocop`
   - `fengshui-shifu-ui`: **no test/lint script exists.** Run
     `npm run build:web` as the closest available check, and say so explicitly
     rather than silently skipping verification.
5. **Docs & Changelog** — update README/API docs if user- or API-facing;
   changelog entry always, in whichever repo(s) changed. If the API/UI contract
   changed, flag that `fengshui-shifu-ui/src/services/api.ts` needs a matching
   update (per `CLAUDE.md`, this is not automatic).
6. **Manual Test Instructions** — concrete, copy-pasteable steps, per repo.
7. **Summary & Handoff** — summary, and hand off to `/commit-message` for the
   actual commit message(s).

## Principles

- **Git is hands-off.** Never create or delete branches and never commit — no
  `git checkout -b`, `git branch`, `git branch -d`, `git commit`, or `git push`.
  Work on whatever branch is already checked out. If on branch `main`,
  ask user if this is on purpose or need a new branch. Checkpoints are
  commit-sized units of work, not commits to run: report each as ready and hand
  over its message. The user runs every git write themselves.

## Notes carried over from CLAUDE.md worth remembering mid-workflow

- The API is currently stateless (no DB, no models) — any feature needing
  persistence is a bigger architectural change, not a normal checkpoint.
- Kua number logic is astrological-year-based and gender-branching; treat as
  domain logic requiring correctness review, not routine code review.
- CORS is wide open by design (mobile clients) — don't "fix" this in an
  unrelated feature's checkpoint.
