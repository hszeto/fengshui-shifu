# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This directory is **not** a git repository. It holds two independent repos that are deployed separately:

- `fengshui-shifu-api/` — Rails 8 API-only backend (Ruby ≥ 3.2, CI on 3.3)
- `fengshui-shifu-ui/` — Expo / React Native Web frontend (TypeScript)

Always run commands from inside the relevant sub-repo. Each repo has its own git
history and is committed to separately.

## Git — hands off

**Never create or delete git branches, and never commit.** These are the user's
to run, always.

- Do not run `git checkout -b`, `git branch`, `git branch -d`, or `git commit`.
- Work on whatever branch is currently checked out — never switch or create one.
  If `main` is checked out and the work warrants a branch, say so and stop.
- When a change is ready, stage nothing and commit nothing — instead say the work
  is ready, and provide the commit message for the user to use.
- If a change seems to warrant its own branch, say so and stop.

This overrides any default behaviour about branching before committing on a
default branch, or committing at natural checkpoint boundaries. `/commit-message`
produces a message; it does not authorise running `git commit`.

## Product vision & roadmap

This is a feng shui analysis app named **fengshui-shifu**. The current API/UI
(BaZi + Kua calculation from birth date/time/gender) is an early slice of a
larger planned product.

**Planned user inputs (not all implemented yet):**
- Address, birthday, sex — used for personal feng shui calculations (Kua
  number, personal directions) — *partially implemented today*
- Optional: photos/video of the house and individual rooms — *not implemented*
- Optional: floor plan upload — *not implemented*. This must support both
  formal blueprints AND informal hand-drawn sketches (e.g. drawn on a napkin),
  which have very different image quality/structure and will need different
  parsing strategies (likely AI vision-assisted interpretation of room layout,
  doors, windows, orientation).

**Implication for current architecture:** the API is currently stateless
(see below — no DB, no models). Supporting photo/floor-plan uploads and
per-user history will require introducing persistence (ActiveRecord +
Postgres) and file storage (e.g. S3) from scratch. When that work begins,
this CLAUDE.md needs a significant update — a lot of "no models exist"
statements below will become stale.

**Domain note:** feng shui calculations (BaZi, Kua, Ba Zhai/Eight Mansions
method, house facing/sitting direction from address) are traditional/rule-based
systems, not statistical models — treat changes to `bazi_calculator_service.rb`
as domain-logic changes requiring correctness review, not just code review.

## Commands

### API (`fengshui-shifu-api`)

```bash
docker-compose up --build          # Rails + Postgres, API on :3000
bundle install
bin/rails server                   # native, requires Postgres on localhost
bundle exec rspec                  # full suite
bundle exec rspec spec/services/bazi_calculator_service_spec.rb        # one file
bundle exec rspec spec/requests/api/v1/bazi_spec.rb:10                 # one example by line
bundle exec rubocop                # rubocop-rails, config in .rubocop.yml
```

RSpec loads the full Rails environment (`config/environment.rb` with the ActiveRecord railtie), so **a reachable Postgres is required to run tests even though nothing touches the database**. Override connection details with `POSTGRES_HOST` / `POSTGRES_USER` / `POSTGRES_PASSWORD`.

### UI (`fengshui-shifu-ui`)

```bash
npm install
npm run web                        # Expo web dev server on :8081
npx expo start                     # Metro bundler; scan QR with Expo Go for iOS/Android
npm run build:web                  # expo export -p web -> dist/
```

There is no test runner, linter, or type-check script configured for the UI.

## Architecture

### Stateless backend

The API has **no `db/` directory, no models, and no migrations**. `pg`, `solid_cache`, and `solid_queue` are in the Gemfile and `config/database.yml` is populated, but every request is computed in-process and nothing is persisted. Don't assume an ActiveRecord layer exists; if a feature needs one, it has to be introduced from scratch.

Two routes only (`config/routes.rb`):

- `GET /api/v1/health` → version/status payload
- `POST /api/v1/bazi/calculate` → BaZi + Kua computation

