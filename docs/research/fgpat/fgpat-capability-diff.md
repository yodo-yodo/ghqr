# ghqr Fine-Grained PAT Capability Diff

この資料における `ghqr` は、リポジトリルートから実行する `./bin/linux_amd64/ghqr` の短縮表記である。検証中は shell セッション限定で `alias ghqr='./bin/linux_amd64/ghqr'` を使う前提とし、`.bashrc` は変更しない。

## 1. Summary

Fine-grained PAT は、少なくとも今回の `example-org/fgpat-validation-repo` に対する repository / organization scan では classic PAT と同等に動作した。

ただし、FGPAT は classic PAT の完全上位互換ではない。GitHub API endpoint ごとの対応状況、必要 permission、Resource owner、Repository access、organization approval policy、enterprise 権限により差が出る。今回の実測でも enterprise discovery / enterprise scan / audit log は両 token とも `read:enterprise` または `admin:enterprise` 不足で制限された。

現時点で「REST / GraphQL 単体では成功するが `ghqr` だけ失敗する」箇所は確認していない。従って、必須の実装修正ではなく README / warning / troubleshooting 改善を主対応とする。

## 2. Methodology

- source of truth:
  - `audit_data/ghqr-fgpat-pr-plan.md`
- 追加参照:
  - `audit_data/ghqr-fgpat-full-validation.md`
  - `audit_data/fgpat-validation-repo-design.md`
- 実行セッション:
  - `Codex shell`
  - `GH_CONFIG_DIR=/tmp/gh-fgpat-org`
- token 取り扱い:
  - token 値、token の一部、prefix、末尾は記録しない
  - `gh auth token` の戻り値を一時的に `GITHUB_TOKEN` に渡す
  - `echo` / `env` / `printenv` で token を表示しない
- 比較方法:
  - Classic PAT と FGPAT で同じ `ghqr` コマンドを実行
  - 必要に応じて `gh api --silent` または `gh api graphql` で REST / GraphQL 単体を確認
  - exit code、stdout/stderr 要約、生成物を記録

## 3. ghqr source code endpoint inventory

### Token / client initialization

- file:
  - `internal/config/github.go`
  - `internal/pipeline/stage_initialization.go`
- functions:
  - `config.NewClients`
  - `InitializationStage.Execute`
- behavior:
  - `GH_TOKEN` を優先して読む
  - 空なら `GITHUB_TOKEN` を読む
  - token 種別判定はしない
  - `oauth2.StaticTokenSource` で共通 HTTP client を作る
  - REST / GraphQL client は同じ token transport を共有する
  - 認証確認は REST `GET /user` 相当の `Users.Get(ctx, "")`

### Repository scan

- files:
  - `internal/pipeline/stage_repository_scan.go`
  - `internal/scanners/batch.go`
  - `internal/scanners/graphql_client.go`
  - `internal/scanners/repository.go`
  - `internal/scanners/ruleset.go`
- API:
  - GraphQL `repository(owner:, name:)`
  - GraphQL `repository.rulesets(first:, includeParents:)`
  - GraphQL `defaultBranchRef.branchProtectionRule`
  - GraphQL `vulnerabilityAlerts`
  - GraphQL `collaborators`
  - GraphQL `deployKeys`
  - GraphQL `object(expression: "HEAD:...")`
- notes:
  - repository 明示指定は GraphQL batch query で取得する
  - ruleset / branch protection は GraphQL で補完する
  - org-wide batch scan では complexity 回避のため一部 connection を省略している

### Organization scan / discovery

- files:
  - `internal/pipeline/stage_organization_discovery.go`
  - `internal/pipeline/stage_organization_scan.go`
  - `internal/pipeline/stage_org_repository_scan.go`
  - `internal/scanners/organization.go`
  - `internal/scanners/graphql_client.go`
