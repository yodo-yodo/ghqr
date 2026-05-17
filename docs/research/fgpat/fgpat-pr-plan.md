# ghqr Fine-Grained PAT Improvement PR Plan

この資料における `ghqr` は、リポジトリルートから実行する `./bin/linux_amd64/ghqr` の短縮表記である。

## 1. Executive summary

現時点の最終判断は **A. 実装修正不要。README / warning 改善で足りる** とする。

理由は次のとおり。

- `ghqr` は token 種別を判定せず、`GH_TOKEN` / `GITHUB_TOKEN` を Bearer token として GitHub API に渡しているだけである
- explicit repository scan は、適切に設定した FGPAT で実測成功している
- 最初の失敗は `ghqr` 不具合ではなく、FGPAT の `Resource owner` / `Repository access` 設定ミスで説明できる
- 一方で organization discovery、enterprise、audit log、Copilot、security endpoint は GitHub API の FGPAT 対応状況、permission、org approval policy の影響を受ける可能性が高く、classic PAT と完全同一挙動は保証できない

したがって、改善 PR の主軸は次の順になる。

1. README の FGPAT 手順と制約を明確化する
2. warning / troubleshooting を FGPAT 向けに具体化する
3. 追加実測で `ghqr` の実装起因の不一致が確認された場合のみ、最小コード変更を検討する

## 2. Current verified facts

### 2.1 source of truth

- 主資料:
  - `audit_data/ghqr-fgpat-full-validation.md`
- 参照資料:
  - `audit_data/fgpat-validation-repo-design.md`

### 2.2 検証済みの事実

- Fine-grained PAT で対象 private repository の REST 可視性確認は成功済み
- 実行コマンド:
```bash
GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api repos/example-org/fgpat-validation-repo --jq .full_name
```
- 結果:
```text
example-org/fgpat-validation-repo
```

- Fine-grained PAT で explicit repository scan は成功済み
- 実行コマンド:
```bash
env GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-org gh auth token)" \
ghqr scan \
--repository example-org/fgpat-validation-repo
```
- 実際の結果:
  - GraphQL repository 解決成功
  - ruleset enrichment 成功
  - evaluation 成功
  - report rendering 成功
  - `results=6`
  - scan 完走
- 生成物:
  - `example-output.json`
  - `example-output.xlsx`
  - `example-output.md`
- 生成場所:
  - `~/dev/lab/oss/ghqr`

- 最初の FGPAT 失敗は個人 owner token による設定不整合だった
- 実行コマンド:
```bash
GH_CONFIG_DIR=/tmp/gh-fgpat gh api repos/example-org/fgpat-validation-repo --jq .full_name
```
- 結果:
  - `404 Not Found`
- 実行コマンド:
```bash
env GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat gh auth token)" \
ghqr scan \
--repository example-org/fgpat-validation-repo
```
- 結果:
  - GraphQL の repository resolve 失敗
  - `No scan results to render`

### 2.3 未確認の事実

- organization scan
- organization auto-discovery
- enterprise scan
- enterprise discovery
- audit log endpoint
- Copilot endpoint
- security endpoint
- REST 単体成功だが `ghqr` 失敗の具体例

## 3. Classic PAT と FGPAT の仕様差

### 3.1 GitHub 仕様上の差

- FGPAT は classic PAT の完全上位互換ではない
- FGPAT は単一の user または organization owner に紐づく
- FGPAT は repository access を個別指定できるが、その分、対象外 resource は見えない
- FGPAT は endpoint ごとに必要 permission が異なる
- organization が approval policy を有効化している場合、FGPAT は承認前だと private resource に使えない
- GitHub Docs では、FGPAT が classic PAT の全機能をまだ完全にはサポートしていないと明記されている

### 3.2 この PR 計画書での解釈

- 「FGPAT でも適切な設定なら classic PAT と一切同じ挙動になる」とは書かない
- 正しい表現は次のとおり:
  - 一部機能では同等に動く
  - ただし GitHub API の FGPAT 対応状況、permission、org approval policy により差が出る可能性がある