`BaziController` uses `wrap_parameters false` so params arrive flat (`birth_date`, `birth_time`, `gender`) rather than nested under a root key. It validates only that `birth_date` is present and rescues `ArgumentError` from `Date.parse` into a 422.

### Domain logic

All of it lives in `app/services/bazi_calculator_service.rb`, with reference tables (heavenly stems, earthly branches, Kua direction profiles) in `app/services/bazi_calculator_service/constants.rb`. Key behaviours to preserve when editing:

- **Day pillar** is derived from the Julian day number: `(date.jd + 5) % 60`, then `% 10` for the stem and `% 12` for the branch.
- **Kua number** uses the *astrological* solar year — dates before ~Feb 4 (Lichun) belong to the previous year. Male and female formulas differ, and both branch on pre-2000 vs post-2000 birth years. This has been corrected more than once; changes here need matching spec updates.
- **Kua is nil when gender is absent**, which makes `kua_number` and `kua_profile` null in the response. Frontend code must tolerate that.
- **`birth_time` is optional.** When present it adds `birth_time` and `hour_branch` keys to the response; when absent those keys are omitted entirely, not null.
- **`today_luck_teaser` is randomized** (`rand(82..98)`), so specs must not assert its exact text.

### API/UI contract

`fengshui-shifu-ui/src/services/api.ts` declares TypeScript interfaces (`BaziCalculationResult`, `HealthResponse`) that hand-mirror the hash built by `BaziCalculatorService#build_calculation_result`. Any change to the response shape must be made in both repos. The client also normalizes several error shapes (`error`, `message`, bare HTTP status) into `{ success: false, error }` — the backend currently only emits `error`.

`API_BASE_URL` comes from `EXPO_PUBLIC_API_URL`, defaulting to `http://localhost:3000/api/v1`. It is baked in at build time by the CI workflow, not read at runtime.

### UI structure

`App.tsx` is a single 350-line component holding all state and the calculate flow. Styling is centralized: `src/styles/colors.ts` defines the dark palette, `src/styles/appStyles.ts` builds the `StyleSheet` from it. Add colors to the palette rather than inlining hex values. The app is dark-mode only (`userInterfaceStyle: "dark"`, background `#0F172A`).

## Deployment

Both repos deploy from `main` via GitHub Actions to Render.

- **API** (`.github/workflows/deploy.yml`): RSpec against a Postgres service container, then triggers a Render deploy via webhook.
- **UI**: Render auto-deploys on push — `expo export -p web` with `EXPO_PUBLIC_API_URL=https://api.fengshui-shifu.com/api/v1`.

*Originally deployed to AWS (ECR/ECS/CloudFront) from Aug–Sept 2026. Migrated to Render for cost and simplicity. See [`docs/aws-infrastructure-legacy.md`](docs/aws-infrastructure-legacy.md) for the historical AWS setup.*

### Port 3000 everywhere — do not "fix" this to 80

**The container must listen on 3000.** The production `Dockerfile` creates a
non-root `rails` user (`USER rails:rails`) before starting Puma, and a non-root
process cannot bind a privileged port (<1024). Setting `-p 80` makes Puma die at
boot with `Permission denied - bind(2) for "0.0.0.0" port 80 (Errno::EACCES)`.

The commit history contains **five** commits flipping this between 80 and 3000.
Every 80 version was incapable of running. This constraint is not platform-specific
— it's the Unix permission model. Render sets `PORT=3000` as an env var; the
Dockerfile is untouched.

`docker-compose.yml` also runs the dev server on 3000, so dev and prod agree.

### CORS

Wide open by design — `origins '*'` in `config/initializers/cors.rb`, to support
mobile clients. The UI and API are on separate hostnames (Render services), so
requests are cross-origin; CORS permissiveness makes that work.

If a request to the API fails, always run
`curl -i https://api.fengshui-shifu.com/api/v1/health` first — if it's not a 200,
the backend is down.
