# Migrating from AWS to Render

**Status: not started.** Production still runs on AWS. This is the plan of
record — work through it top to bottom and update the status line as you go.

[`infrastructure.md`](infrastructure.md) describes the AWS setup this replaces.
Keep it until teardown is finished; it is the only record of what exists.

## Why

Not the money — $21.48/mo is not the problem. The problem is that seven
hand-configured AWS services with no infrastructure-as-code produced a
weeks-long outage in Aug 2026, caused by two target groups silently
disagreeing about which one the listener pointed at.

Render collapses that to two services and a YAML file per repo.

| | AWS today | Render |
|---|---|---|
| Services to configure | 7 | 2 |
| Defined in code | none | `render.yaml` in each repo |
| Cost | ~$20.50/mo (excl. `zteeli.com`) | ~$7/mo |

**The $15 you are paying for the ALB and its public IPv4 address is more than
the $5.06 of Fargate compute it balances.**

## Why this migration is unusually low-risk

**The API has no database.** No `db/`, no models, no migrations — every request
is computed in-process. There is no data to move, no dump/restore window, no
dual-write period. Statelessness is the whole reason this is a two-evening job
rather than a two-week one.

Everything below is also reversible up to the moment you delete AWS resources,
because DNS is the only cutover mechanism and DNS changes roll back.

## What maps to what

| AWS | Render |
|---|---|
| ECR + ECS cluster + service + task definition | Web Service, built from the existing `Dockerfile` |
| ALB + listener + target group + health check | Built in — `healthCheckPath` |
| CloudFront + ACM certificate | Built in — free managed TLS |
| S3 bucket + CloudFront default behaviour | Static Site publishing `dist/` |
| CloudWatch log group | Render service logs |
| `SECRET_KEY_BASE` as plaintext task-def env | Render environment variable (encrypted) |
| `aws ecs update-service` in GitHub Actions | Render deploy hook, still gated on RSpec |

## The only code change: `PORT`

The `Dockerfile` ends with `CMD ["bin/rails", "server", "-b", "0.0.0.0", "-p", "3000"]`.

Render injects `$PORT` (default 10000) and health-checks whatever port it
detects. Rather than edit the `CMD`, **set `PORT=3000` in Render's environment
variables.** Render then health-checks 3000 and the `Dockerfile` is untouched.

This keeps the rule in [`../CLAUDE.md`](../CLAUDE.md) intact — container listens
on 3000, non-root user can bind it because 3000 is above 1024. That constraint
was never about AWS.

## Before you start

- A Render account with both repos connected via GitHub.
- The current `SECRET_KEY_BASE`, read from the live task definition **before you
  tear anything down**:
  ```bash
  aws ecs describe-task-definition --task-definition fengshui-shifu-api-task:4 \
    --region us-east-1 \
    --query 'taskDefinition.containerDefinitions[0].environment'
  ```
  Copy every variable it lists, not just `SECRET_KEY_BASE`. Reuse the existing
  128-character value — it has never been committed, logged, or exposed.
- Access to the DNS registrar where `fengshui-shifu.com` is managed. **It is not
  Route 53** — there is no hosted zone for this domain in the AWS account.
- Confirm Render's current pricing yourself. Starter is $7/mo per web service
  and static sites are free, but Render has restructured workspace plans before.

> **Do not use Render's free tier for the API.** It spins down after inactivity
> and cold starts take ~50 seconds, which is fatal for an app whose value is an
> instant reading.

---

## Phase 1 — Stand up the API on Render

No traffic moves in this phase. AWS keeps serving everything.