## 4. ghqr の現在の token 取り扱い

### 4.1 確認済みの実装

- `internal/config/github.go`
  - `GH_TOKEN` を優先して読む
  - 空なら `GITHUB_TOKEN` を読む
  - token が空なら初期化エラーにする
  - `oauth2.StaticTokenSource` から共通 HTTP client を作る
  - 同じ認証 transport で REST / GraphQL client を作る

- `internal/pipeline/stage_initialization.go`
  - `Users.Get(ctx, "")` で認証確認する

### 4.2 重要な解釈

- token 種別判定はしていない
- classic PAT 前提の token 分岐はない
- Bearer token として GitHub API に渡しているだけである
- そのため、repository visibility や endpoint access の差分は、まず GitHub API / permission / org policy の影響を疑うべきである

## 5. 機能別の検証結果表

| feature | classic PAT result | FGPAT result | same behavior? | likely cause if different | ghqr code change required? |
| --- | --- | --- | --- | --- | --- |
| explicit repository scan | 成功 | 成功 | Yes | 差分なし | No |
| organization scan | 未検証 | 未検証 | 未確認 | org-level permission または endpoint 制約の可能性 | 未判定 |
| organization auto-discovery | 未検証 | 未検証 | 未確認 | `/user/orgs` 系の FGPAT 制約または org visibility 差の可能性 | 実装より UX 改善の可能性が高い |
| enterprise scan | 未検証 | 未検証 | 未確認 | enterprise endpoint / admin 権限 / GitHub plan 制約の可能性 | 未判定 |
| enterprise discovery | 未検証 | 未検証 | 未確認 | GraphQL `viewer.enterprises` の可視性制約の可能性 | 未判定 |
| audit log | 未検証 | 未検証 | 未確認 | enterprise audit 権限または GitHub API 制限の可能性 | 未判定 |
| Copilot | 未検証 | 未検証 | 未確認 | org Copilot permission または API 制限の可能性 | 未判定 |
| security endpoint | 未検証 | 未検証 | 未確認 | Dependabot / code scanning / secret scanning permission の可能性 | 未判定 |
| ruleset / branch protection | repository scan では成功 | repository scan では成功 | Yes | 差分なし | No |
| report rendering | 成功 | 成功 | Yes | 差分なし | No |
| REST 単体成功だが ghqr 失敗の箇所 | 未確認 | 未確認 | 未確認 | 具体例未取得 | 未判定 |

## 6. 実装修正が不要な場合の理由

現時点では、実装修正不要と判断する理由が十分ある。

- explicit repository scan は FGPAT で成功しており、コアの token 取り扱いに不整合は見えていない
- `ghqr` は token 種別による分岐をしておらず、認証方式自体に classic PAT 前提のバグは確認できていない
- 既知の失敗は FGPAT 設定不整合で説明できる
- 未確認領域の多くは enterprise / org-level / security / Copilot といった、GitHub API 側の permission 依存が強い領域である

この段階でコード変更を入れると、GitHub 仕様差を `ghqr` の不具合と誤認したまま修正する危険がある。

## 7. 実装修正が必要な場合の最小変更案

現時点では必須ではないが、今後の追加実測で `ghqr` 実装問題が確認された場合は、次の順で最小変更を提案する。

### 7.1 実装変更が必要になる条件

- REST 単体では成功するのに `ghqr` の同機能だけ失敗する
- GraphQL 単体では成功するのに `ghqr` の batch query または stage 制御だけ失敗する
- FGPAT では取得不能な endpoint を `ghqr` が必須扱いして scan 全体を落としている

### 7.2 最小変更候補

- organization auto-discovery が空配列のときの warning をさらに具体化する
- FGPAT で発生しやすい `403` / `404` / empty result に対して、permission / approval / repository access 不足の可能性を明示する
- enterprise / Copilot / security endpoint の失敗を stage 全体の fatal にせず、補足 warning に寄せる

