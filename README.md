# ci-workflows

Reusable GitHub Actions composite actions shared across naoki1510 projects.

v2 は composite action 100% 構成。複数 step を 1 つの runner 内で連続実行できるので setup の重複が無く、caller が job 構造を自由に組み立てられる。

## Versioning

- **`v2`** (current) — composite action 100%。reusable workflow は無し。
- **`v1`** (legacy) — `rails-ci.yml` / `vite-ci.yml` / `kamal-deploy.yml` の reusable workflow セット。タグとして残しているので既存 caller は `@v1` でそのまま動く。新規 caller は v2 を使う。
- 互換更新は対応する `vN` tag を force-push。breaking change は新メジャー bump。
- 厳密にピン留めするなら commit SHA を使う。

## Available composite actions

### Setup

| Action | 内容 |
|---|---|
| [`actions/setup-rails`](.github/actions/setup-rails/action.yml) | system packages (任意) + Ruby + bundler-cache |
| [`actions/setup-pnpm-node`](.github/actions/setup-pnpm-node/action.yml) | pnpm + Node + 各キャッシュ + `pnpm install --frozen-lockfile` |

### Rails

| Action | 内容 |
|---|---|
| [`actions/rails-lint`](.github/actions/rails-lint/action.yml) | RuboCop → Brakeman (rubocop fail でも brakeman は走る) |
| [`actions/rails-test`](.github/actions/rails-test/action.yml) | `db:test:prepare` → RSpec (caller が DB service container を提供) |
| [`actions/check-swagger`](.github/actions/check-swagger/action.yml) | rswag で swagger.yaml 再生成 → `git diff --exit-code` で最新性検証 |

### Frontend (pnpm)

| Action | 内容 |
|---|---|
| [`actions/vite-lint`](.github/actions/vite-lint/action.yml) | ESLint → Prettier check (eslint fail でも prettier は走る) |
| [`actions/vite-build`](.github/actions/vite-build/action.yml) | typecheck → test → build |
| [`actions/check-codegen`](.github/actions/check-codegen/action.yml) | frontend client (例: @hey-api/openapi-ts) を再生成 → `git diff --exit-code` で最新性検証 |

### Deploy

| Action | 内容 |
|---|---|
| [`actions/kamal-deploy`](.github/actions/kamal-deploy/action.yml) | SSH 設定 + (任意) Docker buildx + `bin/kamal deploy` 実行 |

## Conventions

- Ruby version は `ruby/setup-ruby` が `.ruby-version` / `.tool-versions` / `mise.toml` から自動検出
- Node version は repo root の `.node-version` から読む
- pnpm version は repo root の `package.json` の `packageManager` から読む。`packageManager` が無い repo は `setup-pnpm-node` の `pnpm-version` input で指定する
- Frontend は **pnpm** 前提 (npm は未対応)
- Action versions: `actions/checkout@v6`, `actions/setup-node@v6`, `pnpm/action-setup@v4`, `ruby/setup-ruby@v1`, `docker/setup-buildx-action@v4`
- `actions/rails-test` / `actions/check-swagger` の `*-cmd` 系 input はシェルコマンド文字列を `eval` する。caller が trusted な前提
- DB service container は caller の job 側で定義する (composite action は service を持てない)
- `actions/kamal-deploy` 等で kamal に環境変数を渡したいときは、caller の **job-level** `env:` ブロックに書く (composite 内に env 転送機構は無く、step-level env の伝播は保証されない)

## Caller settings (推奨)

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

同じブランチへの連続 push で古い run が自動キャンセルされ、CI 時間とコストを削減できる。

## Usage

### Rails + frontend を 2 job で完結させる例

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  rails:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: app_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd="pg_isready"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    env:
      RAILS_ENV: test
      POSTGRES_HOST: localhost
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    steps:
      - uses: actions/checkout@v6
      - uses: naoki1510/ci-workflows/.github/actions/setup-rails@v2
        with:
          working-directory: backend
          install-system-packages: build-essential git libpq-dev libyaml-dev pkg-config
      - uses: naoki1510/ci-workflows/.github/actions/rails-lint@v2
        with: { working-directory: backend }
      - uses: naoki1510/ci-workflows/.github/actions/rails-test@v2
        with: { working-directory: backend }
      - uses: naoki1510/ci-workflows/.github/actions/check-swagger@v2
        with:
          working-directory: backend
          swagger-path: backend/swagger/v1/swagger.yaml

  vite:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: naoki1510/ci-workflows/.github/actions/setup-pnpm-node@v2
      - uses: naoki1510/ci-workflows/.github/actions/vite-lint@v2
        with: { pnpm-filter: frontend }
      - uses: naoki1510/ci-workflows/.github/actions/vite-build@v2
        with: { pnpm-filter: frontend }
      - uses: naoki1510/ci-workflows/.github/actions/check-codegen@v2
        with:
          pnpm-filter: frontend
          generated-code-path: frontend/src/lib/api/generated/
```

2 runners (rails, vite) のみ。各 setup は 1 回ずつ。

### Frontend のみのリポジトリ

```yaml
jobs:
  vite:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: naoki1510/ci-workflows/.github/actions/setup-pnpm-node@v2
      - uses: naoki1510/ci-workflows/.github/actions/vite-lint@v2
      - uses: naoki1510/ci-workflows/.github/actions/vite-build@v2
```

### lint と test を別 job に分けたい場合

並列実行や Required Status Check の粒度を細かくしたい場合、caller 側で job を分ければよい。

```yaml
jobs:
  rails-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: naoki1510/ci-workflows/.github/actions/setup-rails@v2
        with: { working-directory: backend, install-system-packages: ... }
      - uses: naoki1510/ci-workflows/.github/actions/rails-lint@v2
        with: { working-directory: backend }

  rails-test:
    runs-on: ubuntu-latest
    services: { postgres: ... }
    env: { RAILS_ENV: test, ... }
    steps:
      - uses: actions/checkout@v6
      - uses: naoki1510/ci-workflows/.github/actions/setup-rails@v2
        with: { working-directory: backend, install-system-packages: ... }
      - uses: naoki1510/ci-workflows/.github/actions/rails-test@v2
        with: { working-directory: backend }
```

job を分けると setup-rails は別 runner で 2 回走るが、bundler-cache のおかげで 2 回目は早い。

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
    runs-on: ubuntu-latest
    # kamal 実行時に必要な env は job-level で定義 (composite 内の step に確実に伝播する)
    env:
      RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
      DATABASE_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
    steps:
      - uses: actions/checkout@v6
      - uses: naoki1510/ci-workflows/.github/actions/kamal-deploy@v2
        with:
          working-directory: backend
          ssh-private-key: ${{ secrets.VPS_SSH_KEY }}
          ssh-known-hosts: ${{ secrets.VPS_SSH_KNOWN_HOSTS }}
```

env は job-level の `env:` ブロックに書く。job 内の全 step に伝播するため、composite 内の `bin/kamal deploy` も同じ env を受け取る。v1 にあった `env-vars` (マルチライン KEY=value を `GITHUB_ENV` に転送する仕組み) は廃止。env injection の入り口を構造的に塞いだ。
