# ci-workflows

Reusable GitHub Actions workflows shared across naoki1510 projects.

## Available workflows

| Workflow | Purpose |
|---|---|
| [`rails-ci.yml`](.github/workflows/rails-ci.yml) | Brakeman + RuboCop + RSpec for Rails apps. Switches between PostgreSQL and MySQL service containers via input. |
| [`vite-ci.yml`](.github/workflows/vite-ci.yml) | ESLint + Prettier + tsc + Vitest + Vite build for pnpm-managed TypeScript projects. |
| [`kamal-deploy.yml`](.github/workflows/kamal-deploy.yml) | Deploy a Rails app to a VPS via Kamal. SSH key + arbitrary env vars are passed as secrets. |

All workflows are pinned to `v1` for caller stability. Update the `v1` tag (force-push) for backward-compatible changes; bump to `v2` for breaking changes.

## Conventions enforced by these workflows

- Ruby version is auto-detected by `ruby/setup-ruby` from `.ruby-version` / `.tool-versions` / `mise.toml` under the working directory.
- Node version is read from `.node-version` at the repo root.
- Frontend uses **pnpm** (no npm support — npm-based projects must migrate first).
- Action versions: `actions/checkout@v6`, `actions/setup-node@v6`, `pnpm/action-setup@v4`, `ruby/setup-ruby@v1`, `docker/setup-buildx-action@v4`.

## Recommended caller settings

caller 側で `concurrency` を設定すると、同じブランチへの連続 push で古い run が自動キャンセルされ、CI 時間とコストを削減できる。

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## Usage

### Rails CI (PostgreSQL example)

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]

jobs:
  rails:
    uses: naoki1510/ci-workflows/.github/workflows/rails-ci.yml@v1
    with:
      working-directory: backend
      db-engine: postgres
      install-system-packages: build-essential git libpq-dev libyaml-dev pkg-config
```

### Rails CI (MySQL example)

```yaml
jobs:
  rails:
    uses: naoki1510/ci-workflows/.github/workflows/rails-ci.yml@v1
    with:
      working-directory: backend
      db-engine: mysql
      mysql-database: app_test
      test-prepare-cmd: |
        bundle exec rails db:create
        bundle exec rails db:schema:load
```

### Vite CI (pnpm workspaces, filtered)

```yaml
jobs:
  vite:
    uses: naoki1510/ci-workflows/.github/workflows/vite-ci.yml@v1
    with:
      pnpm-filter: frontend
```

### Vite CI (single-package repo)

```yaml
jobs:
  vite:
    uses: naoki1510/ci-workflows/.github/workflows/vite-ci.yml@v1
```

### Kamal deploy

```yaml
# .github/workflows/deploy-backend.yml
name: Deploy Backend
on:
  push:
    branches: [main]
    paths:
      - "backend/**"
      - ".github/workflows/deploy-backend.yml"

jobs:
  deploy:
    uses: naoki1510/ci-workflows/.github/workflows/kamal-deploy.yml@v1
    with:
      working-directory: backend
    secrets:
      ssh-private-key: ${{ secrets.VPS_SSH_KEY }}
      ssh-known-hosts: ${{ secrets.VPS_SSH_KNOWN_HOSTS }}
      env-vars: |
        RAILS_MASTER_KEY=${{ secrets.RAILS_MASTER_KEY }}
        DATABASE_PASSWORD=${{ secrets.DATABASE_PASSWORD }}
```

`env-vars` の各行は `KEY=value` 形式。`#` で始まる行と空行は無視される。caller 側で `${{ secrets.* }}` を expression として展開する形なので、reusable workflow 側で個別 secret を宣言する必要がない。

## Versioning

- `v1` tag is the stable line. Backward-compatible changes update `v1` (force-push the tag).
- Breaking changes bump to `v2`, leaving `v1` callers untouched.
- For exact pinning, use a commit SHA instead of `v1`.
