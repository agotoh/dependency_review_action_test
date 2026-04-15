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
- [x] `.github/workflows/dependency-review.yml` 作成（pull_request トリガー）
- [x] 設定オプション検討（`fail-on-severity: moderate`, `comment-summary-in-pr: always`）
- [x] 動作確認用の作業ブランチを作成（`verify/safe-dep-add`）

## Phase 3: 動作確認（正常系）
- [x] 無害な依存追加 PR を作成（backend: rack-attack / frontend: clsx、PR #1）
- [x] Action が成功することを確認（初回は Dependency graph 無効で失敗 → Dependabot security updates を有効化して再実行で pass）
- [x] PR URL を記録（レポートにはスクショではなく PR リンクを記載する方針）
  - 正常系 PR: https://github.com/agotoh/dependency_review_action_test/pull/1

### メモ
- 新規 public リポジトリでも Dependency Review Action 実行に必要な "Dependency graph" は自動有効化されない場合がある
- `gh api -X PUT repos/{owner}/{repo}/vulnerability-alerts` または Settings → Code security で有効化が必要

## Phase 4: 動作確認（異常系）
- [x] 既知脆弱性のある Ruby gem を選定（`rubyzip 1.2.2` / CVE-2018-1000544 Zip Slip、GHSA-5m2v-hc64-56h6）
  - ※当初は `nokogiri 1.13.9` を検討したが Rails 7.2 の依存要件と衝突したため変更
- [x] 該当 gem を `Gemfile` に追加した PR を作成（PR #2）
- [x] Action が検知・ブロックすることを確認（moderate severity で fail）
- [x] 既知脆弱性のある npm package を選定（`lodash 4.17.15` / CVE-2019-10744, CVE-2020-8203）
- [x] 該当 package を `package.json` に追加した PR を作成（PR #2 に同梱）
- [x] Action が検知・ブロックすることを確認（rubyzip / lodash 双方検知、fail）
- [x] 各 PR URL を記録
  - 異常系 PR: https://github.com/agotoh/dependency_review_action_test/pull/2
  - Actions run: https://github.com/agotoh/dependency_review_action_test/actions/runs/24439778731

### 検知結果（Job Summary より）
**apps/backend/Gemfile.lock**
- `rubyzip 1.2.2` — Rubyzip denial of service（moderate）

**apps/frontend/package-lock.json**
- `lodash 4.17.15` — 複数の Advisory を検知
  - Command Injection in lodash（high）
  - Prototype Pollution in lodash（high）
  - Code Injection via `_.template` imports key names（high）
  - Regular Expression Denial of Service (ReDoS)（moderate）
  - Prototype Pollution in `_.unset` / `_.omit`（moderate、2件）

→ 合計 2 packages / 7 advisories を検知し、設定通り `fail-on-severity: moderate` で PR をブロック。
OpenSSF Scorecard の警告も 25 件表示されており、依存の健全性評価も併せて確認可能。

## Phase 5: 代替ツール軽検証（合計 1 人日目安。超過時は机上比較へ切替）
### bundler-audit
- [x] GitHub Actions に bundler-audit ジョブ追加（`.github/workflows/bundler-audit.yml`）
- [x] 脆弱 gem を含む PR で検知できるか確認
  - rubyzip 1.2.2 / CVE-2019-16892 / Medium を検知 → job fail
- [x] 結果 Actions run URL を記録
  - https://github.com/agotoh/dependency_review_action_test/actions/runs/24440305618

### Trivy
- [x] GitHub Actions に Trivy fs スキャンジョブ追加（`.github/workflows/trivy.yml`、`aquasecurity/trivy-action@master`）
- [x] Ruby / Node 双方の脆弱性を検知できるか確認
  - rubyzip 1.2.2: 1 件（MEDIUM, CVE-2019-16892）
  - lodash 4.17.15: 7 件（HIGH × 4 / MEDIUM × 3、CVE-2020-8203 / CVE-2021-23337 等）
- [x] 結果 Actions run URL を記録
  - https://github.com/agotoh/dependency_review_action_test/actions/runs/24440305626

### Dependabot alerts
- [x] リポジトリ設定で Dependabot alerts を有効化（Phase 3 で実施済み）
- [x] 既存の脆弱依存に対してアラートが発生するか確認
  - **Dependabot は default branch（main）のみスキャン**するため、feature branch 上の rubyzip / lodash は alert 対象外
  - main に検出されたのは `aquasecurity/trivy-action` のサプライチェーン侵害 alert 1 件（fixed 状態）
- [ ] ~~Dependabot security updates の PR 自動作成挙動を確認~~（main に脆弱依存が無いため PR 未発生）
- [x] 結果 URL を記録
  - Dependabot alerts 画面: https://github.com/agotoh/dependency_review_action_test/security/dependabot

### Phase 5 サマリ（3ツール比較）
| ツール | 検知タイミング | rubyzip | lodash | カバー言語 | PR ブロック |
|---|---|---|---|---|---|
| Dependency Review Action | PR 時点 | ✅ 1件 | ✅ 6件 | 多言語 | ✅ |
| bundler-audit | PR 時点 | ✅ 1件 | ❌（Ruby 専用） | Ruby のみ | ✅ |
| Trivy (fs) | PR 時点 | ✅ 1件 | ✅ 7件 | 多言語 | ✅ |
| Dependabot alerts | push 後（default branch のみ） | N/A | N/A | 多言語 | ❌（検知のみ） |

- **Dependency Review Action と Trivy はカバレッジ・PR ブロック機能の両面で近い**。Trivy はさらに詳細（CVE 番号・fix version 列）で、lodash は 1 件多く検知（GitHub Advisory と Trivy DB のソース差）
- **bundler-audit** は Ruby 単機能だが導入が容易。Ruby 単体プロジェクトなら十分
- **Dependabot alerts** は PR ゲートにはならず「マージ後の継続監視」用途

## Phase 6: 比較・レポート作成
- [ ] `docs/report.md` 作成
- [ ] 比較表（導入難易度 / 検知力 / PR UX / ノイズ / 費用）を記載
- [ ] **各検証 PR の URL をレポートに記載**（スクショは取らず、PR リンクで参照）
  - 正常系 PR: https://github.com/agotoh/dependency_review_action_test/pull/1
  - 異常系 PR（Ruby）: （Phase 4 で取得）
  - 異常系 PR（npm）: （Phase 4 で取得）
  - bundler-audit / Trivy / Dependabot 検証 PR: （Phase 5 で取得）
- [ ] Dependency Review Action を本番導入するかの所感・推奨を記載
- [ ] main にマージしチーム共有

## Phase 7: 振り返り
- [ ] 想定と実際の差分メモ
- [ ] 本番プロジェクト導入時の注意点を列挙
