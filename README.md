# ci-workflows

Reusable GitHub Actions workflows + composite actions shared across naoki1510 projects.

## Two layers of reuse

| Layer | Where | Use case |
|---|---|---|
| **Composite actions** (`.github/actions/`) | run in the **caller's** runner | setup を共有しつつ caller 側 job に自由に step を足したい時 |
| **Reusable workflows** (`.github/workflows/`) | each call spins up its **own** runner | 完成された job を丸ごと取り込みたい時 (lint / test / build 等) |

caller 側で「rspec の後ろに独自 step を足す」のような自由な組み合わせが必要なら **composite action**、`uses:` 一発で job 1 個分のお膳立てが欲しいなら **reusable workflow** を使う。

## Versioning

- **`v2`** (current) — job 単位の reusable workflow に分割 + composite action でセットアップ共通化。後述。
- **`v1`** (legacy) — `rails-ci.yml` / `vite-ci.yml` で 2 job バンドル。`v2` の `rails-lint` + `rails-test` 等に分割された。caller の移行が完了するまで `v1` も維持する。
- `v1` の互換更新は force-push で `v1` を更新する。breaking change は次のメジャーへ bump。
- 厳密にピン留めするなら commit SHA を使う。

## Available composite actions (v2)

| Action | Purpose |
|---|---|
| [`actions/setup-rails`](.github/actions/setup-rails/action.yml) | System packages (optional) + `ruby/setup-ruby` (bundler-cache 有効)。caller は事前に `actions/checkout` する必要あり。DB は含まない (caller の job 側で service container を立てる) |
| [`actions/setup-pnpm-node`](.github/actions/setup-pnpm-node/action.yml) | `pnpm/action-setup` + `actions/setup-node` (pnpm キャッシュ) + node_modules キャッシュ + `pnpm install --frozen-lockfile` |

## Available reusable workflows

### v2 (job 単位)

| Workflow | Job 構成 |
|---|---|
| [`rails-lint.yml`](.github/workflows/rails-lint.yml) | RuboCop + Brakeman |
| [`rails-test.yml`](.github/workflows/rails-test.yml) | RSpec (postgres / mysql service container 切替) |
| [`vite-lint.yml`](.github/workflows/vite-lint.yml) | ESLint + Prettier |
| [`vite-build.yml`](.github/workflows/vite-build.yml) | Typecheck + Vitest + Vite build |
| [`check-generated-code.yml`](.github/workflows/check-generated-code.yml) | rswag swaggerize + hey-api codegen の最新性検証 (rails + rspec + @hey-api/openapi-ts 専用) |

### v1 (legacy, 2 job バンドル)

| Workflow | Purpose |
|---|---|
| [`rails-ci.yml`](.github/workflows/rails-ci.yml) | `rails-lint` + `rails-test` を 1 workflow に同梱 |
| [`vite-ci.yml`](.github/workflows/vite-ci.yml) | `vite-lint` + `vite-build` を 1 workflow に同梱 |

### 共通

| Workflow | Purpose |
|---|---|
| [`kamal-deploy.yml`](.github/workflows/kamal-deploy.yml) | Deploy a Rails app to a VPS via Kamal |

## Conventions enforced by these workflows

- Ruby version is auto-detected by `ruby/setup-ruby` from `.ruby-version` / `.tool-versions` / `mise.toml` under the working directory.
- Node version is read from `.node-version` at the repo root.
- Frontend uses **pnpm** (no npm support — npm-based projects must migrate first).
- Action versions: `actions/checkout@v6`, `actions/setup-node@v6`, `pnpm/action-setup@v4`, `ruby/setup-ruby@v1`, `docker/setup-buildx-action@v4`.
- `rails-test.yml` / `check-generated-code.yml` の service container は `db-engine` に応じて未使用側の image が空文字列になる。GitHub Actions runner は空 image のコンテナをスキップする (undocumented だが実動作で確認済み)。将来 runner の挙動が変わった場合は db-engine 別の workflow 分割を検討する。
- `rails-test.yml` / `check-generated-code.yml` の `*-cmd` 系 input (`test-cmd`, `test-prepare-cmd`, `swaggerize-cmd` 等) はシェルコマンド文字列を受け、env 経由で `eval` する。caller は信頼できる前提だが、untrusted な caller を想定する場合はコマンド固定 + 引数 input への refactor が必要。

## Recommended caller settings

caller 側で `concurrency` を設定すると、同じブランチへの連続 push で古い run が自動キャンセルされ、CI 時間とコストを削減できる。

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## Usage (v2)

### Rails (lint + test を別 job として呼ぶ)

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  rails-lint:
    uses: naoki1510/ci-workflows/.github/workflows/rails-lint.yml@v2
    with:
      working-directory: backend
      install-system-packages: build-essential git libpq-dev libyaml-dev pkg-config

  rails-test:
    uses: naoki1510/ci-workflows/.github/workflows/rails-test.yml@v2
    with:
      working-directory: backend
      db-engine: postgres
      install-system-packages: build-essential git libpq-dev libyaml-dev pkg-config
```

### Vite (lint + build を別 job として呼ぶ)

```yaml
jobs:
  vite-lint:
    uses: naoki1510/ci-workflows/.github/workflows/vite-lint.yml@v2
    with:
      pnpm-filter: frontend

  vite-build:
    uses: naoki1510/ci-workflows/.github/workflows/vite-build.yml@v2
    with:
      pnpm-filter: frontend
```

### Check generated code (rails + rspec + hey-api 専用)

`rswag` で `swagger.yaml` を生成し、`@hey-api/openapi-ts` で TypeScript クライアントを生成しているリポジトリ向け。1 job 内で連続実行することで、ruby setup と pnpm setup の重複を避けて runner cost を抑える。

```yaml
jobs:
  check-generated-code:
    uses: naoki1510/ci-workflows/.github/workflows/check-generated-code.yml@v2
    with:
      rails-working-directory: backend
      install-system-packages: build-essential git libpq-dev libyaml-dev pkg-config
      db-engine: postgres
      swagger-path: backend/swagger/v1/swagger.yaml
      pnpm-filter: frontend
      generated-code-path: frontend/src/lib/api/generated/
```

### 独自 job で composite action を使う (setup を共有して step を自由に追加)

reusable workflow では足りない処理を caller の独自 job で書くケース。`setup-rails` / `setup-pnpm-node` を呼べば既存 reusable workflow と同じ setup を使い回せる。

```yaml
jobs:
  custom-rails-task:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    services:
      postgres:
        image: postgres:17
        env: { POSTGRES_USER: postgres, POSTGRES_PASSWORD: postgres }
        ports: ["5432:5432"]
        options: >-
          --health-cmd="pg_isready" --health-interval=10s --health-timeout=5s --health-retries=5
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
      - run: bin/rails db:test:prepare
      - run: bundle exec rake my:custom:task
```

## Usage (v1, legacy)

### Rails CI (PostgreSQL example)

```yaml
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

### Vite CI

```yaml
jobs:
  vite:
    uses: naoki1510/ci-workflows/.github/workflows/vite-ci.yml@v1
    with:
      pnpm-filter: frontend
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
