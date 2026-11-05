# workflows

Reusable GitHub Actions workflows for the aerixity deploys. Public on purpose:
`aerixity-labs/dither-studio` lives under another owner and could not call them from a private repo.
**No secrets live here.** Callers pass theirs with `secrets: inherit` (a caller in another
organization, like dither-studio, must pass `DEPLOY_SSH_KEY_V2` explicitly instead), and repo
variables resolve in the caller.

Pin `@v2`. The workflows call each other and `.github/actions/setup` at `@v2` too, so a release is
"merge to main, then move the `v2` tag". `v1` (Docker images, blue/green) stays frozen for history.

| Workflow | What it does |
| --- | --- |
| `ci-next.yml` | Parallel `lint`, `typecheck`, `test` and `build` jobs. The build job can also stamp and upload a static export. |
| `deploy-node.yml` | Builds the Next.js standalone server, uploads it with rsync and has the server release it and reload it in PM2. Smoke test through Cloudflare, Sentry deploy marker. |
| `deploy-static.yml` | Builds the export with `ci-next.yml`, uploads it with rsync as release `<sha>-<run_number>`, then smoke-tests `/version.txt`. |

## Server contract

Every call goes to one forced command on the server (`server/deploy` in `dm3yb/infra`), reached as
`DEPLOY_USER@DEPLOY_HOST` with the key `DEPLOY_SSH_KEY_V2` and the pinned `DEPLOY_KNOWN_HOSTS`.

| Call | What happens |
| --- | --- |
| `rsync … <name>/` | The upload lands in `/srv/incoming/<name>/` (rrsync, write-only). It carries a `RELEASE` file with its id. |
| `node <app> <sha>` | The upload becomes `/srv/apps/<app>/releases/<sha>`, `current` switches, PM2 reloads the app and `/api/health` must report the SHA, or it switches back. **A rollback** names a release still on the server (the newest 5 are kept) and uploads nothing. |
| `static <site> <release>` | The same for `/srv/static/<site>`. A bare SHA picks that commit's newest build. |

Runtime secrets live only in `/srv/env/<app>.env` on the server; PM2 passes them with `--env-file`.

## Secrets and variables (in the calling repo)

| Name | Kind | Used by |
| --- | --- | --- |
| `DEPLOY_SSH_KEY_V2` | secret | deploy-node, deploy-static |
| `SENTRY_AUTH_TOKEN` | secret, optional | deploy-node (source maps, release check, deploy marker) |
| `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` | secret, optional | deploy-node (embedded in the build) |
| `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_KNOWN_HOSTS` | variables | deploy-node, deploy-static |

Permissions on the calling job: `contents: read`.

## Inputs

**`ci-next.yml`:**
- `package-manager` (`pnpm`), `node-version` or `node-version-file`, `working-directory` (`.`);
- `env` and `build-env`: JSON objects;
- `lint`, `typecheck`, `test`, `build`: commands; an empty one skips its job;
- `export-dir`, `release`: static sites only.

**`deploy-node.yml`:**

| Input | Default | What |
| --- | --- | --- |
| `app` | required | Name on the server (`/srv/apps/<app>`, the PM2 process) |
| `public-url` | required | Its `/api/health` must report the new SHA |
| `sha` | empty | Rollback: switch to this commit's release without building |
| `app-dir` | `.` | The Next.js app's folder from the repo root (`apps/ledger` in a monorepo) |
| `build` | `pnpm build` | Build command, run from the repo root |
| `build-env` | `{}` | Build-time values: `NEXT_PUBLIC_*` and placeholders the env check needs |
| `package-manager`, `node-version` | `pnpm`, `24` | Toolchain (the server runs Node 24) |

`GIT_SHA`, `SENTRY_RELEASE` and `VERCEL_DEPLOYMENT_ID` are set to the commit during the build. PM2
sets the same at runtime.

**`deploy-static.yml`:** `site`, `public-url` and `sha`, plus the `ci-next.yml` inputs. `build` is
required, and `export-dir` defaults to `out`.

## Example: a Node app

```yaml
# .github/workflows/deploy.yml
name: deploy
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      sha:
        description: Roll back to this commit (its release is still on the server)
        required: false
        default: ''
permissions:
  contents: read
jobs:
  deploy:
    uses: dm3yb/workflows/.github/workflows/deploy-node.yml@v2
    secrets: inherit
    with:
      app: slotwise
      public-url: https://slotwise.aerixity.com
      sha: ${{ inputs.sha }}
      build-env: |
        {"NEXT_PUBLIC_SITE_URL": "${{ vars.NEXT_PUBLIC_SITE_URL }}", "SKIP_ENV_VALIDATION": "1"}
```