- API:
  - REST `GET /user/orgs` 相当
  - REST `GET /orgs/{org}`
  - REST `GET /orgs/{org}/actions/permissions`
  - REST `GET /orgs/{org}/actions/permissions/workflow`
  - REST `GET /orgs/{org}/dependabot/alerts`
  - REST `GET /orgs/{org}/code-scanning/alerts`
  - REST `GET /orgs/{org}/secret-scanning/alerts`
  - REST `GET /orgs/{org}/security-managers`
  - REST `GET /orgs/{org}/external-groups`
  - REST `GET /orgs/{org}/copilot/billing`
  - GraphQL `organization(login:) { repositories(...) }`
  - GraphQL `organization(login:) { enterprise { ownerInfo { samlIdentityProvider } } }`
- notes:
  - organization discovery が空でも `--organization` / `--repository` 明示指定は成功し得る
  - org sub-scan の一部 endpoint は 403 / 404 を debug 扱いまたは warning 扱いにして継続する

### Enterprise scan / discovery

- files:
  - `internal/pipeline/stage_enterprise_discovery.go`
  - `internal/pipeline/stage_enterprise_scan.go`
  - `internal/scanners/enterprise.go`
- API:
  - GraphQL `viewer { enterprises(first:, after:) }`
  - GraphQL `enterprise(slug:)`
  - GraphQL `enterprise(slug:) { organizations(...) }`
  - GraphQL `enterprise(slug:) { ownerInfo { samlIdentityProvider } }`
  - REST `GET /enterprises/{enterprise}/audit-log`
  - REST `GET /enterprises/{enterprise}/dependabot/alerts`
  - REST `GET /enterprises/{enterprise}/code-scanning/alerts`
  - REST `GET /enterprises/{enterprise}/secret-scanning/alerts`
  - REST `GET /enterprises/{enterprise}/code_security/settings`
- notes:
  - enterprise discovery / settings は GraphQL `read:enterprise` 相当が必要
  - audit log は `admin:enterprise` 相当が必要

## 4. GitHub official FGPAT support matrix

Sources:

- GitHub Docs: Managing personal access tokens
  - https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- GitHub Docs: Endpoints available for fine-grained personal access tokens
  - https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens
- GitHub Docs: Permissions required for fine-grained personal access tokens
  - https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
- GitHub Docs: Organization personal access token policy
  - https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization
- GitHub Docs: Managing requests for personal access tokens in your organization
  - https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization
- GitHub Docs: Forming calls with GraphQL
  - https://docs.github.com/en/graphql/guides/forming-calls-with-graphql

| Area | FGPAT support / permission interpretation | Current confidence |
| --- | --- | --- |
| Repository metadata / visibility | FGPAT 対応。Resource owner と repository access が正しければ private repo も参照可能 | High |
| Repository GraphQL | GraphQL は token の resource visibility / permission に依存 | High |
| Rulesets / branch protection | Repository administration read 相当が必要な可能性。今回の FGPAT では成功 | Medium |
| Organization metadata | Organization access / approval policy に依存 | Medium |
| Organization actions permissions | `admin:org` または actions policies fine-grained permission が必要 | High |
| Copilot billing | FGPAT でも到達可能。org Copilot 権限や plan に依存 | Medium |
| Dependabot / code scanning / secret scanning alerts | 対応可能だが各 security permission / plan に依存 | Medium |
| Enterprise discovery / settings | `read:enterprise` 相当が必要 | High |
| Enterprise audit log | `admin:enterprise` 相当が必要 | High |
| Org approval policy | approval 未承認の場合は private resource に使えない | High |

## 5. Classic PAT vs FGPAT behavior comparison

