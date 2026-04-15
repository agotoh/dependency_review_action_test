# Dependency Review Action 事前検証レポート

- **検証期間**: 2026-04-15
- **検証リポジトリ**: https://github.com/agotoh/dependency_review_action_test（public）
- **対象構成**: モノレポ（`apps/backend`: Rails 7.2 + MySQL 8.4、`apps/frontend`: React + TypeScript + Vite、Docker 環境）
- **関連ドキュメント**: [plan.md](./plan.md) / [tasks.md](./tasks.md)

## 1. 目的
PR で新規に追加される依存パッケージに既知脆弱性が含まれる場合、マージ前に自動検知してブロックする仕組みを評価する。`reference-app` と同等の Rails + React プロジェクトへの導入を前提に、GitHub Dependency Review Action が本番導入に値するか、および無料代替ツール（bundler-audit / Trivy / Dependabot alerts）との比較情報を揃える。

## 2. 検証観点
1. PR に脆弱依存が**含まれない**ときに正しく pass するか（正常系）
2. PR に脆弱依存が**含まれる**ときに fail してマージをブロックできるか（異常系）
3. 検知カバレッジ（Ruby / npm 双方、severity、CVE 数）
4. 導入・運用コスト（セットアップ手順、費用、ノイズ、UX）
5. 無料代替ツールとの差分

