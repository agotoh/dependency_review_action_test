# Dependency Review Action 事前検証 計画書

## 1. 目的
- GitHub の Dependency Review Action を PR に導入し、新規依存に含まれる既知脆弱性を自動検知・ブロックする運用をチームで評価する。
- 本番導入（GitHub Code Security: $30/月/アクティブコミッター）の費用対効果を判断するため、個人アカウントの public リポジトリで動作イメージを共有する。
- あわせて無料の代替ツール（bundler-audit / Trivy / Dependabot alerts）を軽く試し、比較材料を集める。

## 2. スコープ
- **対象リポジトリ**: `dependency_review_action_test`（public 化済み）
- **構成**: モノレポ + Docker 環境
  - `apps/backend/` … Ruby on Rails + MySQL（最小の CRUD が動く簡易アプリ）
  - `apps/frontend/` … Node + TypeScript + React（最小の画面が動く簡易アプリ）
  - `docker-compose.yml` でまとめて起動（backend / frontend / db(MySQL)）
  - ネイティブに Ruby / Node をインストールせず、すべて Docker 上で動作させる
  - `reference-app` の構成を参考にしつつ、企業固有情報は持ち込まず一般的な内容にする。
- **検証の主役**: Dependency Review Action の導入と動作確認。
- **サブ検証（合計 1 人日程度で収まる範囲）**: bundler-audit / Trivy / Dependabot alerts の導入と動作確認。収まらない場合は机上比較に切り替える。
- **期間**: 2 営業日以内。

## 3. 非スコープ
- 本番プロジェクトへの Dependency Review Action 本導入作業。
- 企業固有の設定・秘匿情報の反映。
- 高度なアプリ機能の実装（検証に必要な最小機能のみ）。

## 4. 進め方の全体像
1. **環境準備** — モノレポの骨格を作り、Rails / React の最小動作アプリを配置する。
2. **Dependency Review Action 導入** — `.github/workflows/dependency-review.yml` を作成し PR で発火させる。
3. **動作確認（正常系）** — 脆弱性のない PR で Action が pass することを確認。
4. **動作確認（異常系）** — 既知脆弱性のあるパッケージ（Ruby gem / npm package）を追加した PR を立て、Action が検知・ブロックすることを確認。
   - 使用する脆弱パッケージは検証時に選定（例: 古い `nokogiri`、`lodash@4.17.15` など既知 CVE のあるもの）。
5. **代替ツール軽検証** — 以下を時間枠内で導入・実行。
   - bundler-audit（Ruby/Gemfile 向け、GitHub Actions で実行）
   - Trivy（ファイルシステム/依存スキャン）
   - Dependabot alerts（リポジトリ設定で有効化）
6. **比較と共有レポート作成** — 各ツールの検知能力・セットアップ負荷・ノイズ・費用を表にまとめ、チーム共有用 Markdown を `docs/` に作成。

## 5. 検証観点（レポートに含める項目）
- 導入ステップ数・難易度
- 検知できた脆弱性 / 見逃した脆弱性
- PR 上の UX（コメント・差分表示・ブロック挙動）
- 誤検知・ノイズ量
- 費用（public/private での違いを含む）
- 本番プロジェクトへの適用可否の所感

## 6. 成果物
- `docs/plan.md`（本書）
- `docs/tasks.md`（チェックボックス形式の実行タスク）
- `backend/` / `frontend/` のサンプルアプリ
- `.github/workflows/*.yml`（Dependency Review Action ほか）
- `docs/report.md`（検証結果レポート・比較表・スクショ）

## 7. リスク / 留意点
- **時間超過**: 代替ツールの導入で時間を使いすぎた場合、途中で切り上げて机上比較に切り替える。
- **脆弱性パッケージ選定**: 検知される既知 CVE を持つバージョンの選定に試行錯誤が発生する可能性あり。
- **public リポジトリ公開情報**: 企業情報や内部構成を反映させない点に注意。