## 8. README / warning / troubleshooting 改善案

### 8.1 README

- FGPAT は利用可能だが、classic PAT と完全同一挙動は保証しないと明記する
- 最低限必要な設定例を示す
  - `Resource owner`
  - `Repository access`
  - 必要 permission
  - org approval policy の確認
- `--organization` や `--repository` の明示指定を推奨する
- enterprise / audit log / Copilot / security endpoint は追加 permission または API 制限の可能性があると明記する

### 8.2 warning

- auto-discovery 失敗時:
  - organization list が空でも、explicit `--organization` / `--repository` は成功しうることを示す
- `404 Not Found`:
  - repository access 不足、Resource owner 不一致、org approval 未承認の可能性を示す
- `403 Forbidden`:
  - permission 不足または endpoint 非対応の可能性を示す

### 8.3 troubleshooting

- まず `gh api repos/{owner}/{repo}` で repository visibility を確認する手順
- organization policy による approval requirement の確認手順
- `gh auth token` を一時的に `GITHUB_TOKEN` へ渡す安全な実行例

## 9. GitHub Issue / PR description draft

### 9.1 Draft

`ghqr` の Fine-grained PAT 対応について、現時点の調査では explicit repository scan は適切な token 設定で正常動作することを確認しました。  
一方で、Fine-grained PAT は classic PAT の完全上位互換ではなく、GitHub API の endpoint ごとの対応状況、repository access、permission、organization approval policy の影響を受けます。

今回の調査では、最初の失敗は `ghqr` の不具合ではなく、Fine-grained PAT の `Resource owner` と `Repository access` 設定不整合により説明できました。  
その後、対象 organization を owner にし、対象 repository を明示した Fine-grained PAT では、repository visibility の REST API と `ghqr scan --repository` の両方が成功しました。

したがって現時点では、`ghqr` の必須コード修正よりも、README、warning、troubleshooting を改善し、Fine-grained PAT の前提条件と制約を明確にする方針が妥当です。  
追加検証が必要な領域としては、organization discovery、enterprise scan、audit log、Copilot、security endpoint が残っています。

### 9.2 Draft issue bullets

- explicit repository scan は FGPAT で成功
- classic PAT と完全同一挙動は保証しない
- org approval policy と endpoint ごとの permission 差を明記すべき
- README / warning / troubleshooting を先に改善するのが妥当

## 10. Remaining unknowns

- organization scan の Classic PAT / FGPAT 比較結果
- organization auto-discovery の Classic PAT / FGPAT 比較結果
- enterprise scan / discovery の比較結果
- audit log endpoint の FGPAT 可否
- Copilot endpoint の FGPAT 可否
- security endpoint の FGPAT 可否
- REST 単体成功だが `ghqr` 失敗となる具体例の有無

## 11. 実行ログ

### 11.1 この計画書作成

- 実行セッション:
  - `Codex shell`
- 実行コマンド:
```bash
sed -n '1,320p' audit_data/ghqr-fgpat-full-validation.md
sed -n '1,260p' audit_data/fgpat-validation-repo-design.md
test -f audit_data/ghqr-fgpat-pr-plan.md && sed -n '1,260p' audit_data/ghqr-fgpat-pr-plan.md || true
```
- 結果:
  - source of truth の読み込み成功
  - 既存の `audit_data/ghqr-fgpat-pr-plan.md` は未作成だった
- exit code:
  - `0`
- stdout/stderr 要約:
  - stdout: 既存の検証メモ内容を確認
  - stderr: なし

### 11.2 最終判断

**A. 実装修正不要。README / warning 改善で足りる**

補足:

- この判断は「explicit repository scan 成功」と「token 取り扱い実装に classic PAT 固有分岐がないこと」に基づく
- 未確認領域で `ghqr` 実装問題が見つかった場合のみ、最小変更案へ切り替える
