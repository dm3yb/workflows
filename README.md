# workflows

Reusable GitHub Actions workflows for the aerixity deploys. Public on purpose:
`aerixity-labs/dither-studio` lives under another owner and could not call them from a private repo.
**No secrets live here.** Callers pass theirs with `secrets: inherit` (a caller in another organization, like dither-studio, must pass `DEPLOY_SSH_KEY_V2` explicitly instead), and repo variables resolve in
the caller.

Pin `@v1`. The workflows call each other and `.github/actions/setup` at `@v1` too, so a release is
"merge to main, then move the `v1` tag".

| Workflow | What it does |
| --- | --- |
| `ci-next.yml` | Parallel `lint`, `typecheck`, `test` and `build` jobs, plus an optional Docker build without push. The build job can also stamp and upload a static export. |
| `deploy-node.yml` | Builds and pushes `ghcr.io/dm3yb/<app>:<sha>`, then deploys blue/green over SSH. Smoke test, Sentry deploy marker, keeps 3 image versions. |
| `deploy-static.yml` | Builds the export with `ci-next.yml`, ships it as release `<sha>-<run_number>` over SSH, then smoke-tests `/version.txt`. |

## Server contract

Every deploy goes through one forced command on the server (`server/deploy` in `dm3yb/infra`),
reached as `DEPLOY_USER@DEPLOY_HOST` with the key `DEPLOY_SSH_KEY_V2` and the pinned
`DEPLOY_KNOWN_HOSTS`.

| Call | stdin |
| --- | --- |
| `env <app>` | the app's env file (KEY=VALUE lines) |
| `node <app> <sha>` | a registry token for one GHCR pull. **Empty for a rollback**: the server uses the image it already has |
| `static <site> <release>` | a tar.gz of the export. **Empty for a rollback**: the server switches to that release if it still has it (the newest 5 are kept); a bare SHA picks that commit's newest build |

## Secrets and variables (in the calling repo)

