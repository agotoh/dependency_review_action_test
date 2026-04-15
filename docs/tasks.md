# タスクリスト

計画書: [plan.md](./plan.md)
進捗は `[ ]` → `[x]` で消し込むこと。

## Phase 1: 環境準備（Docker ベース）
- [x] モノレポの骨格作成（`apps/backend/` `apps/frontend/` ディレクトリ、ルート README 更新）
- [x] `.gitignore` 整備（Ruby / Node / Docker 対応）
- [x] `apps/backend/Dockerfile.dev` 作成（Ruby + Rails + MySQL クライアント）
- [x] `apps/frontend/Dockerfile.dev` 作成（Node + TypeScript + React）
- [x] ルートに `docker-compose.yml` 作成（backend / frontend / db(MySQL)）
- [x] `apps/backend/` に Rails 最小アプリを作成（Docker 経由で `rails new`）
- [x] Rails に簡単な CRUD リソースを 1 つ追加（scaffold: Post）
- [x] `docker compose up` で backend の起動確認（http://localhost:3000/posts → 200）
- [x] `apps/frontend/` に React + TypeScript 最小アプリを作成（Docker 経由で Vite 雛形）
- [x] `docker compose up` で frontend の起動確認（http://localhost:5173 → 200）
- [x] backend / frontend / db がまとめて起動することを確認
- [x] README に Docker での起動手順を記載
- [x] 初期状態を main に push（PR ベース検証の土台）

## Phase 2: Dependency Review Action 導入
- [ ] `.github/workflows/dependency-review.yml` 作成（pull_request トリガー）
- [ ] 設定オプション検討（`fail-on-severity`, `comment-summary-in-pr` など）
- [ ] 動作確認用の作業ブランチを作成

## Phase 3: 動作確認（正常系）
- [ ] 無害な依存追加 PR を作成（例: よく使われる安全なバージョンの gem / npm package）
- [ ] Action が成功することを確認
- [ ] PR 上の表示（サマリコメント等）をスクショ取得

## Phase 4: 動作確認（異常系）
- [ ] 既知脆弱性のある Ruby gem を選定（例: 古い nokogiri 等）
- [ ] 該当 gem を `Gemfile` に追加した PR を作成
- [ ] Action が検知・ブロックすることを確認
- [ ] 既知脆弱性のある npm package を選定（例: `lodash@4.17.15` 等）
- [ ] 該当 package を `package.json` に追加した PR を作成
- [ ] Action が検知・ブロックすることを確認
- [ ] 各 PR のスクショ・ログ取得

## Phase 5: 代替ツール軽検証（合計 1 人日目安。超過時は机上比較へ切替）
### bundler-audit
- [ ] GitHub Actions に bundler-audit ジョブ追加
- [ ] 脆弱 gem を含む PR で検知できるか確認
- [ ] 結果をスクショ/ログで記録

### Trivy
- [ ] GitHub Actions に Trivy fs スキャンジョブ追加
- [ ] Ruby / Node 双方の脆弱性を検知できるか確認
- [ ] 結果をスクショ/ログで記録

### Dependabot alerts
- [ ] リポジトリ設定で Dependabot alerts を有効化
- [ ] 既存の脆弱依存に対してアラートが発生するか確認
- [ ] Dependabot security updates の PR 自動作成挙動を確認
- [ ] 結果をスクショで記録

## Phase 6: 比較・レポート作成
- [ ] `docs/report.md` 作成
- [ ] 比較表（導入難易度 / 検知力 / PR UX / ノイズ / 費用）を記載
- [ ] 各ツールのスクショを添付
- [ ] Dependency Review Action を本番導入するかの所感・推奨を記載
- [ ] main にマージしチーム共有

## Phase 7: 振り返り
- [ ] 想定と実際の差分メモ
- [ ] 本番プロジェクト導入時の注意点を列挙