## 3. 検証 1: 正常系（Dependency Review Action）
- PR: [#1 Add safe deps (rack-attack / clsx)](https://github.com/agotoh/dependency_review_action_test/pull/1)
- 追加依存:
  - backend: `rack-attack`
  - frontend: `clsx`
- **結果**: Action **pass**（27秒）、PR コメントで追加依存のサマリが表示される

### ハマりどころ
初回実行時は `Dependency review is not supported on this repository. Please ensure that Dependency graph is enabled` で fail。新規 public リポジトリでも Dependency graph が有効化されていないケースがあり、**Settings → Code security** から Dependency graph を有効化する必要があった。本番導入手順に含めるべき前提条件。

## 4. 検証 2: 異常系（Dependency Review Action）
- PR: [#2 Add vulnerable deps (rubyzip 1.2.2 / lodash 4.17.15)](https://github.com/agotoh/dependency_review_action_test/pull/2)
- 追加依存（意図的な脆弱性パッケージ）:
  - backend: `rubyzip 1.2.2`（CVE-2018-1000544 "Zip Slip" / CVE-2019-16892）
  - frontend: `lodash 4.17.15`（CVE-2019-10744 / CVE-2020-8203 ほか）
- **結果**: Action **fail**、PR ブロック

### 検知内容（Job Summary）
**apps/backend/Gemfile.lock**
| Package | Severity | Advisory |
|---|---|---|
| rubyzip 1.2.2 | moderate | Rubyzip denial of service |

**apps/frontend/package-lock.json**
| Package | Severity | Advisory |
|---|---|---|
| lodash 4.17.15 | high | Command Injection in lodash |
| lodash 4.17.15 | high | Prototype Pollution in lodash |
| lodash 4.17.15 | high | Code Injection via `_.template` imports key names |
| lodash 4.17.15 | moderate | ReDoS in lodash |
| lodash 4.17.15 | moderate | Prototype Pollution in `_.unset` / `_.omit`（2 件） |

合計 **2 packages / 7 advisories** を検知し、`fail-on-severity: moderate` 設定により PR をブロック。

## 5. 代替ツール検証

同じ PR #2 に対して以下のワークフローを並列実行して比較。

### bundler-audit
- ワークフロー: [`.github/workflows/bundler-audit.yml`](../.github/workflows/bundler-audit.yml)
- Actions run: https://github.com/agotoh/dependency_review_action_test/actions/runs/24440305618
- 結果: rubyzip 1.2.2 / CVE-2019-16892 / Medium を検知 → fail
- **スコープは Ruby（Gemfile.lock）のみ**。lodash は当然対象外

### Trivy
- ワークフロー: [`.github/workflows/trivy.yml`](../.github/workflows/trivy.yml)（`aquasecurity/trivy-action@master`）
- Actions run: https://github.com/agotoh/dependency_review_action_test/actions/runs/24440305626
- 結果:
  - rubyzip: 1 件（MEDIUM, CVE-2019-16892）
  - lodash: 7 件（HIGH × 4 / MEDIUM × 3）
- CVE 番号・Fixed Version の列まで詳細表示

### Dependabot alerts
- 事前検証時に Settings から有効化（Phase 3）
- **default branch（main）のみスキャン**するため、feature branch 上の rubyzip / lodash は alert 対象外
- main 上で別途 `aquasecurity/trivy-action` のサプライチェーン alert（fixed）が 1 件検知された → Actions の健全性監視にも有用
- Dependabot alerts 画面: https://github.com/agotoh/dependency_review_action_test/security/dependabot

## 6. 4ツール比較

| ツール | 検知タイミング | rubyzip | lodash | カバー言語 | PR ブロック | 強み | 弱み |
|---|---|---|---|---|---|---|---|
| **Dependency Review Action** | PR 時点 | ✅ 1 | ✅ 6 | 多言語 | ✅ | GitHub 標準、PR コメント UX 良好、OpenSSF Scorecard 連携、license チェック | private repo で有料（Code Security $30/月/committer） |
| **bundler-audit** | PR 時点 | ✅ 1 | ❌ 対象外 | Ruby のみ | ✅ | 無料、導入が最も容易、ruby-advisory-db 即時反映 | Ruby 専用、多言語モノレポでは不十分 |
| **Trivy (fs)** | PR 時点 | ✅ 1 | ✅ 7 | 多言語 | ✅ | 無料、最広カバー、CVE 詳細情報豊富、コンテナ/IaC もスキャン可能 | 出力ノイズが多め、初期設定と severity 調整が必要 |
| **Dependabot alerts** | push 後（default branch のみ） | N/A | N/A | 多言語 | ❌ | 無料、継続監視・自動 PR 提案（security updates） | PR ゲートにならない、feature branch 未対応 |

### カバレッジ差について
Dependency Review Action（GitHub Advisory Database）と Trivy（Trivy DB、GitHub Advisory + NVD + ディストリビュータ情報）でソースが異なるため、lodash の検知件数に 1 件の差が出た。どちらも実務上の保護性能は同等レベルだが、**Trivy のほうが網羅性で若干上回る**。

## 7. 費用とライセンス

| ツール | public repo | private repo |
|---|---|---|
| Dependency Review Action | **無料** | GitHub Advanced Security / **Code Security $30/月/アクティブコミッター** |
| bundler-audit | 無料 | 無料 |
| Trivy | 無料 | 無料 |
| Dependabot alerts | 無料 | 無料 |

`reference-app` は private repo のため、Dependency Review Action を単独で導入する場合は全アクティブコミッター分のライセンスが必要。

## 8. 推奨

### 結論
`reference-app` 本番導入にあたっては、以下の **2 案を提示**する。

#### 案 A: Dependency Review Action + Dependabot alerts（有償導入）
- **強み**: GitHub 標準連携、PR コメント UX、license チェック、Scorecard 可視化、運用負荷が最も軽い
- **コスト**: アクティブコミッター数 × $30/月
- **推奨ケース**: セキュリティ運用負荷を最小化したい、license チェックも必要、コミッター数が少ない

#### 案 B: Trivy + bundler-audit + Dependabot alerts（無償で代替）
- **強み**: コスト 0、検知カバレッジは Dependency Review Action と同等以上（Trivy 網羅性）、コンテナ/IaC も将来スキャン可能
- **弱み**: PR コメントの UX は Dependency Review Action より見劣りする、ワークフロー維持の運用が必要、license チェックは別途要対応
- **推奨ケース**: コスト優先、既に Trivy を他用途でも使っている、多言語モノレポ

### 筆者（検証実施者）のおすすめ
**案 B（Trivy + bundler-audit + Dependabot alerts）** を推奨する。主な理由:
1. **検知性能面で Dependency Review Action に劣らず、むしろ lodash で 1 件多く検知**した
2. コミッター数が一定以上になると年間コストが無視できない
3. Trivy はコンテナイメージ・IaC・シークレットも同ツールでカバー可能で将来の拡張性が高い
4. GitHub Advanced Security の license 機能が必要になったタイミングで案 A に昇格する余地は常にある

ただし、案 A の **PR 上 UX（サマリコメント・Scorecard 情報）の運用体験は実際に良い**ため、セキュリティ運用に慣れていないチームや、コスト許容余地があるチームには案 A を強くおすすめする。

## 9. 本番導入時の注意点

1. **Dependency graph の事前有効化**が必要（新規リポジトリで自動有効化されない場合あり）
2. **`fail-on-severity` は moderate が妥当**（今回の検証で rubyzip が moderate だったため low に下げると過剰、high にすると moderate 脆弱性を見逃す）
3. **`comment-summary-in-pr: always`** を推奨（PR レビュー時の確認性向上）
4. Trivy を使う場合、**`aquasecurity/trivy-action` のバージョンピン**に注意（本検証では `@0.28.0` が存在せず `@master` で回避したが、本番では SHA ピンを推奨）
5. Dependabot alerts の **アラート通知先を明確化**（Slack 連携等）

## 10. 参考 URL

- 正常系 PR（safe deps）: https://github.com/agotoh/dependency_review_action_test/pull/1
- 異常系 PR（vulnerable deps）: https://github.com/agotoh/dependency_review_action_test/pull/2
- bundler-audit Actions run: https://github.com/agotoh/dependency_review_action_test/actions/runs/24440305618
- Trivy Actions run: https://github.com/agotoh/dependency_review_action_test/actions/runs/24440305626
- Dependabot alerts: https://github.com/agotoh/dependency_review_action_test/security/dependabot
- GitHub Dependency Review Action: https://github.com/actions/dependency-review-action
- bundler-audit: https://github.com/rubysec/bundler-audit
- Trivy Action: https://github.com/aquasecurity/trivy-action