| Name | Kind | Used by |
| --- | --- | --- |
| `DEPLOY_SSH_KEY_V2` | secret | deploy-node, deploy-static |
| `APP_ENV_FILE` (or the name given in `env-file-secret`) | secret | deploy-node |
| `SENTRY_AUTH_TOKEN` | secret, optional | deploy-node (source maps, release check, deploy marker) |
| `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` | secret, optional | deploy-node (BuildKit secret `server_actions_key`) |
| `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_KNOWN_HOSTS` | variables | deploy-node, deploy-static |
| `SMOKE_VIA_ORIGIN` | variable | `true` sends the smoke test straight to the server (before a domain's cutover) |

Permissions on the calling job: `contents: read` for every workflow, plus `packages: write` for
`deploy-node` (push, pull, retention).

## Inputs

### ci-next.yml

| Input | Default | Meaning |
| --- | --- | --- |
| `package-manager` | `pnpm` | `pnpm`, `bun` or `yarn` (yarn 1); pnpm and bun read their versions from `package.json` |
| `node-version` / `node-version-file` | – | One of them; the file is relative to `working-directory` |
| `working-directory` | `.` | Where `package.json` and the lockfile are |
| `env` | `{}` | JSON object of environment values for every job, e.g. `{"SKIP_ENV_VALIDATION":"1"}` |
| `lint`, `typecheck`, `test`, `build` | empty | Commands. Each one non-empty becomes a parallel job |
| `build-env` | `{}` | JSON object of public build-time values (`NEXT_PUBLIC_*`), build job only |
| `export-dir` | empty | Static export dir: checked for `index.html` and `404.html` |
| `release` | empty | When set, writes `<export-dir>/version.txt` and uploads the export as artifact `site` |
| `docker-file`, `docker-context`, `docker-build-args`, `docker-cache-scope` | empty, `.`, empty, `app` | Optional image build without push (`GIT_SHA` is always passed) |

### deploy-node.yml

| Input | Default | Meaning |
| --- | --- | --- |
| `app` | required | App name on the server and in GHCR |
| `public-url` | required | `/api/health` there must report the new SHA |
| `dockerfile`, `context` | `Dockerfile`, `.` | Image build |
| `build-args` | empty | One `KEY=value` per line; `GIT_SHA` is always set |
| `env-file-secret` | `APP_ENV_FILE` | Name of the secret with the runtime env file |
| `sha` | empty | Rollback: deploy this commit's image without building |
| `keep-images` | `3` | Image versions kept in GHCR |

Concurrency is built in: `build-<app>` cancels an older build when a newer commit arrives;
`deploy-<app>` never cancels a running deploy, and GitHub keeps only the newest pending one. A
freshness guard skips the deploy when `main` has already moved past the commit. Rollbacks bypass it.

### deploy-static.yml

`site`, `public-url` and `build` are required. `sha` is the rollback target: a release id or a bare
SHA. The other inputs are the same as for `ci-next.yml` (`export-dir` defaults to `out`).

## Callers

### A Node app (slotwise)

```yaml
# .github/workflows/ci.yml
name: ci
on: pull_request
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  ci:
    uses: dm3yb/workflows/.github/workflows/ci-next.yml@v1
    with:
      node-version: '24'
      env: '{"SKIP_ENV_VALIDATION":"1"}'
      lint: pnpm lint
      typecheck: pnpm typecheck
      test: pnpm test
      docker-file: Dockerfile
      docker-cache-scope: slotwise
      docker-build-args: |
        NEXT_PUBLIC_SITE_URL=${{ vars.NEXT_PUBLIC_SITE_URL }}
        NEXT_PUBLIC_SENTRY_DSN=${{ vars.NEXT_PUBLIC_SENTRY_DSN }}
        NEXT_PUBLIC_TURNSTILE_SITE_KEY=${{ vars.NEXT_PUBLIC_TURNSTILE_SITE_KEY }}
        STORAGE_ENDPOINT=${{ vars.STORAGE_ENDPOINT }}
```

```yaml
# .github/workflows/deploy.yml
name: deploy
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      sha:
        description: Roll back to this commit (its image is still on the server)
        default: ''
jobs:
  checks:
    if: inputs.sha == ''
    uses: dm3yb/workflows/.github/workflows/ci-next.yml@v1
    with:
      node-version: '24'
      env: '{"SKIP_ENV_VALIDATION":"1"}'
      lint: pnpm lint
      typecheck: pnpm typecheck
      test: pnpm test
  deploy:
    needs: checks
    if: always() && (needs.checks.result == 'success' || (inputs.sha != '' && needs.checks.result == 'skipped'))
    permissions:
      contents: read
      packages: write
    uses: dm3yb/workflows/.github/workflows/deploy-node.yml@v1
    secrets: inherit
    with:
      app: slotwise
      public-url: https://slotwise.aerixity.com
      sha: ${{ inputs.sha }}
      build-args: |
        NEXT_PUBLIC_SITE_URL=${{ vars.NEXT_PUBLIC_SITE_URL }}
        NEXT_PUBLIC_SENTRY_DSN=${{ vars.NEXT_PUBLIC_SENTRY_DSN }}
        NEXT_PUBLIC_TURNSTILE_SITE_KEY=${{ vars.NEXT_PUBLIC_TURNSTILE_SITE_KEY }}
        STORAGE_ENDPOINT=${{ vars.STORAGE_ENDPOINT }}
```

### The monorepo (netcore-app: vault and ledger)

There is one deploy file per app, with path filters, so only the app a change touches is built and
deployed. Here is the ledger one; the vault one is the same with `apps/vault`,
`app: netcore-vault`, `public-url: https://api.netcoreapp.com` and `APP_ENV_FILE_VAULT`. The CI
callers are split the same way.

```yaml
# .github/workflows/deploy-ledger.yml
name: deploy-ledger
on:
  push:
    branches: [main]
    paths: ['apps/ledger/**', 'packages/**', 'pnpm-lock.yaml', '.github/workflows/deploy-ledger.yml']
  workflow_dispatch:
    inputs:
      sha:
        description: Roll back to this commit
        default: ''
jobs:
  checks:
    if: inputs.sha == ''
    uses: dm3yb/workflows/.github/workflows/ci-next.yml@v1
    with:
      node-version: '24'
      lint: pnpm --filter @netcore/ledger lint
      typecheck: pnpm --filter @netcore/ledger typecheck
      test: pnpm --filter @netcore/ledger test
  deploy:
    needs: checks
    if: always() && (needs.checks.result == 'success' || (inputs.sha != '' && needs.checks.result == 'skipped'))
    permissions:
      contents: read
      packages: write
    uses: dm3yb/workflows/.github/workflows/deploy-node.yml@v1
    secrets: inherit
    with:
      app: netcore-ledger
      dockerfile: apps/ledger/Dockerfile
      public-url: https://netcoreapp.com
      env-file-secret: APP_ENV_FILE_LEDGER
      sha: ${{ inputs.sha }}
      build-args: |
        NEXT_PUBLIC_APP_URL=${{ vars.NEXT_PUBLIC_APP_URL }}
        NEXT_PUBLIC_BETTER_AUTH_URL=${{ vars.NEXT_PUBLIC_BETTER_AUTH_URL }}
        NEXT_PUBLIC_SENTRY_DSN=${{ vars.NEXT_PUBLIC_SENTRY_DSN }}
        NEXT_PUBLIC_CLOUDFLARE_TURNSTILE_SITE_KEY=${{ vars.NEXT_PUBLIC_CLOUDFLARE_TURNSTILE_SITE_KEY }}
        SENTRY_ORG=${{ vars.SENTRY_ORG }}
        SENTRY_PROJECT=${{ vars.SENTRY_PROJECT }}
```

### A static site rebuilt on Sanity publish (oknailsalon-web)

```yaml
# .github/workflows/deploy.yml
name: deploy
on:
  push:
    branches: [main]
  repository_dispatch:
    types: [sanity-publish]
  workflow_dispatch:
    inputs:
      sha:
        description: Roll back to a release id (<sha>-<run>) or SHA still on the server
        default: ''
jobs:
  deploy:
    permissions:
      contents: read
    uses: dm3yb/workflows/.github/workflows/deploy-static.yml@v1
    secrets: inherit
    with:
      site: oknailsalon-web
      public-url: https://oknailsalon.com
      sha: ${{ inputs.sha }}
      package-manager: bun
      node-version-file: .nvmrc
      lint: bun run lint
      typecheck: bun run typecheck
      build: bun run build
      build-env: '{"NEXT_PUBLIC_SANITY_PROJECT_ID":"${{ vars.NEXT_PUBLIC_SANITY_PROJECT_ID }}","NEXT_PUBLIC_SANITY_DATASET":"${{ vars.NEXT_PUBLIC_SANITY_DATASET }}"}'
```

The Sanity webhook sends `POST https://api.github.com/repos/dm3yb/oknailsalon-web/dispatches` with
the body `{"event_type": "sanity-publish"}`.