| Feature | Classic PAT result | FGPAT result | Same behavior? | Classification |
| --- | --- | --- | --- | --- |
| explicit repository scan | 成功、exit code 0、reports 生成 | 成功、exit code 0、reports 生成 | Yes | できる |
| organization scan | 成功、exit code 0、org 1 / repo 1 | 成功、exit code 0、org 1 / repo 1 | Yes | できる |
| organization auto-discovery | 成功、org `example-org` 検出 | 成功、org `example-org` 検出 | Yes in this org | できる |
| enterprise discovery | `read:enterprise` 不足 warning | `read:enterprise` 不足 warning | Yes | GitHub仕様上の権限不足 |
| enterprise scan | enterprise settings は失敗、後続 org scan は成功 | enterprise settings は失敗、後続 org scan は成功 | Yes | GitHub仕様上の権限不足 |
| audit log | REST 単体 403 | REST 単体 403 | Yes | GitHub仕様上の権限不足 |
| Copilot | REST 単体 exit code 0 | REST 単体 exit code 0 | Yes | できる |
| security endpoints | REST 単体 exit code 0 | REST 単体 exit code 0 | Yes | できる |
| org actions permissions | REST 単体 403 | REST 単体 403 | Yes | permission 不足 |
| ruleset / branch protection | scan 内で成功 | scan 内で成功 | Yes | できる |
| REST 単体成功だが ghqr だけ失敗 | 未検出 | 未検出 | N/A | ghqr実装差分なし |

## 6. REST/GraphQL単体とghqr実行結果の差分

### No ghqr-only failure found

今回確認した範囲では、REST / GraphQL 単体では成功するが `ghqr` だけ失敗する箇所は見つかっていない。

### REST / GraphQL standalone probes

| Probe | Classic PAT | FGPAT | Interpretation |
| --- | --- | --- | --- |
| `GET /repos/example-org/fgpat-validation-repo` | 既存検証で成功 | 成功 | FGPAT repository visibility OK |
| GraphQL `repository(owner,name)` | exit code 0、repo 名取得 | exit code 0、repo 名取得 | ghqr repository scan と整合 |
| GraphQL `organization(login).repositories` | exit code 0、repo count 1 | exit code 0、repo count 1 | ghqr organization scan と整合 |
| `GET /orgs/example-org/actions/permissions` | 403 | 403 | token type 差ではなく権限不足 |
| `GET /orgs/example-org/copilot/billing` | exit code 0 | exit code 0 | 両方到達可能 |
| `GET /orgs/example-org/dependabot/alerts` | exit code 0 | exit code 0 | 両方到達可能 |
| `GET /orgs/example-org/code-scanning/alerts` | exit code 0 | exit code 0 | 両方到達可能 |
| `GET /orgs/example-org/secret-scanning/alerts` | exit code 0 | exit code 0 | 両方到達可能 |
| `GET /orgs/example-org/security-managers` | exit code 0 | exit code 0 | 両方到達可能 |
| `GET /enterprises/example-org/audit-log` | 403 | 403 | `admin:enterprise` 不足 |
| `GET /enterprises/example-org/code_security/settings` | 404 | 404 | endpoint / enterprise resource 不一致または権限不足 |

## 7. FGPATでできること

- 対象 org を Resource owner にした FGPAT で private repository を REST 参照できる
- `ghqr scan --repository example-org/fgpat-validation-repo` を実行できる
- `ghqr scan --organization example-org` を実行できる
- 今回の条件では organization auto-discovery で `example-org` を検出できる
- GraphQL repository query が成功する
- GraphQL organization repositories query が成功する
- ruleset / branch protection enrichment が成功する
- Copilot billing endpoint に到達できる
- Dependabot / code scanning / secret scanning / security managers endpoint に到達できる
- JSON / Markdown / Excel reports を生成できる

## 8. FGPATでできないこと

現在の FGPAT 設定では次はできない。

- `GET /orgs/example-org/actions/permissions`
  - 403
  - 原因仮説: `admin:org` または actions policies fine-grained permission 不足
- enterprise discovery / enterprise settings
  - GraphQL `read:enterprise` 不足
- enterprise audit log
  - REST 403
  - 原因仮説: `admin:enterprise` 不足

これは `ghqr` 固有の失敗ではなく、REST / GraphQL 単体でも同じ制限が出ている。

## 9. GitHub仕様上できない可能性が高いこと

- FGPAT を classic PAT の完全上位互換として扱うこと
- organization approval policy を無視して private resource にアクセスすること
- Resource owner / Repository access 外の private repository を読むこと
- enterprise discovery / audit log / enterprise GHAS settings を repository-scoped FGPAT だけで classic PAT と同等に扱うこと
- endpoint ごとの permission 不足を `ghqr` 側だけで回避すること

## 10. ghqr側の実装修正が必要な可能性があること