1. **Add `render.yaml` to `fengshui-shifu-api`** (contents in
   [Appendix A](#appendix-a--renderyaml-for-the-api)).

2. **Create the service.** Render dashboard → New → Blueprint → pick the API
   repo. It reads `render.yaml`.

3. **Set the secrets.** `SECRET_KEY_BASE` is declared `sync: false`, so Render
   prompts for it rather than reading it from git. Paste the value from the task
   definition. Add any other variables you copied.

4. **Deploy and verify against Render's own hostname:**
   ```bash
   curl -i https://fengshui-shifu-api.onrender.com/api/v1/health
   ```
   Expect `200` and a JSON body with `status`, `rails_version`, `ruby_version`.

5. **Verify the real endpoint too** — health checks pass on broken apps:
   ```bash
   curl -sS -X POST https://fengshui-shifu-api.onrender.com/api/v1/bazi/calculate \
     -H 'Content-Type: application/json' \
     -d '{"birth_date":"1985-07-14","gender":"male","birth_time":"14:30"}'
   ```
   Compare the `day_master`, `day_branch` and `kua_number` against the same
   request to `https://api.fengshui-shifu.com/api/v1/bazi/calculate`. They must
   match exactly. `today_luck_teaser` will differ — it is `rand(82..98)`.

6. **Add the custom domain** `api.fengshui-shifu.com` to the Render service.
   Render will show it as unverified because DNS still points at CloudFront.
   That is expected; leave it.

**Gate: do not continue until step 5 produces identical BaZi output.**

## Phase 2 — Stand up the UI on Render

Still no traffic moved.

1. **Add `render.yaml` to `fengshui-shifu-ui`**
   ([Appendix B](#appendix-b--renderyaml-for-the-ui)).

   Note it sets `EXPO_PUBLIC_API_URL` to `https://api.fengshui-shifu.com/api/v1`
   — **the same value CI bakes in today.** This is deliberate. The UI keeps
   talking to whatever that hostname resolves to, so cutting the API over later
   requires no rebuild of the UI.

2. **Create the Static Site** from the blueprint.

3. **Open Render's URL** (`https://fengshui-shifu-ui.onrender.com`) and run a
   real calculation. At this point the Render-hosted UI is calling the
   **AWS-hosted** API, which proves the build and the API contract independently
   of the API migration.

4. **Add the custom domains** `fengshui-shifu.com`, `www.` and `app.` to the
   Render static site. Unverified for now, same as before.

### A note on CORS

Today the UI and API share one CloudFront distribution, so requests are
same-origin. On Render they are two hostnames, so requests become cross-origin
again. This already works: `config/initializers/cors.rb` is `origins '*'` by
design, to support mobile clients.

Do **not** be tempted to use a Render static-site rewrite to proxy `/api/*` and
restore same-origin. A relative `EXPO_PUBLIC_API_URL` would break the iOS and
Android builds — React Native's `fetch` requires an absolute URL, and
`src/services/api.ts` interpolates the value directly.

## Phase 3 — Cut DNS

This is the actual migration. Each record is independent and independently
reversible.

Use the exact record values Render shows in each service's Settings → Custom
Domains. Do not copy IP addresses from documentation — Render's anycast address
has changed before.

**Cut the API first**, alone:

| Host | Change to |
|---|---|
| `api` | CNAME → the value Render shows for the API service |

Lower the TTL on these records to 300s **at least a day beforehand**, or
rollback will take as long as the old TTL.

Then watch:

```bash
dig +short api.fengshui-shifu.com
curl -i https://api.fengshui-shifu.com/api/v1/health
```

The AWS-hosted UI at `app.fengshui-shifu.com` is now calling the Render API.
Load it and run a calculation. **Leave it a full day.**

Then cut the UI:

| Host | Change to |
|---|---|
| `app` | CNAME → the value Render shows for the static site |
| `www` | CNAME → same |
| `@` (apex) | A / ALIAS → the value Render shows |

Verify all four hostnames serve and the app works end to end.

**Stop here for a week before Phase 4.** Nothing below is reversible.

## Phase 4 — Tear down AWS

Order matters. Deleting CloudFront while DNS still points at it is an outage,
which is why this phase comes after a week of stable Render traffic.

1. **CloudFront** — the slowest, so start it first.
   ```bash
   aws cloudfront get-distribution-config --id E2BAXAQRJHI45M
   ```
   Set `Enabled: false`, update, wait for the status to return to `Deployed`
   (~15 minutes), *then* delete. The console does this in two clicks and is less
   error-prone than assembling the update payload by hand.

2. **ECS** — scale to zero, then remove.
   ```bash
   aws ecs update-service --cluster fengshui-shifu-cluster \
     --service fengshui-shifu-api-task --desired-count 0 --region us-east-1
   aws ecs delete-service --cluster fengshui-shifu-cluster \
     --service fengshui-shifu-api-task --force --region us-east-1
   aws ecs delete-cluster --cluster fengshui-shifu-cluster --region us-east-1
   ```

3. **Load balancer** — listener, then ALB, then target group. **This is the
   $9.09.**
   ```bash
   aws elbv2 describe-load-balancers --names fengshui-api-alb --region us-east-1
   aws elbv2 delete-load-balancer --load-balancer-arn <arn> --region us-east-1
   aws elbv2 delete-target-group --target-group-arn <arn> --region us-east-1
   ```
   Deleting the ALB also releases its two public IPv4 addresses — most of the
   $6.21. The third was the Fargate task's, released in step 2.

4. **S3.**
   ```bash
   aws s3 rm s3://fengshui-shifu-ui-prod --recursive
   aws s3api delete-bucket --bucket fengshui-shifu-ui-prod --region us-east-1
   ```

5. **ECR.**
   ```bash
   aws ecr delete-repository --repository-name fengshui-shifu-api \
     --force --region us-east-1
   ```

6. **CloudWatch logs.**
   ```bash
   aws logs delete-log-group --log-group-name /ecs/fengshui-shifu-api-task \
     --region us-east-1
   ```

7. **ACM certificate** — free, but delete it once CloudFront is gone; it cannot
   be deleted while still associated.

8. **Rotate the deploy credentials.** The `AWS_ACCESS_KEY_ID` /
   `AWS_SECRET_ACCESS_KEY` in both repos' GitHub secrets can now push images and
   invalidate a distribution that no longer exists. Delete the IAM access key in
   AWS, then delete the GitHub secrets. Do this even though the resources are
   gone — the key may carry broader permissions than it needed.

### Leave alone

- **Route 53 (`zteeli.com`, `www.zteeli.com`)** — $1.01/mo, belongs to an
  unrelated legacy project. Deleting a hosted zone breaks that project's DNS.
  Decide separately.
- **SES** — 200/24h quota, zero sent. Not billed, and identity verification is
  tedious to redo.
- **Secrets Manager** — already empty.
- **The default VPC, subnets, security groups, internet gateway** — all free.
  There is no VPC line item to cut; what Cost Explorer files under "Amazon
  Virtual Private Cloud" is public IPv4 addresses, released in steps 2 and 3.

### Confirm it worked

Check the following month's bill by usage type. `LoadBalancerUsage`,
`PublicIPv4:InUseAddress`, `Fargate-*` and ECR should all be gone.

```bash
aws ce get-cost-and-usage --time-period Start=YYYY-MM-01,End=YYYY-MM-DD \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE
```

Expect roughly $1.01 remaining, all of it `zteeli.com`.

## Phase 5 — Clean up the repos

Only after Phase 4.

- **`fengshui-shifu-api/README.md`** — lines 87–257 are an AWS debugging guide
  (ECS Exec, `describe-target-health`, ALB triage). Nearly all of it becomes
  wrong. Replace with the Render equivalent: dashboard logs, Render Shell, and
  the same `curl` health check, which is still the correct first move.
- **`../CLAUDE.md`** — the Deployment section, the target-group half of the
  "Port 3000 everywhere" section, and the CORS section's ALB explanation. Keep
  the port-3000 rule itself; the non-root binding constraint still applies.
- **`infrastructure.md`** — delete it, or retitle it as a historical record.
  Do not leave it looking current.
- **`../README.md`** — collapse the "Deploys to (today) / Target" columns back
  to one, and drop the migration callout.
- Delete `fengshui-shifu-ui/.github/workflows/` entirely — Render's `autoDeploy`
  replaces it. Keep the API workflow; see Appendix C.

## Rollback

| Phase | How to undo |
|---|---|
| 1–2 | Delete the Render services. Nothing else changed. |
| 3 | Point the DNS records back. Bounded by TTL — lower it to 300s first. |
| 4 | **None.** Rebuilding means redoing the 2026-08-13 console work by hand. |

---

## Appendix A — `render.yaml` for the API

Create at `fengshui-shifu-api/render.yaml`:

```yaml
services:
  - type: web
    name: fengshui-shifu-api
    runtime: docker
    dockerfilePath: ./Dockerfile
    plan: starter
    region: virginia          # closest to the current us-east-1
    branch: main
    healthCheckPath: /api/v1/health
    autoDeploy: false         # GitHub Actions deploys, but only after RSpec passes
    envVars:
      - key: PORT
        value: 3000           # matches the Dockerfile CMD; see CLAUDE.md
      - key: SECRET_KEY_BASE
        sync: false           # prompted for in the dashboard, never stored in git
```

## Appendix B — `render.yaml` for the UI

Create at `fengshui-shifu-ui/render.yaml`:

```yaml
services:
  - type: web
    name: fengshui-shifu-ui
    runtime: static
    branch: main
    autoDeploy: true          # no test suite to gate on
    buildCommand: npm ci && npx expo export -p web
    staticPublishPath: ./dist
    envVars:
      - key: EXPO_PUBLIC_API_URL
        value: https://api.fengshui-shifu.com/api/v1
      - key: NODE_VERSION
        value: 18             # matches CI; bump separately, not during migration
    routes:
      - type: rewrite
        source: /*
        destination: /index.html
```

`EXPO_PUBLIC_API_URL` is baked in at build time, so changing it requires a
redeploy, not just a restart.

## Appendix C — API workflow

Keep the `test` job exactly as it is. Replace the whole `deploy` job with:

```yaml
  deploy:
    name: Trigger Render Deploy
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Render deploy
        run: curl -fsS -X POST "${{ secrets.RENDER_DEPLOY_HOOK_URL }}"
```

Get the hook URL from Render → the API service → Settings → Deploy Hook, and
store it as a GitHub secret. **This is why `autoDeploy` is `false` for the API:**
Render's own auto-deploy fires on push regardless of whether RSpec passed, which
would be a real regression given how much domain logic lives in
`bazi_calculator_service.rb`.

## Appendix D — known issue, not caused by this migration

`fengshui-shifu-api/config/environments/` **does not exist** — there is no
`production.rb`, `development.rb` or `test.rb`. Rails boots anyway, applying only
`config.load_defaults 8.0` from `application.rb`, which is why this has gone
unnoticed.

The consequence is that `RAILS_ENV` currently changes almost nothing: no eager
loading, no production logging config, no `force_ssl`. Carry the existing
environment variables over to Render **unchanged** so the migration is a true
like-for-like move, then fix this separately. Introducing a `production.rb`
during a platform migration means two variables changing at once, and a failure
you cannot attribute.
