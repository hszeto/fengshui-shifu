# ☯️ FengShui-Shifu (风水师傅)

AI-assisted feng shui and BaZi analysis. This directory is the **workspace root**
— it is not the application. It holds shared Claude Code configuration plus two
independently deployed repositories.

## What's here

```
fengshui-shifu/                 ← this repo: shared config only
├── CLAUDE.md                   # project instructions loaded by Claude Code
├── .claude/
│   ├── commands/               # /feature-spec, /feature-plan,
│   │                           #   /feature-implement, /commit-message
│   └── skills/asdd/SKILL.md    # the ASDD workflow definition
│
├── fengshui-shifu-api/         ← separate repo, gitignored here
└── fengshui-shifu-ui/          ← separate repo, gitignored here
```

**Three repositories, not one.** This one tracks `CLAUDE.md` and `.claude/`,
which are shared by both sub-projects and belong to neither. The two sub-repos
have their own git histories, their own CI, and deploy separately. They are
listed in `.gitignore` on purpose — without that, git records them as gitlinks
with no submodule config, which breaks clones.

Always run git commands from inside the relevant sub-repo. Committing "the
project" is never a single operation.

| Repo | Stack | Deploys to |
|---|---|---|
| `fengshui-shifu-api` | Rails 8, API-only, Ruby 3.3, RSpec | ECR → ECS Fargate behind an ALB |
| `fengshui-shifu-ui` | Expo / React Native Web, TypeScript | S3 + CloudFront |

Both deploy from `main` via GitHub Actions.

## Running it locally

```bash
# API — Rails + Postgres on :3000
cd fengshui-shifu-api && docker-compose up --build

# UI — Expo web on :8081
cd fengshui-shifu-ui && npm install && npm run web
```

The UI reads `EXPO_PUBLIC_API_URL`, defaulting to `http://localhost:3000/api/v1`.
In production that value is baked in at build time by CI, not read at runtime.

## Production

| | |
|---|---|
| Frontend | https://app.fengshui-shifu.com |
| API | https://api.fengshui-shifu.com |
| Health check | `curl -i https://api.fengshui-shifu.com/api/v1/health` |

**If the site is broken, start with that health check.** A CORS error in the
browser console is usually *not* a CORS problem — when the load balancer returns
its own error page, that page carries no `Access-Control-Allow-Origin` header and
Chrome reports a failed preflight. See the debugging section in
`fengshui-shifu-api/README.md`, and the port and target-group notes in
`CLAUDE.md`. Both document outages that have already happened here.

## The ASDD workflow

Features are built spec-first, with two approval gates. Nothing is implemented
without an approved spec and an approved plan.

```
/feature-spec <requirement>   → ai/feature-specs/<name>.md   ⏸ approve
   (answer any Open Questions, in the file or in chat)
/feature-plan <name>          → ai/plans/<name>.md           ⏸ approve
/feature-implement <name>     → one checkpoint at a time     ⏸ commit each
/commit-message               → message, summary, PR description
```

Specs and plans live in `ai/` **inside each sub-repo**, since they are scoped per
repo and belong in that repo's history. Cross-cutting features get a driver spec
in the repo where the change originates and a short companion spec in the other.

**Git stays in the user's hands.** The workflow never creates branches and never
commits — it reports work as ready and hands over the message. See the
"Git — hands off" section of `CLAUDE.md`.

The generic, project-independent version of these commands lives in the
`asdd-workflow-template` repo. Improvements should be made there first, then
copied here.

## Where things are

| Looking for | Go to |
|---|---|
| BaZi / Kua domain logic | `fengshui-shifu-api/app/services/bazi_calculator_service.rb` |
| Reference tables (stems, branches, Kua directions) | `…/bazi_calculator_service/constants.rb` |
| API/UI contract | `fengshui-shifu-ui/src/services/api.ts` — hand-mirrors the API response; **update both sides together** |
| All UI state and screens | `fengshui-shifu-ui/App.tsx` (single component) |
| Colors and styles | `fengshui-shifu-ui/src/styles/` — add to the palette, never inline hex |
| Production debugging | `fengshui-shifu-api/README.md` |
| Conventions and traps | `CLAUDE.md` |

## Domain note

BaZi, Kua and Ba Zhai are traditional rule-based systems — lookup tables and
modular arithmetic over a calendar, not statistical models. Changes to the
calculation logic are **domain changes needing correctness review**, not routine
refactors. The Kua formulas in particular branch on gender and on the
astrological solar year (which begins at Lichun, ~Feb 4, so January birthdays
belong to the previous year). That has been got wrong more than once.