現時点で必須の実装修正は確認していない。

ただし、改善余地はある。

- enterprise discovery の `read:enterprise` 不足 warning は GitHub の scope 文言がそのまま出るため、FGPAT 利用者向けには分かりにくい
- org actions permissions の 403 は `ghqr` 実行時には debug / silent 扱いになり得るため、必要な情報が report から読み取りにくい
- 403 / 404 / empty result のときに、FGPAT の Resource owner、Repository access、permission、org approval policy を確認する案内が不足している

最小変更案:

- README / troubleshooting に FGPAT の設定例と確認手順を追加する
- organization auto-discovery が空のとき、`--organization` / `--repository` 明示指定を促す warning を維持・強化する
- endpoint 単位で 403 / 404 を握りつぶす箇所は、debug だけでなく summary warning または report metadata に残すことを検討する

## 11. README / warning / troubleshooting 改善案

### README

- FGPAT は classic PAT と完全同一挙動を保証しないと明記する
- repository scan の最小設定例を記載する
  - Resource owner: target organization
  - Repository access: selected repositories or all target repositories
  - Repository permissions: Metadata read, Contents read, Administration read
- organization / enterprise / audit / Copilot / security は追加 permission または GitHub plan / org policy に依存すると明記する
- まず `gh api repos/{owner}/{repo}` で repository visibility を確認する手順を載せる

### warning

- enterprise discovery 失敗時:
  - `read:enterprise` が必要であり、FGPAT では token 設定や enterprise 権限の制限を受ける可能性があると補足する
- 403:
  - permission 不足、org approval 未承認、endpoint 非対応の可能性を案内する
- 404:
  - Resource owner / Repository access 不一致、対象 resource 非可視の可能性を案内する

### troubleshooting

- repository visibility:
```bash
GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api repos/example-org/fgpat-validation-repo --jq .full_name
```
- FGPAT scan:
```bash
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-org gh auth token)" ghqr scan --repository example-org/fgpat-validation-repo
```
- org scan:
```bash
GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-org gh auth token)" ghqr scan --organization example-org
```

## 12. PR or Issue draft

### PR draft

This PR documents the current Fine-grained PAT behavior for `ghqr`.

The verified behavior is that repository and organization scans can work with a properly configured Fine-grained PAT. The token must be scoped to the correct resource owner, include the target repository in Repository access, and include the required read permissions.

Fine-grained PATs should not be described as fully equivalent to classic PATs. GitHub applies endpoint-specific permission rules, organization approval policies, repository access restrictions, and enterprise-level permission requirements. In this validation, enterprise discovery and audit log access failed for both classic PAT and Fine-grained PAT due to missing enterprise permissions, while repository and organization scans succeeded for both.

No `ghqr`-only failure was observed: REST / GraphQL standalone probes matched the `ghqr` behavior. The recommended improvement is documentation and user-facing warning/troubleshooting guidance rather than a required functional code change.

### Issue draft

Fine-grained PAT support needs clearer documentation.

Verified:

- `ghqr scan --repository ...` works with a correctly configured FGPAT
- `ghqr scan --organization ...` works in the tested org
- ruleset / branch protection enrichment works in the tested repo
- report rendering works

Limitations:

- FGPAT is not a complete classic PAT replacement
- enterprise discovery needs enterprise-level permission
- audit log needs enterprise admin-level permission
- org approval policy and repository access settings can block private resources
- endpoint-specific permissions may cause partial data

Recommended resolution:

- Update README and troubleshooting
- Improve warnings for FGPAT-style 403 / 404 / empty discovery results
- Avoid claiming full classic PAT parity

## 13. Execution log

### 13.1 Source and code inspection

- session:
  - `Codex shell`
