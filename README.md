# dependency_review_action_test
GitHub Dependency Review Action の事前検証用リポジトリ。

## 構成
- `apps/backend/` … Ruby on Rails（Ruby 3.3 / Rails 7.2 / MySQL 8.4）
- `apps/frontend/` … React + TypeScript（Vite）
- `docker-compose.yml` … backend / frontend / db(MySQL) をまとめて起動

ローカルに Ruby / Node をインストールせず、すべて Docker 上で動作します。

## 必要なもの
- Docker Desktop（Docker Engine + Docker Compose v2）

## 起動手順
```bash
# 初回ビルド
docker compose build

# DB 作成 / マイグレーション（初回のみ）
docker compose run --rm backend bin/rails db:create db:migrate

# 起動
docker compose up
```

- backend: http://localhost:3000
- frontend: http://localhost:5173
- db (MySQL): localhost:13306（ホスト 3306 との競合回避のため）

## 関連ドキュメント
- [docs/plan.md](./docs/plan.md) - 事前検証の計画書
- [docs/tasks.md](./docs/tasks.md) - タスクリスト