- commands:
```bash
sed -n '1,260p' audit_data/ghqr-fgpat-pr-plan.md
sed -n '1,340p' audit_data/ghqr-fgpat-full-validation.md
sed -n '1,260p' audit_data/fgpat-validation-repo-design.md
test -f audit_data/ghqr-fgpat-capability-diff.md && sed -n '1,260p' audit_data/ghqr-fgpat-capability-diff.md || true
rg -n "NewRequest\\(|Organizations\\.List|Organizations\\.Get|Users\\.Get|GetCopilotBilling|graphql:\\\"|query |enterprise\\(|organization\\(|repository\\(|audit-log|dependabot/alerts|code-scanning/alerts|secret-scanning/alerts|security-managers|actions/permissions|external-groups|rulesets|vulnerabilityAlerts|collaborators|deployKeys" internal cmd
sed -n '1,220p' internal/config/github.go
sed -n '1,220p' internal/pipeline/stage_initialization.go
git status --short
```
- exit code:
  - `0`
- stdout/stderr summary:
  - source documents loaded
  - endpoint inventory found in `internal/pipeline` and `internal/scanners`
  - worktree had unrelated local changes outside this research branch

### 13.2 Alias failure and correction

- session:
  - `Codex shell`
- failed command:
```bash
bash -lc 'shopt -s expand_aliases
alias ghqr="./bin/linux_amd64/ghqr"
env GITHUB_TOKEN="$(gh auth token)" ghqr scan --repository example-org/fgpat-validation-repo --output-name audit_data/example_output'
```
- exit code:
  - `127`
- stdout/stderr summary:
  - `env: 'ghqr': No such file or directory`
- cause:
  - alias expansion did not apply when `ghqr` was passed as an argument to `env`
- correction:
  - use shell assignment form: `GITHUB_TOKEN="..." ghqr scan ...`

### 13.3 ghqr scan comparison

| Command | Session | Exit code | stdout/stderr summary | Generated outputs |
| --- | --- | --- | --- | --- |
| `GITHUB_TOKEN="$(gh auth token)" ghqr scan --repository example-org/fgpat-validation-repo --output-name audit_data/example_output` | Codex shell | 0 | repo GraphQL fetch succeeded, ruleset enrichment succeeded, results=6 | `audit_data/example-output.json`, `.xlsx`, `.md` |
| `GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-org gh auth token)" ghqr scan --repository example-org/fgpat-validation-repo --output-name audit_data/example_output` | Codex shell + `GH_CONFIG_DIR=/tmp/gh-fgpat-org` | 0 | repo GraphQL fetch succeeded, ruleset enrichment succeeded, results=6 | `audit_data/example-output.json`, `.xlsx`, `.md` |
| `GITHUB_TOKEN="$(gh auth token)" ghqr scan --organization example-org --output-name audit_data/example_output` | Codex shell | 0 | organization scan succeeded, repo count=1, results=12 | `audit_data/example-output.json`, `.xlsx`, `.md` |
| `GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-org gh auth token)" ghqr scan --organization example-org --output-name audit_data/example_output` | Codex shell + `GH_CONFIG_DIR=/tmp/gh-fgpat-org` | 0 | organization scan succeeded, repo count=1, results=12 | `audit_data/example-output.json`, `.xlsx`, `.md` |
| `GITHUB_TOKEN="$(gh auth token)" ghqr scan --output-name audit_data/example_output` | Codex shell | 0 | enterprise discovery warned for `read:enterprise`; org discovery found `example-org`; scan succeeded | `audit_data/example-output.json`, `.xlsx`, `.md` |
| `GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-org gh auth token)" ghqr scan --output-name audit_data/example_output` | Codex shell + `GH_CONFIG_DIR=/tmp/gh-fgpat-org` | 0 | enterprise discovery warned for `read:enterprise`; org discovery found `example-org`; scan succeeded | `audit_data/example-output.json`, `.xlsx`, `.md` |
| `GITHUB_TOKEN="$(gh auth token)" ghqr scan --enterprise example-org --output-name audit_data/example_output` | Codex shell | 0 | enterprise settings failed for `read:enterprise`; following org scan succeeded | `audit_data/example-output.json`, `.xlsx`, `.md` |
| `GITHUB_TOKEN="$(GH_CONFIG_DIR=/tmp/gh-fgpat-org gh auth token)" ghqr scan --enterprise example-org --output-name audit_data/example_output` | Codex shell + `GH_CONFIG_DIR=/tmp/gh-fgpat-org` | 0 | enterprise settings failed for `read:enterprise`; following org scan succeeded | `audit_data/example-output.json`, `.xlsx`, `.md` |

### 13.4 REST / GraphQL standalone probes

| Command | Token source | Exit code | stdout/stderr summary |
| --- | --- | --- | --- |
| `gh api --silent orgs/example-org/actions/permissions` | Classic PAT | 1 | 403, requires org admin or actions policies fine-grained permission |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api --silent orgs/example-org/actions/permissions` | FGPAT | 1 | 403, requires org admin or actions policies fine-grained permission |
| `gh api --silent orgs/example-org/copilot/billing` | Classic PAT | 0 | success, no stdout due `--silent` |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api --silent orgs/example-org/copilot/billing` | FGPAT | 0 | success, no stdout due `--silent` |
| `gh api --silent orgs/example-org/dependabot/alerts?state=open&per_page=1` | Classic PAT | 0 | success, no stdout due `--silent` |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api --silent orgs/example-org/dependabot/alerts?state=open&per_page=1` | FGPAT | 0 | success, no stdout due `--silent` |
| `gh api --silent orgs/example-org/code-scanning/alerts?state=open&per_page=1` | Classic PAT | 0 | success, no stdout due `--silent` |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api --silent orgs/example-org/code-scanning/alerts?state=open&per_page=1` | FGPAT | 0 | success, no stdout due `--silent` |
| `gh api --silent orgs/example-org/secret-scanning/alerts?state=open&per_page=1` | Classic PAT | 0 | success, no stdout due `--silent` |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api --silent orgs/example-org/secret-scanning/alerts?state=open&per_page=1` | FGPAT | 0 | success, no stdout due `--silent` |
| `gh api --silent orgs/example-org/security-managers` | Classic PAT | 0 | success, no stdout due `--silent` |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api --silent orgs/example-org/security-managers` | FGPAT | 0 | success, no stdout due `--silent` |
| `gh api --silent enterprises/example-org/audit-log?include=all&per_page=1` | Classic PAT | 1 | 403, requires `admin:enterprise` |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api --silent enterprises/example-org/audit-log?include=all&per_page=1` | FGPAT | 1 | 403, requires `admin:enterprise` |
| `gh api --silent enterprises/example-org/code_security/settings` | Classic PAT | 1 | 404 |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api --silent enterprises/example-org/code_security/settings` | FGPAT | 1 | 404 |
| `gh api graphql ... repository(owner,name) ...` | Classic PAT | 0 | returned `fgpat-validation-repo` |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api graphql ... repository(owner,name) ...` | FGPAT | 0 | returned `fgpat-validation-repo` |
| `gh api graphql ... organization(login).repositories ...` | Classic PAT | 0 | returned repository count `1` |
| `GH_CONFIG_DIR=/tmp/gh-fgpat-org gh api graphql ... organization(login).repositories ...` | FGPAT | 0 | returned repository count `1` |

## 14. Final judgment

### A. ghqr実装修正不要。GitHub仕様差なのでREADME/warning改善で対応

Applicable for the currently observed behavior.

理由:

- repository scan、organization scan、auto-discovery は FGPAT でも成功した
- REST / GraphQL 単体と `ghqr` 実行結果は整合している
- token handling は token 種別非依存で、classic PAT 専用分岐はない

### B. ghqr実装修正必要。REST/GraphQL単体では成功するがghqrだけ失敗するため

Not supported by current evidence.

理由:

- REST / GraphQL 単体で成功して `ghqr` だけ失敗する箇所は未検出
- 失敗した endpoint は REST / GraphQL 単体でも失敗している

### C. GitHub仕様上、FGPATではclassic PAT完全同等は不可。Issue文書化が必要

Applicable.

理由:

- GitHub Docs 上、FGPAT は classic PAT の完全上位互換ではない
- Resource owner、Repository access、permission、org approval policy の影響を受ける
- enterprise / audit log / org actions policy などは endpoint-specific permission が必要

最終結論:

- **A と C を採用**
- **B は現時点では採用しない**
